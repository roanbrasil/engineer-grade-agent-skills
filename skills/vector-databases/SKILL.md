---
name: vector-databases
description: Use when selecting, configuring, or querying a vector database — covers pgvector, Chroma, Weaviate, Pinecone, and Qdrant with indexing, filtering, and hybrid search
---

# Vector Databases

## What Vector Databases Do

A vector database stores high-dimensional float vectors alongside metadata, and supports fast Approximate Nearest Neighbor (ANN) search: given a query vector, find the k stored vectors most similar to it.

```
Traditional DB query:        SELECT * FROM docs WHERE category = 'finance'
Vector DB query:             Find 10 documents most semantically similar to this embedding
Hybrid query:                Find 10 documents most similar AND where category = 'finance'
```

The core challenge: exact nearest neighbor in high dimensions is O(n·d) per query. With 10M vectors of 1536 dimensions, that is 15 billion float comparisons per query. ANN indexes reduce this to milliseconds at 95–99% recall.

---

## ANN Algorithms

### HNSW (Hierarchical Navigable Small World)

The default for most production deployments. Builds a multi-layer graph where higher layers make long jumps and lower layers refine locally.

```
Layer 2:   *─────────────────────────────*
           (few nodes, long-range links)

Layer 1:   *───*───────────*────────*───*
           (medium density)

Layer 0:   *─*─*─*─*─*─*─*─*─*─*─*─*─*
           (all nodes, short links)

Search: Start at Layer 2, find rough neighborhood.
        Drop to Layer 1, refine. Drop to Layer 0, retrieve k-NN.
```

Parameters:
- `m`: number of bidirectional links per node. Higher = better recall, more memory. Default: 16.
- `ef_construction`: candidates considered during index build. Higher = better quality, slower build. Default: 64.
- `ef_search` (sometimes called `ef`): candidates at query time. Higher = better recall, slower query.

Tradeoffs:
```
m=8,  ef_construction=64   → Fast build, low memory, 92% recall
m=16, ef_construction=128  → Balanced default, 97% recall
m=32, ef_construction=200  → Best recall, 2× memory, slow build
```

### IVF (Inverted File Index)

Clusters vectors into `nlist` Voronoi cells. Query: find the nearest `nprobe` cells, then search within those cells only.

```
nlist=100 cells:   [cell 1] [cell 2] ... [cell 100]
query → nearest 10 cells → search those 10 cells only
```

Good for > 10M vectors where HNSW memory cost is prohibitive. IVF does not require loading the entire index into RAM.

### Flat Index

Exact brute-force search. 100% recall. Scales as O(n). Use only below 100K vectors or when recall is more important than latency.

### Algorithm Selection

```
Scale              Algorithm     Memory    Notes
────────────────────────────────────────────────────────────────────
< 100K vectors     Flat          Low       Exact search, fast enough
100K–10M           HNSW          High      Best recall/latency; needs RAM
> 10M              IVF-PQ        Low       Quantization reduces memory 10–50×
Very large + GPU   IVF-FLAT/GPU  VRAM      Use Faiss with GPU
```

---

## pgvector

PostgreSQL extension for vector similarity search. Best choice when you already run PostgreSQL and need transactions, joins, or complex SQL filters alongside vector search.

### Setup

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Add vector column to existing table
ALTER TABLE documents ADD COLUMN embedding vector(1536);

