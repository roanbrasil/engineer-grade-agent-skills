---
name: spring-ai
description: Expert Spring AI (1.x) skill — invoke when building AI-powered Java/Kotlin applications with Spring Boot, including chat integrations, RAG pipelines, tool/function calling, streaming, advisors, embedding, multimodal, or swapping LLM providers without code changes.
---

# Spring AI — Expert Reference (1.x)

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Spring AI Stack                              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                       ChatClient                             │    │
│  │   (fluent API — the primary entry point for applications)    │    │
│  └────────────────────────┬─────────────────────────────────────┘    │
│                           │                                          │
│       ┌───────────────────┼────────────────────┐                     │
│       │                   │                    │                     │
│  ┌────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐             │
│  │ Advisors │      │  ChatModel  │     │  Prompts /  │             │
│  │ (AOP-    │      │ (provider   │     │  Templates  │             │
│  │  like    │      │  abstraction│     │             │             │
│  │  layer)  │      │  layer)     │     └─────────────┘             │
│  └──────────┘      └──────┬──────┘                                  │
│                           │                                          │
│          ┌────────────────┼────────────────┐                         │
│          │                │                │                         │
│   ┌──────▼───┐   ┌────────▼──────┐  ┌─────▼──────┐                 │
│   │ OpenAI   │   │  Anthropic    │  │   Ollama   │                 │
│   │  Model   │   │  (Claude)     │  │  (local)   │  + more         │
│   └──────────┘   └───────────────┘  └────────────┘                 │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  RAG Pipeline                                                │    │
│  │  DocumentReader → DocumentTransformer → VectorStore          │    │
│  │                                            │                 │    │
│  │                              QuestionAnswerAdvisor ──────────┘    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Observability: Micrometer metrics + traces (all model calls) │   │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

## Dependencies (Maven)

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Core Spring AI + Spring Boot -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <!-- Or Anthropic Claude -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
    </dependency>
    <!-- Or Ollama (local models) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
    </dependency>

    <!-- Vector store (choose one) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-redis-store-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

```kotlin
// Gradle (Kotlin DSL)
implementation(platform("org.springframework.ai:spring-ai-bom:1.0.0"))
implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter")
```

---

## Configuration

```yaml
# application.yml

# OpenAI
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 2048
      embedding:
        options:
          model: text-embedding-3-small

# Anthropic Claude
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-opus-4-5
          max-tokens: 4096

# Ollama (local)
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2

# pgvector
spring:
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
```

---

## ChatClient — Fluent API

`ChatClient` is the primary API for interacting with chat models. It wraps `ChatModel` with a fluent builder that handles prompts, advisors, tools, and structured output.

### Basic Usage

```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;

@Service
public class AssistantService {

    private final ChatClient chatClient;

    public AssistantService(ChatModel chatModel) {
        this.chatClient = ChatClient.create(chatModel);
    }

    public String ask(String question) {
        return chatClient
            .prompt()
            .system("You are a helpful assistant.")
            .user(question)
            .call()
            .content();  // returns String
    }

    // With variable substitution in prompt templates
    public String explainTopic(String topic) {
        return chatClient
            .prompt()
            .system("You are a technical educator.")
            .user(u -> u.text("Explain {topic} in simple terms").param("topic", topic))
            .call()
            .content();
    }
}
```

```kotlin
// Kotlin
@Service
class AssistantService(chatModel: ChatModel) {
    private val chatClient = ChatClient.create(chatModel)

    fun ask(question: String): String =
        chatClient.prompt()
            .system("You are a helpful assistant.")
            .user(question)
            .call()
            .content()
}
```

### Shared ChatClient with Defaults

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatModel chatModel) {
        // Set defaults once — reused across all calls
        return ChatClient.builder(chatModel)
            .defaultSystem("You are a helpful assistant for Acme Corp.")
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(new InMemoryChatMemory()),
                new SimpleLoggerAdvisor()
            )
            .defaultOptions(
                OpenAiChatOptions.builder()
                    .withTemperature(0.7f)
                    .build()
            )
            .build();
    }
}

@Service
public class SupportService {

    private final ChatClient chatClient;

    public SupportService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String handleTicket(String issue) {
        // Overrides system only for this call
        return chatClient
            .prompt()
            .system("You are a technical support specialist.")
            .user(issue)
            .call()
            .content();
    }
}
```

---

## Structured Output

```java
import org.springframework.ai.chat.client.ChatClient;

