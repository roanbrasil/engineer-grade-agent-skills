---
name: cqrs-event-sourcing
description: Apply CQRS and Event Sourcing when designing systems with complex write/read separation, audit requirements, or event-driven workflows — invoked when discussing command/query separation, event stores, projections, sagas, or eventual consistency.
---

# CQRS + Event Sourcing

CQRS (Command Query Responsibility Segregation) and Event Sourcing are distinct patterns that compose powerfully. CQRS separates the write model from the read model. Event Sourcing makes the event log the source of truth, replacing the current-state snapshot with a complete history.

Neither requires the other — but together they address a class of problems that traditional CRUD cannot.

---

## The Core Mental Model

```
Traditional CRUD:
  Client ──→ [Single Model] ──→ Database (current state only)
                 ↑
             Read & Write
             mixed together

CQRS:
  Client ──→ [Write Model] ──→ Database (optimized for writes)
     │
     └──────→ [Read Model]  ──→ Database (optimized for reads)

CQRS + Event Sourcing:
  Client ──→ [Write Model]
                 │
                 ▼
           [Event Store]    ← system of record (append-only log)
                 │
                 ▼
           [Projections]   ──→ [Read Model DB] ──→ Client queries
                 │
                 ▼
           [Sagas/Process Managers] → external side effects
```

---

## The Three Message Types

Every action in a CQRS+ES system is represented by one of three message types. Getting these right is fundamental.

### Commands

A command is a **request to change state**. It can be rejected. It is named in the imperative.

```java
// Commands: imperative names, validated, can fail
public record PlaceOrder(
    UUID customerId,
    List<OrderItem> items,
    PaymentMethod paymentMethod
) implements Command {
    public PlaceOrder {
        Objects.requireNonNull(customerId, "customerId required");
        if (items == null || items.isEmpty()) throw new IllegalArgumentException("items required");
    }
}

public record CancelOrder(UUID orderId, String reason) implements Command {}
public record ApplyDiscount(UUID orderId, Percentage discount) implements Command {}
```

```python
from dataclasses import dataclass, field
from typing import List
from uuid import UUID

@dataclass(frozen=True)
class PlaceOrder:
    customer_id: UUID
    items: List['OrderItem']
    payment_method: 'PaymentMethod'

    def __post_init__(self):
        if not self.items:
            raise ValueError("Order must have at least one item")

@dataclass(frozen=True)
class CancelOrder:
    order_id: UUID
    reason: str
```

**Command rules:**
- Imperative names: `PlaceOrder`, `CancelOrder`, `ApplyDiscount`
- Can be rejected (throw exception or return a failure result)
- Sent to exactly one handler
- Contains enough data to execute the operation
- Immutable

### Events

An event is a **fact that already happened**. It cannot be rejected. It is named in the past tense.

```java
// Events: past-tense names, immutable facts
public record OrderPlaced(
    UUID orderId,
    UUID customerId,
    List<OrderItem> items,
    Money total,
    Instant occurredAt
) implements DomainEvent {}

public record OrderCancelled(
    UUID orderId,
    String reason,
    Instant occurredAt
) implements DomainEvent {}

public record DiscountApplied(
    UUID orderId,
    Percentage discount,
    Money newTotal,
    Instant occurredAt
) implements DomainEvent {}
```

**Event rules:**
- Past-tense names: `OrderPlaced`, `OrderCancelled`, `PaymentProcessed`
- Immutable — never changed after creation
- Published to 0..N subscribers
- Contains all data needed to react to the event (self-describing)
- Always include `occurredAt` timestamp
- Always include aggregate ID

### Queries

A query is a **request for data**. It has no side effects. It always returns something.

```java
public record GetOrderById(UUID orderId) implements Query<OrderView> {}
public record GetOrdersByCustomer(UUID customerId, OrderStatus status) implements Query<List<OrderSummary>> {}
public record GetMonthlyRevenue(YearMonth month) implements Query<RevenueReport> {}
```

