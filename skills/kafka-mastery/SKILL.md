---
name: kafka-mastery
description: Deep Apache Kafka expertise — architecture, producers, consumers, Streams, Schema Registry, exactly-once semantics, and production tuning. Trigger on Kafka config questions, consumer lag, rebalancing, EOS, schema evolution, or Kafka Streams topology design.
---

# Kafka Mastery

You are an expert Apache Kafka engineer. When answering Kafka questions, apply production-grade reasoning: never recommend `enable.auto.commit=true` in production, always reason about ordering guarantees vs throughput tradeoffs, and explain the "why" behind every config recommendation.

---

## Core Architecture

```
                        KAFKA CLUSTER
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │   Broker 1          Broker 2          Broker 3      │
  │  ┌──────────┐      ┌──────────┐      ┌──────────┐  │
  │  │Topic-A P0│◄─────│Topic-A P0│      │          │  │
  │  │ (Leader) │      │(Follower)│      │          │  │
  │  ├──────────┤      ├──────────┤      ├──────────┤  │
  │  │          │      │Topic-A P1│◄─────│Topic-A P1│  │
  │  │          │      │ (Leader) │      │(Follower)│  │
  │  ├──────────┤      ├──────────┤      ├──────────┤  │
  │  │Topic-A P2│      │          │      │Topic-A P2│◄─┤
  │  │(Follower)│      │          │      │ (Leader) │  │
  │  └──────────┘      └──────────┘      └──────────┘  │
  │                                                      │
  └──────────────────────────────────────────────────────┘
       ▲                                       ▼
  ┌──────────┐                          ┌──────────────┐
  │ Producer │                          │Consumer Group│
  │          │                          │  ┌────────┐  │
  │ key→hash │                          │  │Csmr C1 │  │
  │  → P0,P1 │                          │  │ P0     │  │
  │  or P2   │                          │  ├────────┤  │
  └──────────┘                          │  │Csmr C2 │  │
                                        │  │ P1, P2 │  │
                                        │  └────────┘  │
                                        └──────────────┘
```

### Key Primitives

| Concept | Definition |
|---|---|
| **Topic** | Logical category; append-only log split into partitions |
| **Partition** | Ordered, immutable sequence of records; unit of parallelism |
| **Offset** | Monotonically increasing record position within a partition |
| **Consumer Group** | Set of consumers that collectively consume a topic; each partition assigned to exactly one member |
| **Broker** | Kafka server process; one broker is the leader for each partition |
| **Replication Factor** | Number of partition copies across brokers (minimum 3 in production) |
| **ISR** | In-Sync Replicas — followers caught up within `replica.lag.time.max.ms` |

---

## Partitioning Strategy

### Key-Based Partitioning
```
partition = hash(key) % num_partitions
```

Use key-based partitioning when:
- Order guarantees are required per entity (e.g., all events for `userId=123` go to the same partition)
- Stateful stream processing needs co-located data

**Pitfall — Hot Partitions**: If key cardinality is low (e.g., `country_code`), a few partitions receive all traffic. Solution: compound key (`country_code + user_id`) or custom partitioner.

### Partition Count Tradeoffs

```
More partitions  →  higher throughput (more parallelism)
                 →  more open file handles on brokers
                 →  longer leader election on failure
                 →  higher end-to-end latency (replication fan-out)

Rule of thumb: start with max(target_throughput / throughput_per_partition, num_consumers)
Never reduce partition count after creation.
```

---

## Producer Configuration

### Durability Profiles

```properties
# --- MAXIMUM THROUGHPUT (fire-and-forget, data loss acceptable) ---
acks=0
retries=0
compression.type=lz4
batch.size=65536          # 64 KB
linger.ms=5
buffer.memory=67108864

# --- BALANCED (most production workloads) ---
acks=1
retries=3
retry.backoff.ms=100
compression.type=lz4
batch.size=32768
linger.ms=20
enable.idempotence=false

# --- MAXIMUM DURABILITY (financial, audit) ---
acks=all
enable.idempotence=true    # forces retries=MAX_INT, max.in.flight.requests.per.connection=5
min.insync.replicas=2      # broker-side; set on topic or broker
compression.type=zstd
batch.size=16384
linger.ms=0
```

### Compression Comparison

| Codec | CPU Cost | Ratio | Best For |
|---|---|---|---|
| `none` | Zero | 1.0x | Already-compressed payloads |
| `lz4` | Very low | ~2.0x | High-throughput, latency-sensitive |
| `snappy` | Low | ~2.2x | General purpose |
| `gzip` | Medium | ~3.0x | Bandwidth-constrained, large messages |
| `zstd` | Medium | ~3.5x | Best ratio/CPU tradeoff; recommended for new deployments |

