---
name: llm-observability
description: Use when setting up tracing, monitoring, logging, or cost tracking for LLM applications — covers LangSmith, LangFuse, Arize/Phoenix, metrics, alerting, and prompt versioning
---

# LLM Observability

## Why LLM Observability Is Different

Traditional APM assumes deterministic behavior: same input → same output, predictable latency, fixed cost per request. LLM applications break all three assumptions.

```
Traditional Service             LLM Application
─────────────────────           ──────────────────────────────
Deterministic output     vs.    Non-deterministic output
Microsecond latency      vs.    100ms–30s latency variance
Fixed cost per call      vs.    Cost scales with token count
No prompt concept        vs.    Prompt version matters as much as code version
Input = data             vs.    Input = prompt + data + model choice
```

What you MUST track that traditional observability ignores:
- Which prompt version generated which output
- Token counts (input and output separately — they have different prices)
- Model name and version at the time of each call
- Whether the request hit a cache
- LLM-specific errors: rate limits, context length exceeded, refusals, content filter blocks

---

## What to Trace Per LLM Call

Every LLM call trace record should include all of these fields:

```python
{
    # Identity
    "trace_id": "uuid-v4",           # Unique ID for the full request chain
    "span_id": "uuid-v4",            # ID for this specific LLM call
    "session_id": "session-abc123",  # Groups calls in one user session
    "user_id": "user-42",            # Which user triggered this

    # Model
    "model": "claude-3-5-sonnet-20241022",
    "provider": "anthropic",

    # Prompt
    "system_prompt": "You are a helpful assistant...",
    "user_message": "What is the capital of France?",
    "prompt_version": "v2.3.1",      # Treat prompts like code

    # Response
    "response": "The capital of France is Paris.",
    "finish_reason": "stop",         # stop | length | content_filter | tool_calls

    # Tokens and cost
    "input_tokens": 42,
    "output_tokens": 11,
    "total_tokens": 53,
    "cost_usd": 0.000159,            # Compute and store; don't trust vendor dashboards

    # Latency
    "latency_ms": 843,
    "time_to_first_token_ms": 210,   # Streaming: when did tokens start flowing?

    # Context
    "tags": ["production", "feature-search", "experiment-a"],
    "cache_hit": false,
    "temperature": 0.7,
    "timestamp": "2025-01-15T14:23:11Z"
}
```

ASCII trace structure for a RAG pipeline:

```
Request ──► [trace_id: abc-123]
               │
               ├── span: query_rewrite (latency: 210ms)
               │       input_tokens: 55, output_tokens: 12
               │
               ├── span: retrieval (latency: 45ms)
               │       [not an LLM call — but still trace it]
               │
               └── span: generation (latency: 1240ms)
                       model: gpt-4o
                       input_tokens: 1820, output_tokens: 312
                       cost_usd: 0.00267
```

---

## LangSmith

LangSmith is LangChain's tracing, evaluation, and dataset platform. It works with both LangChain code and plain Python.

### Auto-Tracing (LangChain)

Set environment variables — all LangChain/LangGraph calls are traced automatically:

```bash
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
export LANGCHAIN_API_KEY=ls__your_key_here
export LANGCHAIN_PROJECT=my-production-app
```

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate

# Traced automatically — no code changes needed
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer concisely."),
    ("human", "{question}")
])
chain = prompt | llm

result = chain.invoke({"question": "What is RAG?"})
# LangSmith now has: input, output, latency, tokens, cost
```

### Manual Tracing with `@traceable`

For non-LangChain Python code:

```python
from langsmith import traceable
from anthropic import Anthropic

client = Anthropic()

@traceable(
    name="answer_question",
    tags=["production", "v2"],
    metadata={"prompt_version": "2.3.1"}
)
def answer_question(question: str, user_id: str) -> str:
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=[{"role": "user", "content": question}]
    )
    return response.content[0].text

# The decorator wraps the function and sends trace data to LangSmith
result = answer_question("What is Paris?", user_id="user-42")
```

### RunTree for Fine-Grained Control

```python
from langsmith.run_trees import RunTree

