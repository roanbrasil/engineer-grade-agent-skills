---
name: redpanda
description: Use when working with Redpanda — setting up clusters, configuring topics, tuning performance, migrating from Kafka, or integrating Redpanda Connect pipelines
---

# Redpanda Expert

Redpanda is a Kafka-compatible streaming platform written in C++ using the Seastar framework. It is a drop-in Kafka replacement with no JVM, no ZooKeeper, and native Raft consensus built in.

## What Redpanda Is

- **100% Kafka API compatible**: use existing Kafka clients (Java, Python, Go, .NET) unchanged — just point `bootstrap.servers` at Redpanda
- **Written in C++ with Seastar**: async, thread-per-core, no JVM, no GC pauses
- **No ZooKeeper, no KRaft dependency**: Raft consensus is built directly into every node
- **Single binary**: broker + metadata controller in one process; no separate controller nodes to manage
- **Performance**: 10x lower p99 latency vs Apache Kafka; lower compute and storage cost per GB throughput

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                Kafka Clients (unchanged)              │
│     Producers          Consumers          Admin       │
└───────────┬───────────────┬───────────────┬──────────┘
            │               │               │ Kafka Wire Protocol
┌───────────▼───────────────▼───────────────▼──────────┐
│                 Redpanda Cluster                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │            │
│  │ (Leader) │  │(Follower)│  │(Follower)│            │
│  └──────────┘  └──────────┘  └──────────┘            │
│         ↕ Raft consensus — no ZooKeeper               │
│         ↕ Thread-per-core (Seastar shards)            │
└──────────────────────────────────────────────────────┘
           Storage: local NVMe per node
```

### Thread-Per-Core Architecture (Why Redpanda Is Fast)

Each CPU core runs an independent **shard**:

```
CPU Core 0  → Shard 0 → owns partitions {0, 3, 6, ...} + own memory + own NIC queue
CPU Core 1  → Shard 1 → owns partitions {1, 4, 7, ...} + own memory + own NIC queue
CPU Core 2  → Shard 2 → owns partitions {2, 5, 8, ...} + own memory + own NIC queue
```

- No shared memory between shards — zero mutex contention, zero lock overhead
- Partitions are pinned to shards — cache-hot, predictable access pattern
- No JVM GC — Kafka's GC causes p99 latency spikes; Redpanda never pauses
- Cross-shard communication uses **message passing** (Seastar futures/promises), not locks

**Consequence**: Redpanda's p99 latency is flat under load where Kafka degrades.

---

## RPK CLI (Redpanda Keeper)

`rpk` is the primary CLI for all Redpanda operations.

```bash
# ── Cluster ──────────────────────────────────────────────────────────────────
rpk cluster info                          # broker list, controller, cluster ID
rpk cluster health                        # overall health check
rpk cluster config get                    # dump all cluster config
rpk cluster config set log_segment_size 134217728   # 128 MiB segments
rpk cluster config set retention_bytes 10737418240  # 10 GiB retention

# ── Topics ───────────────────────────────────────────────────────────────────
rpk topic create orders --partitions 12 --replicas 3
rpk topic create events --partitions 24 --replicas 3 \
  --topic-config retention.ms=86400000 \
  --topic-config compression.type=lz4
rpk topic list
rpk topic describe orders                 # partition leaders, offsets, ISR
rpk topic delete orders

# ── Produce / Consume (testing) ───────────────────────────────────────────────
rpk topic produce orders <<< '{"event":"order.created","id":"123"}'
rpk topic produce orders -f '%v\n' < events.jsonl   # batch from file
rpk topic consume orders --from-beginning --num 10
rpk topic consume orders --group my-group --format '%v\n'

# ── Consumer Groups ───────────────────────────────────────────────────────────
rpk group list
rpk group describe my-group               # lag per partition
rpk group seek my-group --to-offset 0 --topics orders     # reset to start
rpk group seek my-group --to-timestamp 1700000000000      # reset by time
rpk group delete my-group

