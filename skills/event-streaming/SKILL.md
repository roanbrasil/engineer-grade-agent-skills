---
name: event-streaming
description: Event streaming architecture expertise — event-driven patterns, stream processing, windowing, stateful processing, exactly-once semantics, schema evolution, stream-table duality, and technology selection (Kafka Streams vs Flink vs Spark). Trigger on EDA design questions, stream processing patterns, watermarks, backpressure, multi-DC replication, or event catalog governance.
---

# Event Streaming Architecture

You are a senior distributed systems architect specializing in event streaming. Apply production-grade reasoning: always ask about event time vs processing time, push back on unbounded state, and flag schema evolution risks early.

---

## Event Streaming vs Traditional Messaging

```
TRADITIONAL MESSAGE QUEUE
┌──────────┐   push/pull   ┌──────────────┐   consume   ┌──────────┐
│ Producer │──────────────►│    Queue     │────────────►│ Consumer │
└──────────┘               └──────────────┘             └──────────┘
                                  │
                          message deleted
                          after consumption
                          (no replay, no independent consumers)

EVENT STREAMING LOG
┌──────────┐   append   ┌─────────────────────────────────────┐
│ Producer │───────────►│  Partition  [E1][E2][E3][E4][E5]... │
└──────────┘            └─────────────────────────────────────┘
                               ▲          ▲          ▲
                        Consumer A   Consumer B   Consumer C
                        (offset=3)   (offset=5)   (offset=1)
                        
Key differences:
  - Retention:    Messages kept for hours/days/forever (log compaction)
  - Replay:       Any consumer can re-read from any offset
  - Independence: Multiple consumer groups, independent offsets
  - Ordering:     Guaranteed within partition
```

---

## Event-Driven Architecture Styles

### 1. Event Notification

```
Service A ──► "OrderPlaced" ──► Service B
                                (queries Service A for details)

Characteristics:
+ Decoupled trigger
- Tight coupling via query-back (chatty)
- Service A must be available when B processes notification
Use when: lightweight notifications, polling reduction
```

### 2. Event-Carried State Transfer (ECST)

```
Service A ──► "OrderPlaced { orderId, userId, items, total, address }" ──► Service B
                (full state embedded in event)

Characteristics:
+ Full autonomy — B never calls back to A
+ A can be offline while B processes
- Larger payloads
- Schema coupling — B depends on A's internal model
Use when: downstream services need data autonomy, high availability requirements
```

### 3. Event Sourcing

```
WRITE PATH                          READ PATH
┌──────────┐                       ┌──────────────────┐
│ Command  │                       │  Projection /    │
│ Handler  │──► Event Store ──────►│  Read Model      │
│          │   [OrderPlaced]       │  (Materialized   │
│          │   [ItemAdded]         │   View)          │
│          │   [OrderShipped]      │                  │
└──────────┘   [OrderCancelled]    └──────────────────┘
                    │
                    └── Source of truth is the EVENT LOG
                        Current state = replay of all events

Benefits: full audit trail, temporal queries, event replay for new projections
Pitfalls: eventual consistency, projection rebuild time, schema evolution complexity
```

---

## Stream Processing Patterns

```
RAW STREAM
───[E1]──[E2]──[E3]──[E4]──[E5]──►

FILTER: keep events matching predicate
───[E1]──────[E3]──────[E5]──►  (E2, E4 dropped)

MAP/TRANSFORM: apply function to each event
───[E1']─[E2']─[E3']─[E4']──►  (each event transformed)

FLATMAP: one event → zero or more events
───[E1a][E1b]──[E3a][E3b][E3c]──►  (explode arrays, etc.)

ENRICH: join with external data
───[E1+lookup]──[E2+lookup]──►

AGGREGATE: reduce many events to one value
───count=5──average=42.3──sum=1050──►

BRANCH: route to multiple outputs
──┬──[E1]──[E3]──►  (stream A: orders > $100)
  └──[E2]──[E4]──►  (stream B: orders ≤ $100)
```

### Joining Streams

```
STREAM-STREAM JOIN (windowed — both sides are unbounded)
Orders:    ──[O1]────────[O2]────►
Payments:  ──────[P1]────────[P2]►
                 JOIN on orderId within 5-min window
Result:    ──────[O1+P1]──[O2+P2]►
⚠ Requires buffering both sides; memory grows with window size

STREAM-TABLE JOIN (non-windowed — table side is compacted state)
Orders:    ──[O1]────[O2]────►
Customers: compacted KTable (latest value per customerId)
Result:    ──[O1+C]──[O2+C]──►  (enrichment pattern)

TABLE-TABLE JOIN (both sides are KTables)
Result: another KTable (latest joined state)
```

