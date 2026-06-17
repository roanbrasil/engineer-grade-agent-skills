---
name: domain-events
description: Design and implement Domain Events — event structure, intra-process and inter-process dispatch, the Outbox Pattern, event versioning, and the connection to Event Sourcing — when decoupling aggregates or integrating across services.
---

# Domain Events

## Core Philosophy

A domain event represents something that happened in the domain — a fact, immutable and in the past. It is the primary mechanism for decoupling aggregates within a bounded context and for integrating between bounded contexts. Events are the seam at which the domain tells the rest of the system: "Something significant just occurred."

**Three rules:**
1. Domain events are named in past tense: `OrderPlaced`, not `PlaceOrder`; `PaymentFailed`, not `FailPayment`
2. Domain events are raised by aggregates *after* invariants are enforced and state is changed
3. Domain events are immutable — they describe history, which cannot be revised

---

## What a Domain Event Is

```
SOMETHING HAPPENED IN THE DOMAIN
             |
             | expressed as
             v
+--------------------------------+
|  OrderConfirmed                |  <-- past tense name
|                                |
|  eventId:    UUID              |  <-- globally unique ID
|  orderId:    OrderId           |  <-- which aggregate raised this
|  customerId: CustomerId        |  <-- relevant context
|  total:      Money             |  <-- payload: what happened
|  occurredAt: Instant           |  <-- when it happened (domain time)
|  version:    1                 |  <-- schema version for evolution
+--------------------------------+
             |
             | triggers
             v
+--------------------------------+
|  Zero or more handlers         |
|  - Send confirmation email     |
|  - Update inventory            |
|  - Notify warehouse            |
|  - Publish to billing context  |
+--------------------------------+
```

Domain events answer: "What happened?" — not "What should happen next?" The causality flows from the event, but the event itself is neutral.

---

## When to Raise Events

Events are raised from within aggregate methods, after the aggregate has enforced its invariants and changed its own state.

```kotlin
class Order private constructor(
    val id: OrderId,
    private var status: OrderStatus,
    private val _items: MutableList<OrderItem>,
    val customerId: CustomerId
) {
    // Internal event accumulator — events raised during this unit of work
    private val _events: MutableList<OrderDomainEvent> = mutableListOf()
    val events: List<OrderDomainEvent> get() = _events.toList()

    fun confirm() {
        // 1. Enforce invariants FIRST
        require(status == OrderStatus.DRAFT) {
            "Cannot confirm order ${id} in status ${status}"
        }
        require(_items.isNotEmpty()) {
            "Cannot confirm empty order ${id}"
        }
        require(total <= creditLimit) {
            "Order ${id} total ${total} exceeds credit limit ${creditLimit}"
        }

        // 2. Change state
        status = OrderStatus.CONFIRMED

        // 3. Raise event (after state is valid)
        _events += OrderConfirmed(
            orderId = id,
            customerId = customerId,
            total = total(),
            itemCount = _items.size,
            occurredAt = Instant.now()
        )
    }

    fun cancel(reason: CancellationReason) {
        require(status in listOf(OrderStatus.DRAFT, OrderStatus.CONFIRMED)) {
            "Cannot cancel order in status ${status}"
        }
        val previous = status
        status = OrderStatus.CANCELLED
        _events += OrderCancelled(
            orderId = id,
            customerId = customerId,
            previousStatus = previous,
            reason = reason,
            occurredAt = Instant.now()
        )
    }
}
```

**Why accumulate events in the aggregate?**
- The aggregate raises events, but doesn't dispatch them — separation of concerns
- The Application Service collects events after the operation and dispatches them
- This keeps the aggregate framework-agnostic

---

## Event Structure

Every domain event should carry enough information for a handler to act without loading the aggregate again.

```kotlin
// Base event — all domain events share this structure
sealed class DomainEvent {
    abstract val eventId: EventId          // Unique ID for this event occurrence
    abstract val occurredAt: Instant       // When it happened
    abstract val version: Int              // Schema version
}

// Rich, self-contained event
data class OrderConfirmed(
    override val eventId: EventId = EventId.generate(),
    override val occurredAt: Instant = Instant.now(),
    override val version: Int = 1,
    // Aggregate identity
    val orderId: OrderId,
    val customerId: CustomerId,
    // Payload — enough for handlers to act without re-loading the order
    val total: Money,
    val itemCount: Int,
    val shippingAddress: Address
) : DomainEvent()

data class PaymentDeclined(
    override val eventId: EventId = EventId.generate(),
    override val occurredAt: Instant = Instant.now(),
    override val version: Int = 1,
    val orderId: OrderId,
    val customerId: CustomerId,
    val attemptedAmount: Money,
    val declineCode: String,
    val gatewayRef: String?
) : DomainEvent()
```