# ── Schema Registry ───────────────────────────────────────────────────────────
rpk registry subject list
rpk registry schema get --id 1
rpk registry subject describe orders-value
```

---

## Local Development: Docker Compose

```yaml
version: "3.8"
services:
  redpanda:
    image: redpandadata/redpanda:latest
    command:
      - redpanda start
      - --overprovisioned           # dev mode: relaxed resource checks
      - --smp 1                     # 1 CPU shard for dev
      - --memory 1G
      - --reserve-memory 0M
      - --node-id 0
      - --check=false
      - --kafka-addr 0.0.0.0:9092
      - --advertise-kafka-addr localhost:9092
      - --pandaproxy-addr 0.0.0.0:8082
      - --advertise-pandaproxy-addr localhost:8082
      - --schema-registry-addr 0.0.0.0:8081
      - --rpc-addr 0.0.0.0:33145
      - --advertise-rpc-addr redpanda:33145
    ports:
      - "9092:9092"    # Kafka protocol
      - "8082:8082"    # Pandaproxy (HTTP)
      - "8081:8081"    # Schema Registry
      - "9644:9644"    # Admin API (REST)
    volumes:
      - redpanda:/var/lib/redpanda/data

  console:
    image: redpandadata/console:latest
    environment:
      KAFKA_BROKERS: redpanda:9092
      KAFKASCHEMAREGISTRY_ENABLED: "true"
      KAFKASCHEMAREGISTRY_URLS: "http://redpanda:8081"
    ports:
      - "8080:8080"
    depends_on:
      - redpanda

volumes:
  redpanda:
```

### Three-Node Local Cluster

```yaml
version: "3.8"
services:
  redpanda-0:
    image: redpandadata/redpanda:latest
    command:
      - redpanda start
      - --node-id 0
      - --smp 1
      - --memory 512M
      - --overprovisioned
      - --kafka-addr 0.0.0.0:9092
      - --advertise-kafka-addr redpanda-0:9092
      - --rpc-addr 0.0.0.0:33145
      - --advertise-rpc-addr redpanda-0:33145
      - --seeds redpanda-0:33145,redpanda-1:33145,redpanda-2:33145
    ports: ["19092:9092"]
    volumes: ["redpanda-0:/var/lib/redpanda/data"]

  redpanda-1:
    image: redpandadata/redpanda:latest
    command:
      - redpanda start
      - --node-id 1
      - --smp 1
      - --memory 512M
      - --overprovisioned
      - --kafka-addr 0.0.0.0:9092
      - --advertise-kafka-addr redpanda-1:9092
      - --rpc-addr 0.0.0.0:33145
      - --advertise-rpc-addr redpanda-1:33145
      - --seeds redpanda-0:33145,redpanda-1:33145,redpanda-2:33145
    ports: ["29092:9092"]
    volumes: ["redpanda-1:/var/lib/redpanda/data"]

  redpanda-2:
    image: redpandadata/redpanda:latest
    command:
      - redpanda start
      - --node-id 2
      - --smp 1
      - --memory 512M
      - --overprovisioned
      - --kafka-addr 0.0.0.0:9092
      - --advertise-kafka-addr redpanda-2:9092
      - --rpc-addr 0.0.0.0:33145
      - --advertise-rpc-addr redpanda-2:33145
      - --seeds redpanda-0:33145,redpanda-1:33145,redpanda-2:33145
    ports: ["39092:9092"]
    volumes: ["redpanda-2:/var/lib/redpanda/data"]

volumes:
  redpanda-0:
  redpanda-1:
  redpanda-2:
```

---

## Kafka Client Integration (Zero Code Change)

Redpanda speaks the Kafka wire protocol. Change only `bootstrap.servers`.

### Java (kafka-clients)

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.*;

// ── Producer ──────────────────────────────────────────────────────────────────
Properties producerProps = new Properties();
producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "redpanda:9092");
producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
// Durability: wait for all in-sync replicas
producerProps.put(ProducerConfig.ACKS_CONFIG, "all");
// Idempotent producer (exactly-once within a session)
producerProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
producerProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps);
ProducerRecord<String, String> record =
    new ProducerRecord<>("orders", "order-123", "{\"event\":\"order.created\"}");

producer.send(record, (metadata, ex) -> {
    if (ex != null) log.error("Failed to send", ex);
    else log.info("Sent to {}-{} @ offset {}", metadata.topic(),
                  metadata.partition(), metadata.offset());
});
producer.flush();

// ── Consumer ──────────────────────────────────────────────────────────────────
Properties consumerProps = new Properties();
consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "redpanda:9092");
consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processors");
consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // manual commit

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
consumer.subscribe(List.of("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
    for (ConsumerRecord<String, String> r : records) {
        processOrder(r.value());
    }
    consumer.commitSync();  // commit after batch processed
}
```

### Kotlin (kafka-clients)

