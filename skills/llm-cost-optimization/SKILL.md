---
name: llm-cost-optimization
description: LLM cost optimization for production systems — invoke when reducing API spend, designing caching strategies, routing requests, or budgeting token usage
---

# LLM Cost Optimization

## Cost Components

```
Total Cost = (Input Tokens × Input Price/M)
           + (Output Tokens × Output Price/M)
           + (Cached Input Tokens × Cache Price/M)   [if caching]
           + (Batch Discount if applicable)

Key ratio: Output tokens cost 3-5× more than input tokens.
Minimize output aggressively. Input is cheap; output is expensive.
```

### Reference Pricing (approximate — always verify current pricing)

```
Model                   Input /M    Output /M   Cache /M
────────────────────────────────────────────────────────
Claude Haiku 3.5        $0.80       $4.00       $0.08
Claude Sonnet 4.5       $3.00       $15.00      $0.30
Claude Opus 4           $15.00      $75.00      $1.50
GPT-4o mini             $0.15       $0.60       $0.075
GPT-4o                  $2.50       $10.00      $1.25
Gemini 1.5 Flash        $0.075      $0.30       varies
Gemini 1.5 Pro          $1.25       $5.00       varies
```

---

## Model Tier Strategy

### Routing Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  INCOMING REQUEST                                            │
└──────────────┬───────────────────────────────────────────────┘
               │
               v
┌──────────────────────────────────────────────────────────────┐
│  COMPLEXITY CLASSIFIER  (fast, cheap model or rule-based)   │
│                                                              │
│  Signals:                                                    │
│  • Query length and structure                                │
│  • Task type (extract vs. generate vs. reason)               │
│  • Required output length                                    │
│  • Domain complexity                                         │
└──────────┬───────────────────────────────┬───────────────────┘
           │                               │
     simple/cheap                    complex/expensive
           │                               │
           v                               v
┌──────────────────────┐     ┌─────────────────────────────┐
│  SMALL MODEL         │     │  LARGE MODEL                │
│  Haiku / GPT-4o mini │     │  Sonnet / GPT-4o            │
│  Classification      │     │  Multi-step reasoning       │
│  Extraction          │     │  Long-form generation       │
│  Simple Q&A          │     │  Complex code generation    │
│  Summarization       │     │  Nuanced analysis           │
└──────────────────────┘     └─────────────────────────────┘
```

### Task-to-Model Mapping

```
Task Type                       Recommended Tier    Notes
─────────────────────────────────────────────────────────────────
Sentiment classification        Small               High accuracy at all tiers
Entity extraction               Small               Rule-based may be even cheaper
Simple summarization            Small               Quality plateau at small tier
FAQ / intent detection          Small + cache       Cache system prompt + docs
Translation                     Small               GPT-4o mini excels here
Code completion (autocomplete)  Small               Speed matters more than power
RAG answer generation           Medium              Depends on context complexity
Code review / audit             Large               Requires deep understanding
Long-form content generation    Large               Quality delta is significant
Multi-step planning             Large               Reasoning depth required
Novel problem solving           Large               No precedent to lean on
```

### Routing Implementation

```python
import anthropic

client = anthropic.Anthropic()

COMPLEXITY_SIGNALS = {
    "simple": ["what is", "define", "list", "summarize", "translate"],
    "complex": ["analyze", "design", "compare", "evaluate", "debug", "explain why"]
}

def classify_request_complexity(user_message: str) -> str:
    """Rule-based fast classifier — replace with ML model at scale."""
    message_lower = user_message.lower()
    word_count = len(user_message.split())

    if word_count > 500:
        return "complex"

    for signal in COMPLEXITY_SIGNALS["complex"]:
        if signal in message_lower:
            return "complex"

    return "simple"