**Payload design principles:**
- Include the aggregate ID so handlers can look up the aggregate if needed
- Include the key facts needed by the most common handlers (avoid N+1 loads)
- Do not embed mutable objects (embed Value Objects only, not Entities)
- Do not include infrastructure concerns (HTTP headers, session IDs)
- Metadata (correlation ID, causation ID, user ID) belongs in an envelope, not the event payload

### Event Envelope (for inter-process)

```kotlin
data class EventEnvelope(
    val eventId: String,
    val eventType: String,           // "order.confirmed"
    val aggregateId: String,
    val aggregateType: String,       // "Order"
    val occurredAt: String,          // ISO-8601
    val version: Int,
    val correlationId: String?,      // Trace the originating request
    val causationId: String?,        // The event that caused this one
    val payload: Map<String, Any?>   // Serialized event data
)
```

---

## Intra-Process Dispatch (Same Transaction)

Handlers that run in the same process and (usually) the same transaction as the event producer.

```
+------------------+       event      +-------------------+
|  Application     |----------------> |  Domain Event     |
|  Service         |                  |  Dispatcher       |
+------------------+                  +-------------------+
         |                                     |
         | 1. load aggregate                   | 2. find handlers
         | 3. call aggregate method            | 4. call each handler
         | 5. collect events                   |    synchronously
         | 6. save aggregate                   |
         | 7. dispatch events                  |
```

```kotlin
// Synchronous dispatcher — Spring ApplicationEventPublisher or custom
@Transactional
class PlaceOrderUseCase(
    private val orderRepository: OrderRepository,
    private val customerRepository: CustomerRepository,
    private val eventDispatcher: DomainEventDispatcher
) {
    fun execute(command: PlaceOrderCommand): OrderId {
        val customer = customerRepository.findById(command.customerId)
            ?: throw CustomerNotFoundException(command.customerId)

        val order = Order.create(customer.id, customer.creditLimit)
        command.items.forEach { order.addItem(it.product, it.quantity) }
        val event = order.confirm()

        // Save FIRST, then dispatch (still same transaction)
        orderRepository.save(order)

        // Dispatch intra-process — handlers run in same TX
        eventDispatcher.dispatch(order.events)
        order.clearEvents()

        return order.id
    }
}

// Intra-process handler: update a read-model projection in the same TX
class OrderConfirmedProjectionHandler(
    private val projectionStore: OrderSummaryProjectionStore
) : DomainEventHandler<OrderConfirmed> {
    override fun handle(event: OrderConfirmed) {
        projectionStore.upsert(
            OrderSummaryProjection(
                orderId = event.orderId.value,
                customerId = event.customerId.value,
                total = event.total.amount,
                status = "CONFIRMED",
                confirmedAt = event.occurredAt
            )
        )
    }
}
```

**When intra-process is sufficient:**
- Updating read models or projections in the same database
- Cascading operations within the same bounded context
- Audit logging
- Enforcing eventual consistency within one service

---

## Inter-Process Dispatch (Message Broker)

When events must cross service or bounded context boundaries, they go through a message broker (Kafka, RabbitMQ, SQS/SNS, Pulsar).

**The naive approach (broken):**
```
save to DB ──> publish to Kafka   <-- WRONG: DB commits but Kafka publish fails
                                       or Kafka publishes but DB rolls back
                                       = split brain
```

**Two-phase commit** — theoretically correct but impractical for most systems.

**The correct approach: Outbox Pattern.**

---

## The Outbox Pattern

The Outbox Pattern guarantees that the event is durably persisted alongside the state change in the same local transaction. A separate relay process then publishes to the broker.

