---
name: apache-pulsar
description: Use when working with Apache Pulsar — designing topic hierarchies, configuring subscriptions, implementing geo-replication, deploying BookKeeper, writing Pulsar Functions, or deciding between Pulsar and Kafka
---

# Apache Pulsar Expert

Apache Pulsar is a cloud-native distributed messaging and streaming platform originally built at Yahoo for planet-scale workloads. It separates compute (brokers) from storage (BookKeeper), making each independently scalable.

## What Pulsar Is

- **Unified messaging model**: streaming (like Kafka) + queuing (like RabbitMQ) in one system, via subscription types
- **Compute/storage separation**: stateless brokers + Apache BookKeeper (stateful storage); scale each dimension independently
- **Multi-tenancy first-class**: `tenant → namespace → topic` hierarchy with quotas, policies, and auth per namespace
- **Built-in geo-replication**: async cross-datacenter replication configured per namespace; no MirrorMaker equivalent needed
- **Tiered storage**: automatic segment offload to S3/GCS/Azure Blob after configurable threshold
- **Pulsar Functions**: lightweight stream-processing lambdas deployed directly into the cluster
- **Schema Registry**: enforced at broker level; prevents incompatible producers from publishing

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Producers / Consumers / Admin              │
│         (Pulsar Client SDK: Java, Python, Go, C++)       │
└───────────────────────┬─────────────────────────────────┘
                        │ Pulsar Binary Protocol (TCP/TLS)
┌───────────────────────▼─────────────────────────────────┐
│              Pulsar Brokers (stateless)                   │
│  - Route messages to correct ledger                      │
│  - Manage subscription cursors (in ZooKeeper)            │
│  - Enforce schema, auth, quotas                          │
│  - Host Pulsar Functions (optional)                      │
└───────────────────────┬─────────────────────────────────┘
                        │ BookKeeper wire protocol
┌───────────────────────▼─────────────────────────────────┐
│          Apache BookKeeper (stateful storage)             │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐               │
│  │ Bookie 1│   │ Bookie 2│   │ Bookie 3│               │
│  │ ledger  │   │ ledger  │   │ ledger  │               │
│  └─────────┘   └─────────┘   └─────────┘               │
│  Ensemble Qe / Write Quorum Qw / Ack Quorum Qa          │
│  e.g. Qe=3, Qw=3, Qa=2 → striped write + 2-of-3 ack   │
└───────────────────────┬─────────────────────────────────┘
                        │ (async, after retention threshold)
┌───────────────────────▼─────────────────────────────────┐
│      Tiered Storage (S3 / GCS / Azure Blob / HDFS)       │
│  Old ledger segments offloaded; brokers proxy on read    │
└─────────────────────────────────────────────────────────┘
     ZooKeeper (metadata: topic ownership, cursors, config)
```

### Key Architectural Properties

- **Brokers are stateless**: a broker can be added or removed without data migration; partitions are reassigned
- **BookKeeper write model**: entries are striped across an ensemble of Bookies; durability is achieved when `Qa` bookies acknowledge
- **Cursor persistence**: consumer positions (subscription cursors) live in ZooKeeper/metadata store, not the broker — so broker restarts don't lose consumer progress

---

## Tenancy Model

```
persistent://tenant/namespace/topic

Examples:
  persistent://acme/payments/transactions
  persistent://acme/payments/refunds
  persistent://acme/orders/created
  persistent://acme/orders/fulfilled

  non-persistent://acme/notifications/push   ← in-memory; no BookKeeper write
```

```
acme  (tenant)
├── payments  (namespace)  ← retention: 30d, replication: us-east + eu-west
│   ├── transactions
│   └── refunds
├── orders    (namespace)  ← retention: 7d, replication: us-east only
│   ├── created
│   └── fulfilled
└── notifications  (namespace)  ← retention: 1h, non-persistent OK
    └── push
```

Policies set **per namespace** apply to all topics inside:
- Message retention (time + size)
- Geo-replication clusters
- Authentication / authorization
- Throughput quotas (dispatch / publish rate limits)
- Schema enforcement strategy

---

## Subscription Types

```
EXCLUSIVE:
  P → [broker] → [C1]          # only 1 consumer; error if C2 tries
  Guarantees ordered delivery per topic partition.

SHARED (Round Robin):
  P → [broker] → [C1, C2, C3]  # messages distributed round-robin
  Like a Kafka consumer group. NO ordering guarantee.
  Best for parallel processing with many workers.