---

## Time in Streams

```
THREE TIME DOMAINS:

Event Time     = when the event actually happened (from event payload)
                 Most correct; suffers from out-of-order delivery
                 
Ingestion Time = when the broker received the event
                 Between event time and processing time
                 
Processing Time = when the stream processor sees the event
                  Easiest but wrong for historical reprocessing

EXAMPLE — out-of-order events:
Real world:        E1(t=10) E2(t=12) E3(t=11) E4(t=15)
Arrives at broker: ──[E1]──[E2]──[E3]──[E4]──►
                          E3 arrived late (t=11 but arrived after t=12)
                          
Without watermarks: window [10,12] closes before E3 arrives → E3 dropped
With watermarks:    allow lateness up to 2 seconds → E3 included
```

### Watermarks

```
Watermark(t) = "I am confident no event with event_time < t will arrive"

LOW_WATERMARK (bounded out-of-orderness):
  watermark = max_event_time_seen - max_lateness_allowed
  
  Events:    ──[t=10]──[t=14]──[t=11]──[t=15]──►
  Max seen:      10       14      14       15
  Watermark(2s): 8        12      12       13
  
  Window [0,10] triggers when watermark > 10 → at t=14 arrival

IDLE PARTITION PROBLEM:
  If one partition receives no events, its watermark never advances.
  → Use per-partition idle timeout to advance watermarks for silent partitions.
```

---

## Stateful Stream Processing

```
STATELESS: each event processed independently
  map, filter, flatMap — no memory required

STATEFUL: current event depends on past events
  count, sum, join, sessionize — requires state store

LOCAL STATE ARCHITECTURE:
┌─────────────────────────────────────────────────┐
│  Stream Task (owns partitions P0, P1)            │
│                                                  │
│  ┌───────────────┐    ┌────────────────────────┐│
│  │ Processing    │◄──►│ Local State Store      ││
│  │ Logic         │    │ (RocksDB on disk)      ││
│  │               │    │ Key: userId            ││
│  │               │    │ Val: {count, sum, ...} ││
│  └───────────────┘    └────────────────────────┘│
│                                ▲                 │
│                    changelog   │ mirror every    │
│                    topic       │ write           │
└────────────────────────────────┼─────────────────┘
                                 ▼
                    ┌────────────────────────┐
                    │ Changelog Topic (Kafka)│
                    │ Durable state backup   │
                    │ Used for restoration   │
                    └────────────────────────┘
```

### State Store Access Patterns

```java
// Kafka Streams — queryable state store
ReadOnlyKeyValueStore<String, Long> store = streams
    .store(StoreQueryParameters.fromNameAndType(
        "word-count-store",
        QueryableStoreTypes.keyValueStore()));

Long count = store.get("kafka"); // local query only; use Interactive Queries for distributed
```

---

## Exactly-Once Processing End-to-End

```
EXACTLY-ONCE SCOPE:

Level 1 — Within Kafka (broker):
  Idempotent producer deduplicates retries
  Transactional producer: atomic write across partitions

Level 2 — Read-Process-Write (Kafka→Process→Kafka):
  Kafka Streams EOS: atomic {consume offset + produce output}
  enable.processing.guarantee=exactly_once_v2

Level 3 — External systems (Kafka→Database):
  NO built-in EOS — requires:
    - Idempotent sink (upsert by primary key, not insert)
    - Or: outbox + dedup table
    - Or: two-phase commit (rare, complex)

PRACTICAL RULE: If the sink is Kafka, use transactions.
If the sink is a database, design for at-least-once + idempotent writes.
```

---

## Schema Evolution — The Hardest Problem

```
WHY IT'S HARD:
  Producers and consumers deploy independently.
  Old consumers must read new events.
  New consumers must read old events in replays.

SCHEMA REGISTRY COMPATIBILITY MODES:

BACKWARD  new schema → reads old data
  ✓ Add field with default
  ✗ Remove field
  ✗ Change field type

FORWARD   old schema → reads new data
  ✓ Add field with default
  ✗ Add required field without default
  ✗ Remove field consumers depend on

FULL      both directions simultaneously (most restrictive)
  ✓ Only: add optional field with default
  ✗ Everything else

BREAKING CHANGES THAT SEEM SAFE BUT AREN'T:
  - Rename field (breaks all consumers reading by name)
  - Reuse a field name with a different type
  - Change enum values (adding is OK; removing breaks serialization)
  - Change numeric precision (int → long is backward; long → int is not)

MIGRATION STRATEGIES:
  1. Expand-contract: add new field alongside old → migrate consumers → remove old field
  2. Topic versioning: orders-v1, orders-v2 with a migration consumer
  3. Avro aliases: rename with backward compatibility via alias declaration
```