**Query rules:**
- No side effects (Command-Query Separation)
- Hits the read model (projection), not the write model (aggregate)
- Can return stale data (acceptable given eventual consistency)

---

## Event Sourcing: The Event Store as System of Record

In a traditional system, the database holds **current state**. In an event-sourced system, the event store holds the **complete history of changes**. Current state is derived by replaying events.

```
Traditional: What is the current balance?
  accounts table: { id: 123, balance: 850.00 }

Event Sourced: What is the current balance?
  events for account 123:
    AccountOpened    { balance: 1000.00 }  t=2024-01-01
    WithdrawalMade   { amount:  200.00  }  t=2024-01-15
    DepositMade      { amount:   50.00  }  t=2024-01-20
    → replay → balance = 1000 - 200 + 50 = 850.00

Benefits:
  ✓ Complete audit trail — every state was caused by an event
  ✓ Time travel — rebuild state at any point in time
  ✓ Event replay — rebuild projections from scratch
  ✓ Debugging — understand exactly how you got to this state
  ✓ Integration — other systems subscribe to domain events
```

### Aggregate: Rebuild from Events

```java
public class Order {
    private UUID id;
    private UUID customerId;
    private List<OrderLine> lines = new ArrayList<>();
    private OrderStatus status;
    private Money total;

    // Private constructor — aggregates are loaded from events only
    private Order() {}

    // Factory: create new aggregate, emitting the first event
    public static List<DomainEvent> place(PlaceOrder command) {
        // validate
        if (command.items().isEmpty()) throw new EmptyOrderException();

        var event = new OrderPlaced(
            UUID.randomUUID(),
            command.customerId(),
            command.items(),
            calculateTotal(command.items()),
            Instant.now()
        );
        return List.of(event);
    }

    // Command handler: emits events or throws
    public List<DomainEvent> cancel(CancelOrder command) {
        if (status == OrderStatus.DELIVERED) {
            throw new CannotCancelDeliveredOrderException(id);
        }
        return List.of(new OrderCancelled(id, command.reason(), Instant.now()));
    }

    // Event applier: mutates state; must not throw; deterministic
    public void apply(OrderPlaced event) {
        this.id = event.orderId();
        this.customerId = event.customerId();
        this.lines = new ArrayList<>(event.items());
        this.total = event.total();
        this.status = OrderStatus.PLACED;
    }

    public void apply(OrderCancelled event) {
        this.status = OrderStatus.CANCELLED;
    }

    // Rebuild from event stream
    public static Order reconstitute(List<DomainEvent> history) {
        var order = new Order();
        for (var event : history) {
            order.applyEvent(event);
        }
        return order;
    }

    private void applyEvent(DomainEvent event) {
        switch (event) {
            case OrderPlaced e -> apply(e);
            case OrderCancelled e -> apply(e);
            default -> throw new UnknownEventException(event.getClass());
        }
    }
}
```

```python
from dataclasses import dataclass, field
from typing import List, Union
from enum import Enum

class Order:
    def __init__(self):
        self.id = None
        self.customer_id = None
        self.lines: list = []
        self.status: OrderStatus = None
        self.total: Money = None

    @classmethod
    def place(cls, command: PlaceOrder) -> List[DomainEvent]:
        """Factory — returns events, does not mutate state yet."""
        if not command.items:
            raise EmptyOrderError()
        return [OrderPlaced(
            order_id=uuid4(),
            customer_id=command.customer_id,
            items=command.items,
            total=calculate_total(command.items),
            occurred_at=datetime.utcnow()
        )]

    def cancel(self, command: CancelOrder) -> List[DomainEvent]:
        if self.status == OrderStatus.DELIVERED:
            raise CannotCancelDeliveredOrderError(self.id)
        return [OrderCancelled(self.id, command.reason, datetime.utcnow())]

    def apply(self, event: DomainEvent) -> None:
        """Apply an event to mutate state. Must not raise."""
        match event:
            case OrderPlaced():
                self.id = event.order_id
                self.customer_id = event.customer_id
                self.lines = list(event.items)
                self.total = event.total
                self.status = OrderStatus.PLACED
            case OrderCancelled():
                self.status = OrderStatus.CANCELLED

    @classmethod
    def reconstitute(cls, history: List[DomainEvent]) -> 'Order':
        order = cls()
        for event in history:
            order.apply(event)
        return order
```

