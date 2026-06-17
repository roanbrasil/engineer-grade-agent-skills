---
name: langchain-patterns
description: Expert LangChain patterns for Python (v0.2/v0.3+) — invoke when building RAG pipelines, agent chains, LCEL expressions, tool-calling agents, memory-backed conversations, or migrating from LangChain v0.1.
---

# LangChain Patterns — Expert Reference (v0.2/v0.3+)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LangChain Stack                              │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │  Prompt  │  │  Model   │  │  Output  │  │    Retrievers &    │ │
│  │Templates │→ │ (LLM /   │→ │  Parser  │  │    Vector Stores   │ │
│  │          │  │  Chat)   │  │          │  │                    │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘ │
│       │              │              │                │              │
│       └──────────────┴──────────────┴────────────────┘              │
│                              │                                      │
│                    ┌─────────▼──────────┐                           │
│                    │  LCEL / Runnable   │ ← pipe syntax `|`         │
│                    │  (invoke / stream  │                           │
│                    │   batch / async)   │                           │
│                    └─────────┬──────────┘                           │
│                              │                                      │
│              ┌───────────────┼───────────────┐                      │
│              │               │               │                      │
│       ┌──────▼──────┐ ┌──────▼──────┐ ┌─────▼──────┐              │
│       │   Chains    │ │   Agents    │ │  Tools &   │              │
│       │ (RAG, etc.) │ │ (ReAct,     │ │ Tool Calls │              │
│       │             │ │  tool-call) │ │            │              │
│       └─────────────┘ └─────────────┘ └────────────┘              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Callbacks / LangSmith tracing (cross-cuts everything)       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## Package Structure (v0.3+)

```
langchain-core          ← Runnable protocol, base abstractions (no LLM deps)
langchain               ← Chains, agents, memory (thin orchestration layer)
langchain-community     ← Third-party integrations (retrievers, tools, loaders)
langchain-openai        ← OpenAI-specific: ChatOpenAI, OpenAIEmbeddings
langchain-anthropic     ← Anthropic: ChatAnthropic
langchain-google-*      ← Google Vertex / GenAI
langgraph               ← Stateful graph-based agents (separate package)
langsmith               ← Observability and tracing
```

Install only what you need:
```bash
pip install langchain-core langchain langchain-openai langsmith
```

---

## LCEL — LangChain Expression Language

LCEL is the composable, lazy-evaluated pipeline syntax. It gives you streaming, async, batching, and LangSmith tracing for free — without changing your chain logic.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Prompt  │ ──▶ │  Model   │ ──▶ │  Parser  │
│ Template │  |  │ ChatOpenAI│  |  │  Str/JSON│
└──────────┘  |  └──────────┘  |  └──────────┘
              pipe operator connects Runnables
```

### Runnable Protocol

Every component in LangChain implements `Runnable`:

| Method     | Signature                              | Use case                        |
|------------|----------------------------------------|---------------------------------|
| `invoke`   | `invoke(input) -> output`              | Single synchronous call         |
| `stream`   | `stream(input) -> Iterator[chunk]`     | Streaming tokens to stdout/UI   |
| `batch`    | `batch([input1, input2]) -> [outputs]` | Parallel processing             |
| `ainvoke`  | `await ainvoke(input) -> output`       | Async single call               |
| `astream`  | `async for chunk in astream(input)`    | Async streaming                 |
| `abatch`   | `await abatch([...]) -> [...]`         | Async parallel                  |

### Basic LCEL Chain

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# Each component is a Runnable
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant specializing in {domain}."),
    ("human", "{question}"),
])
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = StrOutputParser()

# Pipe | builds a RunnableSequence
chain = prompt | model | parser

# invoke — synchronous
result = chain.invoke({"domain": "Python", "question": "What is a generator?"})
print(result)

# stream — yields string chunks as they arrive
for chunk in chain.stream({"domain": "Python", "question": "What is a generator?"}):
    print(chunk, end="", flush=True)

# batch — runs inputs in parallel (threads under the hood)
results = chain.batch([
    {"domain": "Python", "question": "What is a generator?"},
    {"domain": "Python", "question": "What is a decorator?"},
])

# async
import asyncio
async def main():
    result = await chain.ainvoke({"domain": "Python", "question": "What is asyncio?"})
    async for chunk in chain.astream({"domain": "Python", "question": "What is asyncio?"}):
        print(chunk, end="", flush=True)

asyncio.run(main())
```

### RunnableParallel — Fan-Out

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

# Run two sub-chains in parallel and merge results
chain = RunnableParallel(
    summary=prompt_summary | model | parser,
    keywords=prompt_keywords | model | parser,
)
result = chain.invoke({"text": "Long document..."})
# result = {"summary": "...", "keywords": "..."}

