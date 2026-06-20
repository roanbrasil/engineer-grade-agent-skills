---
name: iggy-streaming
description: Expert Iggy message streaming — architecture, Rust SDK producers/consumers, consumer groups, offset management, server config, CLI, and production tuning. Trigger on Iggy setup questions, Rust streaming code, Iggy vs Kafka decisions, partition/offset strategy, or deployment config.
---

# Iggy Streaming

You are a production Iggy streaming engineer. When answering questions, apply engineering rigor: reason about partition strategy, offset commit discipline, connection lifecycle, and always explain the "why" behind config choices. Iggy is NOT Kafka — do not map Kafka concepts 1:1.

---

## What Iggy Is

[Iggy](https://iggy.rs) is an open-source, high-performance persistent message streaming platform written entirely in Rust.

Key distinctions:
- **No JVM, no ZooKeeper, no KRaft** — single binary, minimal ops surface
- **Not Kafka-compatible by design** — simpler mental model, purpose-built protocol
- **Three simultaneous protocols**: TCP binary (default, fastest), HTTP REST (easy integration), QUIC (low-latency, unreliable networks)
- **Append-only log persistence**: configurable retention by size or time
- **Performance**: millions of messages/sec on commodity hardware; sub-millisecond p99 latency on TCP

---

## Core Abstractions

```
Stream: "orders"
  └── Topic: "order-events" (4 partitions)
        ├── Partition 0: [msg0, msg1, msg4, ...]  ← append-only log
        ├── Partition 1: [msg2, msg5, ...]
        ├── Partition 2: [msg3, msg6, ...]
        └── Partition 3: [msg7, ...]

            Consumer Group "processors"
              ├── Consumer 1 ← assigned Partitions 0, 1
              └── Consumer 2 ← assigned Partitions 2, 3

Offset tracking (per consumer, per partition):
  Consumer 1 / Partition 0: committed_offset = 4
  Consumer 1 / Partition 1: committed_offset = 2
```

| Concept | Definition |
|---|---|
| **Stream** | Top-level namespace — named, unique ID; analogous to a Kafka cluster namespace |
| **Topic** | Named channel within a stream; configurable partition count; ordering within partition |
| **Partition** | Ordered, append-only log; messages routed by key hash or round-robin |
| **Consumer Group** | Group of consumers sharing a topic; each message delivered to exactly one member |
| **Offset** | Monotonically increasing position within a partition; committed by consumers |
| **Message** | ID (u128, auto-generated or custom), optional key (bytes), payload (bytes), headers (key-value map) |

**Critical: Iggy vs Kafka mental model**

```
Kafka:     cluster → topic → partition → consumer group
Iggy:      stream  → topic → partition → consumer group

Kafka broker = Iggy server (single process)
Kafka topic = Iggy topic (within a stream)
Kafka __consumer_offsets = Iggy server-side offset store per (consumer, partition)
```

---

## Server Setup

### Configuration (`server.toml`)

```toml
[system]
path = "/var/lib/iggy"          # persistence root; use fast disk (NVMe preferred)

[server]
address = "0.0.0.0:8090"       # TCP binary protocol

[http]
enabled = true
address = "0.0.0.0:3000"       # REST API; disable in high-throughput prod if not needed

[quic]
enabled = true
address = "0.0.0.0:8080"       # QUIC; good for edge/mobile clients

[message_saver]
enabled = true
enforce_sync = true             # fsync on every interval — durability guarantee
interval = "1s"                 # tune: lower = more durable, higher = more throughput

[logging]
level = "info"                  # use "warn" in prod for reduced I/O
path = "/var/log/iggy"
```

**`enforce_sync = true` is non-negotiable for production** — without it, messages in the OS page cache are lost on crash.

### Docker Compose (production-ready)

```yaml
version: "3.9"

services:
  iggy:
    image: iggyrs/iggy:latest
    restart: unless-stopped
    ports:
      - "8090:8090"   # TCP binary
      - "3000:3000"   # HTTP REST
      - "8080:8080/udp"  # QUIC
    volumes:
      - iggy-data:/var/lib/iggy
      - ./server.toml:/etc/iggy/server.toml:ro
    command: ["--config", "/etc/iggy/server.toml"]
    environment:
      RUST_LOG: info
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/"]
      interval: 10s
      timeout: 5s
      retries: 5
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

volumes:
  iggy-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/iggy    # mount fast disk here
```

---

## Rust SDK — Producer

```toml
# Cargo.toml
[dependencies]
iggy = "0.6"
tokio = { version = "1", features = ["full"] }
anyhow = "1"
```

```rust
use iggy::client::Client;
use iggy::clients::builder::IggyClientBuilder;
use iggy::identifier::Identifier;
use iggy::messages::send_messages::{Message, Partitioning};
use std::str::FromStr;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // TCP binary protocol — fastest; use HTTP only for debug/tooling
    let client = IggyClientBuilder::new()
        .with_tcp()
        .with_server_address("127.0.0.1:8090")
        .build()?;

    client.connect().await?;
    client.login_user("iggy", "iggy").await?;

    // Idempotent resource creation — safe to call on every startup
    client.create_stream("orders", None).await.ok();
    client
        .create_topic(
            &Identifier::named("orders")?,
            "order-events",
            4,     // partition count — set once; cannot be reduced later
            None,  // replication factor (reserved for future clustering)
            None,  // message expiry (None = retain forever)
            None,  // max topic size (None = unlimited)
            None,  // topic ID (None = auto-assign)
        )
        .await
        .ok();

    // Key-based partitioning: same key → same partition → ordering guarantee
    let mut messages = vec![
        Message::from_str("order.created", r#"{"id":"order-123","amount":99.99,"customer":"cust-456"}"#)?,
    ];

    client
        .send_messages(
            &Identifier::named("orders")?,
            &Identifier::named("order-events")?,
            &Partitioning::key_based(b"cust-456"),  // key → deterministic partition
            &mut messages,
        )
        .await?;

    println!("Message sent");
    Ok(())
}
```

### Producer — Batch Send (High Throughput)

```rust
use iggy::messages::send_messages::{Message, Partitioning};

async fn send_batch(client: &impl Client, orders: Vec<Order>) -> anyhow::Result<()> {
    // Batch by partition key to preserve per-customer ordering
    let mut buckets: std::collections::HashMap<String, Vec<Message>> =
        std::collections::HashMap::new();

    for order in orders {
        let payload = serde_json::to_vec(&order)?;
        let msg = Message::new(None, payload.into(), None)?;
        buckets
            .entry(order.customer_id.clone())
            .or_default()
            .push(msg);
    }

    for (customer_id, mut msgs) in buckets {
        client
            .send_messages(
                &Identifier::named("orders")?,
                &Identifier::named("order-events")?,
                &Partitioning::key_based(customer_id.as_bytes()),
                &mut msgs,
            )
            .await?;
    }
    Ok(())
}
```

---

## Rust SDK — Consumer

### Simple Poll (Single Consumer, No Group)

```rust
use iggy::consumer::Consumer;
use iggy::messages::poll_messages::PollingStrategy;

let messages = client
    .poll_messages(
        &Identifier::named("orders")?,
        &Identifier::named("order-events")?,
        Some(0),                         // partition_id — explicit for single consumer
        &Consumer::new(Identifier::named("my-consumer")?),
        &PollingStrategy::offset(0),     // start from beginning
        100,                             // max messages per poll
        true,                            // auto-commit offset after poll
    )
    .await?;

for msg in &messages.messages {
    let payload = std::str::from_utf8(&msg.payload)?;
    println!("partition={} offset={} id={} payload={}",
        msg.partition_id, msg.offset, msg.id, payload);
}
```

### Consumer Group — At-Least-Once Processing

```rust
use iggy::consumer::Consumer;
use iggy::messages::poll_messages::PollingStrategy;

// Create the consumer group once (idempotent)
client
    .create_consumer_group(
        &Identifier::named("orders")?,
        &Identifier::named("order-events")?,
        "processors",
        None, // group ID (None = auto)
    )
    .await
    .ok();

// Join the group — server assigns partitions automatically
client
    .join_consumer_group(
        &Identifier::named("orders")?,
        &Identifier::named("order-events")?,
        &Identifier::named("processors")?,
    )
    .await?;

loop {
    let messages = client
        .poll_messages(
            &Identifier::named("orders")?,
            &Identifier::named("order-events")?,
            None,                            // None = server assigns partition from group
            &Consumer::group("processors"),
            &PollingStrategy::next(),        // from last committed offset
            100,
            false,                           // manual commit — process first, then commit
        )
        .await?;

    if messages.messages.is_empty() {
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        continue;
    }

    for msg in &messages.messages {
        process_order(msg).await?;  // do your work first

        // Commit offset only after successful processing
        client
            .store_consumer_offset(
                &Consumer::group("processors"),
                &Identifier::named("orders")?,
                &Identifier::named("order-events")?,
                Some(msg.partition_id),
                msg.offset,
            )
            .await?;
    }
}
```

### Exactly-Once Pattern (Idempotent Receiver)

```rust
use std::collections::HashSet;

struct IdempotentProcessor {
    seen_ids: HashSet<u128>,  // in production: use Redis or DB
}

impl IdempotentProcessor {
    async fn process(&mut self, msg: &iggy::models::messages::PolledMessage) -> anyhow::Result<()> {
        if self.seen_ids.contains(&msg.id) {
            tracing::warn!(id = msg.id, "duplicate message — skipping");
            return Ok(());
        }

        // Process message
        let payload = std::str::from_utf8(&msg.payload)?;
        do_work(payload).await?;

        self.seen_ids.insert(msg.id);
        Ok(())
    }
}
```

---

## HTTP REST API

All endpoints require `Authorization: Basic <base64(user:password)>`.

```bash
BASE=http://localhost:3000

# Create stream
curl -X POST "$BASE/streams" \
  -H "Authorization: Basic aWdneTppZ2d5" \
  -H "Content-Type: application/json" \
  -d '{"stream_id": null, "name": "orders"}'

# Create topic
curl -X POST "$BASE/streams/orders/topics" \
  -H "Authorization: Basic aWdneTppZ2d5" \
  -H "Content-Type: application/json" \
  -d '{"topic_id": null, "name": "order-events", "partitions_count": 4, "message_expiry": null}'

# Publish messages
curl -X POST "$BASE/streams/orders/topics/order-events/messages" \
  -H "Authorization: Basic aWdneTppZ2d5" \
  -H "Content-Type: application/json" \
  -d '{
    "partitioning": {"kind": "message_key", "value": "Y3VzdC00NTY="},
    "messages": [
      {"id": null, "payload": "eyJldmVudCI6Im9yZGVyLmNyZWF0ZWQifQ==", "headers": null}
    ]
  }'

# Poll messages
curl "$BASE/streams/orders/topics/order-events/messages?consumer_group=processors&partition_id=0&strategy=next&count=10&auto_commit=true" \
  -H "Authorization: Basic aWdneTppZ2d5"
```

Note: payload and partition key must be base64-encoded in HTTP API.

---

## CLI (`iggy-cli`)

```bash
# Authentication (set once per session or use env vars)
export IGGY_URL=http://localhost:3000
export IGGY_USERNAME=iggy
export IGGY_PASSWORD=iggy

# Stream management
iggy stream create orders
iggy stream list
iggy stream get orders

# Topic management
iggy topic create --stream orders order-events --partitions 4
iggy topic list --stream orders
iggy topic stats --stream orders order-events   # message count, size, lag

# Consumer groups
iggy consumer-group create --stream orders --topic order-events processors
iggy consumer-group list --stream orders --topic order-events
iggy consumer-group get --stream orders --topic order-events processors  # see member assignments

# Message operations
iggy message send \
  --stream orders \
  --topic order-events \
  --partition-key cust-123 \
  '{"event":"order.created","id":"123"}'

iggy message poll \
  --stream orders \
  --topic order-events \
  --consumer-group processors \
  --strategy next \
  --count 10

# Offset inspection
iggy offset get \
  --stream orders \
  --topic order-events \
  --consumer-group processors \
  --partition 0
```

---

## Consumer Groups and Partition Assignment

```
Topic "order-events": 4 partitions

Initial state (2 consumers):
  Consumer A → Partitions 0, 1   (server assigns)
  Consumer B → Partitions 2, 3

Consumer C joins (rebalance triggered):
  Consumer A → Partition 0
  Consumer B → Partition 2
  Consumer C → Partitions 1, 3   (redistributed)

Consumer B crashes (rebalance triggered after heartbeat timeout):
  Consumer A → Partitions 0, 2
  Consumer C → Partitions 1, 3
```

**Rules:**
- Partition count is the maximum parallelism ceiling — adding more consumers than partitions is wasteful
- Server tracks offset per `(consumer_group, partition)` pair
- A consumer leaving the group triggers rebalance; in-flight uncommitted messages may be redelivered — design handlers to be idempotent

---

## Iggy vs Kafka

| Dimension | Iggy | Kafka (KRaft) |
|---|---|---|
| Runtime | Single Rust binary (~10 MB) | JVM + multiple JARs (500 MB+) |
| Protocol | TCP binary / HTTP / QUIC | Custom Kafka binary TCP |
| Min memory | ~50 MB | ~1 GB (JVM heap) |
| Ops complexity | Low — one process, one config | High — tune JVM, KRaft quorum |
| Ecosystem | Growing; young (2023–) | Mature; vast connector ecosystem |
| Kafka Connect | No equivalent yet | Yes (Kafka Connect) |
| Kafka Streams | No equivalent | Yes |
| Schema Registry | No native; use HTTP API | Confluent / Apicurio |
| Multi-language SDKs | Rust SDK + HTTP REST | Rich (Java, Python, Go, .NET, …) |
| Throughput | Millions msg/sec | Millions msg/sec |
| Latency (p99) | Sub-millisecond (TCP) | ~1–5 ms |
| Use case | Greenfield Rust; edge; low-ops | Enterprise; large ecosystem needs |

---

## When to Use Iggy

**Use Iggy when:**
- Greenfield project with no Kafka ecosystem dependency
- Rust-first stack — native SDK, no JVM overhead
- Need operational simplicity: no ZooKeeper, no KRaft quorum, single binary
- Edge or embedded streaming where JVM memory (~1 GB+) is prohibitive
- Kafka's operational complexity (connector configs, JVM tuning, broker/topic sprawl) is overkill for your scale
- You need HTTP and QUIC out of the box without a separate proxy

**Do NOT use Iggy when:**
- You depend on Kafka Connect sources/sinks (no equivalent)
- You need Kafka Streams or ksqlDB
- You have existing Kafka consumer code you cannot rewrite (different protocol)
- You need a mature ecosystem of third-party connectors (Debezium, Flink, Spark, etc.)
- You need enterprise support SLAs

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `auto_commit=true` in critical consumers | Offsets committed before processing — data loss on crash | Set `auto_commit=false`; commit only after successful processing |
| Ignoring send errors | Network blip causes silent message loss | Always `await` send result; implement retry with backoff |
| One partition for ordered global stream | No parallelism — single consumer bottleneck | Use key-based partitioning; accept per-key ordering, not global |
| More consumers than partitions | Extra consumers are idle; wasted connections | Set consumer count ≤ partition count |
| Not recreating stream/topic idempotently | Startup fails if stream/topic already exists | Call `create_*` with `.ok()` to ignore "already exists" errors |
| Unbounded in-memory buffering before send | OOM under backpressure | Use bounded `tokio::sync::mpsc` channels; apply backpressure upstream |
| HTTP REST for high-throughput producers | HTTP overhead adds latency; TCP binary is 3–5x faster | Use TCP binary SDK in production; HTTP for tooling/debug only |
| Storing offsets only in-memory | Offsets lost on restart; reprocess from beginning | Use server-side offset commit (`store_consumer_offset`) |
| Partition count set too low | Cannot scale consumers later; partition count is immutable | Overprovision partitions at topic creation (e.g., 16 for moderate load) |

---

## Production Checklist

### Server

- [ ] `message_saver.enforce_sync = true` — durability on crash
- [ ] Persistence path on fast disk (NVMe); separate from OS volume
- [ ] `ulimit -n` ≥ 65536 on the server process
- [ ] QUIC disabled if not needed (reduces attack surface)
- [ ] Firewall: restrict port 8090 (TCP) to trusted CIDRs; expose 3000 (HTTP) only if needed
- [ ] Log level set to `warn` in production; `info` only during rollouts
- [ ] Monitor disk usage — append-only log grows; set retention policy (`message_expiry` or `max_topic_size`)
- [ ] Health check on HTTP `/` endpoint in orchestrator (k8s liveness probe)

### Producers

- [ ] Always `await` and check send results; log and retry on error
- [ ] Use key-based partitioning when per-entity ordering is required
- [ ] Pre-create streams and topics at service startup (idempotent)
- [ ] Use batch send for high-throughput paths
- [ ] Instrument with metrics: messages sent, send latency, error rate

### Consumers

- [ ] `auto_commit = false` for at-least-once guarantees
- [ ] Commit offsets only after successful downstream write/processing
- [ ] Implement idempotent message handlers (duplicate delivery on rebalance)
- [ ] Handle empty poll response with backoff sleep (avoid busy-loop)
- [ ] Join consumer group at startup; handle rebalance events
- [ ] Instrument with metrics: messages consumed, processing latency, lag (poll offset vs latest offset)
- [ ] Graceful shutdown: leave consumer group before process exit to trigger immediate rebalance

### Observability

```bash
# Topic stats (message count, bytes, per-partition detail)
iggy topic stats --stream orders order-events

# Consumer group lag per partition
iggy consumer-group get --stream orders --topic order-events processors

# Server-level metrics (if HTTP enabled)
curl http://localhost:3000/stats
```

Expose key metrics to Prometheus via `/metrics` endpoint (check current Iggy release for availability) or scrape HTTP stats endpoint and parse JSON.

---

## Message Schema Design

```rust
// Use a versioned envelope — Iggy has no native schema registry
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct EventEnvelope {
    schema_version: u8,        // bump on breaking changes
    event_type: String,        // "order.created", "order.cancelled"
    aggregate_id: String,      // used as partition key
    timestamp_ms: i64,
    payload: serde_json::Value,
}

// Producer
let envelope = EventEnvelope {
    schema_version: 1,
    event_type: "order.created".into(),
    aggregate_id: order.id.clone(),
    timestamp_ms: chrono::Utc::now().timestamp_millis(),
    payload: serde_json::to_value(&order)?,
};
let payload = serde_json::to_vec(&envelope)?;
let msg = Message::new(None, payload.into(), None)?;

// Consumer — tolerant reader pattern
let envelope: EventEnvelope = serde_json::from_slice(&msg.payload)?;
match envelope.schema_version {
    1 => handle_v1(&envelope),
    v => tracing::warn!("unknown schema version {v} — skipping"),
}
```
