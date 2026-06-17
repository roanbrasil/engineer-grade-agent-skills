---
name: rag-production
description: Use when building, debugging, or improving a RAG (Retrieval-Augmented Generation) pipeline — covers document loading, chunking, embedding, retrieval, reranking, augmentation, and evaluation
---

# Production RAG Pipelines

## Architecture Overview

```
                         RAG Pipeline
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Documents                                                   │
│  ─────────                                                   │
│  PDF / HTML / DOCX                                           │
│       │                                                      │
│       ▼                                                      │
│  [Document Loader]  ──► metadata: source, page, heading     │
│       │                                                      │
│       ▼                                                      │
│  [Chunker]          ──► 256–512 tokens, 10–20% overlap      │
│       │                                                      │
│       ▼                                                      │
│  [Embedder]         ──► text → float[1536]                  │
│       │                                                      │
│       ▼                                                      │
│  [Vector Store]     ──► pgvector / Chroma / Qdrant          │
│                                                              │
│  ─── Query time ────────────────────────────────────────    │
│                                                              │
│  User Query                                                  │
│       │                                                      │
│       ▼                                                      │
│  [Query Transform]  ──► HyDE / multi-query / step-back      │
│       │                                                      │
│       ▼                                                      │
│  [Retriever]        ──► dense + sparse → hybrid RRF         │
│       │                                                      │
│       ▼                                                      │
│  [Reranker]         ──► top-20 → top-5                      │
│       │                                                      │
│       ▼                                                      │
│  [Augmentation]     ──► format context + citations          │
│       │                                                      │
│       ▼                                                      │
│  [LLM Generation]   ──► answer with sources                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Document Loading

### File Type Strategy

```
Format     Library              Notes
─────────────────────────────────────────────────────────────────
PDF        pdfplumber           Better than PyPDF2; preserves tables and layout
HTML       trafilatura          Removes nav/footer/ads; extracts main content
DOCX       python-docx          Preserves heading structure
Markdown   markdown-it-py       Respects code blocks and header hierarchy
CSV        pandas               Load as rows; include column headers in each chunk
```

```python
import pdfplumber
import trafilatura
from pathlib import Path

def load_pdf(path: str) -> list[dict]:
    """Extract text with metadata from PDF."""
    documents = []
    with pdfplumber.open(path) as pdf:
        for page_num, page in enumerate(pdf.pages, start=1):
            text = page.extract_text()
            if text and text.strip():
                documents.append({
                    "content": text,
                    "metadata": {
                        "source": path,
                        "page": page_num,
                        "total_pages": len(pdf.pages),
                        "file_type": "pdf"
                    }
                })
    return documents

def load_html(url: str) -> dict | None:
    """Extract main content from HTML page, stripping boilerplate."""
    downloaded = trafilatura.fetch_url(url)
    if not downloaded:
        return None
    text = trafilatura.extract(
        downloaded,
        include_tables=True,
        include_links=False,
        output_format="txt"
    )
    if not text:
        return None
    return {
        "content": text,
        "metadata": {
            "source": url,
            "file_type": "html"
        }
    }
```

### Incremental Loading

Never re-embed everything on update. Detect changes with content hashes:

```python
import hashlib
import json
from datetime import datetime

def compute_doc_hash(content: str) -> str:
    return hashlib.sha256(content.encode()).hexdigest()

class IncrementalLoader:
    def __init__(self, state_file: str):
        self.state_file = state_file
        self.state = self._load_state()

    def _load_state(self) -> dict:
        try:
            return json.loads(Path(self.state_file).read_text())
        except FileNotFoundError:
            return {}

    def _save_state(self):
        Path(self.state_file).write_text(json.dumps(self.state, indent=2))

    def needs_update(self, doc_id: str, content: str) -> bool:
        current_hash = compute_doc_hash(content)
        if doc_id not in self.state:
            return True
        return self.state[doc_id]["hash"] != current_hash

    def mark_processed(self, doc_id: str, content: str):
        self.state[doc_id] = {
            "hash": compute_doc_hash(content),
            "processed_at": datetime.utcnow().isoformat()
        }
        self._save_state()

