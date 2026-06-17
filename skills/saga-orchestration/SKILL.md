---
name: saga-orchestration
description: Design, implement, or debug distributed sagas for multi-service transactions — use when someone asks about distributed transactions, compensating actions, saga patterns, or replacing 2PC.
---

# Saga Orchestration — Distributed Transactions Without 2PC

## Why Sagas Exist

### The Problem With Distributed Transactions (2PC)

Two-Phase Commit (2PC) requires all participants to hold locks until the coordinator says commit or rollback:

```
Coordinator
  |-- PREPARE --> ServiceA  (ServiceA locks rows, holds lock)
  |-- PREPARE --> ServiceB  (ServiceB locks rows, holds lock)
  |-- PREPARE --> ServiceC  (ServiceC locks rows, holds lock)
  |
  Wait for all votes...
  |
  |-- COMMIT  --> ServiceA  (releases lock)
  |-- COMMIT  --> ServiceB  (releases lock)
  |-- COMMIT  --> ServiceC  (releases lock)
```

Problems:
- **Blocking**: all participants hold locks while waiting for the coordinator; a slow participant blocks everyone
- **Coordinator SPOF**: if the coordinator crashes after PREPARE but before COMMIT, participants are stuck waiting
- **Scalability**: locks across services kill throughput; microservices often don't share a DB
- **Cross-technology**: 2PC requires XA support; Kafka, Redis, and most NoSQL stores don't support it

### The Saga Alternative

A Saga is a sequence of **local transactions**. Each step commits locally and publishes an event or message. If any step fails, **compensating transactions** undo previously completed steps.

Key properties:
- No distributed locks; each service commits independently
- Eventual consistency instead of strong consistency
- Every step must have a defined compensation
- Messages must be idempotent (at-least-once delivery)

---

## Saga Styles

### Choreography — Event-Driven, No Central Coordinator

Each service listens for events and reacts by doing its work and emitting the next event.

```
OrderService          InventoryService        PaymentService        ShippingService
     |                       |                       |                     |
     |-- OrderPlaced ------> |                       |                     |
     |                       |-- StockReserved ----> |                     |
     |                       |                       |-- PaymentProcessed ->|
     |                       |                       |                     |-- OrderShipped
     |
[Compensation path]
     |                       |                       |
     |                       |                    PaymentFailed
     |                       |<-- ReleaseStock ------|
     |<-- OrderCancelled ----|
```

**Pros:**
- Loose coupling; services don't know about each other, only the events
- No single point of failure
- Easy to add new participants (just subscribe to the event)

**Cons:**
- Hard to understand the overall flow — it is spread across many services
- Difficult to debug: where in the saga did it fail?
- Risk of cyclic event dependencies
- Hard to enforce sequencing: InventoryService must not ship before PaymentService charges

**When to use choreography:** simple sagas with 2-4 steps; teams that value service independence; when you can tolerate distributed state tracking.

---

### Orchestration — Central Saga Orchestrator

A dedicated orchestrator sends commands to participants and waits for replies. The orchestrator owns the state machine.

```
                   +------------------+
                   |  Saga Orchestrator|
                   |  State: PENDING  |
                   +--------+---------+
                            |
             1. ReserveStock|
                            v
                   +------------------+
                   |  InventoryService|
                   +--------+---------+
                            |
             2. StockReserved
                            |
                   +--------+---------+
                   |  Saga Orchestrator|
                   |  State: STOCK_RESERVED
                   +--------+---------+
                            |
             3. ChargePayment
                            v
                   +------------------+
                   |  PaymentService  |
                   +--------+---------+
                            |
             4. PaymentCharged
                            |
                   +--------+---------+
                   |  Saga Orchestrator|
                   |  State: PAYMENT_CHARGED
                   +--------+---------+
                            |
             5. ShipOrder
                            v
                   +------------------+
                   |  ShippingService |
                   +------------------+
```

**Pros:**
- Explicit, traceable flow: read the orchestrator to understand the saga
- Error handling centralized: one place to add retries, timeouts, compensation logic
- Easy to add monitoring and alerting on saga state
- Natural place to implement timeouts per step