-- Or create table with vector column
CREATE TABLE documents (
    id          BIGSERIAL PRIMARY KEY,
    content     TEXT NOT NULL,
    embedding   vector(1536),
    source      TEXT,
    page_num    INT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### Inserting Vectors

```python
import psycopg2
import numpy as np
from openai import OpenAI

openai = OpenAI()
conn = psycopg2.connect("postgresql://user:pass@localhost/mydb")

def embed(text: str) -> list[float]:
    response = openai.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

def insert_document(content: str, source: str, page_num: int):
    embedding = embed(content)
    with conn.cursor() as cur:
        cur.execute(
            """
            INSERT INTO documents (content, embedding, source, page_num)
            VALUES (%s, %s::vector, %s, %s)
            """,
            (content, embedding, source, page_num)
        )
    conn.commit()
```

### Similarity Search

Three distance operators:
- `<=>` cosine distance (most common for text embeddings)
- `<->` L2 (Euclidean) distance
- `<#>` negative inner product (for normalized vectors, equivalent to cosine)

```sql
-- Cosine similarity search: find 10 most similar documents
SELECT
    id,
    content,
    source,
    page_num,
    1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- With metadata filter (combine vector search + SQL WHERE)
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE source = 'annual_report_2024.pdf'
  AND page_num BETWEEN 10 AND 50
ORDER BY embedding <=> $1::vector
LIMIT 5;
```

```python
def search(query: str, source_filter: str = None, k: int = 10) -> list[dict]:
    query_embedding = embed(query)

    sql = """
        SELECT id, content, source, page_num,
               1 - (embedding <=> %s::vector) AS similarity
        FROM documents
        {where}
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """.format(where=f"WHERE source = %s" if source_filter else "")

    params = [query_embedding, query_embedding, k]
    if source_filter:
        params = [query_embedding, source_filter, query_embedding, k]

    with conn.cursor() as cur:
        cur.execute(sql, params)
        rows = cur.fetchall()

    return [
        {"id": r[0], "content": r[1], "source": r[2], "page": r[3], "similarity": r[4]}
        for r in rows
    ]
```

### HNSW Index (Recommended)

```sql
-- Create HNSW index for cosine distance
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Index for L2 distance
CREATE INDEX ON documents
USING hnsw (embedding vector_l2_ops);

-- Tune search quality vs speed at query time
SET hnsw.ef_search = 100;  -- Default: 40. Higher = better recall, slower.
```

### IVFFlat Index (Memory-Efficient)

```sql
-- IVFFlat: cheaper memory, requires training (needs data first)
-- Rule: nlist = sqrt(row_count)
CREATE INDEX ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);  -- sqrt(10000 rows) ≈ 100

-- Tune probe count at query time
SET ivfflat.probes = 10;  -- Default: 1. Higher = better recall, slower.
```

### Hybrid Search (Vector + Full-Text)

```sql
-- Combine vector similarity with PostgreSQL full-text search
SELECT
    d.id,
    d.content,
    (1 - (d.embedding <=> $1::vector)) * 0.7 +
    ts_rank(to_tsvector('english', d.content), plainto_tsquery($2)) * 0.3 AS combined_score
FROM documents d
WHERE to_tsvector('english', d.content) @@ plainto_tsquery($2)
   OR (1 - (d.embedding <=> $1::vector)) > 0.7
ORDER BY combined_score DESC
LIMIT 10;
```

Java with Spring Data JPA:

```java
@Repository
public interface DocumentRepository extends JpaRepository<Document, Long> {

    @Query(value = """
        SELECT id, content, source, page_num,
               1 - (embedding <=> CAST(:embedding AS vector)) AS similarity
        FROM documents
        ORDER BY embedding <=> CAST(:embedding AS vector)
        LIMIT :k
        """, nativeQuery = true)
    List<DocumentSimilarityResult> findSimilar(
        @Param("embedding") String embedding,
        @Param("k") int k
    );
}
```

---

## Chroma

Embedded (in-process) or client-server vector database. Python-native. Best for development, prototyping, and applications below 1M vectors.

### Setup

```python
# Embedded mode (in-process, no server needed)
import chromadb
client = chromadb.Client()  # In-memory

# Persistent embedded mode
client = chromadb.PersistentClient(path="/path/to/chroma")

# Client-server mode (run `chroma run` separately)
client = chromadb.HttpClient(host="localhost", port=8000)
```

### Collections

```python
from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction

embedding_fn = OpenAIEmbeddingFunction(
    api_key="sk-...",
    model_name="text-embedding-3-small"
)

# Create or get collection
collection = client.get_or_create_collection(
    name="documents",
    embedding_function=embedding_fn,
    metadata={"hnsw:space": "cosine"}  # Distance metric
)
```

### Adding Documents

```python
# Add with automatic embedding (embedding_function set on collection)
collection.add(
    documents=["HNSW is a graph-based ANN algorithm", "RAG combines retrieval and generation"],
    ids=["doc-001", "doc-002"],
    metadatas=[
        {"source": "textbook.pdf", "page": 42, "section": "algorithms"},
        {"source": "blog.html", "page": 1, "section": "intro"}
    ]
)

# Or add pre-computed embeddings
collection.add(
    embeddings=[[0.1, 0.2, ...], [0.3, 0.4, ...]],
    documents=["raw text 1", "raw text 2"],
    ids=["doc-003", "doc-004"]
)
```

### Querying

```python
# Basic similarity search
results = collection.query(
    query_texts=["What is approximate nearest neighbor search?"],
    n_results=5
)

# With metadata filter
results = collection.query(
    query_texts=["What is HNSW?"],
    n_results=5,
    where={"source": {"$eq": "textbook.pdf"}},           # Exact match
    where={"page": {"$gte": 10, "$lte": 50}},            # Range filter
    where={"$and": [
        {"source": {"$eq": "textbook.pdf"}},
        {"page": {"$gte": 10}}
    ]},
    include=["documents", "metadatas", "distances", "embeddings"]
)

print(results["documents"])   # List of matched texts
print(results["distances"])   # Cosine distances (0 = identical, 2 = opposite)
print(results["metadatas"])   # Metadata dicts
```

---

## Weaviate

Graph-based vector database with built-in hybrid search, multi-tenancy, and vectorizer modules. Best for hybrid search and multi-tenant SaaS.

### Setup and Schema

```python
import weaviate
from weaviate.classes.config import Configure, Property, DataType

client = weaviate.connect_to_local()  # or weaviate.connect_to_wcs(...)

# Create collection (schema)
client.collections.create(
    name="Document",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(
        model="text-embedding-3-small"
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
        Property(name="source", data_type=DataType.TEXT),
        Property(name="page_num", data_type=DataType.INT),
    ]
)
```

### Adding Objects

```python
collection = client.collections.get("Document")

# Single insert (auto-vectorized by Weaviate using text2vec-openai)
collection.data.insert({
    "content": "HNSW is a graph-based ANN algorithm.",
    "source": "textbook.pdf",
    "page_num": 42
})

# Batch insert
with collection.batch.dynamic() as batch:
    for doc in documents:
        batch.add_object({
            "content": doc["content"],
            "source": doc["source"],
            "page_num": doc["page_num"]
        })
```

### Hybrid Search

```python
# Weaviate combines BM25 + vector automatically with alpha parameter
results = collection.query.hybrid(
    query="What is approximate nearest neighbor search?",
    alpha=0.75,   # 0.0 = pure BM25; 1.0 = pure vector; 0.75 = mostly vector
    limit=5,
    return_metadata=weaviate.classes.query.MetadataQuery(score=True, explain_score=True)
)

for obj in results.objects:
    print(obj.properties["content"])
    print(f"Score: {obj.metadata.score}")
```

### Multi-Tenancy

```python
from weaviate.classes.config import Configure

# Enable multi-tenancy on collection
client.collections.create(
    name="TenantDocument",
    multi_tenancy_config=Configure.multi_tenancy(enabled=True)
)

collection = client.collections.get("TenantDocument")

# Create tenants
collection.tenants.create([
    weaviate.classes.tenants.Tenant(name="tenant-acme"),
    weaviate.classes.tenants.Tenant(name="tenant-globex")
])

# Write to specific tenant
acme_collection = collection.with_tenant("tenant-acme")
acme_collection.data.insert({"content": "Acme document"})

# Query within tenant (data is isolated)
results = acme_collection.query.near_text(query="documents", limit=5)
```

---

## Pinecone

Fully managed serverless vector database. No infrastructure to operate. Best for large-scale deployments (100M+ vectors) and teams that want zero ops.

### Setup and Index Creation

```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key="your-api-key")

# Create serverless index
pc.create_index(
    name="production-docs",
    dimension=1536,              # Match your embedding model's output dims
    metric="cosine",             # cosine | euclidean | dotproduct
    spec=ServerlessSpec(
        cloud="aws",
        region="us-east-1"
    )
)

index = pc.Index("production-docs")
```

### Upsert Vectors

```python
import openai

def embed(texts: list[str]) -> list[list[float]]:
    response = openai.embeddings.create(input=texts, model="text-embedding-3-small")
    return [item.embedding for item in response.data]

# Prepare vectors in Pinecone format: [(id, vector, metadata), ...]
vectors = [
    {
        "id": f"doc-{i}",
        "values": embedding,
        "metadata": {
            "content": doc["content"],
            "source": doc["source"],
            "page_num": doc["page_num"]
        }
    }
    for i, (doc, embedding) in enumerate(zip(docs, embed([d["content"] for d in docs])))
]

# Batch upsert (max 100 per request)
batch_size = 100
for i in range(0, len(vectors), batch_size):
    index.upsert(vectors=vectors[i:i + batch_size])
```

### Querying with Metadata Filters

```python
query_embedding = embed(["What is HNSW?"])[0]

# Basic query
results = index.query(
    vector=query_embedding,
    top_k=5,
    include_metadata=True
)

# With metadata filter
results = index.query(
    vector=query_embedding,
    top_k=5,
    include_metadata=True,
    filter={
        "source": {"$eq": "textbook.pdf"},
        "page_num": {"$gte": 10, "$lte": 100}
    }
)

for match in results.matches:
    print(match.score, match.metadata["content"])
```

### Namespaces for Multi-Tenancy

```python
# Write to namespace (tenant isolation)
index.upsert(vectors=vectors, namespace="tenant-acme")
index.upsert(vectors=vectors, namespace="tenant-globex")

# Query within namespace only
results = index.query(
    vector=query_embedding,
    top_k=5,
    namespace="tenant-acme"  # Only searches this tenant's data
)
```

### Sparse-Dense Hybrid

```python
from pinecone_text.sparse import BM25Encoder

# Build BM25 sparse encoder on your corpus
bm25 = BM25Encoder()
bm25.fit([doc["content"] for doc in docs])

# Upsert with both dense and sparse vectors
sparse_vectors = bm25.encode_documents([doc["content"] for doc in docs])
dense_vectors = embed([doc["content"] for doc in docs])

hybrid_vectors = [
    {
        "id": f"doc-{i}",
        "values": dense,
        "sparse_values": sparse,
        "metadata": {"content": doc["content"]}
    }
    for i, (doc, dense, sparse) in enumerate(zip(docs, dense_vectors, sparse_vectors))
]

index.upsert(vectors=hybrid_vectors)

# Hybrid query
query_dense = embed(["What is HNSW?"])[0]
query_sparse = bm25.encode_queries(["What is HNSW?"])

results = index.query(
    vector=query_dense,
    sparse_vector=query_sparse,
    top_k=5,
    include_metadata=True
)
```

---

## Qdrant

High-performance, open-source vector database written in Rust. Supports complex filtering, quantization, and multiple named vectors per point.

### Setup

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# Local (in-memory)
client = QdrantClient(":memory:")

# Persistent local
client = QdrantClient(path="/path/to/qdrant")

# Server
client = QdrantClient(host="localhost", port=6333)
```

### Create Collection

```python
from qdrant_client.models import HnswConfigDiff, OptimizersConfigDiff

client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    hnsw_config=HnswConfigDiff(
        m=16,
        ef_construct=100,
        full_scan_threshold=10000  # Below this, use exact search
    ),
    optimizers_config=OptimizersConfigDiff(
        indexing_threshold=20000   # Build HNSW index after 20K vectors
    )
)
```

### Upsert Points

```python
from qdrant_client.models import PointStruct