```
+------------------+        Single local transaction         +------------------+
|   APPLICATION    | -------- write Order row ----------->  |    DATABASE       |
|   SERVICE        | -------- write to outbox table -------> |                   |
+------------------+                                        | orders            |
                                                            | outbox_events     |
                                                            +------------------+
                                                                    |
                                                         polling or CDC
                                                                    |
                                                                    v
                                                       +------------------+
                                                       |   MESSAGE RELAY  |
                                                       |   (Debezium,     |
                                                       |   custom poller) |
                                                       +------------------+
                                                                    |
                                                                    v
                                                       +------------------+
                                                       |  MESSAGE BROKER  |
                                                       |  (Kafka, Rabbit) |
                                                       +------------------+
                                                                    |
                                                          subscribers consume
                                                                    |
                                                                    v
                                                       +------------------+
                                                       |  OTHER SERVICES  |
                                                       |  (Shipping,      |
                                                       |   Billing, etc.) |
                                                       +------------------+
```

### Outbox Table Schema

```sql
CREATE TABLE outbox_events (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id  TEXT        NOT NULL,
    aggregate_type TEXT       NOT NULL,
    event_type    TEXT        NOT NULL,
    event_version INT         NOT NULL DEFAULT 1,
    payload       JSONB       NOT NULL,
    metadata      JSONB,
    occurred_at   TIMESTAMPTZ NOT NULL,
    published_at  TIMESTAMPTZ,          -- null = not yet published
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    INDEX idx_outbox_unpublished (published_at) WHERE published_at IS NULL
);
```

### Writing to the Outbox (Java/Spring)

```java
@Transactional
public class PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper mapper;

    public OrderId execute(PlaceOrderCommand command) {
        Order order = buildAndConfirmOrder(command);

        // Save aggregate AND events in ONE transaction
        orderRepository.save(order);

        order.events().forEach(event -> {
            OutboxEvent outboxEntry = OutboxEvent.builder()
                .aggregateId(order.id().value())
                .aggregateType("Order")
                .eventType(event.getClass().getSimpleName())
                .eventVersion(event.version())
                .payload(mapper.valueToTree(event))
                .occurredAt(event.occurredAt())
                .build();
            outboxRepository.save(outboxEntry);
        });

        order.clearEvents();
        return order.id();
    }
}
```

### Message Relay (Polling approach)

```kotlin
@Scheduled(fixedDelay = 500)  // run every 500ms
@Transactional
fun relayUnpublishedEvents() {
    val unpublished = outboxRepository.findUnpublished(limit = 100)
    unpublished.forEach { outboxEvent ->
        try {
            val envelope = buildEnvelope(outboxEvent)
            kafkaTemplate.send(
                topicFor(outboxEvent.eventType),
                outboxEvent.aggregateId,
                envelope
            )
            outboxRepository.markPublished(outboxEvent.id)
        } catch (ex: Exception) {
            log.error("Failed to relay event ${outboxEvent.id}", ex)
            // Will retry on next poll; idempotent consumers handle duplicates
        }
    }
}
```

### CDC-Based Relay (Debezium — preferred for high throughput)

```json
// Debezium connector config — reads the outbox table from PostgreSQL WAL
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "postgres",
  "database.dbname": "orders_db",
  "table.include.list": "public.outbox_events",
  "transforms": "outbox",
  "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.table.field.event.id": "id",
  "transforms.outbox.table.field.event.key": "aggregate_id",
  "transforms.outbox.table.field.event.type": "event_type",
  "transforms.outbox.table.field.event.payload": "payload",
  "transforms.outbox.route.by.field": "aggregate_type"
}
```

CDC approach has zero polling latency and no extra DB load — changes are streamed directly from the WAL.

---

## Domain Event Handlers vs Integration Event Handlers

```
                    SAME BOUNDED CONTEXT
+------------------------------------------+
|                                          |
|  Order Aggregate                         |
|      |                                   |
|      | raises OrderConfirmed             |
|      v                                   |
|  Domain Event Handler                    |
|  - Update order summary projection       |  <-- intra-process, same TX
|  - Enforce cross-aggregate consistency   |
|  - Trigger within-context side effects   |
|                                          |
+------------------------------------------+
                    |
                    | Outbox Pattern
                    v
            Message Broker (Kafka)
                    |
                    | OrderConfirmed message
          +---------+---------+
          |                   |
          v                   v
  Shipping Context     Billing Context
  Integration          Integration
  Event Handler        Event Handler
  - Create shipment    - Create invoice
  - Uses Shipping      - Uses Billing
    domain model         domain model
```