---

## The Event Store

The event store is an append-only log of events, indexed by aggregate ID and sequence number.

```
Schema (conceptual):
  events
  ┌────────────────┬─────────────────┬──────────┬─────────────────────┬──────────────────────┐
  │ aggregate_id   │ aggregate_type  │ sequence │ event_type          │ payload (JSON)       │
  ├────────────────┼─────────────────┼──────────┼─────────────────────┼──────────────────────┤
  │ order-abc-123  │ Order           │ 1        │ OrderPlaced         │ { customer_id, ... } │
  │ order-abc-123  │ Order           │ 2        │ DiscountApplied     │ { discount, ... }    │
  │ order-abc-123  │ Order           │ 3        │ OrderCancelled      │ { reason, ... }      │
  └────────────────┴─────────────────┴──────────┴─────────────────────┴──────────────────────┘
```

```java
public interface EventStore {
    void append(UUID aggregateId, List<DomainEvent> events, long expectedVersion);
    List<DomainEvent> load(UUID aggregateId);
    List<DomainEvent> loadFrom(UUID aggregateId, long fromVersion);
}

// Optimistic concurrency: expectedVersion prevents lost updates
public class PostgresEventStore implements EventStore {
    @Override
    public void append(UUID aggregateId, List<DomainEvent> events, long expectedVersion) {
        long currentVersion = getCurrentVersion(aggregateId);
        if (currentVersion != expectedVersion) {
            throw new OptimisticConcurrencyException(aggregateId, expectedVersion, currentVersion);
        }
        // INSERT INTO events ...
        for (int i = 0; i < events.size(); i++) {
            insertEvent(aggregateId, events.get(i), expectedVersion + i + 1);
        }
    }
}
```

---

## Snapshots: When Replay Gets Expensive

For aggregates with thousands of events, full replay on every command becomes slow. Snapshots save the current state periodically and replay only from the snapshot forward.

```java
public class SnapshotAwareOrderRepository implements OrderRepository {
    private final EventStore eventStore;
    private final SnapshotStore snapshotStore;

    @Override
    public Order load(UUID orderId) {
        // Try to load from recent snapshot
        return snapshotStore.findLatest(orderId)
            .map(snapshot -> {
                var order = (Order) snapshot.state();
                var recentEvents = eventStore.loadFrom(orderId, snapshot.version());
                for (var event : recentEvents) order.apply(event);
                return order;
            })
            .orElseGet(() -> {
                // No snapshot: full replay
                var history = eventStore.load(orderId);
                return Order.reconstitute(history);
            });
    }

    @Override
    public void save(Order order, long expectedVersion) {
        eventStore.append(order.id(), order.pendingEvents(), expectedVersion);

        // Snapshot every 50 events
        if (order.version() % 50 == 0) {
            snapshotStore.save(new Snapshot(order.id(), order.version(), order));
        }
    }
}
```

**Snapshot policy:** snapshot every N events (50–500 is typical). Tune based on aggregate event frequency and load test results.

---

## Read-Side Projections

A projection listens to events and builds a denormalized read model optimized for queries.