# Create a parent run (the full request)
parent = RunTree(
    name="rag_pipeline",
    run_type="chain",
    inputs={"query": "How does HNSW work?"},
    project_name="production"
)
parent.post()

# Child: retrieval span
retrieval = parent.create_child(
    name="vector_search",
    run_type="retrieval",
    inputs={"query": "How does HNSW work?", "k": 5}
)
retrieved_docs = vector_store.similarity_search("How does HNSW work?", k=5)
retrieval.end(outputs={"documents": [d.page_content for d in retrieved_docs]})
retrieval.post()

# Child: generation span
generation = parent.create_child(
    name="llm_call",
    run_type="llm",
    inputs={"messages": [{"role": "user", "content": "..."}]}
)
response = llm.invoke("...")
generation.end(outputs={"response": response.content})
generation.post()

parent.end(outputs={"answer": response.content})
parent.post()
```

### Datasets and Evaluation

Collect production traces into evaluation datasets:

```python
from langsmith import Client

client = Client()

# Create dataset from production traces
dataset = client.create_dataset(
    "rag_golden_set",
    description="Production queries with verified answers"
)

# Add examples manually
client.create_examples(
    inputs=[{"question": "What is RAG?"}],
    outputs=[{"answer": "RAG stands for Retrieval-Augmented Generation..."}],
    dataset_id=dataset.id
)

# Or push traces from the UI — mark any production trace as "Add to Dataset"
```

Run evaluation against the dataset:

```python
from langsmith.evaluation import evaluate

def my_app(inputs: dict) -> dict:
    result = chain.invoke({"question": inputs["question"]})
    return {"answer": result.content}

def correctness_evaluator(run, example) -> dict:
    # Use LLM to judge correctness
    score = judge_correctness(run.outputs["answer"], example.outputs["answer"])
    return {"score": score, "key": "correctness"}

results = evaluate(
    my_app,
    data="rag_golden_set",
    evaluators=[correctness_evaluator],
    experiment_prefix="prompt-v2.3"
)
```

### User Feedback

Link thumbs-up/thumbs-down feedback to specific traces:

```python
from langsmith import Client

client = Client()

# After the LLM call, you have the run_id from the trace
client.create_feedback(
    run_id="run-uuid-here",
    key="user_rating",
    score=1,        # 1 = thumbs up, 0 = thumbs down
    comment="Great answer, very concise"
)
```

---

## LangFuse

LangFuse is an open-source LLM observability platform. Self-hostable or cloud. Works with any LLM provider.

### SDK Hierarchy: Traces → Spans → Generations

```
trace (one user request)
  └── span (retrieval step)
  └── span (augmentation step)
      └── generation (the actual LLM call — special span type)
          score (evaluation score attached here)
```

```python
from langfuse import Langfuse

langfuse = Langfuse(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    host="https://cloud.langfuse.com"  # or your self-hosted URL
)

# 1. Create a trace for the full user request
trace = langfuse.trace(
    name="rag_pipeline",
    user_id="user-42",
    session_id="session-abc",
    tags=["production"],
    metadata={"prompt_version": "2.3.1"}
)

# 2. Span for retrieval (non-LLM step)
retrieval_span = trace.span(
    name="vector_retrieval",
    input={"query": "How does HNSW work?"},
)
docs = vector_store.search("How does HNSW work?")
retrieval_span.end(output={"num_docs": len(docs)})

# 3. Generation for the LLM call
generation = trace.generation(
    name="answer_generation",
    model="gpt-4o",
    model_parameters={"temperature": 0.7},
    input=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": f"Context: {docs}\n\nQuestion: How does HNSW work?"}
    ]
)
response = openai_client.chat.completions.create(
    model="gpt-4o",
    messages=[...]
)
generation.end(
    output=response.choices[0].message.content,
    usage={
        "input": response.usage.prompt_tokens,
        "output": response.usage.completion_tokens,
    }
)

# 4. Attach evaluation score
generation.score(
    name="faithfulness",
    value=0.92,
    comment="Answer is grounded in retrieved context"
)

langfuse.flush()  # Ensure all events are sent before process exits
```

### LangChain Integration (Callback)

```python
from langfuse.callback import CallbackHandler