public record MovieReview(
    String title,
    double rating,
    String summary,
    List<String> pros,
    List<String> cons
) {}

@Service
public class ReviewService {

    private final ChatClient chatClient;

    public ReviewService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public MovieReview extractReview(String reviewText) {
        return chatClient
            .prompt()
            .system("Extract movie review information from the text.")
            .user(reviewText)
            .call()
            .entity(MovieReview.class);  // auto-generates format instructions & parses
    }

    // List of entities
    public List<MovieReview> extractMultipleReviews(String text) {
        return chatClient
            .prompt()
            .user(text)
            .call()
            .entity(new ParameterizedTypeReference<List<MovieReview>>() {});
    }
}
```

```kotlin
// Kotlin — data class works directly
data class ProductInfo(
    val name: String,
    val price: Double,
    val inStock: Boolean,
    val description: String,
)

fun extractProduct(text: String): ProductInfo =
    chatClient.prompt()
        .user(text)
        .call()
        .entity(ProductInfo::class.java)
```

### BeanOutputConverter (Lower-level)

```java
import org.springframework.ai.converter.BeanOutputConverter;

var converter = new BeanOutputConverter<>(MovieReview.class);
String formatInstructions = converter.getFormat();

Prompt prompt = new PromptTemplate("""
    Extract movie info.
    {format}
    Text: {text}
    """)
    .create(Map.of("format", formatInstructions, "text", reviewText));

ChatResponse response = chatModel.call(prompt);
MovieReview review = converter.convert(response.getResult().getOutput().getContent());
```

---

## Streaming

```java
import reactor.core.publisher.Flux;

@Service
public class StreamingService {

    private final ChatClient chatClient;

    public Flux<String> streamResponse(String question) {
        return chatClient
            .prompt()
            .user(question)
            .stream()           // returns StreamResponseSpec
            .content();         // returns Flux<String>
    }

    // Full ChatResponse flux (includes metadata, usage)
    public Flux<ChatResponse> streamFullResponse(String question) {
        return chatClient
            .prompt()
            .user(question)
            .stream()
            .chatResponse();
    }
}
```

```java
// Spring WebFlux controller — SSE endpoint
@RestController
public class ChatController {

    private final StreamingService streamingService;

    @GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String question) {
        return streamingService.streamResponse(question);
    }
}
```

```kotlin
// Kotlin coroutines alternative
fun streamResponse(question: String): Flow<String> =
    chatClient.prompt()
        .user(question)
        .stream()
        .content()
        .asFlow()
```

---

## Tool / Function Calling

### @Bean Method with @Description

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Description;

@Configuration
public class ToolConfig {

    @Bean
    @Description("Get the current weather for a given city")
    public Function<WeatherRequest, WeatherResponse> getWeather() {
        return request -> {
            // call real weather API
            return new WeatherResponse(request.city(), "Sunny", 22.0);
        };
    }

    @Bean
    @Description("Search for products in the catalog by name or category")
    public Function<ProductSearchRequest, List<Product>> searchProducts(ProductRepository repo) {
        return request -> repo.findByNameContaining(request.query());
    }
}

// Request/Response records — Spring AI reads field names as parameter schema
record WeatherRequest(
    @JsonPropertyDescription("City name, e.g. 'London'") String city
) {}

record WeatherResponse(String city, String condition, double temperatureCelsius) {}
```

```java
// Register tools on ChatClient call
@Service
public class AgentService {

    private final ChatClient chatClient;

    public String answerWithTools(String question) {
        return chatClient
            .prompt()
            .user(question)
            .tools("getWeather", "searchProducts")  // bean names
            .call()
            .content();
    }
}
```

### FunctionCallback (Programmatic, Dynamic Tools)

```java
import org.springframework.ai.model.function.FunctionCallback;

FunctionCallback dynamicTool = FunctionCallback.builder()
    .function("calculate", (String expression) -> {
        // evaluate expression
        return "42";
    })
    .description("Evaluate a mathematical expression")
    .inputType(String.class)
    .build();

String result = chatClient
    .prompt()
    .user("What is 6 * 7?")
    .tools(dynamicTool)
    .call()
    .content();
```

---

## Advisors

Advisors implement cross-cutting concerns around chat calls — analogous to Spring AOP for LLM calls.