# Pass input through unchanged alongside other operations
chain = RunnableParallel(
    original=RunnablePassthrough(),
    transformed=some_transform_chain,
)
```

### RunnableLambda — Arbitrary Functions

```python
from langchain_core.runnables import RunnableLambda

def preprocess(text: str) -> str:
    return text.strip().lower()

chain = RunnableLambda(preprocess) | prompt | model | parser
```

### Branching with RunnableBranch

```python
from langchain_core.runnables import RunnableBranch

branch = RunnableBranch(
    (lambda x: "code" in x["question"].lower(), code_chain),
    (lambda x: "math" in x["question"].lower(), math_chain),
    default_chain,  # fallback
)
```

---

## Prompt Templates

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    MessagesPlaceholder,
    HumanMessagePromptTemplate,
    SystemMessagePromptTemplate,
)
from langchain_core.messages import HumanMessage, AIMessage

# Standard chat prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an expert in {domain}. Answer concisely."),
    MessagesPlaceholder(variable_name="chat_history"),  # inject list of messages
    ("human", "{question}"),
])

# Partial templates — pre-fill some variables
partial_prompt = prompt.partial(domain="distributed systems")
# Now only requires: chat_history, question

# Format and inspect
messages = prompt.format_messages(
    domain="Python",
    chat_history=[
        HumanMessage(content="Hi"),
        AIMessage(content="Hello! How can I help?"),
    ],
    question="What is asyncio?",
)

# From a template string (for simple cases)
from langchain_core.prompts import PromptTemplate
template = PromptTemplate.from_template("Summarize this: {text}")
```

---

## Output Parsers

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field

# StrOutputParser — extracts .content string from AIMessage
parser = StrOutputParser()

# JsonOutputParser — streams partial JSON as it arrives
parser = JsonOutputParser()
chain = prompt | model | parser
result = chain.invoke(...)  # returns dict

# PydanticOutputParser — validates and returns typed object
class MovieReview(BaseModel):
    title: str = Field(description="Movie title")
    rating: float = Field(description="Rating 0-10")
    summary: str = Field(description="One-line summary")

from langchain_core.output_parsers import PydanticOutputParser
parser = PydanticOutputParser(pydantic_object=MovieReview)

# Inject format instructions into the prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "Extract movie info.\n{format_instructions}"),
    ("human", "{text}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
review: MovieReview = chain.invoke({"text": "Inception is a 9/10 mind-bending thriller..."})

# With structured output (preferred when model supports it — more reliable)
structured_model = model.with_structured_output(MovieReview)
chain = prompt | structured_model
```

---

## Tools

### @tool Decorator

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field

# Simple tool
@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    # real implementation would call a weather API
    return f"Sunny, 22°C in {city}"

# Tool with Pydantic schema for complex inputs
class SearchInput(BaseModel):
    query: str = Field(description="Search query string")
    max_results: int = Field(default=5, description="Maximum number of results")

@tool(args_schema=SearchInput)
def web_search(query: str, max_results: int = 5) -> list[str]:
    """Search the web and return a list of result snippets."""
    # real implementation
    return [f"Result {i} for '{query}'" for i in range(max_results)]

# Inspect the tool schema
print(get_weather.name)          # "get_weather"
print(get_weather.description)   # docstring
print(get_weather.args)          # input schema dict
```

### Tool Calling vs ReAct

```
Tool Calling (preferred):
  ┌──────┐  tool_calls JSON  ┌────────────┐  results  ┌──────┐
  │ LLM  │ ────────────────▶ │ Tool Exec  │ ────────▶ │ LLM  │
  └──────┘                   └────────────┘            └──────┘
  Model natively outputs structured JSON → reliable, fast, parseable

ReAct (text-based, use when model lacks tool calling):
  Thought: I need to look up the weather
  Action: get_weather
  Action Input: {"city": "London"}
  Observation: Sunny, 22°C
  Thought: I now have the answer
  Final Answer: It is sunny and 22°C in London
  → Requires parsing free text → fragile, slower
```

### create_tool_calling_agent

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

model = ChatOpenAI(model="gpt-4o", temperature=0)
tools = [get_weather, web_search]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant. Use tools when needed."),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),  # required for tool call history
])

agent = create_tool_calling_agent(model, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({
    "input": "What is the weather in Paris and London?",
    "chat_history": [],
})
print(result["output"])
```

---

## Memory

```
Memory type selection:
┌─────────────────────────────────────────────────────────┐
│  Context window small? Use BufferWindowMemory (last N)   │
│  Context window large? Use ConversationBufferMemory      │
│  Very long conversations? Use ConversationSummaryMemory  │
└─────────────────────────────────────────────────────────┘
```

```python
from langchain.memory import (
    ConversationBufferMemory,
    ConversationBufferWindowMemory,
    ConversationSummaryMemory,
)

# Buffer: keeps full history — simple, but grows unbounded
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True,  # returns Message objects (not string)
)

