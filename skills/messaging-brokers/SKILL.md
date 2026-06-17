---
name: messaging-brokers
description: Messaging broker expertise across RabbitMQ, Kafka, AWS SQS/SNS, Azure Service Bus, and NATS — primitives, selection criteria, patterns, outbox, CDC, and anti-patterns. Trigger on broker selection questions, queue/topic design, reliability patterns, outbox implementation, or broker topology reviews.
---

# Messaging Brokers

You are an expert in distributed messaging systems. Apply production-grade reasoning: match the broker to the use case, never use dual-write without a compensation strategy, and always design for idempotent consumers.

---

## Message Broker Primitives

```
QUEUE (point-to-point)
┌──────────┐         ┌──────────────┐         ┌──────────┐
│ Producer │────────►│    Queue     │────────►│Consumer A│
└──────────┘         └──────────────┘  (one   └──────────┘
                                        consumer
                                        gets msg)

PUB/SUB (broadcast)
                     ┌──────────────┐────────►│Consumer A│
┌──────────┐         │              │         └──────────┘
│ Producer │────────►│    Topic     │────────►│Consumer B│
└──────────┘         │              │         └──────────┘
                     └──────────────┘────────►│Consumer C│
                                              └──────────┘

PUSH DELIVERY: broker sends to consumer
  + Low latency
  - Consumer must handle rate; may be overwhelmed

PULL DELIVERY: consumer fetches from broker
  + Consumer controls rate (natural backpressure)
  - Polling overhead; requires good fetch tuning

Kafka: pull
RabbitMQ: push (with prefetch as backpressure)
SQS: pull
Service Bus: push + pull
NATS: push (JetStream: pull available)
```

---

## RabbitMQ

### Exchange Types

```
DIRECT EXCHANGE — route by exact routing key
┌──────────┐  routingKey=   ┌──────────────┐  binding=   ┌──────────┐
│ Producer │──"order.new"──►│Direct Exch.  │──"order.new"►│ Queue A  │
└──────────┘                └──────────────┘             └──────────┘

FANOUT EXCHANGE — broadcast to all bound queues
┌──────────┐               ┌──────────────┐────────────►│ Queue A  │
│ Producer │──────────────►│ Fanout Exch. │────────────►│ Queue B  │
└──────────┘               └──────────────┘────────────►│ Queue C  │

TOPIC EXCHANGE — route by pattern matching
  Routing keys:  "order.placed.US", "order.placed.EU", "order.shipped.US"
  Binding patterns:
    "order.placed.*"  → matches US and EU placed
    "order.*.US"      → matches placed.US and shipped.US
    "#"               → matches everything

HEADERS EXCHANGE — route by message headers (not routing key)
  Message headers: {type: "order", priority: "high", region: "EU"}
  Binding args:    {x-match: "all", type: "order", region: "EU"}
  → Matches only if ALL header conditions met (x-match=all)
  → x-match=any: matches if ANY condition met
```

### Queue Types and Features

```
CLASSIC QUEUE (legacy)
  - Stored in Mnesia (per-node, not HA by default)
  - Mirrored queues: deprecated, use quorum queues instead

QUORUM QUEUE (production standard since 3.8)
  - Raft-based replication across 3+ nodes
  - Durability: survives node failures without data loss
  - Tradeoffs: higher latency than classic, no priority queues
  
STREAM (since 3.9) — log-based, like Kafka
  - Retention-based (time/size) vs ack-based deletion
  - Multiple consumers at different offsets
  - Use when replay is needed
```

```python
# Python — Quorum queue declaration
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare quorum queue
channel.queue_declare(
    queue='orders',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',          # quorum queue
        'x-delivery-limit': 5,             # max redelivery attempts before DLX
        'x-dead-letter-exchange': 'orders.dlx',
        'x-dead-letter-routing-key': 'orders.dead',
    }
)

# Set prefetch to limit in-flight messages (backpressure)
channel.basic_qos(prefetch_count=10)

def on_message(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except NonRetryableError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # → DLX
    except RetryableError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)   # redeliver

channel.basic_consume('orders', on_message_callback=on_message)
channel.start_consuming()
```

