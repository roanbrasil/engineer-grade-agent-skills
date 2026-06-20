---
name: nats-jetstream
description: Use when working with NATS messaging or JetStream persistence — pub/sub, streams, consumers, KV store, object store, clustering, or leaf nodes
---

# NATS and JetStream Expert

## What NATS Is

NATS is a lightweight, cloud-native messaging system written in Go. Single binary (~20 MB), no external dependencies, sub-millisecond latency.

```
Architecture layers:

  Core NATS          JetStream
  ──────────         ──────────────────────────────
  Pub/Sub            Streams (persistence)
  Request/Reply      Consumers (durable cursors)
  Queue Groups       Key-Value Store
                     Object Store
                     Exactly-once delivery
```

| Layer | Delivery | Persistence | Use when |
|-------|----------|-------------|----------|
| Core NATS | At-most-once | No | High-frequency telemetry, fire-and-forget |
| JetStream | At-least-once or exactly-once | Yes (file or memory) | Durable queues, event sourcing, auditable logs |

---

## Subject Namespace

Subjects are UTF-8 dot-delimited strings. Design them like a resource path.

```
orders.created               simple subject
orders.>                     wildcard: all tokens under orders (any depth)
orders.*                     single-token wildcard: orders.created, orders.updated
                             but NOT orders.v1.created

_INBOX.AbCdEf               auto-generated inbox for request/reply
$JS.API.>                   JetStream API subjects (reserved)

Examples:
  sensors.us-east.zone1.temperature
  api.v2.users.login
  payments.>                 matches payments.created, payments.refunded, payments.v2.disputed
```

Rule: use `>` in stream subject filters; use `*` in subscriptions for single-level matching.

---

## Core NATS — Pub/Sub

### Go

```go
package main

import (
    "fmt"
    "time"
    "github.com/nats-io/nats.go"
)

func main() {
    nc, err := nats.Connect(nats.DefaultURL)
    if err != nil {
        panic(err)
    }
    defer nc.Drain() // flush + close

    // Subscribe
    sub, err := nc.Subscribe("orders.created", func(msg *nats.Msg) {
        fmt.Printf("received on %s: %s\n", msg.Subject, string(msg.Data))
    })
    if err != nil {
        panic(err)
    }
    defer sub.Unsubscribe()

    // Queue group — only one subscriber in the group receives each message
    nc.QueueSubscribe("orders.created", "processors", func(msg *nats.Msg) {
        fmt.Printf("queue worker got: %s\n", string(msg.Data))
    })

    // Publish
    if err := nc.Publish("orders.created", []byte(`{"id":"123","amount":99.99}`)); err != nil {
        panic(err)
    }
    nc.Flush()

    time.Sleep(100 * time.Millisecond)
}
```

### Python (nats.py)

```python
import asyncio
import nats

async def main():
    nc = await nats.connect("nats://localhost:4222")

    async def handler(msg):
        print(f"subject={msg.subject} data={msg.data.decode()}")

    # Simple subscribe
    sub = await nc.subscribe("orders.created", cb=handler)

    # Queue group
    qsub = await nc.subscribe("orders.created", queue="processors", cb=handler)

    await nc.publish("orders.created", b'{"id":"123","amount":99.99}')
    await asyncio.sleep(0.1)

    await sub.unsubscribe()
    await nc.drain()

asyncio.run(main())
```

### Java (nats.java)

```java
import io.nats.client.*;
import java.nio.charset.StandardCharsets;
import java.time.Duration;

Connection nc = Nats.connect("nats://localhost:4222");

// Subscribe with dispatcher
Dispatcher d = nc.createDispatcher(msg -> {
    String body = new String(msg.getData(), StandardCharsets.UTF_8);
    System.out.printf("subject=%s body=%s%n", msg.getSubject(), body);
});
d.subscribe("orders.created");

// Queue group
d.subscribe("orders.created", "processors");

// Publish
nc.publish("orders.created", "{\"id\":\"123\"}".getBytes(StandardCharsets.UTF_8));
nc.flush(Duration.ofSeconds(1));
```

### Rust (async-nats)