langfuse_handler = CallbackHandler(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    user_id="user-42",
    session_id="session-abc"
)

# Pass as callback — all chain steps are traced automatically
result = chain.invoke(
    {"question": "What is RAG?"},
    config={"callbacks": [langfuse_handler]}
)
```

### Decorator-Based Tracing

```python
from langfuse.decorators import langfuse_context, observe

@observe()  # Creates a span automatically
def retrieve_documents(query: str) -> list[str]:
    docs = vector_store.search(query)
    langfuse_context.update_current_observation(
        output={"num_docs": len(docs)},
        metadata={"index": "production-v3"}
    )
    return docs

@observe()  # Parent trace
def rag_pipeline(question: str) -> str:
    langfuse_context.update_current_trace(
        user_id="user-42",
        tags=["production"]
    )
    docs = retrieve_documents(question)  # Nested span automatically
    answer = generate_answer(question, docs)
    return answer
```

### Self-Hosting LangFuse

```yaml
# docker-compose.yml (minimal)
services:
  langfuse-server:
    image: langfuse/langfuse:latest
    environment:
      DATABASE_URL: postgresql://langfuse:password@db:5432/langfuse
      NEXTAUTH_SECRET: your-secret-32-chars
      SALT: your-salt-32-chars
    ports:
      - "3000:3000"
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_USER: langfuse
      POSTGRES_DB: langfuse
```

---

## Arize / Phoenix

Phoenix (open-source from Arize) focuses on embedding visualization and RAG evaluation using the OpenInference tracing standard.

### OpenInference — Open Tracing Standard

OpenInference defines a vendor-neutral span schema for LLM apps, so traces are portable across tools.

Key span attributes:
```
openinference.span.kind       = LLM | CHAIN | RETRIEVER | RERANKER | AGENT | TOOL
input.value                   = the prompt or query
output.value                  = the response
llm.model_name                = "gpt-4o"
llm.token_count.prompt        = 820
llm.token_count.completion    = 142
retrieval.documents           = [{document.content, document.score, document.id}]
```

```python
import phoenix as px
from openinference.instrumentation.openai import OpenAIInstrumentor
from opentelemetry import trace as otel_trace
from opentelemetry.sdk.trace import TracerProvider

# Start Phoenix locally (opens at http://localhost:6006)
px.launch_app()

# Instrument OpenAI — all calls traced automatically
provider = TracerProvider()
OpenAIInstrumentor().instrument(tracer_provider=provider)

# Your existing code is now traced
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What is HNSW?"}]
)
# Trace appears in Phoenix UI with embeddings, latency, tokens
```

### Embeddings Visualization

Phoenix can project your query embeddings into 2D (UMAP) to identify:
- Clusters of similar queries (topic discovery)
- Outlier queries (edge cases)
- Query drift over time (distribution shift)

```python
import pandas as pd
import phoenix as px

# Export your query embeddings and responses
queries_df = pd.DataFrame({
    "query": ["What is RAG?", "How does HNSW work?", ...],
    "embedding": [embed("What is RAG?"), embed("How does HNSW work?"), ...],
    "response": ["RAG is...", "HNSW is...", ...],
    "latency_ms": [820, 1240, ...]
})

schema = px.Schema(
    prompt_column_names=px.EmbeddingColumnNames(
        vector_column_name="embedding",
        raw_data_column_name="query"
    )
)

dataset = px.Dataset(queries_df, schema, name="production_queries")
px.launch_app(primary=dataset)
# Opens UMAP visualization in browser
```

### RAG Retrieval Evaluation

```python
from phoenix.evals import (
    HallucinationEvaluator,
    QAEvaluator,
    RelevanceEvaluator,
    run_evals
)

