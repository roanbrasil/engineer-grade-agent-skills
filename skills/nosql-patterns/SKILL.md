---
name: nosql-patterns
description: Invoke when selecting a NoSQL database, designing schemas for MongoDB/DynamoDB/Cassandra, modeling access patterns, or evaluating trade-offs between document, key-value, wide-column, graph, and time-series stores.
---

# NoSQL Database Patterns and Selection

## NoSQL vs. Relational: When to Choose

```
Choose PostgreSQL (relational) when:
  - ACID transactions across multiple entities are required
  - Complex ad-hoc queries with JOINs
  - Schema is stable and well-defined
  - Regulatory compliance requires strong consistency

Choose NoSQL when:
  - Schema is flexible or evolves rapidly (document stores)
  - Extreme write throughput (wide-column)
  - Simple access patterns at massive scale (key-value)
  - Deep relationship traversal (graph)
  - Time-indexed sensor/metric data (time-series)
```

---

## NoSQL Categories

```
DOCUMENT          KEY-VALUE         WIDE-COLUMN       GRAPH           TIME-SERIES
---------         ---------         -----------       -----           -----------
MongoDB           Redis             Cassandra         Neo4j           InfluxDB
DynamoDB          DynamoDB          HBase             Amazon Neptune  TimescaleDB
Couchbase         Riak              ScyllaDB          ArangoDB        Prometheus
Firestore         Memcached         Bigtable          TigerGraph      QuestDB

Flexible schema   Simple get/set    Sparse rows       Nodes + edges   Time-indexed
Nested docs       Extremely fast    Write-optimized   Traversal       Downsampling
Range queries     No query flex     Partition+cluster Relationship    Retention
```

---

## MongoDB

### Document Model: Embed vs. Reference

The central design decision in MongoDB. Wrong choice degrades performance by 10x or more.

```
EMBED WHEN:                               REFERENCE WHEN:
- Data is always accessed together        - Data is shared across many documents
- 1-to-few relationship (< ~20 items)     - 1-to-many (unbounded arrays)
- Data is owned by parent document        - Many-to-many relationships
- Sub-documents don't change often        - Sub-document updates are frequent
- Document stays < 4MB total              - Document would exceed 16MB limit
```

#### Embedding Example (Order with line items)

```javascript
// GOOD: Order and its lines always read together — embed
{
  "_id": ObjectId("..."),
  "orderId": "ORD-001",
  "customerId": "USR-123",
  "createdAt": ISODate("2025-01-15T10:00:00Z"),
  "status": "confirmed",
  "lines": [              // Bounded array (~1-100 items) — safe to embed
    { "productId": "PROD-A", "qty": 2, "price": 29.99, "name": "Widget" },
    { "productId": "PROD-B", "qty": 1, "price": 49.99, "name": "Gadget" }
  ],
  "total": 109.97,
  "shippingAddress": {    // Snapshot: preserve address at time of order
    "street": "123 Main St",
    "city": "Springfield",
    "zip": "12345"
  }
}
```

#### Reference Example (Product with reviews)

```javascript
// BAD: unbounded array — reviews could grow to thousands
// WRONG:
{
  "_id": ObjectId("prod-001"),
  "name": "Widget",
  "reviews": [
    { "userId": "u1", "rating": 5, "text": "Great!" },
    // ... potentially thousands of reviews -> document grows unbounded
  ]
}

// RIGHT: Reference from review to product
// product document stays small:
{ "_id": ObjectId("prod-001"), "name": "Widget", "avgRating": 4.3, "reviewCount": 1247 }

// reviews in separate collection:
{
  "_id": ObjectId("..."),
  "productId": ObjectId("prod-001"),   // Reference
  "userId": ObjectId("u-123"),
  "rating": 5,
  "text": "Great product!",
  "createdAt": ISODate("2025-01-15")
}
// Query: db.reviews.find({ productId: ObjectId("prod-001") }).sort({ createdAt: -1 })
```

### Aggregation Pipeline