# Window: keeps last k exchanges only
memory = ConversationBufferWindowMemory(
    k=5,
    memory_key="chat_history",
    return_messages=True,
)

# Summary: LLM summarizes old turns to compress context
memory = ConversationSummaryMemory(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    memory_key="chat_history",
    return_messages=True,
)

# Usage with a chain (manual pattern — preferred in v0.3+)
chat_history = []

def chat(question: str) -> str:
    response = chain.invoke({
        "question": question,
        "chat_history": chat_history,
    })
    chat_history.append(HumanMessage(content=question))
    chat_history.append(AIMessage(content=response))
    return response
```

---

## Retrievers & RAG

### Vector Store Retriever

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.documents import Document

# Index documents
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(documents=docs, embedding=embeddings)

# Basic retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",  # or "mmr" for diversity
    search_kwargs={"k": 5},
)

# Retrieve
docs = retriever.invoke("What is LCEL?")
```

### MultiQueryRetriever — Generates Multiple Queries

```python
from langchain.retrievers import MultiQueryRetriever

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=ChatOpenAI(temperature=0),
    # generates 3 alternate phrasings of the question
    # retrieves for all, deduplicates results
)
```

### ContextualCompressionRetriever — Filters Irrelevant Chunks

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(llm)
retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)
```

### EnsembleRetriever — Combines Dense + Sparse

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25 = BM25Retriever.from_documents(docs, k=5)
dense = vectorstore.as_retriever(search_kwargs={"k": 5})

# Reciprocal Rank Fusion merges both result lists
ensemble = EnsembleRetriever(
    retrievers=[bm25, dense],
    weights=[0.4, 0.6],  # BM25 40%, dense 60%
)
```

### RAG Chain (Full Pattern)

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

def format_docs(docs: list[Document]) -> str:
    return "\n\n".join(doc.page_content for doc in docs)

prompt = ChatPromptTemplate.from_messages([
    ("system", """Answer using only the provided context.
If the answer is not in the context, say 'I don't know'.

Context:
{context}"""),
    ("human", "{question}"),
])

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough(),
    }
    | prompt
    | model
    | StrOutputParser()
)

answer = rag_chain.invoke("How does LCEL enable streaming?")
```

### RAG with Source Documents

```python
from langchain_core.runnables import RunnableParallel

rag_chain_with_sources = RunnableParallel(
    answer=rag_chain,
    sources=retriever,  # returns the raw Document list
)
result = rag_chain_with_sources.invoke("How does LCEL enable streaming?")
print(result["answer"])
for doc in result["sources"]:
    print(doc.metadata["source"])
```

---

## Document Loading & Splitting

```python
from langchain_community.document_loaders import (
    PyPDFLoader, WebBaseLoader, DirectoryLoader
)
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Load
loader = PyPDFLoader("document.pdf")
pages = loader.load()  # list of Document(page_content, metadata)

# Split
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,          # overlap reduces context loss at boundaries
    separators=["\n\n", "\n", " ", ""],  # tries these in order
)
chunks = splitter.split_documents(pages)

# Index
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings(),
    persist_directory="./chroma_db",
)
```

---

## Callbacks

```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult

class TokenStreamHandler(BaseCallbackHandler):
    def on_llm_new_token(self, token: str, **kwargs) -> None:
        print(token, end="", flush=True)

    def on_llm_start(self, serialized, prompts, **kwargs) -> None:
        print("\n[LLM called]")

    def on_chain_error(self, error: Exception, **kwargs) -> None:
        print(f"\n[Error]: {error}")

# Pass at invocation time (preferred — no global state)
result = chain.invoke(
    {"question": "What is Python?"},
    config={"callbacks": [TokenStreamHandler()]},
)

# Or pass at construction time (applies to all invocations)
model_with_callbacks = ChatOpenAI(callbacks=[TokenStreamHandler()])
```

---

## LangSmith Integration

```python
import os

# Set env vars — tracing is automatic once set
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls-..."
os.environ["LANGCHAIN_PROJECT"] = "my-rag-app"  # optional project name

# Every chain.invoke() / chain.stream() is now traced automatically
# View at: https://smith.langchain.com

# Add metadata to a specific run
result = chain.invoke(
    {"question": "..."},
    config={
        "run_name": "rag-question-answering",
        "tags": ["production", "v2"],
        "metadata": {"user_id": "u-123"},
    },
)