```java
// Java — Spring AMQP with DLX
@Configuration
public class RabbitConfig {

    @Bean
    public Queue ordersQueue() {
        return QueueBuilder.durable("orders")
            .withArgument("x-queue-type", "quorum")
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "orders.dead")
            .withArgument("x-message-ttl", 3_600_000)  // 1 hour TTL
            .build();
    }

    @Bean
    public DirectExchange ordersExchange() {
        return new DirectExchange("orders");
    }

    @Bean
    public Binding ordersBinding(Queue ordersQueue, DirectExchange ordersExchange) {
        return BindingBuilder.bind(ordersQueue).to(ordersExchange).with("order.placed");
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("orders.dead").build();
    }

    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange("orders.dlx");
    }

    @Bean
    public Binding deadLetterBinding(Queue deadLetterQueue, DirectExchange deadLetterExchange) {
        return BindingBuilder.bind(deadLetterQueue).to(deadLetterExchange).with("orders.dead");
    }
}

@RabbitListener(queues = "orders")
public void consume(Order order, Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        processOrder(order);
        channel.basicAck(tag, false);
    } catch (Exception e) {
        channel.basicNack(tag, false, false);  // send to DLX
    }
}
```

```kotlin
// Kotlin — RabbitMQ publisher with confirms
@Service
class OrderPublisher(private val rabbitTemplate: RabbitTemplate) {

    fun publish(order: Order) {
        rabbitTemplate.invoke { operations ->
            operations.convertAndSend("orders", "order.placed", order)
            if (!operations.waitForConfirms(5000)) {
                throw MessagingException("Publisher confirm timed out for order ${order.id}")
            }
        }
    }
}
```

### Dead Letter Exchange Topology

```
Normal Flow:
Producer ──► [orders exchange] ──► [orders queue] ──► Consumer
                                         │
                               (nack, reject, TTL expired,
                                max delivery exceeded)
                                         │
                                         ▼
DLX Flow:                     [orders.dlx exchange] ──► [orders.dead queue]
                                                               │
                                                    [DLT Consumer / Alerting]
```

---

## AWS SQS / SNS

### Queue Types

```
STANDARD QUEUE
  - At-least-once delivery (rare duplicates possible)
  - Best-effort ordering (not guaranteed)
  - Nearly unlimited throughput
  - Use when: high throughput, idempotent consumers, order not critical

FIFO QUEUE
  - Exactly-once processing (deduplication window: 5 minutes)
  - Strict ordering within Message Group ID
  - Max 3,000 TPS with batching (300 without)
  - Use when: financial transactions, order processing, inventory updates

VISIBILITY TIMEOUT:
  Consumer receives message → message hidden for N seconds
  If consumer doesn't delete within N seconds → message reappears
  → Set to max(processing_time) + 20% buffer
  → Extend dynamically for long-running jobs (ChangeMessageVisibility)
```

### SNS Fan-out Pattern

```
SNS Fan-out to multiple SQS queues:
┌──────────┐          ┌────────────────┐
│ Producer │─────────►│   SNS Topic    │
│ (publish │          │  "OrderEvents" │
│  once)   │          └────────────────┘
└──────────┘                 │ │ │
                    ┌────────┘ │ └──────────┐
                    ▼          ▼            ▼
              ┌──────────┐ ┌────────┐ ┌──────────┐
              │SQS Queue │ │SQS     │ │SQS Queue │
              │ Email    │ │Fulfil- │ │ Analytics│
              │ Service  │ │ment    │ │          │
              └──────────┘ └────────┘ └──────────┘

Subscription filter policies:
  Email service:    {"eventType": ["OrderPlaced"]}
  Fulfillment:      {"eventType": ["OrderPlaced", "OrderCancelled"]}
  Analytics:        (no filter — all events)
```