```java
// Projection: builds an order summary read model
@Component
public class OrderSummaryProjection {
    private final OrderSummaryRepository readDb;

    @EventHandler
    public void on(OrderPlaced event) {
        readDb.save(new OrderSummaryView(
            event.orderId(),
            event.customerId(),
            event.total(),
            "PLACED",
            event.occurredAt()
        ));
    }

    @EventHandler
    public void on(OrderCancelled event) {
        readDb.findById(event.orderId()).ifPresent(view -> {
            view.setStatus("CANCELLED");
            readDb.save(view);
        });
    }

    @EventHandler
    public void on(DiscountApplied event) {
        readDb.findById(event.orderId()).ifPresent(view -> {
            view.setTotal(event.newTotal());
            readDb.save(view);
        });
    }
}

// Multiple projections from the same events
@Component
public class CustomerOrderHistoryProjection {
    @EventHandler
    public void on(OrderPlaced event) {
        // Build a different read model: customer → orders
        customerOrderHistory.addOrder(event.customerId(), event.orderId(), event.total());
    }
}
```

```python
class OrderSummaryProjection:
    def __init__(self, read_db: OrderSummaryRepository):
        self.read_db = read_db

    def handle(self, event: DomainEvent) -> None:
        match event:
            case OrderPlaced():
                self.read_db.save(OrderSummaryView(
                    order_id=event.order_id,
                    customer_id=event.customer_id,
                    total=event.total,
                    status="PLACED",
                    placed_at=event.occurred_at
                ))
            case OrderCancelled():
                view = self.read_db.find_by_id(event.order_id)
                if view:
                    view.status = "CANCELLED"
                    self.read_db.save(view)
```

**Projection rebuild:** because events are immutable and stored, you can drop and rebuild a projection at any time by replaying all events from the beginning. This means you can add new projections to existing systems with full historical data.

---

## Eventual Consistency

In a CQRS+ES system, the read model is eventually consistent with the write model. After a command is processed:
1. Events are appended to the event store ← **this is the commit point**
2. Projections process the events asynchronously
3. The read model is updated ← **this may lag by milliseconds to seconds**

```
Timeline:
  t=0ms:  PlaceOrder command received
  t=5ms:  OrderPlaced event stored → command returns success
  t=6ms:  Projection receives OrderPlaced event (async)
  t=8ms:  Read model updated with new order
  t=50ms: Client queries order list → sees the order

Between t=5ms and t=8ms: the order exists in write model but not in read model.
This is the consistency window. Design your UI and tests around it.
```

### Communicating Eventual Consistency

```java
// Return the event ID or version so the client can poll
public record PlaceOrderResult(UUID orderId, long eventVersion) {}

// Client polls with version: "give me the order if version >= N"
@GetMapping("/orders/{id}")
public ResponseEntity<OrderView> getOrder(
    @PathVariable UUID id,
    @RequestParam(required = false) Long minVersion
) {
    var view = orderQueryService.findById(id);
    if (minVersion != null && view.version() < minVersion) {
        return ResponseEntity.accepted().build();  // 202: not yet consistent
    }
    return ResponseEntity.ok(view);
}
```

---

## Saga / Process Manager

A saga coordinates a multi-step workflow that spans multiple aggregates or services, reacting to events and issuing commands.

```
Order Fulfillment Saga:
  OrderPlaced
      │
      ▼
  ReserveInventory command ──→ InventoryReserved / InventoryUnavailable
      │                              │
      │                    InventoryReserved
      │                              │
      ▼                              ▼
  ProcessPayment command ──→ PaymentProcessed / PaymentDeclined
      │                              │
      │                    PaymentProcessed
      │                              │
      ▼                              ▼
  ShipOrder command ──────→ OrderShipped

Compensations (on failure):
  PaymentDeclined ──→ ReleaseInventory command
  ShipmentFailed  ──→ RefundPayment + ReleaseInventory commands
```