FAILOVER:
  P → [broker] → [C1 active]   # C2, C3 wait
                  [C2 standby]
                  [C3 standby]
  C2 takes over if C1 disconnects. Ordered delivery guaranteed.
  Best for ordered, fault-tolerant processing with a single active worker.

KEY_SHARED:
  P → [broker] → key=A → C1    # same key always routes to same consumer
                  key=B → C2
                  key=C → C1   # hash ring distribution
  Like per-key ordering in Kafka (partition key). No key races.
  Best for stateful processing keyed by entity ID.
```

Choose based on requirements:

| Need | Subscription Type |
|------|-------------------|
| Strict ordering, single consumer | Exclusive |
| Strict ordering, failover HA | Failover |
| High-throughput parallel (no ordering) | Shared |
| Per-entity ordering, parallel workers | Key_Shared |

---

## Java Client

```xml
<!-- Maven -->
<dependency>
  <groupId>org.apache.pulsar</groupId>
  <artifactId>pulsar-client</artifactId>
  <version>3.2.0</version>
</dependency>
```

```java
import org.apache.pulsar.client.api.*;
import java.util.concurrent.TimeUnit;

// ── Client ────────────────────────────────────────────────────────────────────
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://localhost:6650")
    .tlsTrustCertsFilePath("/path/to/ca.crt")      // TLS (prod)
    .authentication(AuthenticationFactory.token("jwt-token"))  // JWT auth (prod)
    .ioThreads(4)
    .listenerThreads(4)
    .build();

// ── Producer ──────────────────────────────────────────────────────────────────
Producer<String> producer = client.newProducer(Schema.STRING)
    .topic("persistent://acme/orders/created")
    .producerName("order-service")
    .compressionType(CompressionType.LZ4)
    .blockIfQueueFull(true)
    .batchingEnabled(true)
    .batchingMaxPublishDelay(5, TimeUnit.MILLISECONDS)
    .batchingMaxMessages(1000)
    .create();

// Send sync
MessageId msgId = producer.newMessage()
    .key("customer-123")                          // used for KEY_SHARED routing
    .value("{\"orderId\":\"abc\",\"amount\":99.99}")
    .property("eventType", "order.created")       // message properties (headers)
    .eventTime(System.currentTimeMillis())
    .send();

// Send async (non-blocking)
CompletableFuture<MessageId> future = producer.newMessage()
    .key("customer-124")
    .value("{\"orderId\":\"def\",\"amount\":49.99}")
    .sendAsync();

future.whenComplete((id, ex) -> {
    if (ex != null) log.error("Publish failed", ex);
    else log.debug("Published {}", id);
});

// ── Consumer (Shared) ─────────────────────────────────────────────────────────
Consumer<String> consumer = client.newConsumer(Schema.STRING)
    .topic("persistent://acme/orders/created")
    .subscriptionName("order-processors")
    .subscriptionType(SubscriptionType.Shared)
    .receiverQueueSize(1000)
    .negativeAckRedeliveryDelay(30, TimeUnit.SECONDS)
    .deadLetterPolicy(DeadLetterPolicy.builder()
        .maxRedeliverCount(5)
        .deadLetterTopic("persistent://acme/orders/created-dlq")
        .build())
    .subscribe();

// Receive loop
while (true) {
    Message<String> msg = consumer.receive(5, TimeUnit.SECONDS);
    if (msg == null) continue;
    try {
        processOrder(msg.getValue());
        consumer.acknowledge(msg);
    } catch (TransientException e) {
        consumer.negativeAcknowledge(msg);   // redeliver after delay
    } catch (PoisonPillException e) {
        consumer.acknowledge(msg);           // skip; or route to DLQ manually
    }
}

// ── Consumer (Key_Shared) ─────────────────────────────────────────────────────
Consumer<String> keyedConsumer = client.newConsumer(Schema.STRING)
    .topic("persistent://acme/orders/created")
    .subscriptionName("keyed-processors")
    .subscriptionType(SubscriptionType.Key_Shared)
    .keySharedPolicy(KeySharedPolicy.autoSplitHashRange())
    .subscribe();

// ── Reader (exact offset-style access) ───────────────────────────────────────
Reader<String> reader = client.newReader(Schema.STRING)
    .topic("persistent://acme/orders/created")
    .startMessageId(MessageId.earliest)          // or MessageId.latest, or specific ID
    .create();