**Domain Event Handler:**
- Lives in the same bounded context as the event producer
- Can run synchronously in the same transaction
- Uses domain types (not serialized DTOs)
- Raises its own domain events if needed

**Integration Event Handler:**
- Lives in a different bounded context or service
- Consumes from the message broker (asynchronous)
- Translates the integration event into its own domain model (using ACL)
- Must be idempotent — messages may be delivered more than once

### Idempotent Consumer Pattern

```kotlin
// Idempotent handler: safe to call multiple times with the same event
class OrderConfirmedShippingHandler(
    private val shipmentRepository: ShipmentRepository,
    private val processedEventStore: ProcessedEventStore
) {
    fun handle(envelopeJson: String) {
        val envelope = deserialize(envelopeJson)

        // Check if already processed (idempotency check)
        if (processedEventStore.wasProcessed(envelope.eventId)) {
            log.info("Event ${envelope.eventId} already processed, skipping")
            return
        }

        // Process in a transaction that also records the event ID
        transactional {
            val event = translateToShippingDomain(envelope)
            val shipment = Shipment.createFor(event)
            shipmentRepository.save(shipment)
            processedEventStore.markProcessed(envelope.eventId)  // same TX
        }
    }
}
```

---

## Event Versioning

Events are contracts. Consumers depend on them. They must evolve carefully.

### Safe (backward-compatible) changes:
```kotlin
// Version 1
data class OrderConfirmed(
    val orderId: OrderId,
    val total: Money,
    val occurredAt: Instant,
    val version: Int = 1
)

// Version 2 — added optional field, defaults for missing
data class OrderConfirmed(
    val orderId: OrderId,
    val total: Money,
    val occurredAt: Instant,
    val version: Int = 2,
    val itemCount: Int? = null,          // NEW — optional, old consumers ignore it
    val promoCode: String? = null        // NEW — optional, old consumers ignore it
)
```

### Breaking changes — use upcasters:

```kotlin
// Upcaster: transforms old event schema to new schema at read time
class OrderConfirmedV1ToV2Upcaster : EventUpcaster<Map<String, Any?>> {
    override fun supports(eventType: String, version: Int) =
        eventType == "OrderConfirmed" && version == 1

    override fun upcast(event: Map<String, Any?>): Map<String, Any?> {
        // V1 had "amount" and "currency" as separate fields
        // V2 has a nested "total" object
        val amount = event["amount"] as String
        val currency = event["currency"] as String
        return event
            .minus("amount")
            .minus("currency")
            .plus("total" to mapOf("amount" to amount, "currency" to currency))
            .plus("version" to 2)
    }
}
```

### Versioning strategy options:

| Strategy | How | When to use |
|----------|-----|-------------|
| **Additive only** | Never remove/rename fields; only add optional ones | Most cases; simplest |
| **Version field** | Include `version` int in event; consumers branch on it | When schema evolution is expected |
| **Upcasters** | Transform old schema to new at deserialization | Breaking changes that cannot be avoided |
| **New event type** | `OrderConfirmedV2` alongside `OrderConfirmed` | Major structural redesign |
| **Parallel topics** | Publish to both old and new topic during migration | Large organizations, many consumers |

---

## Real-Domain Examples

### E-Commerce

```kotlin
// Order lifecycle events
data class CartAbandoned(val cartId: CartId, val customerId: CustomerId,
    val items: List<CartItem>, val occurredAt: Instant = Instant.now(), val version: Int = 1)

data class OrderPlaced(val orderId: OrderId, val customerId: CustomerId,
    val total: Money, val shippingAddress: Address, val occurredAt: Instant = Instant.now())

data class PaymentAuthorized(val orderId: OrderId, val paymentId: PaymentId,
    val authorizedAmount: Money, val gatewayRef: String, val occurredAt: Instant = Instant.now())

data class PaymentDeclined(val orderId: OrderId, val attemptedAmount: Money,
    val declineCode: String, val occurredAt: Instant = Instant.now())

data class OrderShipped(val orderId: OrderId, val shipmentId: ShipmentId,
    val carrier: String, val trackingNumber: String, val estimatedDelivery: LocalDate,
    val occurredAt: Instant = Instant.now())

data class OrderDelivered(val orderId: OrderId, val deliveredAt: Instant,
    val occurredAt: Instant = Instant.now())

data class ReturnRequested(val orderId: OrderId, val returnId: ReturnId,
    val reason: ReturnReason, val items: List<ReturnItem>, val occurredAt: Instant = Instant.now())

data class RefundProcessed(val returnId: ReturnId, val orderId: OrderId,
    val refundAmount: Money, val method: RefundMethod, val occurredAt: Instant = Instant.now())
```