### Idempotent Producer Mechanics

```
Without idempotence:
  Producer → broker (seq=1) → ACK lost → retry → DUPLICATE

With idempotence (enable.idempotence=true):
  Producer assigns ProducerID + SequenceNumber per partition
  Broker deduplicates based on (PID, partition, seq)
  Exactly-once delivery to broker (single session)
```

---

## Consumer Configuration

### Critical Settings

```properties
# NEVER use in production — commits on a timer, data loss on crash
enable.auto.commit=false

# What to do when no offset exists or offset is out of range
auto.offset.reset=earliest    # replay from beginning (safer)
# auto.offset.reset=latest    # only new messages (risk: gaps on new groups)

# Max records per poll() call — tune to keep processing < max.poll.interval.ms
max.poll.records=500

# How long broker waits before declaring consumer dead and triggering rebalance
session.timeout.ms=45000

# Must be < session.timeout.ms / 3
heartbeat.interval.ms=15000

# Max time between poll() calls — increase for slow batch processing
max.poll.interval.ms=300000

# Fetch tuning
fetch.min.bytes=1
fetch.max.wait.ms=500
max.partition.fetch.bytes=1048576
```

### Manual Offset Commit Pattern

```java
// Java — Spring Kafka with manual ack
@KafkaListener(topics = "orders", containerFactory = "kafkaListenerContainerFactory")
public void consume(ConsumerRecord<String, Order> record, Acknowledgment ack) {
    try {
        processOrder(record.value());
        ack.acknowledge();  // commits offset after successful processing
    } catch (RetryableException e) {
        // do NOT ack — message will be redelivered after session timeout
        throw e;
    } catch (NonRetryableException e) {
        // send to DLT, then ack to skip
        deadLetterTemplate.send("orders.DLT", record);
        ack.acknowledge();
    }
}
```

```python
# Python — confluent-kafka manual commit
from confluent_kafka import Consumer, KafkaError

consumer = Consumer({
    'bootstrap.servers': 'broker1:9092,broker2:9092',
    'group.id': 'order-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
    'session.timeout.ms': 45000,
    'heartbeat.interval.ms': 15000,
})

consumer.subscribe(['orders'])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None:
        continue
    if msg.error():
        handle_error(msg.error())
        continue

    try:
        process(msg.value())
        consumer.commit(msg)  # synchronous commit; use asynchronous=True for throughput
    except Exception as e:
        send_to_dlt(msg)
        consumer.commit(msg)
```

```kotlin
// Kotlin — Spring Kafka
@KafkaListener(topics = ["orders"], groupId = "order-processor")
fun consume(record: ConsumerRecord<String, Order>, ack: Acknowledgment) {
    runCatching { processOrder(record.value()) }
        .onSuccess { ack.acknowledge() }
        .onFailure { ex ->
            when (ex) {
                is NonRetryableException -> {
                    deadLetterTemplate.send("orders.DLT", record)
                    ack.acknowledge()
                }
                else -> throw ex
            }
        }
}
```

---

## Consumer Group Rebalancing

```
EAGER REBALANCE (RangeAssignor / RoundRobinAssignor)
┌──────────────┐     ┌──────────────────────────────────┐
│ C1: P0, P1   │     │ STOP-THE-WORLD                   │
│ C2: P2, P3   │────►│ All consumers revoke ALL partitions│
│              │     │ Coordinator reassigns from scratch │
│              │     │ → high latency spike               │
└──────────────┘     └──────────────────────────────────┘

COOPERATIVE-STICKY REBALANCE (CooperativeStickyAssignor)
┌──────────────┐     ┌──────────────────────────────────┐
│ C1: P0, P1   │     │ INCREMENTAL                      │
│ C2: P2, P3   │────►│ Only moved partitions are revoked│
│ + C3 joins   │     │ C1 keeps P0, gives up P1         │
│              │     │ C3 gets P1 only                  │
└──────────────┘     └──────────────────────────────────┘
```

**Production recommendation**: Always use `CooperativeStickyAssignor`.

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

**Rebalance loop root causes**:
1. Processing takes longer than `max.poll.interval.ms` → increase interval or reduce `max.poll.records`
2. GC pause > `session.timeout.ms` → tune JVM GC, increase timeout
3. Network partition between consumer and broker → check connectivity

---

## Exactly-Once Semantics (EOS)

```
THREE DELIVERY GUARANTEES:

At-most-once:   Producer acks=0. Messages may be lost. Never use for important data.
At-least-once:  Producer retries + consumer processes before committing offset.
                → possible duplicates; idempotent consumers required.
Exactly-once:   Idempotent producer + Transactions + atomic offset commit.
```

### Transactional Producer