while (reader.hasMessageAvailable()) {
    Message<String> msg = reader.readNext();
    // no subscription cursor; manual position control
}
```

### Avro Schema (Java)

```java
// Define schema
SchemaDefinition<OrderEvent> schemaDef = SchemaDefinition.<OrderEvent>builder()
    .withPojo(OrderEvent.class)
    .withAlwaysAllowNull(false)
    .build();

Schema<OrderEvent> avroSchema = Schema.AVRO(schemaDef);

Producer<OrderEvent> producer = client.newProducer(avroSchema)
    .topic("persistent://acme/orders/created")
    .create();

producer.send(new OrderEvent("order-123", 99.99, "CREATED"));
```

---

## Kotlin Client

```kotlin
import org.apache.pulsar.client.api.*
import kotlinx.coroutines.*

// ── Coroutine-friendly producer ───────────────────────────────────────────────
class OrderPublisher(serviceUrl: String) : AutoCloseable {

    private val client = PulsarClient.builder()
        .serviceUrl(serviceUrl)
        .build()

    private val producer = client.newProducer(Schema.STRING)
        .topic("persistent://acme/orders/created")
        .compressionType(CompressionType.LZ4)
        .batchingEnabled(true)
        .batchingMaxPublishDelay(5, java.util.concurrent.TimeUnit.MILLISECONDS)
        .create()

    suspend fun publish(orderId: String, payload: String): MessageId =
        withContext(Dispatchers.IO) {
            producer.newMessage()
                .key(orderId)
                .value(payload)
                .property("eventType", "order.created")
                .send()
        }

    override fun close() {
        producer.close()
        client.close()
    }
}

// ── Coroutine consumer ────────────────────────────────────────────────────────
class OrderConsumer(serviceUrl: String) {

    private val client = PulsarClient.builder()
        .serviceUrl(serviceUrl)
        .build()

    private val consumer = client.newConsumer(Schema.STRING)
        .topic("persistent://acme/orders/created")
        .subscriptionName("order-processors")
        .subscriptionType(SubscriptionType.Shared)
        .subscribe()

    suspend fun consumeLoop(process: suspend (String) -> Unit) = coroutineScope {
        while (isActive) {
            val msg = withContext(Dispatchers.IO) {
                consumer.receive(500, java.util.concurrent.TimeUnit.MILLISECONDS)
            } ?: continue
            try {
                process(msg.value)
                consumer.acknowledge(msg)
            } catch (e: Exception) {
                consumer.negativeAcknowledge(msg)
            }
        }
    }
}
```

### Spring Boot Integration (spring-pulsar)

```xml
<dependency>
  <groupId>org.springframework.pulsar</groupId>
  <artifactId>spring-pulsar-spring-boot-autoconfigure</artifactId>
  <version>1.0.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  pulsar:
    client:
      service-url: pulsar://localhost:6650
    producer:
      topic-name: persistent://acme/orders/created
      compression-type: lz4
    consumer:
      subscription-name: order-processors
      subscription-type: shared
```

```java
@Service
public class OrderEventService {

    private final PulsarTemplate<String> template;

    public void publish(String orderId, String payload) {
        template.send("persistent://acme/orders/created", payload,
            msg -> msg.key(orderId));
    }

    @PulsarListener(
        subscriptionName = "order-processors",
        topics = "persistent://acme/orders/created",
        subscriptionType = SubscriptionType.Shared
    )
    public void consume(String message, Message<String> pulsarMessage) {
        processOrder(message);
        // ack handled automatically by PulsarListener
    }
}
```

---

## Python Client

```python
import pulsar
from pulsar.schema import AvroSchema, Record, String, Double

# ── Schema ─────────────────────────────────────────────────────────────────────
class OrderEvent(Record):
    order_id = String()
    amount   = Double()

# ── Client ─────────────────────────────────────────────────────────────────────
client = pulsar.Client(
    'pulsar://localhost:6650',
    io_threads=4,
    message_listener_threads=4,
)

# ── Producer ───────────────────────────────────────────────────────────────────
producer = client.create_producer(
    'persistent://acme/orders/created',
    schema=AvroSchema(OrderEvent),
    compression_type=pulsar.CompressionType.LZ4,
    batching_enabled=True,
    batching_max_publish_delay_ms=5,
)