**Cons:**
- Orchestrator knows about all participants — creates coupling
- Risk of orchestrator becoming a "smart" service that leaks business logic
- Orchestrator is a new service to operate and scale

**When to use orchestration:** complex sagas with 4+ steps; when auditability matters; when you need timeout control; regulated systems where you need a clear audit trail.

---

## Saga State Machine

### Happy Path

```
PENDING
  |
  |--> [Reserve Stock] --> STOCK_RESERVED
                               |
                               |--> [Charge Payment] --> PAYMENT_CHARGED
                                                              |
                                                              |--> [Ship Order] --> COMPLETED
```

### Compensation Path

```
PENDING
  |
  |--> [Reserve Stock] --> STOCK_RESERVED
                               |
                               |--> [Charge Payment] --> PAYMENT_FAILED
                                                              |
                                                              |--> [Compensation: Release Stock]
                                                                          |
                                                                          |--> STOCK_RELEASED --> CANCELLED
```

### State Table

| State                  | Trigger                  | Next State             | Action                      |
|------------------------|--------------------------|------------------------|-----------------------------|
| PENDING                | Start                    | STOCK_RESERVED         | Send ReserveStock command   |
| STOCK_RESERVED         | StockReserved event      | PAYMENT_CHARGED        | Send ChargePayment command  |
| STOCK_RESERVED         | StockReservationFailed   | CANCELLED              | (nothing to compensate yet) |
| PAYMENT_CHARGED        | PaymentCharged event     | SHIPPED                | Send ShipOrder command      |
| PAYMENT_CHARGED        | PaymentFailed event      | COMPENSATING           | Send ReleaseStock command   |
| COMPENSATING           | StockReleased event      | CANCELLED              | Mark saga cancelled         |
| SHIPPED                | OrderShipped event       | COMPLETED              | Mark saga complete          |

---

## Compensating Transactions

### Rules

1. Every forward step must have a compensation defined **before** you implement the step
2. Compensations are **not rollbacks** — the local transaction already committed; the compensation is a new transaction that undoes the effect
3. Compensations must also be **idempotent** — they may be called more than once
4. The orchestrator must persist which compensations need to run if it crashes mid-compensation

### Compensation Mapping

| Forward Transaction    | Compensation                  | Notes                              |
|------------------------|-------------------------------|------------------------------------|
| ReserveStock           | ReleaseStock                  | Decrement reservation count        |
| ChargePayment          | RefundPayment                 | Issue refund via payment provider  |
| CreateShipment         | CancelShipment                | Only possible before handoff       |
| SendEmail              | (no compensation)             | Pivotal; send apology email instead|
| ShipPhysicalGoods      | (no compensation)             | Pivotal; cannot un-ship            |

### The Pivotal Transaction

The **pivotal transaction** is the point of no return — the step that cannot be compensated (e.g., physical shipment, external API call that is irreversible).

```
Step 1: ReserveStock     <-- can compensate
Step 2: ChargePayment    <-- can compensate
Step 3: ShipGoods        <-- PIVOTAL: cannot compensate
Step 4: SendConfirmation <-- after pivot: retry until success
```

After the pivotal transaction, the only option is to **retry until success** — never compensate.

---

## Process Manager (Complex Sagas)

When sagas have parallel steps, conditional flows, or timeouts, use a **Process Manager** pattern.

```
                +-------------------+
                |  Process Manager  |
                |  saga_id: abc123  |
                |  state: RUNNING   |
                |  step_results: {} |
                |  compensations: []|
                +--------+----------+
                         |
          +--------------+--------------+
          |                             |
    [Reserve Stock]              [Reserve Flight]    <-- parallel
          |                             |
    StockReserved              FlightReserved
          |                             |
          +--------------+--------------+
                         |
                  Both completed?
                         |
                  [ChargePayment]
```