```
Request flow:
  ChatClient.call()
      │
      ▼
  [Advisor 1 — before]  ← e.g. inject memory, retrieve docs
      │
      ▼
  [Advisor 2 — before]  ← e.g. content moderation check
      │
      ▼
  ChatModel.call()      ← actual LLM call
      │
      ▼
  [Advisor 2 — after]   ← e.g. save response to memory
      │
      ▼
  [Advisor 1 — after]   ← e.g. log tokens used
      │
      ▼
  ChatClient response
```

### MessageChatMemoryAdvisor — Conversation History

```java
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.InMemoryChatMemory;

// In-memory (development)
var memory = new InMemoryChatMemory();
var memoryAdvisor = new MessageChatMemoryAdvisor(memory);

// Per-conversation ID (isolates sessions)
String result = chatClient
    .prompt()
    .user("What did I say earlier?")
    .advisors(a -> a
        .advisor(memoryAdvisor)
        .param(CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId)  // per-user/session
        .param(CHAT_MEMORY_RETRIEVE_SIZE_KEY, 10)                // last 10 exchanges
    )
    .call()
    .content();
```

### QuestionAnswerAdvisor — RAG Auto-Injection

```java
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.vectorstore.VectorStore;

@Service
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String askWithContext(String question) {
        return chatClient
            .prompt()
            .user(question)
            .advisors(new QuestionAnswerAdvisor(
                vectorStore,
                SearchRequest.defaults().withTopK(5)  // retrieve 5 most relevant docs
            ))
            .call()
            .content();
        // Advisor automatically:
        // 1. Embeds the question
        // 2. Searches vectorStore
        // 3. Injects results into system prompt
        // 4. Calls model
    }
}
```

### SafeGuardAdvisor — Content Moderation

```java
import org.springframework.ai.chat.client.advisor.SafeGuardAdvisor;

// Blocks messages containing specified sensitive patterns
var safeGuard = new SafeGuardAdvisor(
    List.of("personal data", "credit card", "ssn")
);

chatClient.prompt()
    .user(userInput)
    .advisors(safeGuard)
    .call()
    .content();
```

### Custom Advisor

```java
import org.springframework.ai.chat.client.advisor.api.*;

public class RequestLoggingAdvisor implements CallAroundAdvisor {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingAdvisor.class);

    @Override
    public String getName() {
        return "RequestLoggingAdvisor";
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;  // run first
    }

    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        log.info("LLM request: model={}, messages={}",
            request.chatOptions().getModel(),
            request.messages().size()
        );

        long start = System.currentTimeMillis();
        AdvisedResponse response = chain.nextAroundCall(request);  // call next advisor/model
        long duration = System.currentTimeMillis() - start;

        log.info("LLM response: duration={}ms, tokens={}",
            duration,
            response.response().getMetadata().getUsage().getTotalTokens()
        );
        return response;
    }
}

// Streaming advisor
public class StreamingLoggingAdvisor implements StreamAroundAdvisor {

    @Override
    public String getName() { return "StreamingLoggingAdvisor"; }

    @Override
    public int getOrder() { return 0; }

    @Override
    public Flux<AdvisedResponse> aroundStream(AdvisedRequest request, StreamAroundAdvisorChain chain) {
        log.info("Stream request started");
        return chain.nextAroundStream(request)
            .doOnComplete(() -> log.info("Stream completed"));
    }
}
```

---

## RAG Pipeline

```
┌──────────────┐    ┌─────────────────────┐    ┌─────────────┐
│ DocumentReader│───▶│ DocumentTransformer │───▶│ VectorStore │
│ (PDF, Web,   │    │ (split, enrich,     │    │ (pgvector,  │
│  DB, FS)     │    │  clean)             │    │  Chroma...) │
└──────────────┘    └─────────────────────┘    └──────┬──────┘
                                                       │
                                          ┌────────────▼──────────┐
                              Question ──▶│ QuestionAnswerAdvisor │
                                          │ (embed + search +     │
                                          │  inject into prompt)  │
                                          └───────────────────────┘
```

### Ingestion Pipeline