```python
# Python — SQS producer with deduplication
import boto3
import hashlib, json

sqs = boto3.client('sqs', region_name='us-east-1')

def publish_order(order: dict, queue_url: str):
    message_body = json.dumps(order)
    # FIFO: deduplication ID prevents duplicates within 5-min window
    dedup_id = hashlib.md5(f"{order['orderId']}:{order['version']}".encode()).hexdigest()
    
    response = sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=message_body,
        MessageGroupId=order['userId'],     # FIFO ordering per user
        MessageDeduplicationId=dedup_id,
        MessageAttributes={
            'eventType': {'DataType': 'String', 'StringValue': 'OrderPlaced'},
            'correlationId': {'DataType': 'String', 'StringValue': order['correlationId']},
        }
    )
    return response['MessageId']

def consume_orders(queue_url: str):
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,          # long polling — reduces empty responses
            VisibilityTimeout=60,
            AttributeNames=['All'],
            MessageAttributeNames=['All'],
        )
        for msg in response.get('Messages', []):
            try:
                process(json.loads(msg['Body']))
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=msg['ReceiptHandle']
                )
            except Exception:
                # Message becomes visible again after VisibilityTimeout
                pass  # or extend visibility timeout for long processing
```

```java
// Java — SQS consumer with DLQ
@SqsListener(value = "orders-queue", deletionPolicy = SqsMessageDeletionPolicy.ON_SUCCESS)
public void handleOrder(@Payload Order order,
                        @Header("ApproximateReceiveCount") String receiveCount) {
    if (Integer.parseInt(receiveCount) > 3) {
        log.warn("Message received {} times, sending to DLQ manually", receiveCount);
        // SQS automatically moves to DLQ after maxReceiveCount; just throw
    }
    processOrder(order);  // exception → automatic redelivery up to maxReceiveCount
}
```

---

## Azure Service Bus

### Topology

```
QUEUE (point-to-point with sessions)
┌──────────┐    ┌────────────────────────────┐    ┌──────────┐
│ Producer │───►│  Queue                     │───►│Consumer 1│
└──────────┘    │  [S1: msg1, msg2]          │    └──────────┘
                │  [S2: msg3, msg4]  sessions│
                │  [S3: msg5]                │    ┌──────────┐
                └────────────────────────────┘───►│Consumer 2│
                                                   └──────────┘
TOPIC + SUBSCRIPTIONS (pub/sub with filtering)
                         ┌────────────────────────┐
                         │  Topic: "orders"        │
┌──────────┐             ├────────────────────────┤
│ Producer │────────────►│  Sub: "fulfillment"    │──► Fulfillment consumer
└──────────┘             │  filter: amount > 100  │
                         ├────────────────────────┤
                         │  Sub: "notifications"  │──► Notification consumer
                         │  filter: (all)         │
                         ├────────────────────────┤
                         │  Sub: "fraud-check"    │──► Fraud consumer
                         │  filter: region = 'EU' │
                         └────────────────────────┘
```

### Sessions for Ordering

```
Service Bus Sessions = ordered processing per entity
  SessionId = orderId or userId
  One consumer holds the session lock → processes all messages for that entity in order
  Other consumers process different sessions in parallel

Use sessions when:
  - Order matters per customer/entity
  - State machine processing (events must be ordered)
  - Exactly-one-consumer-per-entity guarantee needed
```

```java
// Java — Service Bus with sessions
ServiceBusClientBuilder builder = new ServiceBusClientBuilder()
    .connectionString(connectionString);

ServiceBusSessionReceiverClient sessionReceiver = builder
    .sessionReceiver()
    .queueName("orders")
    .buildClient();

// Accept next available session
ServiceBusReceiverClient session = sessionReceiver.acceptNextSession();
String sessionId = session.getSessionId();

for (ServiceBusReceivedMessage msg : session.receiveMessages(100)) {
    try {
        processOrderEvent(msg, sessionId);
        session.complete(msg);
    } catch (Exception e) {
        session.deadLetter(msg, new DeadLetterOptions()
            .setDeadLetterReason("ProcessingFailed")
            .setDeadLetterErrorDescription(e.getMessage()));
    }
}
```