```kotlin
import org.apache.kafka.clients.producer.KafkaProducer
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.clients.consumer.KafkaConsumer
import java.time.Duration

// ── Producer ──────────────────────────────────────────────────────────────────
val producer = KafkaProducer<String, String>(mapOf(
    "bootstrap.servers" to "redpanda:9092",
    "key.serializer"    to "org.apache.kafka.common.serialization.StringSerializer",
    "value.serializer"  to "org.apache.kafka.common.serialization.StringSerializer",
    "acks"              to "all",
    "enable.idempotence" to "true",
    "compression.type"  to "lz4",
))

producer.send(ProducerRecord("orders", "order-123", """{"event":"order.created"}"""))
    .get()  // blocking; use async callback in production

// ── Consumer ──────────────────────────────────────────────────────────────────
val consumer = KafkaConsumer<String, String>(mapOf(
    "bootstrap.servers"  to "redpanda:9092",
    "group.id"           to "order-processors",
    "key.deserializer"   to "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer" to "org.apache.kafka.common.serialization.StringDeserializer",
    "auto.offset.reset"  to "earliest",
    "enable.auto.commit" to "false",
))

consumer.subscribe(listOf("orders"))
while (true) {
    consumer.poll(Duration.ofMillis(500)).forEach { record ->
        processOrder(record.value())
    }
    consumer.commitSync()
}
```

### Spring Boot (spring-kafka)

```java
// application.yml — only change is bootstrap address
spring:
  kafka:
    bootstrap-servers: redpanda:9092
    producer:
      acks: all
      compression-type: lz4
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: order-processors
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events"
```

```java
@Service
public class OrderEventService {

    private final KafkaTemplate<String, OrderCreatedEvent> template;

    public void publish(OrderCreatedEvent event) {
        template.send("orders", event.orderId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("Publish failed", ex);
            });
    }

    @KafkaListener(topics = "orders", groupId = "order-processors",
                   containerFactory = "kafkaListenerContainerFactory")
    public void consume(OrderCreatedEvent event,
                        Acknowledgment ack) {
        processOrder(event);
        ack.acknowledge();
    }
}
```

### Python (kafka-python)

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# ── Producer ──────────────────────────────────────────────────────────────────
producer = KafkaProducer(
    bootstrap_servers=["redpanda:9092"],
    acks="all",
    compression_type="lz4",
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    key_serializer=lambda k: k.encode("utf-8"),
)

producer.send("orders", key="order-123", value={"event": "order.created", "id": "123"})
producer.flush()

# ── Consumer ──────────────────────────────────────────────────────────────────
consumer = KafkaConsumer(
    "orders",
    bootstrap_servers=["redpanda:9092"],
    group_id="order-processors",
    auto_offset_reset="earliest",
    enable_auto_commit=False,
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
)

for message in consumer:
    process_order(message.value)
    consumer.commit()
```

---

## HTTP Proxy — Pandaproxy

Produce and consume via REST without any Kafka client library. Useful for serverless functions, languages without a Kafka SDK, or simple scripts.

```bash
# Produce a message
curl -s -X POST http://localhost:8082/topics/orders \
  -H "Content-Type: application/vnd.kafka.json.v2+json" \
  -d '{
    "records": [
      {"key": "order-123", "value": {"event": "order.created"}},
      {"key": "order-124", "value": {"event": "order.created"}}
    ]
  }'

# Consume — create a consumer first
curl -s -X POST http://localhost:8082/consumers/my-group \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  -d '{
    "name": "consumer-1",
    "format": "json",
    "auto.offset.reset": "earliest"
  }'

# Subscribe consumer to topic
curl -s -X POST \
  http://localhost:8082/consumers/my-group/instances/consumer-1/subscription \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  -d '{"topics": ["orders"]}'

# Fetch records
curl -s http://localhost:8082/consumers/my-group/instances/consumer-1/records \
  -H "Accept: application/vnd.kafka.json.v2+json"

# Direct partition read (no consumer group required)
curl -s "http://localhost:8082/topics/orders/partitions/0/records?offset=0&count=10" \
  -H "Accept: application/vnd.kafka.json.v2+json"
```

---

## Redpanda Connect (Benthos-Based Pipelines)

Redpanda acquired Benthos and ships it as **Redpanda Connect** (`rpk connect`). It is a batteries-included data integration framework with 200+ connectors.

```yaml
# pipeline.yaml — enrich orders from Kafka → HTTP sink
input:
  kafka:
    seed_brokers:
      - redpanda:9092
    topics:
      - orders
    consumer_group: connect-processors

pipeline:
  processors:
    - mapping: |
        root        = this
        root.processed_at = now()
        root.region = "us-east-1"

    - http:
        url: "https://enrichment.internal/lookup/${! json("order_id") }"
        verb: GET
        result_map: root.enrichment = content().string()

output:
  kafka:
    seed_brokers:
      - redpanda:9092
    topic: orders-enriched