producer.send(
    OrderEvent(order_id='order-123', amount=99.99),
    partition_key='customer-123',
    properties={'eventType': 'order.created'},
    event_timestamp=int(time.time() * 1000),
)

# ── Consumer ───────────────────────────────────────────────────────────────────
consumer = client.subscribe(
    'persistent://acme/orders/created',
    subscription_name='order-processors',
    consumer_type=pulsar.ConsumerType.Shared,
    schema=AvroSchema(OrderEvent),
    negative_ack_redelivery_delay_ms=30_000,
    dead_letter_policy=pulsar.ConsumerDeadLetterPolicy(
        max_redeliver_count=5,
        dead_letter_topic='persistent://acme/orders/created-dlq',
    ),
)

while True:
    msg = consumer.receive(timeout_millis=1000)
    if msg is None:
        continue
    try:
        process_order(msg.value())
        consumer.acknowledge(msg)
    except TransientError:
        consumer.negative_acknowledge(msg)

client.close()
```

---

## pulsar-admin CLI

```bash
# ── Tenant ─────────────────────────────────────────────────────────────────────
pulsar-admin tenants create acme \
  --admin-roles admin@acme.com \
  --allowed-clusters us-east,eu-west

pulsar-admin tenants list
pulsar-admin tenants get acme

# ── Namespace ──────────────────────────────────────────────────────────────────
pulsar-admin namespaces create acme/orders

# Retention: keep 10 GiB OR 7 days, whichever limit is hit first
pulsar-admin namespaces set-retention acme/orders \
  --size 10G --time 7d

# Backlog quota: reject new messages when backlog exceeds 5 GiB
pulsar-admin namespaces set-backlog-quota acme/orders \
  --limit 5G --policy producer_exception

# Dispatch rate limit per topic in namespace
pulsar-admin namespaces set-dispatch-rate acme/orders \
  --msg-dispatch-rate 100000 --byte-dispatch-rate 104857600

# Geo-replication (set which clusters replicate to each other)
pulsar-admin namespaces set-replication-clusters acme/orders \
  --clusters us-east,eu-west

# ── Topics ─────────────────────────────────────────────────────────────────────
# Non-partitioned topic
pulsar-admin topics create persistent://acme/orders/created

# Partitioned topic (12 partitions; each is a logical sub-topic)
pulsar-admin topics partitioned-create persistent://acme/orders/created \
  --partitions 12

# Stats (producers, consumers, message rates, storage)
pulsar-admin topics stats persistent://acme/orders/created

# Detailed internal stats (ledger layout, cursor positions)
pulsar-admin topics stats-internal persistent://acme/orders/created

# ── Subscriptions ──────────────────────────────────────────────────────────────
pulsar-admin topics subscriptions persistent://acme/orders/created
pulsar-admin topics stats persistent://acme/orders/created \
  | jq '.subscriptions["order-processors"].msgBacklog'

# Reset consumer group to beginning (skip-all then seek)
pulsar-admin topics skip-all persistent://acme/orders/created \
  --subscription order-processors

# Reset to a specific message ID or timestamp
pulsar-admin topics reset-cursor persistent://acme/orders/created \
  --subscription order-processors \
  --time "2024-01-01T00:00:00Z"

# ── Schema ─────────────────────────────────────────────────────────────────────
pulsar-admin schemas get persistent://acme/orders/created
pulsar-admin schemas upload persistent://acme/orders/created \
  --filename schema.json
```

---

## Schema Registry

Schemas are enforced at the **broker** level. A producer publishing a message incompatible with the registered schema is rejected before the message reaches storage.

```json
{
  "type": "AVRO",
  "schema": {
    "type": "record",
    "name": "OrderCreated",
    "namespace": "com.acme.events",
    "fields": [
      {"name": "orderId",   "type": "string"},
      {"name": "amount",    "type": "double"},
      {"name": "customerId","type": "string"},
      {"name": "createdAt", "type": "long",   "logicalType": "timestamp-millis"}
    ]
  },
  "properties": {}
}
```

Schema compatibility strategies per namespace:

```bash
# BACKWARD: new schema can read old data (add optional fields with defaults)
pulsar-admin namespaces set-schema-compatibility-strategy acme/orders \
  --compatibility BACKWARD

# FULL: backward + forward compatible (safest for long-running consumers)
pulsar-admin namespaces set-schema-compatibility-strategy acme/orders \
  --compatibility FULL