def route_request(user_message: str, system_prompt: str) -> str:
    complexity = classify_request_complexity(user_message)

    model = "claude-haiku-3-5" if complexity == "simple" else "claude-sonnet-4-5"
    max_tokens = 512 if complexity == "simple" else 4096

    response = client.messages.create(
        model=model,
        max_tokens=max_tokens,
        system=system_prompt,
        messages=[{"role": "user", "content": user_message}]
    )

    return response.content[0].text
```

**Important**: benchmark quality at each tier before committing to routing logic. Run 100+ real examples through both tiers and measure quality degradation. A 5% quality drop at 80% cost savings is often acceptable; a 20% drop rarely is.

---

## Prompt Caching

### Anthropic Prompt Caching

```
Cost Structure:
  Cache write (first call): 25% more than base input price
  Cache read (subsequent):  10% of base input price (90% savings)
  Cache TTL: 5 minutes (refreshed if accessed within TTL)

Requirement: Minimum ~1024 tokens to be eligible for caching
```

```python
import anthropic

client = anthropic.Anthropic()

LARGE_STATIC_DOCUMENT = "..." # 50,000 token document

def ask_about_document(question: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        system=[
            {
                "type": "text",
                "text": "You are a document analyst. Answer questions using only the provided document."
            },
            {
                "type": "text",
                "text": LARGE_STATIC_DOCUMENT,
                "cache_control": {"type": "ephemeral"}  # cache this block
            }
        ],
        messages=[
            {"role": "user", "content": question}
        ]
    )

    usage = response.usage
    print(f"Cache write: {usage.cache_creation_input_tokens}")
    print(f"Cache read:  {usage.cache_read_input_tokens}")
    print(f"New input:   {usage.input_tokens}")

    return response.content[0].text

# First call: pays for cache write (1.25x input price)
# Second call within 5 min: 90% cheaper for the cached block
```

**Cache placement rule**: static content must be a **prefix**. Put static content (instructions, tools, documents) first; dynamic content (user question, current data) last.

```
┌─────────────────────────────────────────────────────┐
│  [STATIC - cached]                                  │
│  System instructions          ← cache_control here  │
│  Tool definitions             ← or here             │
│  Reference documents          ← or here             │
├─────────────────────────────────────────────────────┤
│  [DYNAMIC - not cached]                             │
│  Conversation history                               │
│  Current user message                               │
└─────────────────────────────────────────────────────┘
```

### OpenAI Automatic Caching

```python
# OpenAI automatically caches prompts > 1024 tokens
# 50% discount on cached input tokens — no code changes required
# Cache lasts ~5-10 minutes of inactivity

# To maximize cache hits:
# 1. Keep the start of your prompt identical across requests
# 2. Put dynamic content at the END of the prompt
# 3. Use the same model and parameters

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        # This large system message will be auto-cached after the first call
        {"role": "system", "content": large_static_system_prompt},
        # Dynamic content goes here — after the static prefix
        {"role": "user", "content": user_question}
    ]
)

# Check cache usage
print(response.usage.prompt_tokens_details.cached_tokens)
```

### Gemini Context Caching

```python
import google.generativeai as genai

# Create a cached content object with explicit TTL
cached_content = genai.caching.CachedContent.create(
    model="models/gemini-1.5-pro-001",
    contents=[large_document],
    ttl=datetime.timedelta(minutes=60),  # explicit TTL
    display_name="product-manual-v2"
)

# Use the cached content in requests
model = genai.GenerativeModel.from_cached_content(cached_content)
response = model.generate_content("What is the warranty period?")

# Clean up when done
cached_content.delete()
```

---

## Output Length Control

Output tokens cost 3-5x more than input tokens. Control output length aggressively.

### API-Level Cap

```python
# Always set max_tokens — never leave it open
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=256,      # hard cap; model stops here even mid-sentence
    messages=[...]
)
```

### Prompt-Level Instruction

```python
# Vague — model may write 500 words
prompt = "Summarize this article."

# Explicit — model targets the constraint
prompt = "Summarize this article in exactly 3 bullet points, each under 15 words."