```

```bash
# Run a pipeline
rpk connect run pipeline.yaml

# Run with environment override
rpk connect run --set input.kafka.seed_brokers=["prod-redpanda:9092"] pipeline.yaml

# List available connectors
rpk connect list inputs
rpk connect list outputs
rpk connect list processors
```

### Common Connect Patterns

```yaml
# S3 → Redpanda (backfill from object store)
input:
  aws_s3:
    bucket: my-data-lake
    prefix: orders/2024/
    codec: lines

output:
  kafka:
    seed_brokers: ["redpanda:9092"]
    topic: orders-backfill

---
# Redpanda → PostgreSQL (CDC sink)
input:
  kafka:
    seed_brokers: ["redpanda:9092"]
    topics: ["order-events"]
    consumer_group: pg-sink

pipeline:
  processors:
    - mapping: |
        root.id       = this.order_id
        root.status   = this.event
        root.updated  = this.timestamp

output:
  sql_insert:
    driver: postgres
    dsn: "postgres://user:pass@db:5432/orders"
    table: order_status
    columns: [id, status, updated]
    args_mapping: root = [this.id, this.status, this.updated]
```

---

## Schema Registry

Redpanda ships a built-in Schema Registry compatible with Confluent's API.

```bash
# Register a schema
curl -X POST http://localhost:8081/subjects/orders-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "{\"type\":\"record\",\"name\":\"Order\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"}'

# Fetch schema by ID
curl http://localhost:8081/schemas/ids/1

# List subjects
curl http://localhost:8081/subjects

# Check compatibility before evolution
curl -X POST http://localhost:8081/compatibility/subjects/orders-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "..."}'
```

Java with Avro + Schema Registry:

```java
// Maven: io.confluent:kafka-avro-serializer (works with Redpanda)
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "redpanda:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
props.put("schema.registry.url", "http://redpanda:8081");
```

---

## Tiered Storage

Offload cold log segments to S3/GCS/Azure Blob. Brokers hold only hot data; old segments are served directly from object storage on demand.

```bash
# Enable tiered storage cluster-wide
rpk cluster config set cloud_storage_enabled true
rpk cluster config set cloud_storage_region us-east-1
rpk cluster config set cloud_storage_bucket my-redpanda-bucket

# Per-topic tiered storage + local retention override
rpk topic alter-config orders \
  --set redpanda.remote.write=true \
  --set redpanda.remote.read=true \
  --set retention.local.target.bytes=536870912  # keep only 512 MiB locally
```

Architecture with tiered storage:

```
Producer → Redpanda Broker (hot segments: local NVMe)
                │
                ▼ (async upload after segment roll)
           S3 / GCS / Azure Blob (cold segments)
                │
Consumer ← Redpanda Broker (serves from S3 transparently on cache miss)
```

---

## Performance Tuning

```bash
# Segment size (larger = fewer files, better throughput; smaller = faster retention cleanup)
rpk cluster config set log_segment_size 268435456    # 256 MiB (default 128 MiB)

# Retention
rpk cluster config set delete_retention_ms 604800000  # 7 days
rpk cluster config set log_retention_bytes 107374182400  # 100 GiB per partition

# Replication
rpk cluster config set default_replication_factor 3
rpk cluster config set min_insync_replicas 2

# Compaction
rpk cluster config set log_compaction_interval_ms 5000

# Producer tuning (client-side)
# linger.ms: batch window — higher = more batching = higher throughput
# batch.size: max bytes per batch
# compression.type: lz4 is best overall; snappy for CPU-constrained
```

Topic-level config overrides:

```bash
rpk topic alter-config high-volume-topic \
  --set retention.ms=3600000 \
  --set segment.bytes=536870912 \
  --set compression.type=lz4 \
  --set min.insync.replicas=2