# Usage
loader = IncrementalLoader("doc_state.json")
for doc in all_documents:
    if loader.needs_update(doc["id"], doc["content"]):
        chunks = chunker.split(doc)
        embedder.embed_and_upsert(chunks)
        loader.mark_processed(doc["id"], doc["content"])
```

---

## Chunking Strategies

### Comparison

```
Strategy              Speed   Quality   Structure-Aware   Use When
──────────────────────────────────────────────────────────────────────────
Fixed-size            Fast    Low       No                Homogeneous text, baseline
Recursive char split  Fast    Medium    Partial           General purpose default
Semantic chunking     Slow    High      Yes               Quality matters more than speed
Markdown-aware        Fast    High      Yes               Markdown/code documentation
Parent-child          Medium  High      Partial           Need precision + context
```

### Recursive Character Splitting (Default Choice)

Split on `\n\n`, then `\n`, then ` `, then character. Respects natural text boundaries.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,         # Target tokens (approximate)
    chunk_overlap=64,       # ~12.5% overlap
    length_function=len,    # Or use tiktoken for exact token counting
    separators=["\n\n", "\n", ". ", " ", ""]  # Priority order
)

chunks = splitter.create_documents(
    texts=[doc["content"] for doc in documents],
    metadatas=[doc["metadata"] for doc in documents]
)
```

### Semantic Chunking

Split when adjacent sentences become semantically dissimilar:

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

chunker = SemanticChunker(
    embeddings=OpenAIEmbeddings(model="text-embedding-3-small"),
    breakpoint_threshold_type="percentile",  # Split at 95th percentile of dissimilarity
    breakpoint_threshold_amount=95
)

# Expensive: embeds every sentence to find break points
chunks = chunker.split_text(document_text)
```

### Markdown-Aware Splitting

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

# First: split on headers to preserve structure
header_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ],
    strip_headers=False  # Keep headers in chunk content
)

header_chunks = header_splitter.split_text(markdown_text)

# Then: split large header sections by size
char_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64
)

final_chunks = char_splitter.split_documents(header_chunks)
# Each chunk now has metadata: {"h1": "Introduction", "h2": "Background"}
```

### Parent-Child Chunking

Index small child chunks for precise retrieval; return the larger parent for full context:

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_chroma import Chroma

# Child chunks: small, for precise vector matching
child_splitter = RecursiveCharacterTextSplitter(chunk_size=256, chunk_overlap=32)

# Parent chunks: large, for rich LLM context
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1024, chunk_overlap=128)

vectorstore = Chroma(embedding_function=embeddings)
docstore = InMemoryStore()  # In production, use Redis or a database

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter
)

retriever.add_documents(documents)