```rust
use async_nats;
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::connect("nats://localhost:4222").await?;

    // Subscribe
    let mut subscriber = client.subscribe("orders.created").await?;

    // Queue group
    let mut queue_sub = client
        .queue_subscribe("orders.created", "processors".to_string())
        .await?;

    // Publish
    client
        .publish("orders.created", r#"{"id":"123","amount":99.99}"#.into())
        .await?;

    // Receive one message
    if let Some(msg) = subscriber.next().await {
        println!("subject={} data={:?}", msg.subject, msg.payload);
    }

    Ok(())
}
```

---

## Request/Reply Pattern

NATS has native request/reply: the requester auto-creates an inbox subject; the responder replies to it.

```
Requester                    NATS Server              Responder
   |                             |                        |
   |-- Publish("api.users.get"   |                        |
   |   Reply-To: _INBOX.abc) --> |                        |
   |                             |-- deliver to sub ------>|
   |                             |                        |-- Respond() -->|
   |                             |<-- Publish(_INBOX.abc) |
   |<-- deliver reply ---------- |
```

### Go

```go
// Responder
nc.Subscribe("api.users.get", func(msg *nats.Msg) {
    var req struct{ ID string `json:"id"` }
    json.Unmarshal(msg.Data, &req)
    user := db.GetUser(req.ID)
    data, _ := json.Marshal(user)
    msg.Respond(data) // publishes to msg.Reply
})

// Requester (blocking, with timeout)
reply, err := nc.Request("api.users.get", []byte(`{"id":"123"}`), 2*time.Second)
if err != nil {
    // timeout or no responders
}
fmt.Printf("user: %s\n", string(reply.Data))
```

### Python

```python
async def responder(msg):
    user = get_user(msg.data)
    await msg.respond(user)

await nc.subscribe("api.users.get", cb=responder)

# Request (awaits reply)
reply = await nc.request("api.users.get", b'{"id":"123"}', timeout=2.0)
print(reply.data.decode())
```

---

## JetStream — Streams

A Stream is a named, durable log that captures messages on one or more subjects.

```
Stream "ORDERS"
  Subjects:  ["orders.>"]
  MaxAge:    24h
  Storage:   File
  Replicas:  3
  Retention: LimitsPolicy | WorkQueuePolicy | InterestPolicy

  seq=1  orders.created   {"id":"100"}
  seq=2  orders.updated   {"id":"100","status":"paid"}
  seq=3  orders.created   {"id":"101"}
  seq=4  orders.cancelled {"id":"100"}
  ...
```

### Retention Policies

| Policy | Behaviour |
|--------|-----------|
| `LimitsPolicy` (default) | Retain by MaxMsgs / MaxBytes / MaxAge |
| `WorkQueuePolicy` | Delete message after any consumer ACKs it |
| `InterestPolicy` | Delete message once ALL consumers have ACKed it |

### Create Stream — Go

```go
js, err := nc.JetStream()
if err != nil {
    panic(err)
}

_, err = js.AddStream(&nats.StreamConfig{
    Name:      "ORDERS",
    Subjects:  []string{"orders.>"},
    MaxAge:    24 * time.Hour,
    Storage:   nats.FileStorage,
    Replicas:  3,
    Retention: nats.LimitsPolicy,
    Discard:   nats.DiscardOld, // drop oldest when limits hit
})
```

### Publish with Acknowledgment — Go

```go
// Synchronous: blocks until server persists
ack, err := js.Publish("orders.created", []byte(`{"id":"123"}`))
if err != nil {
    // message NOT persisted; retry
    panic(err)
}
fmt.Printf("stream=%s seq=%d\n", ack.Stream, ack.Sequence)

// Async publish (higher throughput)
js.PublishAsync("orders.created", []byte(`{"id":"124"}`))
select {
case <-js.PublishAsyncComplete():
    fmt.Println("all async publishes confirmed")
case <-time.After(5 * time.Second):
    fmt.Println("timed out")
}
```

---

## JetStream — Consumers

A Consumer is a named, durable cursor into a stream. It tracks delivery position and redelivers unacked messages.