# Evaluate a batch of RAG responses
results = run_evals(
    dataframe=traces_df,  # columns: input, output, reference (retrieved context)
    evaluators=[
        HallucinationEvaluator(model=judge_model),  # Is the answer in the context?
        QAEvaluator(model=judge_model),              # Does the answer address the query?
        RelevanceEvaluator(model=judge_model),       # Is the context relevant?
    ]
)
```

---

## Metrics to Track

### Latency

```
Metric          Target              Alert Threshold
──────────────────────────────────────────────────────────
P50 latency     < 500ms             > 800ms sustained
P95 latency     < 2000ms            > 4000ms for 5min
P99 latency     < 5000ms            > 10000ms
TTFT (stream)   < 200ms             > 500ms
```

Track latency *per model* and *per chain step* — a slow P99 might be one step, not the whole chain.

### Token Usage

```python
# Compute cost at trace time; don't rely on vendor estimates
PRICING = {
    "gpt-4o": {"input": 2.50, "output": 10.00},          # per 1M tokens
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "claude-3-5-sonnet-20241022": {"input": 3.00, "output": 15.00},
    "text-embedding-3-small": {"input": 0.02, "output": 0.0},
}

def compute_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    prices = PRICING[model]
    return (
        input_tokens * prices["input"] / 1_000_000 +
        output_tokens * prices["output"] / 1_000_000
    )
```

Aggregate dimensions:
- Cost per request (alert on anomalies)
- Cost per user per day (identify heavy users)
- Cost per feature / endpoint
- Daily total (set budget alerts)

### Error Rate

Track these separately — they have different root causes:

```
Error Type              Root Cause              Action
───────────────────────────────────────────────────────────────
API timeout             Network / model load    Retry with backoff
Rate limit (429)        Too many requests       Implement token bucket
Context length (400)    Prompt too long         Truncate or chunk
Content filter block    Policy violation        Log and review
Refusal                 Model policy            Rephrase prompt
JSON parse error        Bad structured output   Add output validation
```

### Cache Hit Rate

```python
# Log whether each call was a cache hit
# Anthropic prompt caching: cache_creation_input_tokens vs cache_read_input_tokens
response = anthropic_client.messages.create(...)
usage = response.usage

cache_hit = usage.cache_read_input_tokens > 0
cache_hit_rate = cache_reads / total_calls  # Track over time

# Cost savings from cache
cache_savings = usage.cache_read_input_tokens * 0.30 / 1_000_000  # 90% cheaper
```

---

## Prompt Versioning

Treat prompts like code. Every change to a prompt is a deployment.

```
Prompt versioning rules:
  1. Store prompts in version control (not hardcoded strings)
  2. Each version has a semver: v2.3.1
  3. Every trace records the prompt version that was used
  4. Never change a prompt in production without creating a new version
  5. A/B test prompt versions before full rollout
```

```python
# prompts/answer_question/v2.3.1.txt
SYSTEM_PROMPT = """
You are a helpful assistant. Answer the user's question based only on the
provided context. If the answer is not in the context, say "I don't know."
Be concise — answer in 2-3 sentences maximum.
"""

# Load prompt by version — never hardcode in application logic
def load_prompt(name: str, version: str) -> str:
    path = f"prompts/{name}/{version}.txt"
    return Path(path).read_text()

# Always tag your trace with the prompt version
@traceable(metadata={"prompt_version": "2.3.1"})
def answer(question: str, context: str) -> str:
    prompt = load_prompt("answer_question", "2.3.1")
    ...
```

LangSmith Prompt Hub (managed prompt versioning):

```python
from langsmith import Client

client = Client()

# Push a prompt version
client.push_prompt(
    "answer-question",
    object=prompt_template,
    description="v2.3.1 — added conciseness instruction"
)

# Pull a specific version in production
prompt = client.pull_prompt("answer-question:v2.3.1")
```

---

## Production Logging

Every LLM call should emit a structured JSON log line:

```python
import json
import logging
import time
from uuid import uuid4

logger = logging.getLogger("llm")