Persistent state in DB:
```sql
CREATE TABLE saga_instance (
    saga_id        UUID PRIMARY KEY,
    saga_type      VARCHAR(100) NOT NULL,
    current_state  VARCHAR(100) NOT NULL,
    payload        JSONB,                    -- saga input data
    step_results   JSONB DEFAULT '{}',       -- results from each step
    compensations  JSONB DEFAULT '[]',       -- compensations to run on failure
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    updated_at     TIMESTAMPTZ DEFAULT NOW(),
    timeout_at     TIMESTAMPTZ              -- when to trigger compensation if stuck
);
```

---

## Implementation: Java + Spring + Kafka (Orchestration)

### Saga State Entity

```java
@Entity
@Table(name = "order_saga")
public class OrderSaga {

    @Id
    private UUID sagaId;

    @Enumerated(EnumType.STRING)
    private SagaState state;

    private UUID orderId;
    private UUID customerId;
    private BigDecimal amount;

    @ElementCollection
    @CollectionTable(name = "saga_compensation_log")
    private List<String> completedSteps = new ArrayList<>();

    private LocalDateTime timeoutAt;

    public enum SagaState {
        PENDING, STOCK_RESERVED, PAYMENT_CHARGED, SHIPPED, COMPLETED,
        COMPENSATING_PAYMENT, COMPENSATING_STOCK, CANCELLED, FAILED
    }
}
```

### Saga Orchestrator Service

```java
@Service
@Slf4j
@Transactional
public class OrderSagaOrchestrator {

    private final OrderSagaRepository sagaRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Clock clock;

    // Step 1: start the saga
    public void startSaga(CreateOrderCommand cmd) {
        OrderSaga saga = new OrderSaga();
        saga.setSagaId(UUID.randomUUID());
        saga.setOrderId(cmd.orderId());
        saga.setCustomerId(cmd.customerId());
        saga.setAmount(cmd.amount());
        saga.setState(SagaState.PENDING);
        saga.setTimeoutAt(clock.instant().plus(Duration.ofMinutes(30)));
        sagaRepository.save(saga);

        // send first command
        kafkaTemplate.send("inventory.commands", new ReserveStockCommand(
            saga.getSagaId(), cmd.orderId(), cmd.items()
        ));

        log.info("Saga {} started for order {}", saga.getSagaId(), cmd.orderId());
    }

    // Step 2: handle stock reserved
    @KafkaListener(topics = "inventory.events")
    public void onInventoryEvent(InventoryEvent event) {
        OrderSaga saga = sagaRepository.findBySagaId(event.sagaId())
            .orElseThrow(() -> new SagaNotFoundException(event.sagaId()));

        if (event instanceof StockReservedEvent reserved) {
            if (saga.getState() != SagaState.PENDING) {
                log.warn("Duplicate StockReservedEvent for saga {} — ignoring", saga.getSagaId());
                return; // idempotency guard
            }
            saga.setState(SagaState.STOCK_RESERVED);
            saga.getCompletedSteps().add("RESERVE_STOCK");
            sagaRepository.save(saga);

            kafkaTemplate.send("payment.commands", new ChargePaymentCommand(
                saga.getSagaId(), saga.getCustomerId(), saga.getAmount()
            ));

        } else if (event instanceof StockReservationFailedEvent) {
            saga.setState(SagaState.CANCELLED);
            sagaRepository.save(saga);
            log.info("Saga {} cancelled: stock unavailable", saga.getSagaId());
        }
    }

    // Step 3: handle payment result
    @KafkaListener(topics = "payment.events")
    public void onPaymentEvent(PaymentEvent event) {
        OrderSaga saga = sagaRepository.findBySagaId(event.sagaId())
            .orElseThrow(() -> new SagaNotFoundException(event.sagaId()));

        if (event instanceof PaymentChargedEvent charged) {
            if (saga.getState() != SagaState.STOCK_RESERVED) {
                log.warn("Duplicate PaymentChargedEvent for saga {} — ignoring", saga.getSagaId());
                return;
            }
            saga.setState(SagaState.PAYMENT_CHARGED);
            saga.getCompletedSteps().add("CHARGE_PAYMENT");
            sagaRepository.save(saga);

            // PIVOTAL TRANSACTION: after this, we retry — never compensate
            kafkaTemplate.send("shipping.commands", new ShipOrderCommand(
                saga.getSagaId(), saga.getOrderId()
            ));

        } else if (event instanceof PaymentFailedEvent) {
            // trigger compensation: reverse stock reservation
            saga.setState(SagaState.COMPENSATING_STOCK);
            sagaRepository.save(saga);

            kafkaTemplate.send("inventory.commands", new ReleaseStockCommand(
                saga.getSagaId(), saga.getOrderId()
            ));
        }
    }

    // Compensation: handle stock released
    @KafkaListener(topics = "inventory.compensation.events")
    public void onStockReleased(StockReleasedEvent event) {
        OrderSaga saga = sagaRepository.findBySagaId(event.sagaId())
            .orElseThrow(() -> new SagaNotFoundException(event.sagaId()));

        saga.setState(SagaState.CANCELLED);
        sagaRepository.save(saga);
        log.info("Saga {} compensated and cancelled", saga.getSagaId());
    }

    // Timeout handler — run periodically
    @Scheduled(fixedDelay = 60_000)
    public void handleTimeouts() {
        List<OrderSaga> timedOut = sagaRepository.findByTimeoutAtBeforeAndStateIn(
            LocalDateTime.now(),
            List.of(SagaState.PENDING, SagaState.STOCK_RESERVED, SagaState.PAYMENT_CHARGED)
        );

        for (OrderSaga saga : timedOut) {
            log.warn("Saga {} timed out in state {}", saga.getSagaId(), saga.getState());
            triggerCompensation(saga);
        }
    }

    private void triggerCompensation(OrderSaga saga) {
        // compensate in reverse order of completedSteps
        List<String> steps = new ArrayList<>(saga.getCompletedSteps());
        Collections.reverse(steps);

        for (String step : steps) {
            switch (step) {
                case "CHARGE_PAYMENT" -> kafkaTemplate.send("payment.commands",
                    new RefundPaymentCommand(saga.getSagaId(), saga.getAmount()));
                case "RESERVE_STOCK" -> kafkaTemplate.send("inventory.commands",
                    new ReleaseStockCommand(saga.getSagaId(), saga.getOrderId()));
            }
        }
        saga.setState(SagaState.COMPENSATING_PAYMENT);
        sagaRepository.save(saga);
    }
}
```