# Retrieval: searches child chunks, returns parent chunks
results = retriever.get_relevant_documents("What is HNSW?")
```

### Chunk Size Guidance

```
Task                    Chunk Size    Overlap     Rationale
──────────────────────────────────────────────────────────────────
Factual Q&A             256–384 tok   10%         Precise retrieval, small context
Reasoning / synthesis   512–1024 tok  15–20%      Need surrounding context
Code retrieval          512 tok       20%          Code blocks must be complete
Long-form summarization 1024–2048 tok 10%          Broad context needed
```

---

## Embedding

### Model Selection

```
Model                      Dims   Speed   Cost      Best For
──────────────────────────────────────────────────────────────────
text-embedding-3-small     1536   Fast    $0.02/1M  General purpose, production default
text-embedding-3-large     3072   Medium  $0.13/1M  Quality-sensitive use cases
nomic-embed-text           768    Fast    Free      Self-hosted, open-source
bge-m3                     1024   Medium  Free      Multilingual (100+ languages)
cohere-embed-v3            1024   Fast    $0.10/1M  Multilingual + reranking bundle
```

### Async Batching for Speed

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def embed_batch(texts: list[str], model: str = "text-embedding-3-small") -> list[list[float]]:
    response = await client.embeddings.create(input=texts, model=model)
    return [item.embedding for item in response.data]

async def embed_all(texts: list[str], batch_size: int = 500) -> list[list[float]]:
    """Embed a large list of texts with parallel batches."""
    batches = [texts[i:i + batch_size] for i in range(0, len(texts), batch_size)]

    # Process up to 10 batches concurrently
    semaphore = asyncio.Semaphore(10)

    async def embed_with_limit(batch):
        async with semaphore:
            return await embed_batch(batch)

    results = await asyncio.gather(*[embed_with_limit(b) for b in batches])
    return [embedding for batch_result in results for embedding in batch_result]

# Usage
embeddings = asyncio.run(embed_all(chunk_texts))
```

### Matryoshka Embeddings (Dimensionality Reduction)

`text-embedding-3-*` supports truncating to lower dimensions with minimal quality loss:

```python
from openai import OpenAI

client = OpenAI()

# Full 1536 dims: highest quality
# 512 dims: 2/3 the storage/compute, ~97% of quality
# 256 dims: 1/6 the storage, ~90% of quality

response = client.embeddings.create(
    input="What is HNSW?",
    model="text-embedding-3-large",
    dimensions=512  # Truncate to 512 dims
)
```

---

## Retrieval

### Dense vs Sparse vs Hybrid

```
Mode        How It Works                          Good At
──────────────────────────────────────────────────────────────────────────
Dense       Cosine similarity on embeddings       Semantic similarity, paraphrases
Sparse      BM25 keyword scoring                  Exact terms, rare words, codes
Hybrid      RRF(dense_rank, sparse_rank)          Best overall; production default
```

### Hybrid Retrieval with RRF

```python
def reciprocal_rank_fusion(
    dense_results: list[tuple[str, float]],  # (doc_id, score)
    sparse_results: list[tuple[str, float]],
    k: int = 60
) -> list[tuple[str, float]]:
    """
    Combine two ranked lists using Reciprocal Rank Fusion.
    k=60 is the standard constant from the original RRF paper.
    """
    scores: dict[str, float] = {}

    for rank, (doc_id, _) in enumerate(dense_results, start=1):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)

    for rank, (doc_id, _) in enumerate(sparse_results, start=1):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)

    return sorted(scores.items(), key=lambda x: x[1], reverse=True)


class HybridRetriever:
    def __init__(self, vector_store, bm25_index):
        self.vector_store = vector_store
        self.bm25_index = bm25_index

    def retrieve(self, query: str, k: int = 20) -> list[dict]:
        # Dense retrieval
        query_embedding = embed(query)
        dense = self.vector_store.similarity_search_with_score(query_embedding, k=k)
        dense_results = [(doc.id, score) for doc, score in dense]

        # Sparse retrieval (BM25)
        sparse = self.bm25_index.search(query, k=k)
        sparse_results = [(doc.id, score) for doc, score in sparse]

        # Fuse ranks
        fused = reciprocal_rank_fusion(dense_results, sparse_results)

        # Return top-k documents in fused order
        return [self.get_doc(doc_id) for doc_id, _ in fused[:k]]
```

### Reranking

Retrieve 20, rerank to 5. Cross-encoders read both query and document together — much more accurate than embedding cosine similarity, but too slow to run on the full corpus.