```kotlin
// Kotlin — Service Bus sender with correlation
@Service
class OrderBusPublisher(private val senderClient: ServiceBusSenderClient) {

    fun publish(order: Order, correlationId: String) {
        val message = ServiceBusMessage(objectMapper.writeValueAsBytes(order))
            .setMessageId(order.id)
            .setCorrelationId(correlationId)
            .setSessionId(order.userId)          // session-based ordering per user
            .setTimeToLive(Duration.ofHours(24))
            .setApplicationProperty("eventType", "OrderPlaced")
            .setApplicationProperty("region", order.region)

        senderClient.sendMessage(message)
    }
}
```

### Lock Renewal for Long Processing

```python
# Python — Azure Service Bus with lock renewal
from azure.servicebus import ServiceBusClient
import threading

with ServiceBusClient.from_connection_string(conn_str) as client:
    with client.get_queue_receiver("orders") as receiver:
        for msg in receiver:
            # Start background thread to renew lock every 30s
            stop_renewal = threading.Event()
            def renew():
                while not stop_renewal.is_set():
                    receiver.renew_message_lock(msg)
                    stop_renewal.wait(30)
            
            renewal_thread = threading.Thread(target=renew, daemon=True)
            renewal_thread.start()
            
            try:
                long_running_process(msg)
                receiver.complete_message(msg)
            except Exception as e:
                receiver.dead_letter_message(msg, reason=str(e))
            finally:
                stop_renewal.set()
```

---

## NATS

### Subject Hierarchy and Wildcards

```
Subject naming (dot-separated hierarchy):
  orders.placed.US
  orders.placed.EU
  orders.shipped.US
  
Wildcards:
  orders.placed.*      → matches orders.placed.US and orders.placed.EU
  orders.>            → matches everything under orders.*
  >                   → matches all subjects

NATS Core (fire-and-forget, no persistence):
  Best for: low-latency internal service mesh, ephemeral notifications
  Latency: sub-millisecond
  Delivery: at-most-once (subscribers must be online)

JetStream (persistent, replay, consumer groups):
  Replaces NATS Streaming (deprecated)
  Delivery: at-least-once or exactly-once (with Nats-Msg-Id dedup)
  Replay: yes (by sequence or time)
  Pull consumers: explicit demand, natural backpressure
  Push consumers: server-initiated, fast but no built-in backpressure
```

```python
# Python — NATS JetStream
import asyncio
import nats

async def main():
    nc = await nats.connect("nats://localhost:4222")
    js = nc.jetstream()

    # Create stream
    await js.add_stream(name="ORDERS", subjects=["orders.>"])

    # Publish with deduplication
    ack = await js.publish(
        "orders.placed.US",
        b'{"orderId": "123"}',
        headers={"Nats-Msg-Id": "order-123-v1"}  # dedup key
    )
    print(f"Published to stream seq={ack.seq}")

    # Pull consumer
    psub = await js.pull_subscribe("orders.>", "order-processor")
    msgs = await psub.fetch(10, timeout=5)
    for msg in msgs:
        process(msg.data)
        await msg.ack()

asyncio.run(main())
```

---

## Broker Selection Guide