# For structured output — controls response size precisely
prompt = """Extract the key data points from this report.
Respond with ONLY a JSON object. No explanation, no preamble.
{"metric": "...", "value": ..., "unit": "..."}"""
```

### Output Length vs. Quality Trade-off

```
Task                        Needed output    Common mistake
──────────────────────────────────────────────────────────────
Sentiment classification    1 word/token     Allowing long explanations
Entity extraction           JSON array       Allowing prose with JSON
Yes/No question             "yes" or "no"    Model explaining reasoning
Summarization (brief)       50-100 words     No explicit word limit set
Code review (critical only) 3-5 issues       Requesting exhaustive review
```

---

## Batching

### Anthropic Batch API

```
Cost savings: 50% discount on all tokens
Turnaround: results available within 24 hours (typically faster)
Use for: nightly pipelines, document processing, eval runs, data labeling
```

```python
import anthropic

client = anthropic.Anthropic()

# Submit a batch
batch_requests = [
    {
        "custom_id": f"ticket-{i}",
        "params": {
            "model": "claude-haiku-3-5",
            "max_tokens": 256,
            "messages": [
                {"role": "user", "content": f"Classify this ticket: {ticket}"}
            ]
        }
    }
    for i, ticket in enumerate(tickets)
]

batch = client.messages.batches.create(requests=batch_requests)
batch_id = batch.id

# Poll for completion (or use webhook in production)
import time

while True:
    batch_status = client.messages.batches.retrieve(batch_id)
    if batch_status.processing_status == "ended":
        break
    time.sleep(60)

# Retrieve results
results = {}
for result in client.messages.batches.results(batch_id):
    if result.result.type == "succeeded":
        results[result.custom_id] = result.result.message.content[0].text
```

### OpenAI Batch API

```python
import json
from openai import OpenAI

client = OpenAI()

# Create JSONL file with requests
requests = [
    {
        "custom_id": f"req-{i}",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-4o-mini",
            "max_tokens": 256,
            "messages": [{"role": "user", "content": text}]
        }
    }
    for i, text in enumerate(texts)
]

with open("batch_requests.jsonl", "w") as f:
    for req in requests:
        f.write(json.dumps(req) + "\n")

# Upload and submit
batch_file = client.files.create(
    file=open("batch_requests.jsonl", "rb"),
    purpose="batch"
)

batch = client.batches.create(
    input_file_id=batch_file.id,
    endpoint="/v1/chat/completions",
    completion_window="24h"
)

# Poll and retrieve (same pattern as above)
```

---

## Semantic Caching

Cache responses by semantic similarity — return a cached answer when a new query is sufficiently similar to a past query.

```
┌─────────────────┐    embed     ┌─────────────────────┐
│  Incoming query │ ──────────> │  Query embedding     │
└─────────────────┘             └──────────┬──────────┘
                                           │
                                    similarity search
                                           │
                                           v
                               ┌─────────────────────────┐
                               │  Vector cache (Redis /  │
                               │  Pinecone / pgvector)   │
                               └──────────┬──────────────┘
                                          │
                              cosine similarity ≥ 0.95?
                                    /           \
                                  YES            NO
                                   │              │
                            return cached    call LLM API
                             response       store in cache
```

### Implementation with Redis + OpenAI Embeddings

```python
import numpy as np
import redis
import json
import openai

client = openai.OpenAI()
redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)

CACHE_TTL = 3600        # 1 hour
SIMILARITY_THRESHOLD = 0.95