```
Consumer "processor" on Stream "ORDERS"
  DeliverPolicy:  All | New | ByStartSequence | ByStartTime | LastPerSubject
  AckPolicy:      Explicit | None | All
  FilterSubject:  "orders.created"
  MaxDeliver:     5          (retry limit; -1 = unlimited)
  AckWait:        30s        (redeliver if not acked within this window)
  MaxAckPending:  1000       (flow control)
  InactiveThreshold: 5m     (delete ephemeral consumer after this idle time)
```

### Push Consumer — Go

```go
sub, err := js.Subscribe(
    "orders.created",
    func(msg *nats.Msg) {
        if err := processOrder(msg.Data); err != nil {
            msg.Nak()        // redeliver immediately
            return
        }
        msg.Ack()
    },
    nats.Durable("processor"),
    nats.AckExplicit(),
    nats.MaxDeliver(5),
    nats.AckWait(30*time.Second),
)
defer sub.Drain()
```

### Pull Consumer — recommended for work queues

Pull consumers give the application control over fetch rate and batch size. Prefer over push for high-throughput or batch processing.

```go
sub, err := js.PullSubscribe(
    "orders.created",
    "processor",
    nats.BindStream("ORDERS"),
)
if err != nil {
    panic(err)
}
defer sub.Unsubscribe()

for {
    msgs, err := sub.Fetch(10, nats.MaxWait(5*time.Second))
    if err != nil && err != nats.ErrTimeout {
        log.Printf("fetch error: %v", err)
        continue
    }
    for _, msg := range msgs {
        if err := processOrder(msg.Data); err != nil {
            msg.NakWithDelay(10 * time.Second) // back-off retry
            continue
        }
        msg.Ack()
    }
}
```

### Pull Consumer — Python

```python
import asyncio
import nats
from nats.js.api import ConsumerConfig, DeliverPolicy, AckPolicy

async def main():
    nc = await nats.connect("nats://localhost:4222")
    js = nc.jetstream()

    # Create or bind consumer
    psub = await js.pull_subscribe(
        "orders.created",
        "processor",
        config=ConsumerConfig(
            deliver_policy=DeliverPolicy.ALL,
            ack_policy=AckPolicy.EXPLICIT,
            max_deliver=5,
            ack_wait=30,
        ),
    )

    while True:
        try:
            msgs = await psub.fetch(10, timeout=5)
            for msg in msgs:
                try:
                    process_order(msg.data)
                    await msg.ack()
                except Exception:
                    await msg.nak()
        except nats.errors.TimeoutError:
            pass

asyncio.run(main())
```

### Pull Consumer — Rust (async-nats)

```rust
use async_nats::jetstream::{self, consumer::{pull, Config}};
use futures::StreamExt;

let client = async_nats::connect("nats://localhost:4222").await?;
let jetstream = jetstream::new(client);

let stream = jetstream.get_stream("ORDERS").await?;
let consumer: pull::Consumer = stream
    .get_or_create_consumer(
        "processor",
        Config {
            durable_name: Some("processor".to_string()),
            filter_subject: "orders.created".to_string(),
            max_deliver: 5,
            ack_wait: std::time::Duration::from_secs(30),
            ..Default::default()
        },
    )
    .await?;

let mut messages = consumer.fetch().max_messages(10).messages().await?;

while let Some(message) = messages.next().await {
    let message = message?;
    match process_order(&message.payload) {
        Ok(_) => message.ack().await?,
        Err(_) => message.ack_with(jetstream::AckKind::Nak(None)).await?,
    }
}
```

---

## Key-Value Store

JetStream KV is a first-class key-value store built on a stream. Supports TTL, history, watch, and CAS.