```
┌─────────────────┬──────────────┬─────────┬──────────┬────────────┬────────┐
│  Requirement    │  RabbitMQ    │  Kafka  │  SQS/SNS │  Svc Bus   │  NATS  │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Throughput      │ 50k msg/s    │ 1M+ /s  │ 3k/s     │ 10k/s      │ 10M/s  │
│                 │ per node     │         │ (FIFO)   │            │        │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Latency         │ <5ms         │ 5-15ms  │ 100ms+   │ 10-50ms    │ <1ms   │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Message replay  │ No           │ Yes     │ No       │ No         │ Yes    │
│                 │ (Streams: Y) │         │          │            │(JS: Y) │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Strict ordering │ Per queue    │ Per     │ FIFO     │ Sessions   │ Per    │
│                 │              │ partition│ queues  │            │subject │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Flexible routing│ Excellent    │ Limited │ Limited  │ Good       │ Good   │
│ (topic/headers) │ (exchanges)  │         │ (filter) │ (filter)   │ (wild) │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Managed service │ CloudAMQP    │ MSK,    │ Native   │ Native     │ Synadia│
│                 │ AmazonMQ     │ Confluent│         │            │ Cloud  │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Protocol        │ AMQP 0-9-1   │ Binary  │ HTTP/SQS │ AMQP 1.0  │ NATS   │
│                 │ AMQP 1.0     │         │          │            │        │
├─────────────────┼──────────────┼─────────┼──────────┼────────────┼────────┤
│ Best for        │ Complex      │ Event   │ AWS-     │ Azure-     │ Cloud- │
│                 │ routing,     │ stream- │ native   │ native,    │ native,│
│                 │ RPC patterns │ ing,    │ simple   │ ordering   │ IoT,   │
│                 │              │ audit   │ workload │ per entity │ edge   │
└─────────────────┴──────────────┴─────────┴──────────┴────────────┴────────┘
```

---

## Message Patterns

### Competing Consumers (Work Queue)

```
               ┌──────────────────────┐
               │       Queue          │
Producer ─────►│ [M1][M2][M3][M4][M5]│
               └──────────────────────┘
                    │     │     │
                    ▼     ▼     ▼
                  [C1]  [C2]  [C3]   ← each message consumed by ONE consumer
                  
Scale consumers horizontally for throughput.
Each consumer processes different messages.
Natural load distribution.
```

### Request-Reply (RPC over Messaging)

```
                           ┌───────────────┐
Requester ──► [request]───►│ Request Queue │──► Worker
                           └───────────────┘        │
                                                    reply
                                                     │
Requester ◄── [response] ◄── [Reply Queue] ◄─────────┘
(correlationId matches request)

correlationId: UUID generated by requester
replyTo:       name of requester's reply queue (exclusive, auto-delete)

Use cases: synchronous-feeling calls over async infrastructure
Pitfall: requester must handle timeout (worker might be dead)
```

```python
# Python — RPC pattern with RabbitMQ
import uuid, pika

class OrderServiceClient:
    def __init__(self):
        self.connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
        self.channel = self.connection.channel()
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue
        self.responses = {}
        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self._on_response,
            auto_ack=True
        )

    def _on_response(self, ch, method, props, body):
        self.responses[props.correlation_id] = body

    def get_order(self, order_id: str, timeout: float = 10.0) -> dict:
        corr_id = str(uuid.uuid4())
        self.channel.basic_publish(
            exchange='',
            routing_key='order.rpc',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=corr_id,
            ),
            body=order_id.encode()
        )
        import time
        deadline = time.time() + timeout
        while corr_id not in self.responses:
            self.connection.process_data_events(time_limit=0.1)
            if time.time() > deadline:
                raise TimeoutError(f"No reply for order {order_id} within {timeout}s")
        return json.loads(self.responses.pop(corr_id))
```

---

## Poisonous Messages: Retry + DLQ Strategy