def get_embedding(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",  # cheap: $0.02/M tokens
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

def semantic_cache_get(query: str) -> str | None:
    query_embedding = get_embedding(query)

    # Scan cached entries (use vector DB at scale)
    for key in redis_client.scan_iter("cache:*"):
        entry = json.loads(redis_client.get(key))
        similarity = cosine_similarity(query_embedding, entry["embedding"])
        if similarity >= SIMILARITY_THRESHOLD:
            return entry["response"]

    return None

def semantic_cache_set(query: str, response: str) -> None:
    embedding = get_embedding(query)
    cache_key = f"cache:{hash(query)}"
    redis_client.setex(
        cache_key,
        CACHE_TTL,
        json.dumps({"embedding": embedding, "response": response, "query": query})
    )

def answer_with_cache(query: str, system_prompt: str) -> str:
    # Check semantic cache first
    cached = semantic_cache_get(query)
    if cached:
        return cached

    # Cache miss — call the API
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": query}
        ]
    )
    answer = response.choices[0].message.content

    # Store in cache
    semantic_cache_set(query, answer)
    return answer
```

**Cache invalidation strategy:**
- TTL-based: automatic expiry (simplest)
- Event-based: invalidate on data change (e.g., product update → clear product FAQ cache)
- Explicit: `/admin/cache/clear?prefix=product_*`

---

## Token Budgeting

### Counting Tokens Before Sending

```python
# Anthropic
import anthropic

client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-sonnet-4-5",
    system=system_prompt,
    messages=messages
)
print(f"Estimated input tokens: {response.input_tokens}")

# OpenAI (tiktoken)
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
token_count = len(enc.encode(prompt_text))

# Google (genai)
model = genai.GenerativeModel("gemini-1.5-pro")
token_count = model.count_tokens(prompt_text).total_tokens
```

### Conversation History Budget

```python
MAX_HISTORY_TOKENS = 4000
TARGET_RESPONSE_TOKENS = 1000
MODEL_CONTEXT = 128000

def truncate_history(history: list[dict], max_tokens: int) -> list[dict]:
    """Keep most recent messages within token budget."""
    total = 0
    kept = []

    for message in reversed(history):
        tokens = count_tokens(message["content"])
        if total + tokens > max_tokens:
            break
        kept.insert(0, message)
        total += tokens

    if len(kept) < len(history):
        # Prepend a summary marker
        kept.insert(0, {
            "role": "user",
            "content": "[Earlier conversation has been truncated to fit context window]"
        })

    return kept
```

### Budget Alerts

```python
import dataclasses

@dataclasses.dataclass
class TokenBudget:
    per_request_limit: int = 10_000
    per_user_daily_limit: int = 500_000
    alert_threshold: float = 0.8  # alert at 80% of budget

class CostTracker:
    def __init__(self, budget: TokenBudget):
        self.budget = budget
        self.usage: dict[str, int] = {}  # user_id -> tokens_today

    def record_usage(self, user_id: str, input_tokens: int, output_tokens: int) -> None:
        total = input_tokens + output_tokens
        self.usage[user_id] = self.usage.get(user_id, 0) + total

        utilization = self.usage[user_id] / self.budget.per_user_daily_limit
        if utilization >= self.budget.alert_threshold:
            self._send_alert(user_id, utilization)

    def _send_alert(self, user_id: str, utilization: float) -> None:
        print(f"ALERT: User {user_id} at {utilization:.0%} of daily token budget")
```

---

## Cost Monitoring

Track cost at multiple dimensions to catch anomalies and attribute spend.

```python
# Middleware to track LLM costs
import time
from dataclasses import dataclass

# Approximate pricing ($/M tokens) — update from provider pricing pages
MODEL_PRICING = {
    "claude-haiku-3-5": {"input": 0.80, "output": 4.00, "cache_read": 0.08},
    "claude-sonnet-4-5": {"input": 3.00, "output": 15.00, "cache_read": 0.30},
    "gpt-4o-mini": {"input": 0.15, "output": 0.60, "cache_read": 0.075},
    "gpt-4o": {"input": 2.50, "output": 10.00, "cache_read": 1.25},
}