points = [
    PointStruct(
        id=i,
        vector=embedding,
        payload={                  # Qdrant calls metadata "payload"
            "content": doc["content"],
            "source": doc["source"],
            "page_num": doc["page_num"],
            "category": doc["category"]
        }
    )
    for i, (doc, embedding) in enumerate(zip(docs, embeddings))
]

client.upsert(collection_name="documents", points=points)
```

### Querying with Rich Payload Filtering

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

query_embedding = embed("What is HNSW?")

# Basic search
results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    limit=5
)

# With payload (metadata) filter
results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(key="source", match=MatchValue(value="textbook.pdf")),
            FieldCondition(key="page_num", range=Range(gte=10, lte=100))
        ],
        should=[
            FieldCondition(key="category", match=MatchValue(value="algorithms"))
        ]
    ),
    limit=5,
    with_payload=True,
    score_threshold=0.7           # Minimum similarity score
)

for result in results:
    print(result.score, result.payload["content"])
```

### Named Vectors (Multi-Modal)

```python
# Collection with two vector spaces (e.g., title embedding + content embedding)
from qdrant_client.models import VectorsConfig

client.create_collection(
    collection_name="multimodal_docs",
    vectors_config={
        "title": VectorParams(size=1536, distance=Distance.COSINE),
        "content": VectorParams(size=1536, distance=Distance.COSINE),
    }
)

# Upsert with multiple vectors
client.upsert(
    collection_name="multimodal_docs",
    points=[
        PointStruct(
            id=1,
            vector={
                "title": embed(doc["title"]),
                "content": embed(doc["content"])
            },
            payload={"title": doc["title"], "content": doc["content"]}
        )
    ]
)

# Query against specific vector
results = client.search(
    collection_name="multimodal_docs",
    query_vector=("content", embed("HNSW algorithm")),
    limit=5
)
```