```java
// Java — Transactional producer
Properties props = new Properties();
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-processor-tx-1");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.ACKS_CONFIG, "all");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("output-topic", key, value));
    // Atomically commit offset + output in one transaction
    producer.sendOffsetsToTransaction(offsetsMap, consumerGroupMetadata);
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException e) {
    // Fatal — this producer instance is invalid
    producer.close();
} catch (KafkaException e) {
    producer.abortTransaction();
}
```

```python
# Python — confluent-kafka transactions
producer = Producer({
    'bootstrap.servers': 'broker1:9092',
    'transactional.id': 'order-processor-tx-1',
    'enable.idempotence': True,
})

producer.init_transactions()
producer.begin_transaction()
try:
    producer.produce('output-topic', key=key, value=value)
    producer.send_offsets_to_transaction(
        offsets,           # {TopicPartition: OffsetAndMetadata}
        consumer.consumer_group_metadata()
    )
    producer.commit_transaction()
except Exception as e:
    producer.abort_transaction()
    raise
```

---

## Kafka Streams

### Topology Primitives

```
KStream  = unbounded, append-only event stream (every message is an event)
KTable   = changelog stream; latest value per key (stateful; backed by RocksDB + changelog topic)
GlobalKTable = replicated to ALL stream tasks (for broadcast lookups; small datasets only)
```

### Windowing Types

```
TUMBLING WINDOW (fixed, non-overlapping):
time: 0────10────20────30
       [  W1  ] [  W2  ] [  W3  ]
Each event belongs to exactly one window.
Use: per-minute counters

HOPPING WINDOW (fixed size, overlapping):
size=10, advance=5
time: 0────10────20
       [  W1  ]
            [  W2  ]
                 [  W3  ]
An event can appear in multiple windows.
Use: rolling averages

SESSION WINDOW (activity-based, variable size):
Grouped by inactivity gap (e.g., gap=5min)
Events:  E1──E2──────gap>5min──E3──E4
Windows: [  Session 1  ]       [Ses2]
Use: user sessions, clickstream
```

### Stateful Processing Example

```java
// Java — Kafka Streams word count with windowing
StreamsBuilder builder = new StreamsBuilder();

builder.stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()))
    .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
    .groupBy((key, word) -> word)
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count(Materialized.<String, Long, WindowStore<Bytes, byte[]>>as("word-counts")
        .withValueSerde(Serdes.Long()))
    .toStream()
    .map((windowedKey, count) ->
        KeyValue.pair(windowedKey.key(), count.toString()))
    .to("output-topic", Produced.with(Serdes.String(), Serdes.String()));

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
```

```kotlin
// Kotlin — Stream-Table join
val builder = StreamsBuilder()

val ordersStream: KStream<String, Order> = builder.stream("orders")
val customersTable: KTable<String, Customer> = builder.table("customers")

ordersStream
    .join(customersTable,
        { order, customer -> EnrichedOrder(order, customer) },
        Joined.with(Serdes.String(), orderSerde, customerSerde)
    )
    .to("enriched-orders")
```

### Changelog Topics and Fault Tolerance

```
State Store (RocksDB, local)
        │
        │ every write mirrored to
        ▼
Changelog Topic (Kafka)
        │
        │ on task restart/rebalance
        ▼
State restored from changelog (replay)
```

---

## Schema Registry + Avro/Protobuf

### Schema Evolution Rules

```
BACKWARD COMPATIBLE  (new schema reads old data)
  - Add optional field with default
  - Remove field with no default (readers ignore unknown fields)

FORWARD COMPATIBLE   (old schema reads new data)
  - Add field with default
  - Remove required field

FULL COMPATIBLE      (both directions)
  - Only add optional fields with defaults
  - NEVER rename or change field types

BREAKING CHANGES (never do in production):
  - Rename field without alias
  - Change field type (int → string)
  - Remove field that has no default
```

### Avro Schema Example

```json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.example",
  "fields": [
    {"name": "orderId",   "type": "string"},
    {"name": "userId",    "type": "string"},
    {"name": "amount",    "type": "double"},
    {"name": "currency",  "type": "string", "default": "USD"},
    {"name": "createdAt", "type": "long",   "logicalType": "timestamp-millis"}
  ]
}
```

```java
// Java — Schema Registry producer
props.put("schema.registry.url", "http://schema-registry:8081");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
    KafkaAvroSerializer.class.getName());
// Subject naming strategy: TopicNameStrategy (default), RecordNameStrategy, TopicRecordNameStrategy
props.put("value.subject.name.strategy",
    "io.confluent.kafka.serializers.subject.TopicRecordNameStrategy");
```

---

## Lag Monitoring