### Idempotent Consumer Pattern

```java
@KafkaListener(topics = "inventory.commands")
public void handleReserveStock(ReserveStockCommand cmd) {
    // idempotency check: have we already processed this saga step?
    if (processedCommandRepository.existsBySagaIdAndStep(cmd.sagaId(), "RESERVE_STOCK")) {
        log.info("Duplicate ReserveStockCommand for saga {} — publishing previous result", cmd.sagaId());
        // re-publish the result so the orchestrator can continue
        republishResult(cmd.sagaId(), "RESERVE_STOCK");
        return;
    }

    // process
    try {
        inventoryService.reserve(cmd.orderId(), cmd.items());
        processedCommandRepository.save(new ProcessedCommand(cmd.sagaId(), "RESERVE_STOCK", "SUCCESS"));
        kafkaTemplate.send("inventory.events", new StockReservedEvent(cmd.sagaId()));
    } catch (InsufficientStockException e) {
        processedCommandRepository.save(new ProcessedCommand(cmd.sagaId(), "RESERVE_STOCK", "FAILED"));
        kafkaTemplate.send("inventory.events", new StockReservationFailedEvent(cmd.sagaId(), e.getMessage()));
    }
}
```

---

## Alternative Implementations

### Axon Framework (Java)

```java
@Saga
public class OrderSaga {

    @Autowired
    private transient CommandGateway commandGateway;

    private String orderId;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void on(OrderPlacedEvent event) {
        this.orderId = event.getOrderId();
        SagaLifecycle.associateWith("sagaId", event.getSagaId());
        commandGateway.send(new ReserveStockCommand(event.getSagaId(), event.getOrderId()));
    }

    @SagaEventHandler(associationProperty = "sagaId")
    public void on(StockReservedEvent event) {
        commandGateway.send(new ChargePaymentCommand(event.getSagaId(), event.getAmount()));
    }

    @SagaEventHandler(associationProperty = "sagaId")
    public void on(PaymentFailedEvent event) {
        commandGateway.send(new ReleaseStockCommand(event.getSagaId(), this.orderId));
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "sagaId")
    public void on(OrderCompletedEvent event) {
        // saga ends; Axon removes it from the saga store
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "sagaId")
    public void on(OrderCancelledEvent event) {
        // compensation complete
    }
}
```