MongoDB's server-side data processing. Always use pipeline over client-side processing.

```javascript
// Revenue by category for last 30 days
db.orders.aggregate([
  // Stage 1: Filter — reduces documents early (use indexed fields here)
  { $match: {
    status: "completed",
    createdAt: { $gte: new Date(Date.now() - 30 * 24 * 3600 * 1000) }
  }},
  // Stage 2: Unwind array into separate documents
  { $unwind: "$lines" },
  // Stage 3: Lookup (JOIN) to products collection
  { $lookup: {
    from: "products",
    localField: "lines.productId",
    foreignField: "_id",
    as: "product"
  }},
  { $unwind: "$product" },
  // Stage 4: Group and aggregate
  { $group: {
    _id: "$product.category",
    totalRevenue: { $sum: { $multiply: ["$lines.qty", "$lines.price"] }},
    orderCount: { $sum: 1 },
    avgOrderValue: { $avg: "$total" }
  }},
  // Stage 5: Sort
  { $sort: { totalRevenue: -1 }},
  // Stage 6: Project (shape output)
  { $project: {
    category: "$_id",
    totalRevenue: { $round: ["$totalRevenue", 2] },
    orderCount: 1,
    avgOrderValue: { $round: ["$avgOrderValue", 2] },
    _id: 0
  }}
])
```

### Indexes

```javascript
// Single field index
db.orders.createIndex({ customerId: 1 })

// Compound index — order matters: equality fields first, range last
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
// Supports: find by customerId; find by customerId + status; sort by createdAt

// Partial index — only index documents matching filter (smaller, faster)
db.orders.createIndex(
  { createdAt: 1 },
  { partialFilterExpression: { status: "pending" } }
)

// Text index for full-text search
db.products.createIndex({ name: "text", description: "text" })
db.products.find({ $text: { $search: "wireless bluetooth headphones" } })

// Multikey index (array field)
db.products.createIndex({ tags: 1 })
db.products.find({ tags: "electronics" })  // Hits multikey index

// Verify index usage
db.orders.find({ customerId: "USR-123" }).explain("executionStats")
// Check: IXSCAN (good) vs COLLSCAN (bad — needs index)
```

### Change Streams (CDC)

```javascript
// React to document changes in real time (like Kafka CDC for MongoDB)
const changeStream = db.collection('orders').watch([
  { $match: { 'fullDocument.status': 'payment_confirmed' } }
], { fullDocument: 'updateLookup' });

changeStream.on('change', async (change) => {
  if (change.operationType === 'update') {
    const order = change.fullDocument;
    await fulfillmentService.enqueueOrder(order._id);
    await notificationService.sendConfirmation(order.customerId);
  }
});

// Use cases: event-driven triggers, cache invalidation, audit logs, search index sync
```

### Schema Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Unbounded arrays | Document grows > 16MB; slow reads | Reference in separate collection |
| Massive number of collections | Collection-level locks; metadata overhead | Group similar entities |
| No indexes on query fields | COLLSCAN on every query | Add indexes, verify with `explain()` |
| Using `_id` as string UUID | Slower than ObjectId; no time component | Use ObjectId unless you need a specific format |
| Storing booleans as strings | `"true"` != `true`; query bugs | Use native BSON types |
| No schema validation | Silent data corruption | Add `$jsonSchema` validator |

---

## DynamoDB

### Core Concepts

```
TABLE: single-table design — ALL entities in one table
  PK (Partition Key): distributes data across physical shards
  SK (Sort Key): enables range queries within a partition

  PK                 SK                    Attributes
  --------           --------              ----------
  USER#123           PROFILE               {name, email, created}
  USER#123           ORDER#2025-01-15#001  {total, status, items}
  USER#123           ORDER#2025-01-15#002  {total, status, items}
  PRODUCT#ABC        DETAILS               {name, price, category}
  PRODUCT#ABC        REVIEW#USER#123       {rating, text, date}
```

### Single-Table Design: Access Patterns Drive Everything