```java
import org.springframework.ai.document.Document;
import org.springframework.ai.reader.pdf.PagePdfDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;

@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;

    public void ingestPdf(Resource pdfResource) {
        // 1. Read
        var reader = new PagePdfDocumentReader(pdfResource);
        List<Document> pages = reader.get();

        // 2. Split into chunks
        var splitter = new TokenTextSplitter(
            512,   // chunk size (tokens)
            128,   // overlap
            5,     // min chunk size
            10_000,// max chunk size
            true   // keep separator
        );
        List<Document> chunks = splitter.apply(pages);

        // 3. Enrich with metadata
        chunks.forEach(doc -> doc.getMetadata().put("source", pdfResource.getFilename()));

        // 4. Embed and store
        vectorStore.add(chunks);  // auto-embeds using configured EmbeddingModel
    }

    public void ingestWebPage(String url) {
        var reader = new WebPageDocumentReader(url);
        var splitter = new TokenTextSplitter();
        vectorStore.add(splitter.apply(reader.get()));
    }
}
```

### Query Pipeline

```java
@Service
public class RagQueryService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String query(String question, String conversationId) {
        return chatClient
            .prompt()
            .system("""
                You are a knowledgeable assistant.
                Answer ONLY using the provided context.
                If the answer is not in the context, say 'I don't have that information.'
                """)
            .user(question)
            .advisors(
                new QuestionAnswerAdvisor(
                    vectorStore,
                    SearchRequest.defaults()
                        .withTopK(5)
                        .withSimilarityThreshold(0.7)  // filter low-relevance docs
                ),
                new MessageChatMemoryAdvisor(chatMemory)
            )
            .advisors(a -> a.param(CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
            .call()
            .content();
    }
}
```

### Supported Vector Stores

| Store            | Starter artifact                                  | Notes                          |
|------------------|---------------------------------------------------|--------------------------------|
| pgvector         | `spring-ai-pgvector-store-spring-boot-starter`    | PostgreSQL extension, prod-ready |
| Redis            | `spring-ai-redis-store-spring-boot-starter`       | In-memory, fast                |
| Chroma           | `spring-ai-chroma-store-spring-boot-starter`      | Open source, easy local setup  |
| Weaviate         | `spring-ai-weaviate-store-spring-boot-starter`    | Cloud-native                   |
| Pinecone         | `spring-ai-pinecone-store-spring-boot-starter`    | Managed, scalable              |
| Qdrant           | `spring-ai-qdrant-store-spring-boot-starter`      | High performance               |
| SimpleVectorStore | (built-in, no extra dep)                         | In-memory, dev/test only       |

```java
// SimpleVectorStore for tests / local development
@Bean
public VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return new SimpleVectorStore(embeddingModel);
}
```

---

## Embedding

```java
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.embedding.EmbeddingRequest;
import org.springframework.ai.embedding.EmbeddingOptions;

@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    // Single text
    public float[] embed(String text) {
        return embeddingModel.embed(text);
    }

    // Batch
    public List<float[]> embedBatch(List<String> texts) {
        EmbeddingResponse response = embeddingModel.embedForResponse(texts);
        return response.getResults()
            .stream()
            .map(r -> r.getOutput())
            .toList();
    }

    // Cosine similarity (manual)
    public double cosineSimilarity(float[] a, float[] b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

---

## Multimodal

```java
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.messages.Media;
import org.springframework.util.MimeTypeUtils;

@Service
public class VisionService {

    private final ChatClient chatClient;

    public String describeImage(Resource imageResource) {
        return chatClient
            .prompt()
            .user(u -> u
                .text("Describe what you see in this image in detail.")
                .media(MimeTypeUtils.IMAGE_JPEG, imageResource)
            )
            .call()
            .content();
    }

    public String analyzeReceipt(Resource receiptImage) {
        return chatClient
            .prompt()
            .user(u -> u
                .text("Extract: merchant name, total amount, date, and line items.")
                .media(MimeTypeUtils.IMAGE_PNG, receiptImage)
            )
            .call()
            .entity(Receipt.class);  // structured output from image
    }
}
```

---

## Prompt Templates

```java
import org.springframework.ai.chat.prompt.PromptTemplate;

@Service
public class TemplateService {

    // Inline template
    public String summarize(String text, String style) {
        var template = new PromptTemplate("""
            Summarize the following text in a {style} style.
            
            Text: {text}
            
            Summary:
            """);
        Prompt prompt = template.create(Map.of("style", style, "text", text));
        return chatModel.call(prompt).getResult().getOutput().getContent();
    }
}
```

```java
// Load template from classpath resource
@Value("classpath:/prompts/summarize.st")
private Resource summarizeTemplate;