---

## Stream-Table Duality

```
Every stream can be viewed as a table:
  Stream: [set(user=1,val=A), set(user=2,val=B), set(user=1,val=C)]
  Table:  {user=1: C, user=2: B}  ← compact to latest per key

Every table can be viewed as a stream:
  Table changes over time → changelog stream of mutations
  INSERT/UPDATE/DELETE → events

This duality is fundamental:
  KTable in Kafka Streams IS backed by a changelog topic
  Database CDC IS a stream of table mutations

STREAM-TABLE DUALITY ENABLES:
  - Materializing external database state into Kafka (CDC)
  - Rebuilding tables from streams after failures
  - Joining real-time streams with slow-moving reference data
```

---

## Technology Comparison: Kafka Streams vs Flink vs Spark Streaming

```
┌─────────────────┬────────────────┬───────────────────┬──────────────────┐
│                 │ Kafka Streams  │  Apache Flink     │  Spark Streaming │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ Deployment      │ Library (no    │ Separate cluster  │ Spark cluster    │
│                 │ cluster needed)│ required          │ required         │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ Latency         │ Milliseconds   │ Milliseconds      │ Seconds (micro-  │
│                 │                │                   │ batch)           │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ Exactly-once    │ Yes (Kafka→    │ Yes (end-to-end   │ Yes (idempotent  │
│                 │ Kafka only)    │ with checkpoints) │ writes required) │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ State           │ RocksDB (local)│ RocksDB + remote  │ In-memory or     │
│ management      │ + changelog    │ state backends    │ external store   │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ Event time /    │ Good           │ Excellent         │ Good (DStream    │
│ Watermarks      │ (basic)        │ (most advanced)   │ limited)         │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ Complex event   │ No             │ Yes (CEP library) │ Limited          │
│ processing      │                │                   │                  │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ SQL support     │ ksqlDB         │ Flink SQL         │ Spark SQL        │
├─────────────────┼────────────────┼───────────────────┼──────────────────┤
│ Best for        │ Kafka-centric  │ Complex stateful  │ Batch + stream   │
│                 │ microservices  │ streaming at scale│ unified pipeline │
└─────────────────┴────────────────┴───────────────────┴──────────────────┘

DECISION GUIDE:
  Already on Kafka, simple topology, want embedded → Kafka Streams
  Complex event patterns, large state, advanced watermarks → Flink
  Batch + streaming same code, Spark ecosystem → Spark Structured Streaming
  SQL-first, Kafka → ksqlDB
```

---

## Backpressure

```
UNBOUNDED QUEUE FAILURE PATTERN:
  Producer ──► [Queue: 1M items] ──► Consumer (slow)
                     ▲
              Memory exhausts → OOM crash → data loss or reprocessing

BACKPRESSURE MECHANISM:
  Producer ──► [Queue: 1000 items MAX] ──► Consumer
                    │                          │
                    └──── "full" signal ────────┘
                         Producer slows down

REACTIVE STREAMS BACKPRESSURE:
  Consumer signals demand: "I can process 100 items"
  Producer sends exactly 100
  Consumer signals next demand
  → demand-driven flow control

KAFKA BACKPRESSURE:
  Kafka does NOT apply backpressure to producers (log is unbounded by design)
  Consumer backpressure: pause() partition consumption
  Processing backpressure: bound thread pool queue, reject + retry

FLINK BACKPRESSURE:
  Network buffers fill → upstream operator slows
  Automatic, end-to-end
  Monitor via Flink Web UI backpressure indicator
```

---

## Dead Letter Topics / Poison Pills

```
POISON PILL: an event that cannot be processed successfully
  - Malformed schema (schema mismatch)
  - Null required fields
  - Business rule violation (orderId not found)
  - Transient error that became permanent after retries

RETRY TOPOLOGY:
Main Topic ──► [Consumer] ──failure──► Retry Topic (delay=5s)
                                          │
                                    [Retry Consumer] ──failure──► Retry Topic (delay=30s)
                                                                      │
                                                              [DLT Consumer] ──► Dead Letter Topic
                                                              
DLT (Dead Letter Topic) naming convention:
  orders           → main
  orders.retry-1   → first retry (5s delay)
  orders.retry-2   → second retry (30s delay)  
  orders.DLT       → permanent failure; requires manual intervention
```