Design the table around queries — not around entities. Identify all access patterns first.

```
Access Patterns → Table Design
1. Get user profile by userId          → PK=USER#123, SK=PROFILE
2. Get all orders for user             → PK=USER#123, SK begins_with ORDER#
3. Get order by orderId                → GSI: PK=ORDER#001, SK=STATUS#confirmed
4. Get all orders by status            → GSI: PK=STATUS#pending, SK=CREATED_AT#...
5. Get product details                 → PK=PRODUCT#ABC, SK=DETAILS
6. Get all reviews for product         → PK=PRODUCT#ABC, SK begins_with REVIEW#
```

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('app-table')

# Put item (idempotent write with condition)
table.put_item(
    Item={
        'PK': 'USER#123',
        'SK': 'PROFILE',
        'name': 'Jane Doe',
        'email': 'jane@example.com',
        'createdAt': '2025-01-15T10:00:00Z',
        'GSI1PK': 'EMAIL#jane@example.com',   # GSI for email lookup
        'GSI1SK': 'USER#123'
    },
    ConditionExpression=Attr('PK').not_exists()  # Fail if already exists
)

# Get all orders for user (sorted by date — SK is ORDER#DATE#ID)
response = table.query(
    KeyConditionExpression=Key('PK').eq('USER#123') & Key('SK').begins_with('ORDER#'),
    ScanIndexForward=False,   # Descending order (newest first)
    Limit=20
)
orders = response['Items']

# Update with optimistic locking
table.update_item(
    Key={'PK': 'ORDER#001', 'SK': 'DETAILS'},
    UpdateExpression='SET #status = :newStatus, updatedAt = :now',
    ConditionExpression='#status = :expectedStatus AND version = :version',
    ExpressionAttributeNames={'#status': 'status'},
    ExpressionAttributeValues={
        ':newStatus': 'shipped',
        ':expectedStatus': 'confirmed',
        ':version': 3,
        ':now': '2025-01-15T14:00:00Z'
    }
)
```

### GSI (Global Secondary Index) vs LSI (Local Secondary Index)

```
GSI (Global Secondary Index)        LSI (Local Secondary Index)
----------------------------        --------------------------
Alternate PK + optional SK          Same PK, alternate SK
Eventual consistency                Strong consistency possible
Can be added after table creation   Must define at table creation
Costs extra RCU/WCU                 Shares table RCU/WCU
Projects subset of attributes       Projects subset of attributes
Use for: different entity lookups   Use for: alternate sort order

Example GSI: email → user lookup
  GSI PK: EMAIL#jane@example.com
  GSI SK: USER#123
  → Query: who has this email?
```

### DynamoDB Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `Scan` in production | Full table scan = expensive + slow | Always query by PK; use GSI |
| Hot partition key | All traffic to one shard; throttling | Add random suffix: `USER#123#1` |
| Too many GSIs (> 5) | Write amplification; cost | Redesign access patterns; composite SK |
| Large items (> 400KB) | DynamoDB limit; poor performance | Store payload in S3, reference in DDB |
| Multi-table design (one table per entity) | Cross-entity transactions expensive | Single-table design |
| Storing booleans as numbers | Confusion; inconsistency | Use native DynamoDB Boolean type |

---

## Cassandra

### Data Modeling Rules

**Rule 1: Queries drive table design.** No JOINs. No ad-hoc queries. Design one table per query.

**Rule 2: Every query MUST filter by the full partition key.** If you can't filter by partition key, you need a different table.

**Rule 3: Denormalization is required and expected.** Duplicate data across tables to enable different query patterns.

### Partition Key + Clustering Key