```python
import cohere

co = cohere.Client("your-api-key")

def retrieve_and_rerank(query: str, vector_store, top_k_retrieve: int = 20, top_k_final: int = 5) -> list[dict]:
    # Step 1: Fast initial retrieval (20 candidates)
    candidates = vector_store.similarity_search(query, k=top_k_retrieve)

    # Step 2: Expensive but accurate reranking (20 → 5)
    rerank_response = co.rerank(
        model="rerank-english-v3.0",
        query=query,
        documents=[c.page_content for c in candidates],
        top_n=top_k_final
    )

    return [
        {
            "content": candidates[r.index].page_content,
            "metadata": candidates[r.index].metadata,
            "relevance_score": r.relevance_score
        }
        for r in rerank_response.results
    ]
```

Open-source alternative (BGE Reranker):

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")  # Multilingual

def rerank(query: str, docs: list[str]) -> list[tuple[str, float]]:
    pairs = [(query, doc) for doc in docs]
    scores = reranker.predict(pairs)
    return sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)
```

---

## Query Transformation

### HyDE (Hypothetical Document Embedding)

Generate a hypothetical ideal answer, embed that, use it for retrieval. Works because the hypothetical answer is closer in embedding space to relevant documents than the short query is.

```python
def hyde_retrieve(query: str, llm, vector_store, k: int = 5) -> list[dict]:
    # Step 1: Generate hypothetical answer
    hypothetical_answer = llm.invoke(
        f"Write a detailed technical answer to this question. "
        f"Be specific and factual:\n\nQuestion: {query}"
    ).content

    # Step 2: Embed the hypothetical answer (not the query)
    hyde_embedding = embed(hypothetical_answer)

    # Step 3: Retrieve using the hypothetical answer embedding
    return vector_store.similarity_search_by_vector(hyde_embedding, k=k)
```

### Multi-Query Retrieval

Generate query variants, retrieve for each, deduplicate by doc ID:

```python
from langchain.retrievers.multi_query import MultiQueryRetriever

# LangChain built-in multi-query
retriever = MultiQueryRetriever.from_llm(
    retriever=vector_store.as_retriever(search_kwargs={"k": 10}),
    llm=llm
)
# Internally generates 3 query variants and merges results

# Manual implementation for full control
def multi_query_retrieve(original_query: str, llm, vector_store) -> list[dict]:
    variants_response = llm.invoke(
        f"Generate 3 different phrasings of this question for document retrieval. "
        f"Return only the questions, one per line.\n\nOriginal: {original_query}"
    )
    queries = [original_query] + variants_response.content.strip().split("\n")

    seen_ids = set()
    all_docs = []
    for q in queries:
        for doc in vector_store.similarity_search(q, k=5):
            if doc.metadata["id"] not in seen_ids:
                seen_ids.add(doc.metadata["id"])
                all_docs.append(doc)
    return all_docs
```

---

## Augmentation

### Context Window Management

```python
import tiktoken

def build_prompt(query: str, docs: list[dict], max_context_tokens: int = 6000) -> str:
    """Pack as many top-ranked docs as fit in the context budget."""
    enc = tiktoken.get_encoding("cl100k_base")

    context_parts = []
    token_count = 0

    for i, doc in enumerate(docs, start=1):
        citation = f"[{i}] Source: {doc['metadata'].get('source', 'unknown')}, Page: {doc['metadata'].get('page', 'N/A')}"
        chunk_text = f"{citation}\n{doc['content']}\n"
        chunk_tokens = len(enc.encode(chunk_text))

        if token_count + chunk_tokens > max_context_tokens:
            break  # Stop before exceeding budget

        context_parts.append(chunk_text)
        token_count += chunk_tokens

    context = "\n---\n".join(context_parts)

    return f"""Answer the question based on the following context.
Cite your sources using [1], [2], etc.
If the answer is not in the context, say "I don't know."

Context:
{context}

Question: {query}

Answer:"""
```

---

## Evaluation with RAGAS

```python
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    faithfulness,         # Answer grounded in context? (no hallucination)
    answer_relevancy,     # Answer addresses the question?
    context_recall,       # Were relevant chunks retrieved?
    context_precision,    # Are retrieved chunks relevant? (no noise)
)