### Quantization

Reduce memory by 4–32× with minimal recall drop:

```python
from qdrant_client.models import ScalarQuantizationConfig, ScalarType

# Scalar quantization: 4× memory reduction (float32 → int8)
client.create_collection(
    collection_name="quantized_docs",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    quantization_config=ScalarQuantizationConfig(
        type=ScalarType.INT8,
        quantile=0.99,      # Clip outliers at 99th percentile
        always_ram=True     # Keep quantized index in RAM; fetch full vectors from disk
    )
)
```

---

## Selection Matrix

```
                  pgvector  Chroma   Weaviate  Pinecone  Qdrant
──────────────────────────────────────────────────────────────────────
Scale             < 10M     < 1M     < 50M     100M+     < 100M
Managed           No        No       Yes       Yes       Self-host
Open Source       Yes       Yes      Yes       No        Yes
Transactions      Yes       No       No        No        No
Hybrid Search     Manual    No       Built-in  Built-in  Manual
Multi-Tenancy     SQL       Manual   Built-in  Namespace Collection
Ops Complexity    Low*      Low      Medium    None      Low
Cost              Infra     Free     Infra     Per req   Infra
Best For          Existing  Dev/     Multi-    Managed   On-prem
                  PG stack  proto    tenant    large     perf
```
*pgvector: low complexity if you already run PostgreSQL.