```sql
-- Time-series sensor readings
CREATE TABLE sensor_readings_by_sensor (
  sensor_id   UUID,
  day         DATE,          -- Wide partition: split by day to limit partition size
  recorded_at TIMESTAMP,
  value       DOUBLE,
  unit        TEXT,
  PRIMARY KEY ((sensor_id, day), recorded_at)   -- Composite partition key + clustering
) WITH CLUSTERING ORDER BY (recorded_at DESC)   -- Newest first in each partition
  AND default_time_to_live = 2592000;           -- Auto-expire after 30 days (TTL)

-- Query: MUST include full partition key
SELECT * FROM sensor_readings_by_sensor
WHERE sensor_id = ? AND day = ?
ORDER BY recorded_at DESC LIMIT 100;

-- WRONG: missing partition key → full cluster scan (disaster)
SELECT * FROM sensor_readings_by_sensor WHERE sensor_id = ?;  -- Missing day!
```

```sql
-- E-commerce: orders per user (denormalized from order table)
CREATE TABLE orders_by_user (
  user_id    UUID,
  created_at TIMESTAMP,
  order_id   UUID,
  status     TEXT,
  total      DECIMAL,
  PRIMARY KEY (user_id, created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC, order_id ASC);

-- Same order data in separate table for different access pattern
CREATE TABLE orders_by_status (
  status     TEXT,
  created_at TIMESTAMP,
  order_id   UUID,
  user_id    UUID,
  total      DECIMAL,
  PRIMARY KEY (status, created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Write Patterns

```python
from cassandra.cluster import Cluster
from cassandra.query import BatchStatement, ConsistencyLevel

cluster = Cluster(['cassandra-node1', 'cassandra-node2'])
session = cluster.connect('ecommerce')

# Prepared statements: compile once, execute many
insert_stmt = session.prepare("""
  INSERT INTO orders_by_user (user_id, created_at, order_id, status, total)
  VALUES (?, ?, ?, ?, ?)
  USING TTL 7776000
""")  # Auto-expire after 90 days

# Batch: atomic write to multiple denormalized tables
batch = BatchStatement(consistency_level=ConsistencyLevel.LOCAL_QUORUM)
batch.add(insert_stmt, (user_id, created_at, order_id, 'pending', total))
batch.add(insert_by_status_stmt, ('pending', created_at, order_id, user_id, total))
session.execute(batch)  # Both tables updated atomically
```

### Tombstones and TTL

```sql
-- WRONG: frequent DELETEs create tombstones that degrade reads
DELETE FROM sensor_readings WHERE sensor_id = ? AND day = ?;

-- RIGHT: use TTL — Cassandra handles expiry without tombstone accumulation
INSERT INTO sensor_readings (sensor_id, day, recorded_at, value)
VALUES (?, ?, ?, ?)
USING TTL 2592000;  -- 30 days in seconds

-- Check tombstone ratio (alert if > 20%)
-- In nodetool: nodetool tablestats ecommerce.orders_by_user | grep "Tombstone"
```

### Cassandra Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `SELECT *` without partition key | Full cluster scan; kills performance | Always filter by partition key |
| Large partitions (> 100MB) | OOM; slow reads; compaction issues | Add time bucket to PK (day/month) |
| Using `ALLOW FILTERING` | Full table scan disguised | Design the right table for the query |
| Frequent deletes without TTL | Tombstone accumulation; read slowdown | Use TTL for time-bounded data |
| Secondary indexes on high-cardinality columns | Scatter-gather read across all nodes | Materialize a new table |
| Batch across different partition keys | Large batch; coordinator bottleneck | Use async writes; UNLOGGED BATCH only for same partition |

---

## Redis Patterns

```python
import redis
import json
from datetime import timedelta

r = redis.Redis(host='redis', port=6379, decode_responses=True)

# Cache-aside pattern
def get_product(product_id: str) -> dict:
    cache_key = f"product:{product_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    product = db.find_product(product_id)  # Slow DB call
    r.setex(cache_key, timedelta(minutes=15), json.dumps(product))
    return product

# Distributed lock (prevent duplicate processing)
def process_order(order_id: str):
    lock_key = f"lock:order:{order_id}"
    with r.lock(lock_key, timeout=30, blocking_timeout=5) as lock:
        # Safe: only one instance processes this order
        process(order_id)