# Prepare evaluation dataset
eval_data = {
    "question": ["What is HNSW?", "What is RAG?"],
    "answer": ["HNSW is...", "RAG is..."],                # LLM-generated answers
    "contexts": [["doc about HNSW..."], ["doc about RAG..."]],  # Retrieved chunks
    "ground_truth": ["HNSW stands for...", "RAG stands for..."]  # Reference answers
}

dataset = Dataset.from_dict(eval_data)

results = evaluate(
    dataset=dataset,
    metrics=[faithfulness, answer_relevancy, context_recall, context_precision]
)

print(results)
# {
#   "faithfulness": 0.92,       # > 0.8 is good; below means hallucinations
#   "answer_relevancy": 0.88,   # > 0.8 is good; below means off-topic answers
#   "context_recall": 0.75,     # > 0.7 is good; below means missing chunks
#   "context_precision": 0.83   # > 0.8 is good; below means noisy retrieval
# }
```

---

## Java Example (Spring AI)

```java
import org.springframework.ai.document.Document;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.chat.ChatClient;

@Service
public class RagService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    public RagService(VectorStore vectorStore, ChatClient chatClient) {
        this.vectorStore = vectorStore;
        this.chatClient = chatClient;
    }

    public void ingestDocuments(List<Document> rawDocuments) {
        // Chunk documents
        TokenTextSplitter splitter = new TokenTextSplitter(512, 64, 5, 10000, true);
        List<Document> chunks = splitter.apply(rawDocuments);

        // Embed and index (embedding happens inside vectorStore.add())
        vectorStore.add(chunks);
    }

    public String answer(String question) {
        // Retrieve top-5 relevant chunks
        List<Document> context = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(5).withSimilarityThreshold(0.7)
        );

        // Build augmented prompt
        String contextText = context.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n---\n"));

        String prompt = """
            Answer based on the context below.
            Cite sources. Say "I don't know" if not in context.

            Context:
            %s

            Question: %s
            """.formatted(contextText, question);

        return chatClient.call(prompt);
    }
}
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| No overlap between chunks | Sentences split across chunks; key phrases lost | Use 10–20% overlap |
| Embedding HTML with nav/headers/footers | Boilerplate pollutes the index | Use trafilatura to extract main content |
| Sending all 20 retrieved chunks to LLM | Context bloat; model ignores far chunks | Rerank to 5; respect token budget |
| No reranking step | Initial cosine retrieval is noisy | Add cross-encoder reranker |
| Re-embedding everything on any update | Hours of unnecessary compute and cost | Incremental loading with content hashes |
| Fixed chunk size for all content | One size never fits all document types | Choose strategy per content type |
| No retrieval quality metrics | You can't improve what you don't measure | Track context_recall and context_precision |
| Ignoring metadata | Can't filter by source, date, or section | Always extract and store metadata |
| k=3 retrieval | Too few candidates; misses relevant docs | Retrieve 20, rerank to 5 |

---

## Production Checklist

- [ ] PDF loading uses pdfplumber, not PyPDF2
- [ ] HTML extraction uses trafilatura (boilerplate stripped)
- [ ] Metadata (source, page, heading) preserved on every chunk
- [ ] Incremental loader: only re-embed changed documents
- [ ] Chunk size appropriate for task (256–512 for factual, 512–1024 for reasoning)
- [ ] Overlap set to 10–20% of chunk size
- [ ] Embedding in async batches of 100–500
- [ ] Hybrid retrieval (dense + sparse + RRF) in production
- [ ] Reranker reduces candidates from 20 → 5
- [ ] Context window budget enforced (don't overflow LLM max)
- [ ] Citations included in prompt; LLM asked to cite sources
- [ ] RAGAS evaluation: faithfulness > 0.8, context_recall > 0.7
- [ ] Evaluation runs on every pipeline change