```java
@Saga
public class OrderFulfillmentSaga {
    private UUID orderId;
    private UUID inventoryReservationId;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void on(OrderPlaced event, CommandGateway commandGateway) {
        this.orderId = event.orderId();
        commandGateway.send(new ReserveInventory(
            UUID.randomUUID(), event.orderId(), event.items()
        ));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(InventoryReserved event, CommandGateway commandGateway) {
        this.inventoryReservationId = event.reservationId();
        commandGateway.send(new ProcessPayment(orderId, event.total()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(InventoryUnavailable event, CommandGateway commandGateway) {
        commandGateway.send(new CancelOrder(orderId, "Inventory unavailable"));
        SagaLifecycle.end();
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(PaymentDeclined event, CommandGateway commandGateway) {
        commandGateway.send(new ReleaseInventory(inventoryReservationId));
        commandGateway.send(new CancelOrder(orderId, "Payment declined: " + event.reason()));
        SagaLifecycle.end();
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void on(OrderShipped event) {
        // Saga complete
    }
}
```

---

## Event Versioning and Schema Evolution

Events are permanent. Once stored, they cannot be changed. When the event schema must evolve:

```
Strategies:

1. Upcasting (recommended for small changes)
   Old event: OrderPlaced { orderId, customerId, total }
   New event: OrderPlaced { orderId, customerId, total, currency }  ← added field

   Upcast on read: if currency is null, default to "USD"

2. Versioned Events (for breaking changes)
   OrderPlacedV1 { orderId, customerId, totalAmount }
   OrderPlacedV2 { orderId, customerId, total: Money(amount, currency) }  ← different shape

   Keep handlers for V1 alongside handlers for V2.
   Upcast V1 → V2 in a migration step.

3. Copy-on-write (for full history migration)
   Run a migration job that reads all V1 events,
   writes V2 events, marks V1 as migrated.
   Dangerous: prefer upcasting when possible.
```

```java
// Upcaster: transforms old event format to new format on read
public class OrderPlacedUpcaster implements EventUpcaster {
    @Override
    public Stream<IntermediateEventRepresentation> upcast(
        Stream<IntermediateEventRepresentation> stream
    ) {
        return stream.map(event -> {
            if (!"OrderPlaced".equals(event.getType().getName())) return event;
            return event.upcastPayload(tree -> {
                if (!tree.has("currency")) {
                    tree.put("currency", "USD");  // backfill with safe default
                }
                return tree;
            });
        });
    }
}
```

---

## When to Use CQRS + Event Sourcing

**Use CQRS alone when:**
- Read and write workloads have very different scale requirements
- The read model requires complex aggregations that are expensive to compute at query time
- Multiple different read representations are needed for the same data

**Use Event Sourcing alone (rare) or CQRS+ES together when:**
- Audit trail is a hard requirement (financial, medical, legal)
- You need to replay history to fix bugs or add new projections
- Temporal queries ("what was the state at time T?") are required
- The domain has complex workflows spanning multiple aggregates (sagas)
- Event-driven integration with other services is a core architectural requirement

---

## When NOT to Use CQRS + Event Sourcing

```
CRUD apps                    → Simple table-per-entity is cleaner and faster to build
Simple domains               → The overhead of events/projections/sagas is not justified
Small teams new to ES        → Very high learning curve; wrong upcasting strategy = data loss
Simple read/write parity     → If read == write, CQRS adds complexity for no gain
Reporting / analytics-first  → Use a data warehouse instead of event sourcing
Prototypes                   → You will rebuild the projection schema 5 times in week 1
```

**The tell-tale sign that CQRS+ES is wrong for you:** you spend most of your time mapping flat CRUD entities to events and back. If events feel artificial, the domain probably does not have the complexity that justifies event sourcing.

---

## Spring / Axon Framework Implementation Sketch