```go
kv, err := js.CreateKeyValue(&nats.KeyValueConfig{
    Bucket:  "config",
    TTL:     1 * time.Hour,
    History: 5,           // keep last 5 revisions per key
    Storage: nats.FileStorage,
    Replicas: 3,
})

// Write
rev, _ := kv.Put("feature.dark-mode", []byte("true"))

// Read
entry, _ := kv.Get("feature.dark-mode")
fmt.Printf("key=%s value=%s rev=%d\n", entry.Key(), entry.Value(), entry.Revision())

// Atomic update (CAS)
kv.Update("feature.dark-mode", []byte("false"), rev) // fails if rev mismatch

// Delete
kv.Delete("feature.dark-mode")
kv.Purge("feature.dark-mode") // remove all history

// Watch a key
watcher, _ := kv.Watch("feature.*")
defer watcher.Stop()
for entry := range watcher.Updates() {
    if entry == nil {
        break // initial values delivered
    }
    fmt.Printf("key=%s value=%s op=%s\n", entry.Key(), entry.Value(), entry.Operation())
}

// Watch all keys
watcher, _ = kv.WatchAll()
```

### Use Cases for KV Store

- Feature flags distributed to all service instances
- Config management with change notifications (watch)
- Distributed locking via CAS (Update with expected revision)
- Leader election: first writer wins on a key
- Service registry: each instance writes `services.{id}` with TTL; auto-expires on crash

---

## Object Store

Large binary blobs stored in JetStream. Chunked internally. Use for artifacts, ML models, config files.

```go
obs, err := js.CreateObjectStore(&nats.ObjectStoreConfig{
    Bucket:   "artifacts",
    Storage:  nats.FileStorage,
    Replicas: 3,
})

// Upload
obs.PutFile("report.pdf")

// Upload with metadata
obs.Put(&nats.ObjectMeta{Name: "model-v2.bin", Description: "trained weights"}, reader)

// Download
obs.GetFile("report.pdf", "local-copy.pdf")

// Get info
info, _ := obs.GetInfo("report.pdf")
fmt.Printf("size=%d chunks=%d\n", info.Size, info.Chunks)

// List
obs.List()  // returns []ObjectInfo

// Delete
obs.Delete("report.pdf")
```

---

## Clustering and Leaf Nodes

```
       NGS Global Supercluster (Synadia)
      ┌──────────────────────────────────┐
      │  US-EAST cluster   EU-WEST cluster│
      │  [A1] [A2] [A3]   [B1] [B2] [B3]│
      └──────┬────────────────────┬──────┘
             │  Gateway           │
      ┌──────┴──────┐      ┌──────┴──────┐
      │ Leaf Node   │      │ Leaf Node   │
      │ (edge/IoT)  │      │ (branch DC) │
      └─────────────┘      └─────────────┘
```

**Cluster server config (nats-server.conf)**

```
cluster {
  name: us-east
  listen: 0.0.0.0:6222
  routes: [
    nats-route://nats-1:6222
    nats-route://nats-2:6222
    nats-route://nats-3:6222
  ]
}

jetstream {
  store_dir: /data/nats
  max_memory_store: 1GB
  max_file_store: 20GB
}
```

**JetStream stream with replication**

```go
js.AddStream(&nats.StreamConfig{
    Name:     "ORDERS",
    Subjects: []string{"orders.>"},
    Replicas: 3, // 3-node quorum; tolerates 1 node failure
    Storage:  nats.FileStorage,
})
```

**Leaf Node config**

```
leafnodes {
  remotes = [{
    url: "nats-leaf://hub.example.com:7422"
    credentials: "/etc/nats/leaf.creds"
  }]
}
```

Leaf nodes work offline; messages buffer locally and sync to hub when reconnected.

---

## NATS vs Kafka vs Redis Pub/Sub

| Dimension | NATS Core | JetStream | Kafka | Redis Pub/Sub |
|-----------|-----------|-----------|-------|---------------|
| Persistence | No | Yes | Yes | No |
| Delivery guarantee | At-most-once | At-least-once / exactly-once | At-least-once | At-most-once |
| Wildcard routing | Yes (`*`, `>`) | Yes | No | Pattern match |
| Request/Reply | Native | Native | DIY via topics | No |
| Key-Value | No | Yes | No | Native |
| Object Store | No | Yes | No | No |
| Consumer groups | Queue Groups | Consumers | Consumer Groups | No |
| Operational weight | Very light | Light | Heavy | Light |
| Multi-tenancy | Accounts | Accounts | ACLs | ACLs |
| Ordering | Per-subject | Per-subject | Per-partition | No |