public String summarizeFromFile(String text) {
    var template = new PromptTemplate(summarizeTemplate);
    return chatModel.call(template.create(Map.of("text", text)))
        .getResult().getOutput().getContent();
}
```

```
# src/main/resources/prompts/summarize.st
You are an expert summarizer.

Summarize the following text concisely:

{text}

Key points:
```

---

## Observability

```yaml
# application.yml — enable Micrometer tracing
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0  # 100% in dev, lower in prod
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

```java
// Spring AI auto-instruments all ChatModel and EmbeddingModel calls:
// - spring.ai.chat.client.operation.seconds (latency histogram)
// - spring.ai.chat.client.token.usage (prompt + completion tokens)
// - spring.ai.embedding.operation.seconds

// Access via Micrometer / Actuator
// Traces appear in Zipkin, Jaeger, or any OTel-compatible backend
```

---

## Testing

```java
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.model.Generation;
import org.springframework.ai.chat.messages.AssistantMessage;

// Unit test with MockChatModel
class AssistantServiceTest {

    @Test
    void askReturnsMockedResponse() {
        // Arrange
        var mockResponse = new ChatResponse(
            List.of(new Generation(new AssistantMessage("Paris")))
        );
        ChatModel mockModel = prompt -> mockResponse;
        var service = new AssistantService(ChatClient.create(mockModel));

        // Act
        String result = service.ask("What is the capital of France?");

        // Assert
        assertThat(result).isEqualTo("Paris");
    }
}
```

```java
// Integration test with real model (requires API key in test env)
@SpringBootTest
class RagIntegrationTest {

    @Autowired
    private RagQueryService ragService;

    @Autowired
    private DocumentIngestionService ingestionService;

    @Test
    void queryReturnRelevantAnswer() {
        // Ingest test document
        ingestionService.ingestPdf(new ClassPathResource("test-data/spring-ai-docs.pdf"));

        // Query
        String answer = ragService.query("What is a ChatClient?", "test-session");

        assertThat(answer).isNotBlank();
        assertThat(answer.toLowerCase()).contains("chatclient");
    }
}
```

```java
// Test with AutoConfigureMockMvc for streaming endpoints
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class StreamingControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void streamChatReturnsFluxOfStrings() {
        webTestClient.get()
            .uri("/chat/stream?question=Hello")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(String.class)
            .hasSize(GreaterThan(0));
    }
}
```

---

## Model Abstraction — Swap Providers Without Code Changes

```
Application code uses ChatClient (provider-agnostic)
        │
        ▼
┌───────────────────────────────────┐
│          ChatClient               │
│  (same API regardless of model)   │
└───────────────┬───────────────────┘
                │  delegates to
        ┌───────▼───────┐
        │   ChatModel   │  ← interface
        └───────────────┘
                │
    ┌───────────┼────────────┐
    │           │            │
┌───▼───┐ ┌───▼───┐ ┌──────▼──────┐
│OpenAI │ │Claude │ │  Ollama     │
│ChatM..│ │ChatM..│ │  ChatModel  │
└───────┘ └───────┘ └─────────────┘

Switch via application.yml only — zero code change:
  spring.ai.anthropic.api-key + model → uses Claude
  spring.ai.openai.api-key + model    → uses OpenAI
  spring.ai.ollama.base-url + model   → uses Ollama
```

---

## Supported Providers

| Provider        | Starter                                          | Chat | Embed | Vision | Functions |
|-----------------|--------------------------------------------------|------|-------|--------|-----------|
| OpenAI          | `spring-ai-openai-spring-boot-starter`           | ✓    | ✓     | ✓      | ✓         |
| Anthropic Claude| `spring-ai-anthropic-spring-boot-starter`        | ✓    |       | ✓      | ✓         |
| Google Gemini   | `spring-ai-vertex-ai-gemini-spring-boot-starter` | ✓    | ✓     | ✓      | ✓         |
| Mistral         | `spring-ai-mistral-ai-spring-boot-starter`       | ✓    | ✓     |        | ✓         |
| Ollama (local)  | `spring-ai-ollama-spring-boot-starter`           | ✓    | ✓     | ✓      | ✓         |
| Azure OpenAI    | `spring-ai-azure-openai-spring-boot-starter`     | ✓    | ✓     | ✓      | ✓         |
| AWS Bedrock     | `spring-ai-bedrock-converse-spring-boot-starter` | ✓    | ✓     | ✓      | ✓         |

---

## When to Use Spring AI vs Raw SDK