```

---

## Pulsar Functions

Lightweight stream processing running inside the Pulsar cluster. No separate Flink/Spark cluster required for simple transformations.

```
Input Topic → [Function] → Output Topic
                   ↕
           State Store (per-function key-value)
           Metrics (automatically exposed)
```

### Java Function

```java
import org.apache.pulsar.functions.api.*;

public class OrderEnricherFunction implements Function<String, String> {

    @Override
    public String process(String input, Context ctx) {
        // ctx.getLogger() — function logger
        // ctx.incrCounter("processed", 1) — metrics counter
        // ctx.getState("key") / ctx.putState("key", value) — persistent state

        OrderEvent event = parseJson(input);
        event.setProcessedAt(Instant.now());
        event.setRegion(ctx.getInstanceId() % 2 == 0 ? "us-east" : "eu-west");

        ctx.incrCounter("orders_processed", 1);
        return toJson(event);
    }
}
```

```bash
# Deploy function
pulsar-admin functions create \
  --jar target/order-functions.jar \
  --classname com.acme.functions.OrderEnricherFunction \
  --inputs persistent://acme/orders/created \
  --output persistent://acme/orders/enriched \
  --name order-enricher \
  --parallelism 4 \
  --log-topic persistent://acme/functions/logs

# Monitor
pulsar-admin functions status --name order-enricher
pulsar-admin functions stats  --name order-enricher

# Restart / stop
pulsar-admin functions restart --name order-enricher
pulsar-admin functions stop    --name order-enricher
```

### Python Function

```python
from pulsar import Function

class WordCountFunction(Function):
    def process(self, input_msg: str, context) -> None:
        for word in input_msg.split():
            context.incr_counter(word, 1)
        # no return value → use sink topic if needed

# Deploy Python function
# pulsar-admin functions create \
#   --py word_count.py \
#   --classname word_count.WordCountFunction \
#   --inputs persistent://acme/text/raw \
#   --name word-count
```

---

## Geo-Replication

Built-in async replication per namespace. No MirrorMaker, no additional infrastructure.

```
  us-east cluster                eu-west cluster
  ┌──────────────┐               ┌──────────────┐
  │   Producers  │               │   Producers  │
  │      ↓       │               │      ↓       │
  │   Brokers    │ ←──replicate──│   Brokers    │
  │      ↓       │───replicate──→│      ↓       │
  │  BookKeeper  │               │  BookKeeper  │
  └──────────────┘               └──────────────┘
```

```bash
# Step 1: ensure both clusters are registered in each other's ZooKeeper
pulsar-admin clusters create us-east \
  --url http://pulsar-us-east:8080 \
  --broker-url pulsar://pulsar-us-east:6650

pulsar-admin clusters create eu-west \
  --url http://pulsar-eu-west:8080 \
  --broker-url pulsar://pulsar-eu-west:6650

# Step 2: create tenant allowed on both clusters
pulsar-admin tenants create acme \
  --allowed-clusters us-east,eu-west

# Step 3: enable replication on the namespace
pulsar-admin namespaces set-replication-clusters acme/orders \
  --clusters us-east,eu-west
```

After step 3, messages produced in `us-east` are automatically replicated to `eu-west` and vice versa. Consumers in either cluster see all messages.

**Replication lag monitoring**:
```bash
pulsar-admin topics stats persistent://acme/orders/created \
  | jq '.replication'
```

---

## Tiered Storage

Automatically offload cold BookKeeper ledger segments to object storage after a configurable threshold. Brokers proxy reads from object store transparently to consumers.

```yaml
# broker.conf
offloadersDirectory=/pulsar/offloaders
managedLedgerOffloadDriver=aws-s3
s3ManagedLedgerOffloadBucket=acme-pulsar-offload
s3ManagedLedgerOffloadRegion=us-east-1
# GCS alternative:
# managedLedgerOffloadDriver=google-cloud-storage
# gcsManagedLedgerOffloadBucket=acme-pulsar-offload
# gcsManagedLedgerOffloadServiceAccountKeyFile=/path/to/key.json
```

```bash
# Set offload threshold per namespace (offload when ledger > 10 GiB)
pulsar-admin namespaces set-offload-threshold acme/orders \
  --size 10G

# Trigger manual offload (normally automatic)
pulsar-admin topics offload persistent://acme/orders/created \
  --size-threshold 0