### Banking

```kotlin
data class AccountOpened(val accountId: AccountId, val customerId: CustomerId,
    val accountType: AccountType, val initialBalance: Money,
    val occurredAt: Instant = Instant.now())

data class DepositMade(val accountId: AccountId, val transactionId: TransactionId,
    val amount: Money, val newBalance: Money, val channel: DepositChannel,
    val occurredAt: Instant = Instant.now())

data class WithdrawalMade(val accountId: AccountId, val transactionId: TransactionId,
    val amount: Money, val newBalance: Money, val occurredAt: Instant = Instant.now())

data class TransferInitiated(val transferId: TransferId,
    val sourceAccountId: AccountId, val destinationAccountId: AccountId,
    val amount: Money, val occurredAt: Instant = Instant.now())

data class TransferCompleted(val transferId: TransferId,
    val occurredAt: Instant = Instant.now())

data class InsufficientFundsDetected(val accountId: AccountId,
    val attemptedAmount: Money, val availableBalance: Money,
    val occurredAt: Instant = Instant.now())

data class AccountFrozen(val accountId: AccountId,
    val reason: FreezeReason, val initiatedBy: UserId,
    val occurredAt: Instant = Instant.now())
```

---

## Domain Events and Event Sourcing

Domain Events are the foundation of Event Sourcing. In a standard domain model, events are *side effects* — the aggregate state is the source of truth. In Event Sourcing, events *are* the source of truth — the current state is *derived* from the event history.

```
STANDARD DDD (events as side effects):
+----------+     save     +----------+
| Aggregate| -----------> | DATABASE |  <-- current state stored
+----------+              | (rows)   |
                          +----------+

EVENT SOURCING (events as source of truth):
+----------+     append   +----------+
| Aggregate| -----------> | EVENT    |  <-- events stored (append-only)
+----------+              | STORE    |
    ^                     +----------+
    |  replay to rebuild       |
    +--------------------------+
```

### Event-Sourced Aggregate (Kotlin)

```kotlin
class Order private constructor(
    val id: OrderId,
    private var status: OrderStatus = OrderStatus.DRAFT,
    private val _items: MutableList<OrderItem> = mutableListOf()
) {
    private val _changes: MutableList<DomainEvent> = mutableListOf()
    val changes: List<DomainEvent> get() = _changes.toList()

    // Command method: validates, then calls apply
    fun confirm() {
        require(status == OrderStatus.DRAFT) { "..." }
        require(_items.isNotEmpty()) { "..." }
        apply(OrderConfirmed(orderId = id, total = total(), occurredAt = Instant.now()))
    }

    // Apply method: updates state from event (used for both new events and replay)
    private fun apply(event: DomainEvent) {
        when (event) {
            is OrderConfirmed -> status = OrderStatus.CONFIRMED
            is OrderCancelled -> status = OrderStatus.CANCELLED
            is ItemAddedToOrder -> _items.add(OrderItem.of(event.sku, event.quantity, event.unitPrice))
            is ItemRemovedFromOrder -> _items.removeIf { it.sku == event.sku }
        }
        _changes += event  // only for new events; replay doesn't add to changes
    }

    companion object {
        // Rebuild aggregate from event history
        fun reconstitute(history: List<DomainEvent>): Order {
            val first = history.first() as? OrderCreated
                ?: throw IllegalStateException("First event must be OrderCreated")
            val order = Order(first.orderId)
            history.drop(1).forEach { order.applyFromHistory(it) }
            return order
        }
    }

    // Called during replay — does not add to _changes
    private fun applyFromHistory(event: DomainEvent) {
        when (event) {
            is OrderConfirmed -> status = OrderStatus.CONFIRMED
            is OrderCancelled -> status = OrderStatus.CANCELLED
            is ItemAddedToOrder -> _items.add(OrderItem.of(event.sku, event.quantity, event.unitPrice))
            is ItemRemovedFromOrder -> _items.removeIf { it.sku == event.sku }
        }
    }
}
```

