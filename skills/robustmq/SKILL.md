---
name: robustmq
description: Expert RobustMQ skill — multi-protocol cloud-native message queue (MQTT, Kafka-compat, AMQP) built in Rust. Trigger on RobustMQ setup, MQTT IoT streaming, Kafka-compatible migration, AMQP/RabbitMQ replacement, multi-protocol broker architecture, or RobustMQ vs alternatives decisions.
---

# RobustMQ

You are a production RobustMQ engineer. When answering questions, apply cloud-native reasoning: think about protocol boundaries, the implications of shared storage, and the maturity caveats of a young project. Always verify feature availability against the current GitHub release — RobustMQ moves fast.

---

## What RobustMQ Is

[RobustMQ](https://github.com/robustmq/robustmq) is an open-source, cloud-native, high-performance multi-protocol message queue written in Rust (Apache-2.0).

**Core value proposition:** One broker, three protocols — replace RabbitMQ (AMQP), Kafka (streaming), and EMQ X (MQTT/IoT) with a single Rust binary cluster.

Key characteristics:
- **Multi-protocol in a single process**: MQTT 3.1/5.0, AMQP 0-9-1, Kafka-compatible wire protocol, HTTP
- **Rust runtime**: no JVM, no GC pauses, low memory footprint
- **Cloud-native architecture**: stateless broker nodes + separate metadata (Placement Center) + separate storage (Journal Server)
- **Pluggable storage**: RocksDB (embedded) or Journal Server (distributed, replicated)
- **Design goal**: operational simplicity — one cluster to manage instead of three

**Maturity warning:** RobustMQ (2024–2025) is production-ready for MQTT workloads; Kafka-compat and AMQP support is stabilizing. Always verify current feature status at https://github.com/robustmq/robustmq before committing to production.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client Layer                            │
│                                                                 │
│   MQTT Devices/Apps    Kafka Producers/Consumers    AMQP Apps   │
│   (IoT sensors,        (existing Kafka code,        (RabbitMQ   │
│    mobile clients)      no SDK change needed)        consumers) │
└──────────┬──────────────────────┬───────────────────────┬───────┘
           │ :1883 / :8883        │ :9092                 │ :5672
           ▼                      ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    RobustMQ Broker Nodes                         │
│  (stateless; horizontally scalable; protocol handlers per port) │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │   MQTT Handler   │  │  Kafka Handler   │  │ AMQP Handler │  │
│  │  QoS 0/1/2       │  │  produce/fetch   │  │ exchanges,   │  │
│  │  retain/will     │  │  consumer groups │  │ queues,      │  │
│  │  shared subs     │  │  topic metadata  │  │ routing keys │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ gRPC
                    ┌──────────────▼──────────────────┐
                    │        Placement Center          │
                    │  (Raft-based metadata cluster)   │
                    │                                  │
                    │  • Broker registration/discovery │
                    │  • Topic/queue/subscription meta │
                    │  • Consumer group coordination   │
                    │  • Cluster membership & health   │
                    │  • Leader election               │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │         Journal Server           │
                    │  (distributed storage layer)     │
                    │                                  │
                    │  • Segment-based append-only log │
                    │  • Replication across nodes      │
                    │  • Tiered storage (planned)      │
                    │  • RocksDB for embedded mode     │
                    └─────────────────────────────────┘
```

### Component Roles

| Component | Role | Analogous To |
|---|---|---|
| **Broker Node** | Protocol handler; stateless; scale horizontally | Kafka Broker (protocol layer only) |
| **Placement Center** | Metadata, coordination, consensus | Kafka KRaft Controller |
| **Journal Server** | Durable, replicated message storage | Kafka log storage |
| **robust-ctl** | CLI for cluster/topic/client management | kafka-topics.sh + kafka-consumer-groups.sh |

---

## Quick Start with Docker Compose

```yaml
version: "3.9"

services:
  placement-center:
    image: robustmq/placement-center:latest
    container_name: robustmq-placement
    ports:
      - "1228:1228"    # gRPC (internal use by brokers)
      - "1227:1227"    # HTTP management API
    volumes:
      - placement-data:/var/lib/robustmq/placement
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:1227/"]
      interval: 10s
      timeout: 5s
      retries: 10

  mqtt-broker:
    image: robustmq/mqtt-server:latest
    container_name: robustmq-mqtt
    ports:
      - "1883:1883"    # MQTT plaintext
      - "8883:8883"    # MQTT over TLS
      - "8083:8083"    # MQTT over WebSocket
      - "8084:8084"    # MQTT over WebSocket + TLS
      - "8080:8080"    # HTTP management / health
    environment:
      PLACEMENT_CENTER_ADDR: "placement-center:1228"
    depends_on:
      placement-center:
        condition: service_healthy
    restart: unless-stopped

  kafka-broker:
    image: robustmq/kafka-server:latest
    container_name: robustmq-kafka
    ports:
      - "9092:9092"    # Kafka protocol
    environment:
      PLACEMENT_CENTER_ADDR: "placement-center:1228"
    depends_on:
      placement-center:
        condition: service_healthy
    restart: unless-stopped

  amqp-broker:
    image: robustmq/amqp-server:latest
    container_name: robustmq-amqp
    ports:
      - "5672:5672"    # AMQP 0-9-1
      - "15672:15672"  # Management UI (if enabled)
    environment:
      PLACEMENT_CENTER_ADDR: "placement-center:1228"
    depends_on:
      placement-center:
        condition: service_healthy
    restart: unless-stopped

volumes:
  placement-data:
```

**Minimum viable cluster:** placement-center + one broker type. Add broker types as needed — they share the same Placement Center.

---

## MQTT Protocol

### MQTT Concepts in RobustMQ

```
Topic hierarchy (MQTT wildcard-based, unlike Kafka's exact-match):
  sensors/+/temperature     ← + matches one level
  sensors/#                 ← # matches any remaining levels

QoS levels:
  QoS 0 — at-most-once   (fire and forget; fastest; no ack)
  QoS 1 — at-least-once  (delivery confirmed; duplicates possible)
  QoS 2 — exactly-once   (four-way handshake; slowest; no duplicates)

Shared subscriptions (MQTT 5.0 — load balancing):
  $share/group-name/topic/path
  Multiple subscribers on same share group → each message to ONE subscriber
  (This is RobustMQ's MQTT equivalent of Kafka consumer groups)
```

### Rust MQTT Client (`rumqttc`)

```rust
// Cargo.toml
// rumqttc = "0.24"
// tokio = { version = "1", features = ["full"] }

use rumqttc::{AsyncClient, EventLoop, MqttOptions, QoS, Event, Packet};
use std::time::Duration;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let mut opts = MqttOptions::new("rust-client-1", "localhost", 1883);
    opts.set_keep_alive(Duration::from_secs(30));
    opts.set_credentials("user", "password");
    opts.set_clean_session(false);  // persist session; resubscribe not needed on reconnect

    let (client, mut eventloop) = AsyncClient::new(opts, 100); // 100 = inflight cap

    // Subscribe before starting the event loop task
    client.subscribe("sensors/+/temperature", QoS::AtLeastOnce).await?;
    client.subscribe("$share/processors/orders/#", QoS::ExactlyOnce).await?;

    // Publish
    client
        .publish(
            "sensors/zone-a/temperature",
            QoS::AtLeastOnce,
            false,  // retain = false
            b"23.5",
        )
        .await?;

    // Event loop — must be polled continuously
    loop {
        match eventloop.poll().await {
            Ok(Event::Incoming(Packet::Publish(msg))) => {
                let topic = &msg.topic;
                let payload = std::str::from_utf8(&msg.payload)?;
                println!("topic={topic} payload={payload} qos={:?}", msg.qos);
            }
            Ok(Event::Incoming(Packet::ConnAck(_))) => println!("Connected"),
            Ok(Event::Incoming(Packet::PubAck(ack))) => {
                println!("QoS1 ack for pkid={}", ack.pkid);
            }
            Ok(_) => {}
            Err(e) => {
                eprintln!("MQTT error: {e}; reconnecting...");
                tokio::time::sleep(Duration::from_secs(1)).await;
            }
        }
    }
}
```

### MQTT 5.0 Features

```rust
use rumqttc::v5::{AsyncClient, MqttOptions, QoS};
use rumqttc::v5::mqttbytes::v5::PublishProperties;

// Message expiry (MQTT 5.0)
let mut props = PublishProperties::default();
props.message_expiry_interval = Some(3600); // seconds

// User properties (custom headers)
props.user_properties = vec![
    ("correlation-id".into(), "req-789".into()),
    ("content-type".into(), "application/json".into()),
];

client
    .publish_with_properties("orders/created", QoS::AtLeastOnce, false, payload, props)
    .await?;
```

---

## Kafka-Compatible API

RobustMQ exposes a Kafka wire-protocol–compatible endpoint at `:9092`. Existing Kafka producers and consumers connect without code changes — only the `bootstrap.servers` address changes.

**Supported (verify against current release):**
- Produce messages (all acks levels)
- Fetch/consume messages
- Consumer groups with offset commit
- Topic creation/deletion via admin API
- List offsets, metadata

**Not yet supported (check GitHub for current status):**
- Kafka Streams API
- Exactly-once transactions (`read_committed` isolation)
- Kafka Connect
- ACL / fine-grained authorization

### Java Producer (no code change needed)

```java
// pom.xml: org.apache.kafka:kafka-clients:3.x
import org.apache.kafka.clients.producer.*;
import java.util.Properties;

Properties props = new Properties();
// Only this line changes from Kafka:
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "robustmq-host:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
    "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
    "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);      // micro-batching

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>(
    "orders",
    "customer-456",  // key → partition routing
    "{\"event\":\"order.created\",\"id\":\"123\"}"
);

producer.send(record, (metadata, ex) -> {
    if (ex != null) {
        log.error("Send failed", ex);
    } else {
        log.info("Sent to partition={} offset={}", metadata.partition(), metadata.offset());
    }
});
producer.flush();
producer.close();
```

### Java Consumer

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.*;

Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "robustmq-host:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processors");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // manual commit
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
    "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
    "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
        for (ConsumerRecord<String, String> record : records) {
            processOrder(record.value());
        }
        // Commit only after processing
        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

### Python Kafka Producer

```python
# pip install kafka-python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['robustmq-host:9092'],
    key_serializer=str.encode,
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    acks='all',
    retries=3,
    linger_ms=5,
)

future = producer.send(
    'orders',
    key='customer-456',
    value={'event': 'order.created', 'id': '123', 'amount': 99.99}
)
record_metadata = future.get(timeout=10)
print(f"partition={record_metadata.partition} offset={record_metadata.offset}")
producer.close()
```

### Python Kafka Consumer

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['robustmq-host:9092'],
    group_id='order-processors',
    auto_offset_reset='earliest',
    enable_auto_commit=False,   # manual commit
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    try:
        process_order(message.value)
        consumer.commit()
    except Exception as e:
        print(f"Processing failed: {e}")
        # Do NOT commit — message will be redelivered
```

---

## AMQP 0-9-1 (RabbitMQ Migration Path)

RobustMQ supports AMQP 0-9-1 on port 5672, making it a drop-in target for RabbitMQ migrations.

```
AMQP concepts in RobustMQ:
  Exchange → routes messages to queues via binding keys
  Queue    → durable or transient message buffer
  Binding  → exchange → queue routing rule
  Consumer → pulls from queue; acks or nacks messages

Exchange types:
  direct  → exact routing key match
  fanout  → broadcast to all bound queues
  topic   → wildcard routing key (*.orders.#)
  headers → match on message header attributes
```

### Python AMQP Producer

```python
# pip install pika
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        port=5672,
        credentials=pika.PlainCredentials('user', 'password'),
        heartbeat=60,
        blocked_connection_timeout=300,
    )
)
channel = connection.channel()

# Declare exchange and queue (idempotent)
channel.exchange_declare(exchange='orders', exchange_type='topic', durable=True)
channel.queue_declare(queue='order-created', durable=True)
channel.queue_bind(exchange='orders', queue='order-created', routing_key='order.created.*')

channel.basic_publish(
    exchange='orders',
    routing_key='order.created.standard',
    body=json.dumps({'id': '123', 'amount': 99.99}),
    properties=pika.BasicProperties(
        delivery_mode=2,        # persistent — survives broker restart
        content_type='application/json',
    )
)
connection.close()
```

### Python AMQP Consumer

```python
import pika

def on_message(channel, method, properties, body):
    try:
        order = json.loads(body)
        process_order(order)
        channel.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        print(f"Processing failed: {e}")
        # Nack without requeue after max retries; send to DLX
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.basic_qos(prefetch_count=10)  # flow control — limit in-flight messages
channel.basic_consume(queue='order-created', on_message_callback=on_message)
channel.start_consuming()
```

### Java AMQP (Spring AMQP)

```java
// Spring Boot application.yml
// spring.rabbitmq.host=robustmq-host
// spring.rabbitmq.port=5672
// (no code change from RabbitMQ)

@Component
public class OrderConsumer {

    @RabbitListener(queues = "order-created")
    public void handleOrder(OrderCreatedEvent event) {
        processOrder(event);
        // Spring AMQP auto-acks on successful return
        // throws exception → nack + requeue (or DLX if configured)
    }
}
```

---

## CLI (`robust-ctl`)

```bash
# Cluster operations
robust-ctl cluster status
robust-ctl cluster nodes list

# MQTT broker management
robust-ctl mqtt topics list
robust-ctl mqtt topics get sensors/temperature
robust-ctl mqtt clients list
robust-ctl mqtt clients disconnect client-id-123
robust-ctl mqtt subscriptions list --client client-id-123

# Kafka-compat operations
robust-ctl kafka topics create orders --partitions 4 --replication-factor 1
robust-ctl kafka topics list
robust-ctl kafka topics describe orders
robust-ctl kafka consumer-groups list
robust-ctl kafka consumer-groups describe order-processors
robust-ctl kafka consumer-groups reset-offset order-processors orders --to-earliest

# AMQP operations
robust-ctl amqp exchanges list
robust-ctl amqp queues list
robust-ctl amqp queues purge order-created

# Placement Center management
robust-ctl placement status
robust-ctl placement leader
```

---

## RobustMQ vs Alternatives

| | RobustMQ | RabbitMQ | Apache Kafka | EMQ X |
|---|---|---|---|---|
| **Language** | Rust | Erlang/OTP | Java/Scala | Erlang |
| **Protocols** | MQTT + Kafka + AMQP | AMQP + MQTT (plugin) | Kafka binary | MQTT + CoAP |
| **Kafka-compat** | Yes | No | Native | No |
| **IoT / MQTT** | Yes (MQTT 3.1/5.0) | Partial (plugin) | No | Yes (primary) |
| **AMQP** | 0-9-1 | 0-9-1 + 1.0 | No | No |
| **Memory floor** | ~50 MB | ~200 MB | ~1 GB (JVM) | ~150 MB |
| **Operational units** | 1 cluster type | 1 cluster | Broker + KRaft | 1 cluster |
| **Maturity** | Young (2024–) | Mature | Mature | Mature |
| **Connectors** | Limited | Many plugins | Kafka Connect | Limited |
| **Production use** | Emerging | Wide enterprise | Wide enterprise | IoT-wide |

---

## When to Use RobustMQ

**Use RobustMQ when:**
- You need MQTT (IoT) + Kafka-style streaming + AMQP from ONE cluster — eliminating EMQ X + Kafka + RabbitMQ
- Rust-native infrastructure — no JVM, no Erlang runtime to manage
- IoT platform with millions of device connections (MQTT shared subscriptions for load balancing)
- Replacing RabbitMQ with something horizontally scalable (AMQP compatibility reduces migration cost)
- New project or greenfield microservices where you can accept a young ecosystem
- Reducing broker sprawl — one ops team, one monitoring stack, one certificate rotation process

**Do NOT use RobustMQ when:**
- You need Kafka Streams, Kafka Connect, or ksqlDB
- You need battle-tested exactly-once Kafka transactions
- You require RabbitMQ plugins (Shovel, Federation, delayed messages without workarounds)
- You need AMQP 1.0 (RobustMQ supports 0-9-1 only)
- Your organization requires enterprise support contracts
- You are running MQTT workloads that need CoAP, LwM2M, or the wider EMQ X rule engine

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Treating Kafka-compat as 100% Kafka | Missing features (transactions, Streams) break at runtime | Test against RobustMQ's Kafka-compat matrix before migrating |
| Skipping `durable=True` on AMQP queues | Queue lost on broker restart | Always declare queues/exchanges as durable in production |
| QoS 0 for critical IoT commands | Message dropped silently under load or disconnect | Use QoS 1 (at-least-once) for commands; QoS 0 only for telemetry |
| AMQP consumer with `prefetch_count=0` | Unbounded in-flight messages; consumer OOM | Set `basic_qos(prefetch_count=N)` — N = your processing capacity |
| No Dead Letter Exchange (DLX) on AMQP queues | Failed messages block queue or are silently dropped | Configure DLX + dead letter queue; alert on DLX queue depth |
| Single Placement Center node | Raft requires quorum — one node = no fault tolerance | Run 3 Placement Center nodes minimum in production |
| MQTT `clean_session=True` on reconnect for persistent consumers | Session state lost; subscriptions must be re-sent | Set `clean_session=False` for durable subscribers |
| Kafka `enable.auto.commit=true` | Offsets committed before processing — data loss on crash | Always `enable.auto.commit=false`; commit after processing |
| Sharing one Kafka topic group across MQTT + Kafka clients | Consumption semantics differ; offsets collide | Keep protocol-specific consumers in separate consumer groups |

---

## Production Checklist

### Cluster

- [ ] Placement Center: 3 nodes minimum (Raft quorum requires majority)
- [ ] Journal Server: 3 nodes minimum with replication factor ≥ 2
- [ ] Broker nodes: 2+ per protocol; behind a load balancer
- [ ] Persistent volumes for Placement Center state (Raft log)
- [ ] TLS enabled for MQTT (`:8883`), AMQP (`:5671`), inter-node communication
- [ ] `ulimit -n` ≥ 65536 on all nodes (MQTT supports millions of persistent connections)
- [ ] Monitoring: Placement Center health, broker memory/CPU, queue depths, consumer group lag
- [ ] Health checks in orchestrator for all service types

### MQTT

- [ ] QoS 1 or 2 for any message where loss is unacceptable
- [ ] Shared subscriptions (`$share/<group>/<topic>`) for consumer-group semantics
- [ ] Max QoS 2 inflight limit tuned to client capacity
- [ ] Client ID uniqueness enforced — duplicate client ID disconnects the prior session
- [ ] Will messages configured for device disconnect detection
- [ ] Topic ACLs configured (deny `#` publish from devices; whitelist specific topics)

### Kafka-Compatible

- [ ] Verify feature matrix against current release before migrating
- [ ] `acks=all` on producers; `retries ≥ 3`
- [ ] `enable.auto.commit=false` on consumers
- [ ] Consumer group IDs unique per application
- [ ] Topic partition count set at creation; plan for scale (cannot reduce)

### AMQP

- [ ] All exchanges and queues declared `durable=True`
- [ ] Dead Letter Exchange (DLX) bound to every production queue
- [ ] `basic_qos` prefetch count set per consumer
- [ ] Message `delivery_mode=2` (persistent) for durability
- [ ] Manual ack (`auto_ack=False`); ack only after successful processing

### Observability

```bash
# Cluster health
robust-ctl cluster status

# MQTT client connections
robust-ctl mqtt clients list | wc -l

# Consumer group lag (Kafka-compat)
robust-ctl kafka consumer-groups describe order-processors

# Queue depth (AMQP)
robust-ctl amqp queues list   # look for "messages" count
```

Key metrics to monitor:
- **MQTT**: active connections, messages in/out per second, subscription count, QoS 2 inflight
- **Kafka-compat**: consumer lag per partition, produce/fetch latency, error rate
- **AMQP**: queue depth, consumer count, ack rate, nack/reject rate
- **Placement Center**: Raft leader elections, proposal latency, node count