**Choose NATS/JetStream when:**

- You need low-latency messaging without Kafka's JVM and ZooKeeper/KRaft overhead
- IoT or edge deployments with leaf nodes bridging to central cluster
- Request/reply service mesh lighter than gRPC for internal comms
- Persistent queue without Kafka operational complexity
- Config/feature flag distribution via built-in KV store

**Stick with Kafka when:**

- You need consumer replay across many independent teams long after the fact
- You have existing Kafka expertise and ecosystem tooling (Kafka Connect, ksqlDB)
- Partition-level ordering is a hard requirement across thousands of topics

---

## Security

### TLS and Credentials

```go
// TLS with client cert
opts := nats.Options{
    Url:         "tls://nats.example.com:4222",
    TLSConfig:   &tls.Config{InsecureSkipVerify: false},
    Secure:      true,
}

// NKey authentication
nc, _ := nats.Connect("nats://localhost:4222",
    nats.NkeyOptionFromSeed(nkeyPath),
)

// JWT + NKey credentials file
nc, _ := nats.Connect("nats://localhost:4222",
    nats.UserCredentials("/etc/nats/user.creds"),
)
```

### Account isolation (server config)

```
accounts {
  SVC_A: {
    users: [{user: svc-a, password: $SVC_A_PASS}]
    exports: [{stream: "orders.>"}]
    jetstream: enabled
  }
  SVC_B: {
    users: [{user: svc-b, password: $SVC_B_PASS}]
    imports: [{stream: {account: SVC_A, subject: "orders.>"}}]
  }
}
```

---

## Production Checklist

- [ ] Use `nc.Drain()` instead of `nc.Close()` — flushes pending messages before shutdown
- [ ] Set `MaxReconnects` and `ReconnectWait` on the connection options
- [ ] Use `nats.ErrorHandler` and `nats.DisconnectErrHandler` for observability
- [ ] JetStream streams: set `Replicas: 3` in production; never 1 for durable data
- [ ] Set `MaxDeliver` on consumers to prevent infinite redelivery loops
- [ ] Set `AckWait` longer than your processing SLA; shorter than your timeout budget
- [ ] Use pull consumers for batch workloads; push consumers for low-latency event handling
- [ ] Monitor `nats server report jetstream` and consumer pending/redelivery counts
- [ ] Use `Discard: nats.DiscardNew` in rate-limiting scenarios to back-pressure producers
- [ ] Authenticate with NKey or JWT+NKey; avoid plain username/password in production
- [ ] Isolate tenants with accounts; export/import only what is needed

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `nc.Close()` on shutdown | In-flight messages lost | Use `nc.Drain()` |
| Push consumer without `Durable` | Consumer lost on restart; redelivers from start | Always name push consumers |
| `MaxDeliver: -1` in production | Poison messages loop forever | Set a finite limit; send to dead-letter stream |
| Publishing without checking JetStream `ack` error | Silent message loss | Always check `err` from `js.Publish` |
| Single-replica stream | No fault tolerance | `Replicas: 3` for production |
| Huge `Fetch` batch with slow processing | `AckWait` expires before processing; redelivery storm | Tune batch size to processing rate * AckWait |
| Wildcards in stream subjects too broad | Stream captures unintended subjects | Be explicit: `orders.>` not `>` |
| Sharing one consumer across multiple app instances | Double-processing; each instance has its own cursor | Use separate consumers or pull-subscribe from one consumer |

---

## Observability

```bash
# Server stats
nats server info
nats server report jetstream

# Stream inspection
nats stream info ORDERS
nats stream ls

# Consumer inspection
nats consumer info ORDERS processor
nats consumer ls ORDERS

# Live message monitor
nats sub "orders.>"

# JetStream consumer pending count (alert if > threshold)
nats consumer report ORDERS
```

Expose metrics via Prometheus with [nats-server /varz and /connz endpoints](https://docs.nats.io/running-a-nats-service/nats_admin/monitoring) or the NATS Prometheus exporter.