**Key difference:** In standard DDD, the repository `save()` persists current state and separately publishes events. In Event Sourcing, the repository `save()` appends new events; current state is reconstructed by replaying them.

---

## Python Example

```python
# domain/events.py
from dataclasses import dataclass, field
from datetime import datetime, timezone
from uuid import UUID, uuid4
from typing import Optional

@dataclass(frozen=True)
class DomainEvent:
    event_id: UUID = field(default_factory=uuid4)
    occurred_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    version: int = 1

@dataclass(frozen=True)
class OrderConfirmed(DomainEvent):
    order_id: UUID = None
    customer_id: UUID = None
    total_amount: str = None     # Use str for Decimal serialization safety
    total_currency: str = None
    item_count: int = 0

@dataclass(frozen=True)
class PaymentDeclined(DomainEvent):
    order_id: UUID = None
    attempted_amount: str = None
    decline_code: str = None
    gateway_ref: Optional[str] = None

# domain/order.py
class Order:
    def __init__(self, order_id: UUID, customer_id: UUID, credit_limit: Decimal):
        self._id = order_id
        self._customer_id = customer_id
        self._credit_limit = credit_limit
        self._status = OrderStatus.DRAFT
        self._items: list[OrderItem] = []
        self._events: list[DomainEvent] = []

    def confirm(self) -> None:
        if self._status != OrderStatus.DRAFT:
            raise DomainError(f"Cannot confirm order in status {self._status}")
        if not self._items:
            raise DomainError("Cannot confirm empty order")

        self._status = OrderStatus.CONFIRMED
        self._events.append(OrderConfirmed(
            order_id=self._id,
            customer_id=self._customer_id,
            total_amount=str(self.total),
            total_currency="BRL",
            item_count=len(self._items)
        ))

    def pull_events(self) -> list[DomainEvent]:
        events = list(self._events)
        self._events.clear()
        return events

# application/place_order_use_case.py
class PlaceOrderUseCase:
    def __init__(self, order_repo: OrderRepository, outbox_repo: OutboxRepository):
        self._order_repo = order_repo
        self._outbox_repo = outbox_repo

    @transactional
    def execute(self, command: PlaceOrderCommand) -> UUID:
        order = Order.create(command.customer_id, command.credit_limit)
        for item in command.items:
            order.add_item(item.product, item.quantity)
        order.confirm()

        # Save aggregate
        self._order_repo.save(order)

        # Write events to outbox — same transaction
        for event in order.pull_events():
            self._outbox_repo.save(OutboxEvent(
                aggregate_id=str(order.id),
                aggregate_type="Order",
                event_type=type(event).__name__,
                payload=asdict(event),
                occurred_at=event.occurred_at
            ))

        return order.id
```

---

## Practical Checklist

### Event Design
- [ ] Event name is past tense and business-meaningful?
- [ ] Event carries enough data for handlers to act without reloading the aggregate?
- [ ] Event has an `eventId` for idempotency?
- [ ] Event has an `occurredAt` timestamp?
- [ ] Event has a `version` field for schema evolution?
- [ ] Payload contains only Value Objects and primitives (no Entity references)?

### When to Raise
- [ ] Event is raised only after invariants are enforced?
- [ ] Event is raised only after state is already changed?
- [ ] Events are accumulated in the aggregate, not dispatched directly?

### Intra-Process Dispatch
- [ ] Events are dispatched after the aggregate is saved (not before)?
- [ ] Intra-process handlers are synchronous and run in the same transaction?

### Outbox Pattern
- [ ] Outbox write is in the same transaction as the aggregate save?
- [ ] Relay is idempotent (duplicate publish doesn't break consumers)?
- [ ] Consumers implement idempotency (using event ID)?
- [ ] Outbox is monitored for stuck events (unpublished for > threshold)?

### Event Versioning
- [ ] New fields are optional with defaults (backward-compatible)?
- [ ] Breaking changes go through upcasters?
- [ ] Consumers ignore unknown fields (lenient deserialization)?

### Integration Events
- [ ] Consumers translate event to their own domain model (ACL)?
- [ ] Consumer records processed event ID in same transaction?
- [ ] Consumer is safe to call twice with same event (idempotent)?