def calculate_cost(model: str, usage) -> float:
    pricing = MODEL_PRICING.get(model, MODEL_PRICING["claude-sonnet-4-5"])
    input_cost  = (usage.input_tokens / 1_000_000) * pricing["input"]
    output_cost = (usage.output_tokens / 1_000_000) * pricing["output"]
    cache_cost  = (getattr(usage, "cache_read_input_tokens", 0) / 1_000_000) * pricing["cache_read"]
    return input_cost + output_cost + cache_cost

def tracked_llm_call(client, model, messages, system, user_id, feature):
    start = time.time()
    response = client.messages.create(
        model=model,
        max_tokens=1024,
        system=system,
        messages=messages
    )
    latency_ms = (time.time() - start) * 1000
    cost = calculate_cost(model, response.usage)

    # Emit metrics to your observability platform
    metrics.record({
        "llm.cost_usd": cost,
        "llm.input_tokens": response.usage.input_tokens,
        "llm.output_tokens": response.usage.output_tokens,
        "llm.latency_ms": latency_ms,
        "llm.model": model,
        "user_id": user_id,
        "feature": feature,
    })

    return response
```

---

## Embedding Cost Optimization

```
Embedding Optimization Strategies:
┌─────────────────────────────────────────────────────────────┐
│ 1. CACHE static document embeddings                         │
│    Don't re-embed the same content on every query           │
│                                                             │
│ 2. CHOOSE model by use case                                 │
│    text-embedding-3-small: $0.02/M  → most use cases        │
│    text-embedding-3-large: $0.13/M  → highest accuracy      │
│                                                             │
│ 3. BATCH embed during ingestion                             │
│    Process documents in bulk; store in vector DB            │
│                                                             │
│ 4. MATRYOSHKA embeddings                                    │
│    Reduce dimensions post-embedding without re-embedding    │
│    3072-dim → 512-dim: 6x storage savings, ~1% quality loss │
└─────────────────────────────────────────────────────────────┘
```

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

# Matryoshka: generate large embedding, then truncate
response = client.embeddings.create(
    model="text-embedding-3-large",
    input=texts,
    dimensions=512  # truncate from 3072 → 512 natively
)

# Or truncate manually and re-normalize
def truncate_embedding(embedding: list[float], dim: int) -> list[float]:
    truncated = embedding[:dim]
    norm = np.linalg.norm(truncated)
    return (np.array(truncated) / norm).tolist()
```

---

## Anti-Patterns

| Anti-Pattern | Cost Impact | Fix |
|---|---|---|
| No `max_tokens` set | Runaway output; unpredictable cost | Always set `max_tokens` |
| Full conversation history every call | O(n²) token growth | Summarize or truncate history |
| Re-embedding static content on every query | 10-100x unnecessary embedding cost | Cache embeddings in vector DB |
| GPT-4 / Sonnet for simple classification | 15-50x overspend | Route to small model tier |
| No caching for repeated identical prompts | Paying 10x for same content | Implement semantic or exact cache |
| Verbose output instructions ("explain thoroughly") | 3-10x more output tokens | Be specific: "in 2 sentences" |
| Streaming for batch jobs | No cost benefit, added complexity | Use batch API for async work |
| No cost attribution | Can't identify high-spend users/features | Tag every call with user_id + feature |
| Single model for all tasks | Overspend or under-quality | Benchmark and route by task type |

---

## Cost Optimization Checklist

- [ ] `max_tokens` is set on every API call
- [ ] Model tier selected based on task complexity benchmark
- [ ] Routing logic tested on real workload samples
- [ ] Static content is cached (Anthropic cache_control or OpenAI auto-cache)
- [ ] Conversation history is truncated or summarized for long sessions
- [ ] Batch API used for async / nightly workloads
- [ ] Semantic cache deployed for repeated similar queries
- [ ] Embedding cache deployed; static docs not re-embedded per query
- [ ] Per-user and per-feature cost tracked in observability platform
- [ ] Budget alerts configured with auto-escalation
- [ ] Output token usage reviewed; verbose prompts trimmed
- [ ] Cost impact of every significant prompt change measured