```
Use Spring AI when:
  ✓ Spring Boot application (natural fit)
  ✓ Need to swap LLM providers without code change
  ✓ Building RAG: advisors handle retrieval automatically
  ✓ Want Micrometer observability out of the box
  ✓ Conversation memory managed declaratively via advisors
  ✓ Streaming via reactive Flux integrated in WebFlux stack

Use raw SDK (openai-java, anthropic-java) when:
  ✓ Non-Spring application (Jakarta EE, Quarkus, plain Java)
  ✓ Need deep provider-specific features (realtime API, extended thinking)
  ✓ Minimal dependency footprint required
  ✓ Ultra-low latency with no framework overhead
```

---

## Anti-Patterns

### 1. Creating ChatClient on Every Request

```java
// BAD: expensive, loses advisor configuration
public String ask(String question) {
    return ChatClient.create(chatModel).prompt().user(question).call().content();
}

// GOOD: inject a pre-configured ChatClient bean
@Service
public class AssistantService {
    private final ChatClient chatClient;  // injected once
    public AssistantService(ChatClient chatClient) { this.chatClient = chatClient; }
}
```

### 2. Using SimpleVectorStore in Production

```java
// BAD: lost on restart, not thread-safe under load
@Bean
public VectorStore vectorStore(EmbeddingModel em) {
    return new SimpleVectorStore(em);  // in-memory, ephemeral
}

// GOOD: use pgvector or Redis in production
// (configured via spring.ai.vectorstore.pgvector.*)
```

### 3. Ignoring Similarity Threshold in RAG

```java
// BAD: returns irrelevant docs when no good match exists
new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults().withTopK(5))

// GOOD: filter below relevance threshold
new QuestionAnswerAdvisor(
    vectorStore,
    SearchRequest.defaults()
        .withTopK(5)
        .withSimilarityThreshold(0.70)  // discard docs below 70% similarity
)
```

### 4. Blocking in Reactive Context

```java
// BAD: blocks the Reactor thread in a WebFlux app
@GetMapping("/chat")
public Mono<String> chat(String q) {
    String result = chatClient.prompt().user(q).call().content();  // blocking!
    return Mono.just(result);
}

// GOOD: use streaming or wrap in subscribeOn
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chat(String q) {
    return chatClient.prompt().user(q).stream().content();  // non-blocking
}
```

### 5. Hardcoding Model Names

```java
// BAD: locked to one model, hard to change
ChatOptions opts = OpenAiChatOptions.builder().withModel("gpt-4o").build();

// GOOD: externalize to application.yml
// spring.ai.openai.chat.options.model=gpt-4o
// Override per-environment without code change
```

### 6. Not Handling Tool Call Failures

```java
// BAD: tool exception propagates as 500
@Bean
@Description("Fetch stock price")
public Function<StockRequest, StockResponse> getStockPrice() {
    return request -> stockApi.getPrice(request.symbol());  // can throw
}

// GOOD: catch and return structured error
@Bean
@Description("Fetch stock price")
public Function<StockRequest, StockResponse> getStockPrice() {
    return request -> {
        try {
            return stockApi.getPrice(request.symbol());
        } catch (StockNotFoundException e) {
            return new StockResponse(request.symbol(), null, "Symbol not found");
        }
    };
}
```

---

## Production Checklist

- [ ] Inject `ChatClient` as a Spring bean — configure defaults once via `ChatClient.builder()`
- [ ] Externalize all model config to `application.yml` (model name, temperature, max-tokens)
- [ ] Use `pgvector` or `Redis` vector store — never `SimpleVectorStore` in production
- [ ] Set `similarityThreshold` on `SearchRequest` to avoid injecting irrelevant context
- [ ] Add `MessageChatMemoryAdvisor` with a unique `conversationId` per user session
- [ ] Use `stream().content()` in WebFlux endpoints — never block a Reactor thread
- [ ] Register Micrometer metrics endpoint and alert on p99 latency and token usage
- [ ] Wrap tool/function implementations in try-catch and return structured error responses
- [ ] Write unit tests with `MockChatModel` and integration tests with `@SpringBootTest`
- [ ] Tune `TokenTextSplitter` chunk size for your domain — test retrieval recall before ship
- [ ] Use `SafeGuardAdvisor` for any user-facing chat that accepts free-form input
- [ ] Pin `spring-ai-bom` version and test upgrades in staging — Spring AI evolves fast