```

---

## Deployment: Docker Compose (Local Dev)

```yaml
version: "3.8"
services:
  pulsar:
    image: apachepulsar/pulsar:3.2.0
    command: bin/pulsar standalone
    ports:
      - "6650:6650"    # Pulsar binary protocol
      - "8080:8080"    # Admin REST API
    volumes:
      - pulsardata:/pulsar/data
      - pulsarconf:/pulsar/conf
    environment:
      PULSAR_MEM: "-Xms512m -Xmx512m -XX:MaxDirectMemorySize=256m"

  pulsar-manager:
    image: apachepulsar/pulsar-manager:v0.4.0
    ports:
      - "9527:9527"
      - "7750:7750"
    environment:
      SPRING_CONFIGURATION_FILE: /pulsar-manager/pulsar-manager/application.properties
    depends_on:
      - pulsar

volumes:
  pulsardata:
  pulsarconf:
```

### Production: Kubernetes (Helm)

```bash
# Add Pulsar Helm chart repo
helm repo add apache https://pulsar.apache.org/charts
helm repo update

# Install with 3 ZK nodes + 3 Bookies + 3 Brokers
helm install pulsar apache/pulsar \
  --namespace pulsar --create-namespace \
  --set initialize=true \
  --set zookeeper.replicaCount=3 \
  --set bookkeeper.replicaCount=3 \
  --set broker.replicaCount=3 \
  --set proxy.replicaCount=2 \
  --values custom-values.yaml
```

---

## Apache Pulsar vs Apache Kafka

| Dimension              | Apache Pulsar                          | Apache Kafka (KRaft)              |
|------------------------|----------------------------------------|-----------------------------------|
| Storage model          | Separate (BookKeeper)                  | Coupled (broker disk)             |
| Scale compute          | Independent of storage                 | Must scale together               |
| Scale storage          | Add Bookies without touching brokers   | Add brokers (rebalance partitions)|
| Multi-tenancy          | Native (tenant/namespace/topic)        | Via ACLs + naming conventions     |
| Geo-replication        | Built-in per namespace                 | MirrorMaker 2 (extra infra)       |
| Subscription modes     | Exclusive / Shared / Failover / Key_Shared | Consumer group only          |
| Functions              | Pulsar Functions built-in              | Need Kafka Streams / Flink        |
| Tiered storage         | Built-in (S3/GCS/HDFS)                | External plugin (Confluent/OSS)   |
| Schema Registry        | Built-in, enforced at broker           | Confluent (separate process)      |
| Ops complexity         | High (ZK + BookKeeper + Brokers + Proxy) | Lower (KRaft)                  |
| Partition reassignment | Transparent (broker is stateless)      | Involves data movement            |
| Kafka compatibility    | Limited (via KoP plugin)              | Native                            |
| Ecosystem maturity     | Smaller than Kafka                    | Very mature                       |
| Language               | Java                                   | Java / Scala                      |

---

## When to Use Apache Pulsar

- **Multi-tenant SaaS platform**: tenant isolation with per-namespace quotas, auth, and retention is a first-class requirement
- **Global active-active**: need built-in geo-replication without running MirrorMaker clusters
- **Mixed messaging semantics**: need both ordered streaming AND queue fanout in a single system (replacing Kafka + RabbitMQ)
- **Infinite retention without big disks**: tiered storage offloads to S3 automatically; topic backlog is not bounded by broker disk
- **Independent compute/storage scaling**: traffic spikes require more broker CPU but not more storage capacity (or vice versa)
- **Regulated data isolation**: compliance requires per-tenant storage boundaries enforced at the platform layer

## When NOT to Use Apache Pulsar

- **Team already deep in Kafka**: migration cost (3 layers to operate + client SDK changes) rarely exceeds Kafka's capability gap
- **Simple use case**: Kafka or Redpanda suffice with significantly lower ops complexity
- **Small team**: ZooKeeper + BookKeeper + Brokers + Proxy = 4 components to monitor, upgrade, and size
- **Kafka Streams or ksqlDB dependency**: no Pulsar equivalent for stateful stream processing; Pulsar Functions are lightweight only
- **Kafka Connect sink/source ecosystem**: Pulsar IO is less mature; many connectors lag behind Kafka Connect

---

## Anti-Patterns

```
WRONG: Using a single namespace for all teams
  Namespaces are the quota + policy boundary. Mixing teams in one namespace means
  one team's backlog or high publish rate affects all others.
  CORRECT: one namespace per team or service domain.