### Temporal (Workflow-as-Code)

```java
@WorkflowImpl
public class OrderSagaWorkflowImpl implements OrderSagaWorkflow {

    private final InventoryActivities inventory = Workflow.newActivityStub(
        InventoryActivities.class,
        ActivityOptions.newBuilder().setStartToCloseTimeout(Duration.ofMinutes(5)).build()
    );
    private final PaymentActivities payment = Workflow.newActivityStub(
        PaymentActivities.class,
        ActivityOptions.newBuilder().setStartToCloseTimeout(Duration.ofMinutes(5)).build()
    );

    @Override
    public void processOrder(OrderRequest request) {
        // Temporal handles retries, timeouts, and crash recovery automatically
        String reservationId = null;
        try {
            reservationId = inventory.reserveStock(request.orderId(), request.items());
            payment.chargePayment(request.customerId(), request.amount());
            // pivotal: no compensation after this
            shipping.shipOrder(request.orderId());
        } catch (PaymentException e) {
            // Temporal compensations: guaranteed to run even if workflow crashes here
            if (reservationId != null) {
                inventory.releaseStock(reservationId);
            }
            throw ApplicationFailure.newFailure("Payment failed, order cancelled", "OrderCancelled");
        }
    }
}
```

---

## Testing Sagas

### Unit Test — Mock Participants, Verify State Transitions

```java
@ExtendWith(MockitoExtension.class)
class OrderSagaOrchestratorTest {

    @Mock KafkaTemplate<String, Object> kafkaTemplate;
    @Mock OrderSagaRepository sagaRepository;
    @InjectMocks OrderSagaOrchestrator orchestrator;

    @Test
    void whenPaymentFails_thenStockCompensationSent() {
        OrderSaga saga = new OrderSaga();
        saga.setSagaId(UUID.randomUUID());
        saga.setState(SagaState.STOCK_RESERVED);
        saga.getCompletedSteps().add("RESERVE_STOCK");

        when(sagaRepository.findBySagaId(saga.getSagaId())).thenReturn(Optional.of(saga));

        orchestrator.onPaymentEvent(new PaymentFailedEvent(saga.getSagaId(), "Insufficient funds"));

        verify(kafkaTemplate).send(eq("inventory.commands"), any(ReleaseStockCommand.class));
        assertThat(saga.getState()).isEqualTo(SagaState.COMPENSATING_STOCK);
    }

    @Test
    void duplicateEvent_isIgnored() {
        OrderSaga saga = new OrderSaga();
        saga.setSagaId(UUID.randomUUID());
        saga.setState(SagaState.STOCK_RESERVED); // already past PENDING

        when(sagaRepository.findBySagaId(saga.getSagaId())).thenReturn(Optional.of(saga));

        orchestrator.onInventoryEvent(new StockReservedEvent(saga.getSagaId()));

        // should not advance state or send new command
        verify(kafkaTemplate, never()).send(anyString(), any());
    }
}
```

### Integration Test — Full Saga with TestContainers