# Sorted set: real-time leaderboard
r.zadd('leaderboard:weekly', {'user:123': 9850, 'user:456': 9200})
top10 = r.zrevrange('leaderboard:weekly', 0, 9, withscores=True)

# Pub/Sub: simple event bus (use Kafka for durable messaging)
# Publisher
r.publish('order_events', json.dumps({'type': 'order_created', 'orderId': 'abc'}))

# Rate limiting with sliding window
def is_rate_limited(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f"rate:{user_id}"
    now = int(time.time())
    pipe = r.pipeline()
    pipe.zadd(key, {str(now): now})
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zcard(key)
    pipe.expire(key, window)
    _, _, count, _ = pipe.execute()
    return count > limit
```

---

## Selection Guide

```
DECISION TREE
=============

Need ACID transactions across multiple entities?
  YES → PostgreSQL (not NoSQL)
  NO  ↓

What is your PRIMARY access pattern?

Simple get/set by ID; extreme speed; caching; pub/sub; leaderboards?
  → Redis

Variable-schema documents; nested data; complex queries; moderate scale?
  → MongoDB

Extreme write throughput (> 100K writes/sec); time-series; IoT; append-only logs?
  → Cassandra / ScyllaDB (Cassandra-compatible, 10x faster)

Serverless; AWS-native; complex access patterns; need managed service?
  → DynamoDB (single-table design)

Entities connected by relationships; traversal queries (friends-of-friends, paths)?
  → Neo4j / Amazon Neptune

Time-indexed sensor, metric, or financial tick data; retention + downsampling?
  → TimescaleDB (PostgreSQL extension) or InfluxDB

Full-text search as primary access pattern?
  → Elasticsearch / OpenSearch (not a primary store — use alongside relational)
```

```
SCALE BENCHMARKS (rough guidance)
==================================
Redis:           < 1ms latency; millions of ops/sec; limited by RAM
MongoDB:         10M+ documents; flexible; seconds-to-minutes analytics
DynamoDB:        unlimited scale; < 10ms at any scale; pay-per-request
Cassandra:       petabyte scale; millions writes/sec; linear scaling with nodes
Neo4j:           billions of nodes/edges; complex traversal in milliseconds
TimescaleDB:     millions of data points/sec; petabyte time-series
```

---

## Multi-Model Anti-Patterns

| Mistake | Symptom | Fix |
|---|---|---|
| Using MongoDB as a relational DB | JOINs everywhere; `$lookup` in every query | Denormalize; or use PostgreSQL |
| Using Redis as primary store | Data loss on restart; no persistence | Use as cache/queue alongside a durable store |
| DynamoDB multi-table design | Transactions across tables; expensive | Single-table design |
| Cassandra for low-cardinality partition keys | Hot partitions; throttling | Add time bucket or random suffix to PK |
| Using Cassandra for < 1M rows | Cassandra overhead not worth it | PostgreSQL or DynamoDB at small scale |
| Neo4j for everything | Most data isn't graph-shaped; poor performance | Use graph only for relationship-heavy use cases |
| No TTL on ephemeral data | Data grows unbounded; storage cost | Set TTL from day one on time-bounded data |

---

## Consistency Models Reference

```
STRONG CONSISTENCY      EVENTUAL CONSISTENCY      CAUSAL CONSISTENCY
------------------      --------------------      ------------------
Read your writes        Reads may be stale        Operations causally
guaranteed              (ms to seconds)           related are ordered

DynamoDB (default)      DynamoDB (GSI reads)      MongoDB (sessions)
Cassandra (QUORUM)      Cassandra (ONE)           Cassandra (LOCAL_SERIAL)
MongoDB (primary)       MongoDB (secondary reads)

Use when:               Use when:                 Use when:
- Financial data        - Read-heavy, tolerate    - Social feeds
- Inventory counts      - Stale data OK           - Shopping cart
- User profile writes   - Max performance         - Collaborative editing
```