WRONG: Setting subscriptionType=Shared and expecting ordered delivery
  Shared distributes messages across all consumers round-robin. There is no
  ordering guarantee. Use Failover or Key_Shared if ordering matters.

WRONG: Not setting backlog quotas
  Without a backlog quota, a slow consumer will accumulate messages indefinitely,
  consuming Bookie disk. The broker will reject new messages only when disk fills.
  Set --backlog-quota with producer_exception policy on every namespace.

WRONG: Storing schema outside Pulsar (manual JSON parsing)
  Bypassing the built-in schema registry means incompatible producers reach
  consumers and cause runtime deserialization failures.
  CORRECT: register schemas; let the broker enforce compatibility.

WRONG: Using non-persistent topics for durable workloads
  non-persistent:// topics exist only in broker memory. A broker restart
  drops all undelivered messages. Use only for fire-and-forget notifications.

WRONG: Ignoring ack timeout vs negative-ack
  Not setting ackTimeout means unacked messages wait indefinitely (or until
  consumer restart). Set negativeAckRedeliveryDelay + deadLetterPolicy
  to bound redelivery behavior.

WRONG: Running Bookies on shared storage (SAN/NFS)
  BookKeeper's write performance depends on predictable fsync latency.
  Shared network storage causes write stalls. Use local NVMe per Bookie.
```

---

## Production Readiness Checklist

```
ZooKeeper
  [ ] Minimum 3 ZooKeeper nodes (5 for critical clusters)
  [ ] Dedicated disks; ZK data and transaction log on separate mounts
  [ ] ZK heap ≤ 8 GB; JVM GC tuned (G1GC)

BookKeeper (Bookies)
  [ ] Minimum 3 Bookies; ensemble size ≥ write quorum ≥ ack quorum
  [ ] Typical: Qe=3, Qw=3, Qa=2 (tolerate 1 Bookie loss without data loss)
  [ ] Dedicated local NVMe per Bookie; journals on separate device from ledgers
  [ ] Read/write cache sized to fit hot working set

Brokers
  [ ] Stateless; can scale horizontally without data migration
  [ ] TLS enabled on all ports (6650→6651 TLS, 8080→8443 HTTPS)
  [ ] Authentication: JWT or mTLS for clients; service accounts per producer/consumer
  [ ] Topic compaction and offload scheduled

Namespaces
  [ ] One namespace per team / domain
  [ ] Retention policy (size + time) explicitly set on every namespace
  [ ] Backlog quota set with producer_exception policy
  [ ] Schema compatibility strategy set (FULL for stable systems)

Topics
  [ ] Partitioned topics for high-throughput (≥ 1 partition per consumer instance)
  [ ] Dead-letter topic configured for every consumer with retries
  [ ] Tiered storage enabled for topics with retention > 7 days

Consumers
  [ ] enableAckTimeout + ackTimeoutMillis set to bound redelivery window
  [ ] negativeAckRedeliveryDelay set < ackTimeout
  [ ] Dead-letter policy with maxRedeliverCount ≤ 10
  [ ] Subscription names stable and versioned (changing name creates new cursor → reprocessing from start)

Observability
  [ ] Prometheus endpoint: broker :8080/metrics, bookie :8000/metrics
  [ ] Alert on: bookie disk usage > 80%, subscription backlog > threshold,
      replication lag > 60s, failed broker / bookie count
  [ ] Pulsar Manager or Streamnative Console for dashboard
```

---

## Observability

```bash
# Broker metrics (Prometheus format)
curl -s http://localhost:8080/metrics | grep pulsar_

# Key metrics
# pulsar_producers_count                       — active producers per topic
# pulsar_consumers_count                       — active consumers per subscription
# pulsar_rate_in / pulsar_rate_out             — msg/s per topic
# pulsar_throughput_in / pulsar_throughput_out — bytes/s
# pulsar_storage_size                          — total storage per topic
# pulsar_msg_backlog                           — unacked messages per subscription
# pulsar_replication_backlog                   — geo-replication lag
# pulsar_storage_offloaded_size                — bytes moved to tiered storage

# Admin REST equivalents
curl -s http://localhost:8080/admin/v2/persistent/acme/orders/created/stats \
  | jq '{msgRateIn: .msgRateIn, msgBacklog: .subscriptions["order-processors"].msgBacklog}'
```