```
EXPONENTIAL BACKOFF RETRY FLOW:

Message arrives ──► Process ──[fail]──► Retry 1 (delay: 1s)
                                             │
                                        [fail]──► Retry 2 (delay: 2s)
                                                       │
                                                  [fail]──► Retry 3 (delay: 4s)
                                                                  │
                                                             [fail]──► DLQ
                                                             
DLQ handling:
  1. Alert ops team (PagerDuty, Slack)
  2. Store with full context (original topic, error, stack trace, timestamp)
  3. Provide replay tool for manual reprocessing after fix
  4. Never silently drop messages

IDEMPOTENCY KEY: always include in message
  - Allows safe retry without duplicate side effects
  - Store processed keys in Redis/DB with TTL
  - Check before processing: "have I seen this key?"
```

---

## Outbox Pattern

```
PROBLEM (dual write):
  ┌──────────────────────────────────────────────────────────┐
  │  Service                                                  │
  │  1. UPDATE orders SET status='PLACED' WHERE id=123  ──┐  │
  │  2. PUBLISH OrderPlaced to broker                  ──┐ │  │
  │                                                      │ │  │
  │  What if step 2 fails after step 1 commits?          │ │  │
  │  → Database updated, event never published           │ │  │
  │  → Inconsistent state between DB and downstream       │ │  │
  └──────────────────────────────────────────────────────────┘

OUTBOX PATTERN (atomic write):
  ┌──────────────────────────────────────────────────────────┐
  │  Transaction (atomic)                                    │
  │  1. UPDATE orders SET status='PLACED' WHERE id=123       │
  │  2. INSERT INTO outbox                                   │
  │     (id, topic, key, payload, created_at, published)     │
  │     VALUES (uuid, 'orders', '123', '{...}', NOW(), false)│
  └──────────────────────────────────────────────────────────┘
            │
            │  (separate process, polling or CDC)
            ▼
  ┌──────────────────┐     ┌──────────────┐
  │  Outbox Poller   │────►│    Broker    │
  │  (or Debezium)   │     └──────────────┘
  │  SELECT * FROM   │
  │  outbox WHERE    │
  │  published=false │
  └──────────────────┘

Guaranteed: if DB commit succeeded, event WILL be published (eventual)
```

```java
// Java — Outbox table insert within transaction
@Transactional
public Order placeOrder(PlaceOrderCommand cmd) {
    Order order = orderRepository.save(new Order(cmd));
    
    OutboxEvent event = OutboxEvent.builder()
        .id(UUID.randomUUID())
        .aggregateType("Order")
        .aggregateId(order.getId())
        .topic("orders")
        .eventType("OrderPlaced")
        .payload(objectMapper.writeValueAsString(new OrderPlacedEvent(order)))
        .createdAt(Instant.now())
        .published(false)
        .build();
    
    outboxRepository.save(event);   // same transaction as order save
    return order;
}

// Separate scheduled poller
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> unpublished = outboxRepository
        .findByPublishedFalseOrderByCreatedAtAsc(PageRequest.of(0, 100));
    
    for (OutboxEvent event : unpublished) {
        kafkaTemplate.send(event.getTopic(), event.getAggregateId(), event.getPayload())
            .get(5, TimeUnit.SECONDS);  // wait for ack
        event.setPublished(true);
        outboxRepository.save(event);
    }
}
```

### Transactional Outbox vs CDC vs Dual Write

```
┌──────────────────┬──────────────────────────┬─────────────────────────────┐
│  Approach        │  Pros                    │  Cons                       │
├──────────────────┼──────────────────────────┼─────────────────────────────┤
│ Outbox + poller  │ Simple, no extra infra   │ Polling delay (1s typical)  │
│                  │ Works with any broker    │ Poller is single point of   │
│                  │                          │ failure (use leader election)│
├──────────────────┼──────────────────────────┼─────────────────────────────┤
│ Outbox + CDC     │ Near real-time (<100ms)  │ Requires Debezium + Kafka   │
│ (Debezium)       │ No polling overhead      │ Operational complexity      │
│                  │ Exactly reads WAL        │ Schema changes need care    │
├──────────────────┼──────────────────────────┼─────────────────────────────┤
│ Dual write       │ Simple code              │ NOT safe — data loss risk   │
│ (NEVER in prod)  │                          │ No atomicity guarantee      │
│                  │                          │ Always use outbox instead   │
└──────────────────┴──────────────────────────┴─────────────────────────────┘
```