```
Consumer Lag = Latest Offset (Log End Offset) - Committed Offset

Partition P0:
  Log End Offset:    1000  ← latest message written
  Committed Offset:   950  ← last offset consumer confirmed
  Lag:                 50  ← messages not yet processed

Total Group Lag = SUM(lag across all partitions)
```

```bash
# CLI lag check
kafka-consumer-groups.sh \
  --bootstrap-server broker:9092 \
  --describe \
  --group my-consumer-group

# Output columns: GROUP, TOPIC, PARTITION, CURRENT-OFFSET, LOG-END-OFFSET, LAG
```

**Alert thresholds**:
- Lag increasing monotonically → consumer falling behind (scale up or optimize)
- Lag suddenly jumps → consumer restart caused rebalance
- Lag = -1 → consumer never committed (check `auto.offset.reset`)

---

## Kafka Connect

```
SOURCE CONNECTOR                    SINK CONNECTOR
┌──────────────┐                  ┌──────────────┐
│ Database /   │                  │ Kafka Topic  │
│ REST API /   │──► Kafka Topic ──►│             │──► Elasticsearch
│ File System  │                  │              │    S3, JDBC DB
└──────────────┘                  └──────────────┘
        │
        └── Single Message Transforms (SMTs) applied inline
            - ExtractField, MaskField, ReplaceField
            - TimestampRouter (route to time-partitioned topics)
            - InsertField (add metadata)
```

```json
// Connector config — Debezium PostgreSQL source
{
  "name": "postgres-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "orders",
    "table.include.list": "public.orders,public.customers",
    "plugin.name": "pgoutput",
    "publication.autocreate.mode": "filtered",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false"
  }
}
```

---

## Production Pitfalls

### 1. Message Too Large
```
RecordTooLargeException: The request included a message larger than the max message size

Fix:
  # Broker
  message.max.bytes=10485760        # 10 MB
  # Topic override
  kafka-configs.sh --alter --topic my-topic \
    --add-config max.message.bytes=10485760

  # Producer
  max.request.size=10485760

  # Consumer
  max.partition.fetch.bytes=10485760

Better fix: store large payloads in S3/blob store, send reference in Kafka message.
```

### 2. Too Many Partitions
```
> 4000 partitions per broker → ZooKeeper/KRaft metadata overhead
> 10k partitions cluster-wide → leader election time on rolling restart becomes minutes

Rule: 10-100 partitions per topic for most workloads.
Use fewer large partitions rather than many small ones.
```

### 3. Consumer Stuck in Rebalance Loop
```
Symptom: "Attempt to heartbeat failed since group is rebalancing" repeating endlessly

Root causes + fixes:
1. max.poll.interval.ms too low for processing time
   → Increase max.poll.interval.ms=600000, reduce max.poll.records=100

2. Consumer dies during rebalance, triggers new rebalance
   → Add shutdown hook, ensure graceful close

3. Too many consumers (> partitions count)
   → Excess consumers sit idle; not a bug but wasteful

4. Switch to CooperativeStickyAssignor to minimize disruption
```

---

## Performance Tuning Checklists

### Throughput-Optimized Producer
- [ ] `acks=1` (or `acks=0` if loss acceptable)
- [ ] `compression.type=lz4` or `zstd`
- [ ] `batch.size=65536` (64 KB)
- [ ] `linger.ms=20` (allow batching)
- [ ] `buffer.memory=134217728` (128 MB)
- [ ] Multiple producer instances across threads

### Latency-Optimized Producer
- [ ] `linger.ms=0`
- [ ] `batch.size=16384`
- [ ] `acks=1`
- [ ] Disable compression if CPU is the bottleneck

### Consumer Throughput
- [ ] `fetch.min.bytes=65536`
- [ ] `fetch.max.wait.ms=500`
- [ ] `max.poll.records=1000`
- [ ] Parallel processing within poll loop (thread pool per record)
- [ ] Separate processing threads from poll thread (use `pause()`/`resume()` for backpressure)

### Broker Tuning
- [ ] `num.io.threads` = 2x number of disks
- [ ] `num.network.threads` = 3–6
- [ ] `log.retention.bytes` per partition to bound disk usage
- [ ] `min.insync.replicas=2` when `acks=all`
- [ ] `unclean.leader.election.enable=false` for financial data

---

## Quick Reference

```
Topic creation with recommended settings:
kafka-topics.sh --create \
  --bootstrap-server broker:9092 \
  --topic orders \
  --partitions 12 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000 \
  --config compression.type=zstd \
  --config max.message.bytes=1048576

Describe topic:
kafka-topics.sh --describe --topic orders --bootstrap-server broker:9092

Reset consumer group offset to earliest:
kafka-consumer-groups.sh \
  --bootstrap-server broker:9092 \
  --group my-group \
  --topic orders \
  --reset-offsets --to-earliest --execute
```