### Decision Tree

```
Do you already run PostgreSQL?
├── Yes → pgvector (unless scale > 10M vectors)
└── No
    ├── Prototyping or < 1M vectors? → Chroma (embedded)
    ├── Need managed service, no ops? → Pinecone
    ├── Need multi-tenancy + hybrid search? → Weaviate
    ├── On-premises, performance-critical? → Qdrant
    └── Multilingual + multi-modal? → Qdrant (named vectors)
```

---

## HNSW Tuning Reference

```
Parameter         Default  Range     Effect
─────────────────────────────────────────────────────────────────────
m                 16       4–64      Links per node; higher = better recall + more memory
ef_construction   64       32–512    Build quality; higher = better graph, slower build
ef_search         40       10–500    Query quality; tune this first in production
```

Tuning process:
1. Build with default `m=16, ef_construction=128`
2. Measure recall@10 on a benchmark set
3. If recall < 95%, increase `ef_search` first (free at query time)
4. If recall still low, rebuild with higher `ef_construction`
5. If memory is a problem, decrease `m` or enable quantization

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| No index (flat search) above 100K vectors | Query becomes O(n); unusable at scale | Create HNSW or IVFFlat index |
| Wrong distance metric | Cosine-trained embeddings with L2 index gives wrong ranking | Match metric to embedding training (usually cosine) |
| Filtering after retrieval in application code | Returns wrong k; must fetch n > k then filter | Use built-in payload/metadata filters |
| No score threshold | Returns irrelevant results when nothing matches | Set minimum similarity threshold (0.6–0.8 typical) |
| Indexing before loading data (IVF) | IVF index trains on empty or small data; bad clusters | Load > 10K vectors before creating IVFFlat index |
| Not batching upserts | 1 vector per request = 1000× slower | Batch 100–500 vectors per upsert call |
| Storing full text in vector DB | Expensive; vector DBs are not optimized for large text | Store content in PostgreSQL/S3; store ID + metadata in vector DB |
| Ignoring ef_search tuning | Default ef_search=40 often gives 90% recall | Benchmark and tune ef_search per use case |

---

## Production Checklist

- [ ] Distance metric matches embedding model (cosine for text-embedding-*)
- [ ] HNSW or IVFFlat index created (never rely on flat search at scale)
- [ ] `ef_search` tuned for target recall (benchmark with ground truth)
- [ ] Metadata filters pushed into DB (not post-filtered in code)
- [ ] Score threshold set to filter low-relevance results
- [ ] Upserts batched (100–500 per request)
- [ ] Incremental upsert on document update (not full re-index)
- [ ] Recall benchmarked periodically (index can drift)
- [ ] Multi-tenancy implemented (namespace / tenant / collection)
- [ ] Quantization evaluated if memory is a constraint
- [ ] Backup strategy in place (for self-hosted deployments)