```java
// Java — Spring Kafka DLT with exponential backoff
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
        (rec, ex) -> new TopicPartition(rec.topic() + ".DLT", rec.partition()));
    
    var backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000L);   // 1s
    backoff.setMultiplier(2.0);          // 2s, 4s
    backoff.setMaxInterval(10000L);      // cap at 10s
    
    return new DefaultErrorHandler(recoverer, backoff);
}
```

```python
# Python — DLT pattern
def process_with_dlt(consumer, producer, dlt_topic):
    msg = consumer.poll(1.0)
    try:
        result = process(msg.value())
        consumer.commit(msg)
    except NonRetryableError as e:
        producer.produce(
            dlt_topic,
            key=msg.key(),
            value=msg.value(),
            headers={
                'original-topic': msg.topic(),
                'original-partition': str(msg.partition()),
                'original-offset': str(msg.offset()),
                'error-message': str(e),
                'failed-at': datetime.utcnow().isoformat(),
            }
        )
        consumer.commit(msg)
```

---

## Multi-DC Replication

```
ACTIVE-PASSIVE (DR):
DC1 ──────────────────────────────────────────────────────► DC2
     MirrorMaker 2 replicates all topics                    (standby)
     DC2 topic naming: dc1.orders, dc1.customers
     Offset translation maintained by MM2
     RTO: minutes (DNS failover + consumer restart)

ACTIVE-ACTIVE (multi-region serving):
DC1 ◄─────────────────────────────────────────────────► DC2
    MirrorMaker 2 in both directions
    Topic naming prevents loops: dc1.orders vs dc2.orders
    Consumers read from local cluster only
    Application-level conflict resolution required
    
AGGREGATION HUB:
DC1 ──►                  
DC2 ──►  Central Kafka  ──► Analytics, ML pipelines
DC3 ──►
    Each DC's topics aggregated for global view

MIRRORMAKER 2 CONFIG:
clusters = dc1, dc2
dc1.bootstrap.servers = dc1-broker:9092
dc2.bootstrap.servers = dc2-broker:9092
dc1->dc2.enabled = true
dc1->dc2.topics = orders, customers, payments
dc1->dc2.groups = .*
sync.topic.configs.enabled = true
sync.topic.acls.enabled = true
```

---

## Event Catalog and Governance

```
EVENT CATALOG COMPONENTS:
┌─────────────────────────────────────────────────────────┐
│  Event Catalog                                          │
│                                                         │
│  events/                                                │
│  ├── orders/                                            │
│  │   ├── OrderPlaced.avsc        (Avro schema)          │
│  │   ├── OrderShipped.avsc                              │
│  │   └── README.md               (ownership, SLA, examples)│
│  ├── payments/                                          │
│  │   └── PaymentProcessed.avsc                         │
│  └── catalog.yaml               (machine-readable index)│
└─────────────────────────────────────────────────────────┘

GOVERNANCE RULES:
  1. Schema Registry as source of truth — no schema in app code
  2. Compatibility mode = FULL (strictest) by default
  3. Breaking change process: RFC → approval → versioned migration
  4. Event ownership: one team owns each topic namespace
  5. Sensitive data: PII fields tagged; masked in non-prod environments
  6. Retention SLA: documented per topic (7 days / 30 days / forever)
  7. Consumer SLA: producers document expected consumer groups and contracts

EVENT METADATA STANDARD (headers):
  event-id:        UUID (idempotency key)
  event-type:      com.example.orders.OrderPlaced
  event-version:   1.0
  source:          order-service
  correlation-id:  trace ID from originating HTTP request
  causation-id:    event-id that caused this event
  timestamp:       ISO-8601 UTC
```

---

## Production Checklist

### Stream Processing Service
- [ ] Use event time, not processing time for business logic
- [ ] Define watermark strategy (bounded out-of-orderness or periodic)
- [ ] Handle late data explicitly (emit update or drop with metric)
- [ ] State store size bounded (use TTL / windowed state)
- [ ] Changelog topic retention >= processing SLA for restoration
- [ ] Dead letter topic per input topic
- [ ] Idempotent output (upsert, not insert) for external sinks
- [ ] Schema compatibility mode enforced in CI pipeline
- [ ] Consumer lag alert configured (threshold: 1 minute of lag)
- [ ] Graceful shutdown with `streams.close(Duration.ofSeconds(30))`

### Schema Governance
- [ ] Schema Registry deployed with HA (3+ instances)
- [ ] Compatibility mode set before first schema registration
- [ ] Schema diff in PR review process
- [ ] Consumer count tracked per schema version (deprecation possible?)
- [ ] PII fields annotated and encrypted at rest