---

## Broker Topology Anti-Patterns

### Anti-Pattern 1: Direct Queue-to-Queue Coupling

```
BAD:
Service A ──► Queue "service-b-input" ──► Service B
                     ▲
              A knows B's internal queue name
              A is coupled to B's deployment topology

GOOD:
Service A ──► Topic/Exchange "order.placed" ──► [Subscription] ──► Service B
              (logical event name)               (B's private queue)
              A knows nothing about B's infrastructure
```

### Anti-Pattern 2: Queue Sprawl

```
BAD (one queue per integration pair):
  order-svc → fulfillment-queue
  order-svc → notification-queue
  order-svc → analytics-queue
  order-svc → fraud-queue
  ... (N queues, each tightly coupled)

GOOD (event topic with subscriptions):
  order-svc → orders.placed topic
               └── fulfillment-sub ──► Fulfillment
               └── notification-sub ──► Notifications
               └── analytics-sub ──► Analytics
               └── fraud-sub ──► Fraud Detection
```

### Anti-Pattern 3: Shared Queue Between Teams

```
BAD:
Team A ──► [shared-orders-queue] ◄── Team B
                                ◄── Team C
  Problem: teams compete for messages
           one team's consumer bug starves others
           no independent scaling

GOOD:
              [orders-topic]
               │        │
[Team A sub]──►│  [Team B sub]──►│  [Team C sub]──►
Independent queues per team namespace.
```

### Anti-Pattern 4: Unbounded Retry Without DLQ

```
BAD:
while True:
    msg = queue.receive()
    try:
        process(msg)
    except:
        queue.nack(msg, requeue=True)  # infinite loop on poison pill

GOOD:
  - Max retry count (3-5)
  - Exponential backoff between retries
  - DLQ after max retries
  - Alert + monitoring on DLQ depth
```

### Anti-Pattern 5: Large Messages in Broker

```
BAD: storing 5 MB PDFs in RabbitMQ/SQS messages
  - Broker memory exhaustion
  - Network saturation
  - SQS max: 256 KB; RabbitMQ: defaults 128 MB but causes OOM

GOOD (Claim Check pattern):
  1. Store payload in S3/Blob storage
  2. Publish lightweight reference message: {"s3Key": "orders/123/invoice.pdf"}
  3. Consumer fetches from S3
```

---

## Production Checklist

### Broker Deployment
- [ ] Minimum 3 nodes for RabbitMQ (quorum queues need quorum)
- [ ] Minimum 3 brokers for Kafka (replication factor 3)
- [ ] SQS/Service Bus: managed HA (no action needed)
- [ ] Network partition tolerance: configure accordingly per CAP theorem

### Message Design
- [ ] Every message has a unique ID (UUID) for deduplication
- [ ] Every message has a correlationId for distributed tracing
- [ ] Payload includes event timestamp (not broker timestamp)
- [ ] Schema versioned and registered in schema registry (if Kafka)
- [ ] Messages are self-describing (include event type in payload or header)

### Reliability
- [ ] DLQ configured for every queue/subscription
- [ ] DLQ depth alert configured (threshold: >0 for critical, >N for others)
- [ ] Retry policy: exponential backoff with jitter, max 5 retries
- [ ] Consumer is idempotent (dedup by message ID)
- [ ] Outbox pattern used for reliable publishing (not dual write)
- [ ] Publisher confirms / acks enabled on producer

### Operations
- [ ] Message processing time SLA defined and monitored
- [ ] Queue/topic depth monitored (alert on growing backlog)
- [ ] Consumer lag alerting (for Kafka)
- [ ] DLQ replay process documented and tested
- [ ] Message TTL set (no immortal messages)
- [ ] Load test at 2x expected peak (with realistic message sizes)