```

---

## Redpanda vs Apache Kafka

| Dimension              | Redpanda                        | Apache Kafka (KRaft)            |
|------------------------|---------------------------------|---------------------------------|
| Language               | C++ (Seastar)                   | Java / Scala                    |
| Runtime overhead       | Native binary, ~50 MB RSS       | JVM, 2–6 GB heap typical        |
| GC pauses              | None                            | G1GC / ZGC; p99 spikes          |
| ZooKeeper              | Not needed                      | KRaft (no ZK since 3.3)         |
| p99 latency            | Sub-millisecond typical         | 5–50 ms typical under load      |
| Operational complexity | Low (single binary)             | Medium (KRaft, JVM tuning)      |
| Kafka API compat.      | 100% (Kafka wire protocol)      | Native                          |
| Kafka Streams          | Not supported                   | Native                          |
| Connect framework      | Redpanda Connect (Benthos)      | Kafka Connect + connectors      |
| Schema Registry        | Built-in                        | Confluent (separate process)    |
| HTTP Proxy             | Pandaproxy built-in             | REST Proxy (separate)           |
| Tiered storage         | Built-in                        | Plugin (Confluent/community)    |
| Ecosystem              | Growing                         | Very mature (10+ years)         |
| License                | BSL 1.1 (source available)      | Apache 2.0                      |

---

## When to Use Redpanda

- Kafka-compatible streaming with lower operational overhead (single binary, no JVM tuning)
- p99 latency is a hard business requirement (fintech, gaming, real-time analytics)
- Small-to-medium engineering team that does not want to tune JVM heap, GC, or ZooKeeper
- Migrating from Kafka: zero client code changes required
- Need Kafka + built-in HTTP proxy + Schema Registry + data pipelines in one distribution
- Edge / IoT deployments where binary size and memory footprint matter

## When to Stick with Apache Kafka

- Need **Kafka Streams** (native stateful stream processing; no equivalent in Redpanda)
- Heavy investment in Kafka Connect with Confluent-specific SMTs or connectors
- Existing team with deep Kafka expertise and tuned operational runbooks
- Confluent Cloud or Confluent Platform contractual requirement
- Require Apache 2.0 license (Redpanda uses BSL 1.1)

---

## Anti-Patterns

```
WRONG: Using --overprovisioned in production
  --overprovisioned relaxes thread and memory scheduling checks designed for dev.
  Production nodes must NOT use this flag.

WRONG: Running Redpanda with shared storage (NFS, EBS multi-attach)
  Each node must have dedicated local NVMe or SSD. Shared storage causes
  severe latency degradation due to Seastar's direct I/O model.

WRONG: Setting replication_factor=1 for anything except dev/scratch topics
  A single replica means data loss if the node fails. Always use RF=3 in production.

WRONG: Ignoring min.insync.replicas
  acks=all without min.insync.replicas=2 degrades to single-replica durability
  when a follower is down. Always set both together.

WRONG: Using auto.create.topics.enable=true in production
  Typos in topic names silently create new topics with default config.
  Disable this; create topics explicitly with rpk.

WRONG: Not setting advertise-kafka-addr to the externally reachable address
  Clients will receive metadata pointing to the internal/loopback address
  and fail to connect after the initial bootstrap.
```

---

## Production Readiness Checklist

```
Cluster
  [ ] Minimum 3 nodes for HA; 5 for rolling upgrades without quorum loss
  [ ] Dedicated local NVMe/SSD per node; no shared storage
  [ ] --overprovisioned NOT set on production nodes
  [ ] advertise-kafka-addr set to externally reachable address per node
  [ ] TLS enabled for inter-broker and client-broker communication
  [ ] SASL/SCRAM or mTLS for client authentication

Topics
  [ ] replication_factor = 3 (default; 5 for critical topics)
  [ ] min.insync.replicas = 2 (enforced via cluster config)
  [ ] auto.create.topics.enable = false
  [ ] Partition count set based on throughput target; not changed after creation
  [ ] Retention policy (time and/or bytes) explicitly configured per topic

Producers
  [ ] acks = all
  [ ] enable.idempotence = true
  [ ] compression.type = lz4 (or snappy/zstd based on CPU budget)
  [ ] retries + retry.backoff.ms set for transient failures

Consumers
  [ ] enable.auto.commit = false; commit after processing
  [ ] Consumer group ID explicitly set (not default)
  [ ] Dead-letter topic for poison messages

Operations
  [ ] Metrics scraped: Prometheus endpoint at :9644/metrics
  [ ] Alert on under-replicated partitions
  [ ] Alert on consumer group lag > threshold
  [ ] rpk cluster health checked in CI/CD health gates
  [ ] Tiered storage configured for topics with long retention requirements
```

---

## Observability

```bash
# Prometheus metrics
curl -s http://localhost:9644/metrics | grep redpanda_kafka

# Admin API
curl -s http://localhost:9644/v1/brokers         # broker status
curl -s http://localhost:9644/v1/partitions       # partition placement
curl -s http://localhost:9644/v1/config/         # cluster config

# Key metrics to alert on
# redpanda_kafka_under_replicated_replicas > 0  → replication problem
# redpanda_kafka_request_latency_seconds{quantile="0.99"} > 0.1 → p99 above 100ms
# redpanda_group_offset_lag > N  → consumer lag
```

Grafana dashboard: Redpanda ships an official Grafana dashboard JSON at `https://grafana.com/grafana/dashboards/18135`.