def traced_llm_call(
    model: str,
    messages: list,
    user_id: str,
    session_id: str,
    prompt_version: str,
    tags: list[str]
) -> dict:
    trace_id = str(uuid4())
    start = time.perf_counter()

    try:
        response = client.messages.create(model=model, messages=messages, max_tokens=1024)
        latency_ms = int((time.perf_counter() - start) * 1000)

        log_record = {
            "event": "llm_call",
            "trace_id": trace_id,
            "user_id": user_id,
            "session_id": session_id,
            "model": model,
            "prompt_version": prompt_version,
            "tags": tags,
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "cost_usd": compute_cost(model, response.usage.input_tokens, response.usage.output_tokens),
            "latency_ms": latency_ms,
            "finish_reason": response.stop_reason,
            "error": None,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        }
        logger.info(json.dumps(log_record))
        return {"content": response.content[0].text, "trace_id": trace_id}

    except Exception as e:
        latency_ms = int((time.perf_counter() - start) * 1000)
        logger.error(json.dumps({
            "event": "llm_call_error",
            "trace_id": trace_id,
            "model": model,
            "error": str(e),
            "error_type": type(e).__name__,
            "latency_ms": latency_ms,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        }))
        raise
```

---

## Alerting

```
Alert                   Condition                           Severity
──────────────────────────────────────────────────────────────────────
Latency spike           P95 > 5s for 5 consecutive minutes  WARNING
Latency critical        P99 > 15s                            CRITICAL
Cost anomaly            Hourly cost > 2× 7-day average       WARNING
Daily budget breach     Daily spend > budget threshold        CRITICAL
Error rate high         Error rate > 5% over 10 minutes      WARNING
Rate limit storm        429 errors > 10% of calls            WARNING
Context length breach   > 5% of calls hit max context        WARNING
Cache hit rate drop     Cache hit rate drops > 20%           INFO
```

Java example with Micrometer:

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import io.micrometer.core.instrument.Counter;

@Service
public class ObservableLlmClient {

    private final MeterRegistry registry;

    public ObservableLlmClient(MeterRegistry registry) {
        this.registry = registry;
    }

    public String call(String model, String prompt, String userId) {
        Timer.Sample sample = Timer.start(registry);
        try {
            String response = llmProvider.call(model, prompt);

            sample.stop(Timer.builder("llm.call.latency")
                .tag("model", model)
                .tag("status", "success")
                .register(registry));

            registry.counter("llm.tokens.input",
                "model", model, "user_id", userId)
                .increment(countTokens(prompt));

            registry.counter("llm.tokens.output",
                "model", model, "user_id", userId)
                .increment(countTokens(response));

            return response;

        } catch (Exception e) {
            sample.stop(Timer.builder("llm.call.latency")
                .tag("model", model)
                .tag("status", "error")
                .tag("error_type", e.getClass().getSimpleName())
                .register(registry));

            registry.counter("llm.errors",
                "model", model,
                "error_type", e.getClass().getSimpleName())
                .increment();

            throw e;
        }
    }
}
```

---

## Observability Platform Comparison

```
Platform      Open Source  Self-Host  LangChain  Cost Model
────────────────────────────────────────────────────────────────
LangSmith     No           No         Native      Per trace
LangFuse      Yes          Yes        Callback    Open-source free
Phoenix/Arize Partial      Yes        Callback    Open-source free
Helicone      No           Partial    Proxy       Per request
Langfuse Cloud Yes         Cloud      Callback    Freemium
```

---

## Production Checklist

- [ ] Every LLM call emits a structured JSON log with all required fields
- [ ] Prompt version is recorded on every trace
- [ ] Latency P50/P95/P99 tracked per model and per chain step
- [ ] Token usage (input/output) tracked per request, per user, per day
- [ ] Cost computed and stored at trace time
- [ ] Error rate tracked by error type, not just total
- [ ] Cache hit rate monitored with weekly trend
- [ ] Alerts configured for latency spike, cost anomaly, error rate
- [ ] Production traces feed into evaluation datasets
- [ ] User feedback linked to traces
- [ ] Dashboard shows daily cost vs budget with projection

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Logging only errors | Blind to latency drift and token bloat | Log every call |
| Tracking total tokens only | Can't optimize; input and output price differently | Track input and output separately |
| No prompt versioning | Can't attribute performance changes to prompt changes | Version prompts in git |
| Vendor dashboard as source of truth | Delayed, aggregated, no custom dimensions | Compute cost at call time |
| No trace IDs | Can't correlate logs with user reports | Generate UUID per request, pass through chain |
| One giant trace per session | Can't identify which step is slow | Span each chain step separately |
| No sampling strategy | At high volume, storing every trace is expensive | Sample 100% errors + 10% successes |