```java
// Command handler (write side)
@Aggregate
public class OrderAggregate {
    @AggregateIdentifier
    private UUID orderId;
    private OrderStatus status;

    @CommandHandler
    public OrderAggregate(PlaceOrder command) {
        apply(new OrderPlaced(command.orderId(), command.customerId(), command.items(), ...));
    }

    @CommandHandler
    public void handle(CancelOrder command) {
        if (status == OrderStatus.DELIVERED) throw new CannotCancelDeliveredOrderException(orderId);
        apply(new OrderCancelled(orderId, command.reason(), Instant.now()));
    }

    @EventSourcingHandler
    public void on(OrderPlaced event) {
        this.orderId = event.orderId();
        this.status = OrderStatus.PLACED;
    }

    @EventSourcingHandler
    public void on(OrderCancelled event) {
        this.status = OrderStatus.CANCELLED;
    }
}

// Query handler (read side)
@Component
public class OrderQueryHandler {
    private final OrderSummaryRepository repository;

    @QueryHandler
    public OrderSummaryView handle(GetOrderById query) {
        return repository.findById(query.orderId())
            .orElseThrow(() -> new OrderNotFoundException(query.orderId()));
    }
}
```

## Python Implementation Sketch (without framework)

```python
class CommandBus:
    def __init__(self):
        self._handlers: dict[type, Callable] = {}

    def register(self, command_type: type, handler: Callable):
        self._handlers[command_type] = handler

    def dispatch(self, command) -> None:
        handler = self._handlers.get(type(command))
        if not handler:
            raise UnhandledCommandError(type(command))
        handler(command)

class PlaceOrderCommandHandler:
    def __init__(self, event_store: EventStore, event_bus: EventBus):
        self.event_store = event_store
        self.event_bus = event_bus

    def __call__(self, command: PlaceOrder) -> None:
        events = Order.place(command)  # domain logic — no I/O
        self.event_store.append(
            aggregate_id=events[0].order_id,
            events=events,
            expected_version=0
        )
        for event in events:
            self.event_bus.publish(event)

class EventBus:
    def __init__(self):
        self._subscribers: dict[type, list[Callable]] = defaultdict(list)

    def subscribe(self, event_type: type, handler: Callable):
        self._subscribers[event_type].append(handler)

    def publish(self, event: DomainEvent) -> None:
        for handler in self._subscribers.get(type(event), []):
            handler(event)
```

---

## Checklist

### Commands
- [ ] Command names are imperative: `PlaceOrder`, not `OrderPlaced` or `OrderPlacer`
- [ ] Commands are immutable value objects
- [ ] Commands are validated before being dispatched
- [ ] Each command has exactly one handler

### Events
- [ ] Event names are past tense: `OrderPlaced`, not `PlaceOrder`
- [ ] Events are immutable and contain all relevant data (self-describing)
- [ ] Events include `occurredAt` timestamp and aggregate ID
- [ ] Events never contain infrastructure types (no JPA entities, no HTTP objects)

### Queries
- [ ] Queries have zero side effects
- [ ] Queries hit the read model, not the event store or aggregate
- [ ] Query handlers never dispatch commands

### Aggregates
- [ ] Aggregates can only be loaded by replaying events (no `new Aggregate()` from the DB)
- [ ] `apply` methods only mutate state — no I/O, no throwing
- [ ] Command handlers validate and emit events — they do not mutate state directly
- [ ] Optimistic concurrency (expected version) is enforced on append

### Projections
- [ ] Each projection handles events idempotently (safe to replay)
- [ ] Projections can be dropped and rebuilt by replaying all events
- [ ] Multiple projections from the same event stream are independent

### Eventual Consistency
- [ ] The team understands and accepts eventual consistency
- [ ] The UI is designed to handle consistency lag (polling, WebSockets, optimistic updates)
- [ ] Tests account for async projection updates

### Event Versioning
- [ ] A strategy for schema evolution is documented before the first event is stored
- [ ] Upcasters are in place for events that have changed shape
- [ ] Event type names are stable identifiers (not class FQNs that will change on refactor)

### Sagas
- [ ] Each saga has a clear start event and end condition
- [ ] Compensating commands exist for every step that can fail
- [ ] Saga state is persisted (not in-memory) so it survives restarts