# Evaluate with LangSmith
from langsmith.evaluation import evaluate

def correctness_evaluator(run, example):
    # custom evaluation logic
    score = 1.0 if example.outputs["answer"] in run.outputs["answer"] else 0.0
    return {"score": score, "key": "correctness"}

evaluate(
    chain.invoke,
    data="my-dataset-name",
    evaluators=[correctness_evaluator],
)
```

---

## Migration: v0.1 → v0.3

| v0.1 (deprecated)                         | v0.3 (current)                              |
|--------------------------------------------|---------------------------------------------|
| `from langchain.chat_models import ChatOpenAI` | `from langchain_openai import ChatOpenAI`   |
| `LLMChain(llm=..., prompt=...)`            | `prompt \| model \| parser` (LCEL)          |
| `ConversationalRetrievalChain`             | Custom LCEL RAG chain                       |
| `initialize_agent(tools, llm, agent=...)`  | `create_tool_calling_agent` + `AgentExecutor` |
| `from langchain.tools import Tool`         | `from langchain_core.tools import tool`     |
| `AgentType.OPENAI_FUNCTIONS`               | `create_tool_calling_agent`                 |
| `langchain.embeddings`                     | `langchain_openai.OpenAIEmbeddings`         |

---

## When to Use LangChain vs Raw SDK

```
Use LangChain when:
  ✓ Building RAG pipelines (retriever → prompt → model → parser)
  ✓ Multi-step agent loops with tool use
  ✓ Swapping LLM providers without rewriting logic
  ✓ Need streaming + async + batching with one API surface
  ✓ Want LangSmith tracing for free

Use raw SDK (openai, anthropic) when:
  ✓ Single LLM call with no chaining
  ✓ Tight latency budget (no abstraction overhead)
  ✓ Deep provider-specific feature usage (realtime API, extended thinking)
  ✓ Team unfamiliar with LangChain patterns
  ✓ Minimal dependency footprint required
```

---

## Anti-Patterns

### 1. Over-Chaining Simple Operations
```python
# BAD: unnecessary LCEL for a single call
chain = prompt | model | parser
result = chain.invoke({"q": "2+2?"})  # overkill if no reuse

# GOOD: just call the model directly
response = model.invoke(prompt.format_messages(q="2+2?"))
```

### 2. Using Deprecated LLMChain
```python
# BAD: deprecated, will be removed
from langchain.chains import LLMChain
chain = LLMChain(llm=model, prompt=prompt)

# GOOD: LCEL
chain = prompt | model | StrOutputParser()
```

### 3. Not Streaming Long Outputs
```python
# BAD: user waits 30s before seeing anything
result = chain.invoke({"question": "Write a 1000-word essay on..."})
print(result)

# GOOD: stream tokens as they arrive
for chunk in chain.stream({"question": "Write a 1000-word essay on..."}):
    print(chunk, end="", flush=True)
```

### 4. Ignoring Tool Call Exceptions
```python
# BAD: no error handling
executor = AgentExecutor(agent=agent, tools=tools)

# GOOD: handle tool errors gracefully
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True,   # recovers from output parse failures
    max_iterations=10,            # prevents infinite loops
    early_stopping_method="generate",
)
```

### 5. Unbounded Memory Growth
```python
# BAD: memory grows forever in long-running app
memory = ConversationBufferMemory()

# GOOD: window or summary memory
memory = ConversationBufferWindowMemory(k=10)
```

### 6. Retriever Without Reranking on Large Corpora
```python
# BAD on >10k docs: pure similarity may return off-topic chunks
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# GOOD: retrieve more, then compress/rerank
retriever = ContextualCompressionRetriever(
    base_compressor=LLMChainExtractor.from_llm(llm),
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)
```

---

## Production Checklist

- [ ] Use `langchain-core` as only dependency where possible (lighter)
- [ ] Set `LANGCHAIN_TRACING_V2=true` and `LANGCHAIN_API_KEY` in production
- [ ] Always pass `max_iterations` to `AgentExecutor` to prevent runaway loops
- [ ] Use `.with_structured_output(Schema)` instead of `PydanticOutputParser` when model supports it
- [ ] Prefer `ChatPromptTemplate.from_messages` over string templates for chat models
- [ ] Set `temperature=0` for deterministic tool-calling agents
- [ ] Use `RetryOutputParser` or `OutputFixingParser` for unreliable JSON parsing
- [ ] Test retrieval quality independently before testing end-to-end RAG
- [ ] Pin `langchain`, `langchain-core`, and provider package versions
- [ ] Use async (`ainvoke`, `astream`) in FastAPI / async web frameworks