```java
@SpringBootTest
@Testcontainers
class OrderSagaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Test
    void happyPath_orderCreated_sagaCompletes() throws Exception {
        // trigger saga
        sagaOrchestrator.startSaga(new CreateOrderCommand(orderId, customerId, items, amount));

        // wait for COMPLETED state (poll with timeout)
        await().atMost(Duration.ofSeconds(30)).until(() ->
            sagaRepository.findByOrderId(orderId)
                .map(s -> s.getState() == SagaState.COMPLETED)
                .orElse(false)
        );

        // verify side effects
        assertThat(inventoryService.getReservation(orderId)).isPresent();
        assertThat(paymentService.getCharge(orderId)).isPresent();
    }

    @Test
    void paymentServiceDown_compensationRunsAndOrderCancelled() throws Exception {
        // simulate payment service being unavailable
        paymentStub.returnError(500);

        sagaOrchestrator.startSaga(new CreateOrderCommand(orderId, customerId, items, amount));

        await().atMost(Duration.ofSeconds(30)).until(() ->
            sagaRepository.findByOrderId(orderId)
                .map(s -> s.getState() == SagaState.CANCELLED)
                .orElse(false)
        );

        // stock should be released
        assertThat(inventoryService.getReservation(orderId)).isEmpty();
    }
}
```

### Chaos Test — Kill Participant Mid-Saga

```bash
# Using Toxiproxy to simulate network partition mid-saga
# 1. Start the saga
curl -X POST /orders -d '{"items": [...], "amount": 99.99}'

# 2. After ReserveStock succeeds, kill the payment service
docker stop payment-service

# 3. Wait for timeout to trigger
sleep 1800  # saga timeout is 30 minutes

# 4. Verify compensation ran
curl /orders/{orderId} | jq '.state'
# Expected: "CANCELLED"

# 5. Verify stock was released
curl /inventory/{itemId}/reservations | jq '.[] | select(.orderId == "...")'
# Expected: empty
```

---

## Anti-Patterns

**Saga Without Idempotency**
- Kafka guarantees at-least-once delivery; duplicate messages will corrupt saga state
- Every step handler must check "have I already processed this?" before acting

**Compensation Without Testing**
- The compensation path is only triggered during failures — rarely exercised in dev/QA
- Test compensation explicitly; many teams discover compensation bugs during incidents

**Too Many Steps**
- Sagas with 10+ steps are fragile: 10 steps with 99% reliability each = 90% overall success
- Split into sub-sagas; group tightly related steps together

**Sharing a Database Between Saga Participants**
- If InventoryService and PaymentService share a DB, use a local transaction — not a saga
- Sagas are for services with independent databases and transports

**Synchronous Compensation**
- Calling compensation steps synchronously (HTTP) makes the orchestrator dependent on participant availability during compensation
- Send compensation commands via durable messaging (Kafka, SQS) so they survive crashes

**Not Persisting Saga State**
- If the orchestrator restarts, it must be able to resume from where it left off
- Always persist state before sending commands; use transactional outbox if needed

---

## Transactional Outbox Pattern (Reliable Message Publishing)

Avoid the dual-write problem: writing to DB and publishing to Kafka in the same operation.

```java
// WRONG: if Kafka publish fails, DB is committed but no event is sent
sagaRepository.save(saga);
kafkaTemplate.send("inventory.commands", command); // can fail independently

// CORRECT: write command to outbox table in the same DB transaction
@Transactional
public void advanceSaga(OrderSaga saga, Object command) {
    sagaRepository.save(saga);
    outboxRepository.save(new OutboxMessage(
        UUID.randomUUID(),
        "inventory.commands",
        objectMapper.writeValueAsString(command),
        Instant.now()
    ));
    // Debezium CDC picks up outbox inserts and publishes to Kafka
}
```

---

## Checklist

- [ ] Every step has a compensating transaction defined
- [ ] All event/command handlers are idempotent
- [ ] Saga state is persisted before sending commands (transactional outbox or equivalent)
- [ ] Timeout is defined for each saga; stuck sagas trigger compensation
- [ ] Pivotal transaction is identified; steps after it use retry-only (no compensation)
- [ ] Duplicate event handling tested explicitly
- [ ] Compensation path has integration tests
- [ ] Saga state is queryable for debugging and support
- [ ] Dead-letter queue is set up for unprocessable messages
