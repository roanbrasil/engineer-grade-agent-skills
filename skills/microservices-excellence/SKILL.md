---
name: microservices-excellence
description: Production-hardened microservices guidance — service decomposition, inter-service communication, API Gateway, distributed tracing, data ownership, and anti-patterns
---

# Microservices Excellence

You are an expert in production microservices architecture. Apply battle-tested decomposition strategies, enforce data ownership boundaries, and prevent the most common distributed systems pitfalls.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         External Clients                         │
│                  (Mobile, Web, Third-party)                      │
└──────────────────────────────┬───────────────────────────────────┘
                               │ HTTPS
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                          API Gateway                             │
│          Auth offloading · Rate limiting · Routing               │
│          Request/Response transformation · TLS termination       │
└────────┬──────────────┬────────────────┬──────────────┬──────────┘
         │              │                │              │
         ▼              ▼                ▼              ▼
┌──────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────┐
│   User       │ │  Order     │ │  Inventory │ │  Notification  │
│   Service    │ │  Service   │ │  Service   │ │  Service       │
│              │ │            │ │            │ │                │
│  [User DB]   │ │  [Order DB]│ │  [Inv DB]  │ │  [Notif DB]    │
└──────────────┘ └─────┬──────┘ └────────────┘ └────────────────┘
                        │
                        │ Domain Events (Kafka/RabbitMQ)
                        ▼
              ┌──────────────────┐
              │   Message Broker │
              │   (Async Bus)    │
              └───────┬──────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│ Inventory       │     │ Notification    │
│ (order.created) │     │ (order.created) │
└─────────────────┘     └─────────────────┘
```

---

## 1. Service Decomposition

### Bounded Context → Service Boundary

```
Domain Model Analysis
       │
       ▼
Identify Bounded Contexts (DDD)
       │
       ├── Each BC has its own ubiquitous language
       ├── Same word means different thing in each BC
       │   (e.g., "Account" in Banking BC vs Auth BC)
       │
       ▼
Map BCs to candidate services
       │
       ▼
Apply decomposition criteria:
  ✓ Single business capability
  ✓ Independent deployability
  ✓ Team can own end-to-end
  ✓ Separate data store
  ✗ If services always deploy together → probably one service
  ✗ If all calls are synchronous chains → probably one service
```

**Decomposition heuristics:**
- Decompose by **business capability**, not by technical layer (not "UserController service")
- Each service should be able to answer its core questions without calling another service
- Start coarser-grained; split only when: different scaling needs, different deployment frequency, different team ownership

### SCS (Self-Contained System) vs Microservice

```
Self-Contained System (SCS):
- Includes its own UI fragment
- Owns a complete vertical slice of functionality
- 2–12 developers own it
- Looser inter-system coupling, each SCS is a mini-monolith
- Better for teams new to distributed systems

Microservice:
- Single business capability
- 1–3 developers typically
- No UI, pure API/event consumer
- Finer granularity, more network calls
- Requires sophisticated infra (service mesh, distributed tracing)

CHOOSE SCS when:    Multiple bounded contexts, limited infra maturity
CHOOSE Microservice when: Clear single capability, strong DevOps culture
```

---

## 2. Inter-Service Communication

### Synchronous: REST (OpenAPI Contract)

```yaml
# openapi.yaml — the contract is the source of truth
openapi: "3.1.0"
info:
  title: Order Service API
  version: "1.0.0"
paths:
  /v1/orders/{orderId}:
    get:
      operationId: getOrder
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Order"
        "404":
          $ref: "#/components/responses/NotFound"
```

```
REST Communication Rules:
- Use OpenAPI contract, generate client SDK from spec (not hand-coded)
- Timeout on every call — never infinite timeout
- Always implement circuit breaker on outbound calls
- Version your API (/v1/, /v2/) — never break existing callers
- Use HATEOAS for resource discovery when appropriate
```

### Synchronous: gRPC (Protobuf)

```protobuf
// inventory.proto
syntax = "proto3";
package inventory.v1;

service InventoryService {
  rpc CheckStock (CheckStockRequest) returns (CheckStockResponse);
  rpc ReserveItems (ReserveItemsRequest) returns (ReserveItemsResponse);
  rpc StreamInventoryUpdates (InventoryFilter) returns (stream InventoryUpdate);
}

message CheckStockRequest {
  repeated string sku_ids = 1;
}

message CheckStockResponse {
  map<string, int32> available_quantities = 1;
}
```

```
gRPC vs REST:
USE gRPC when:
  - Service-to-service (not browser-facing)
  - High throughput, low latency critical
  - Bidirectional streaming needed
  - Strong typing across polyglot services

USE REST when:
  - Browser clients need access
  - Simpler debugging needed (human-readable)
  - External partner integrations
```

### Asynchronous: Domain Events via Message Broker

```
Event-Driven Communication Pattern:

Producer (Order Service)               Broker (Kafka)
┌─────────────────────┐               ┌──────────────┐
│ OrderCreatedEvent   │──publish───►  │ topic:       │
│ {                   │               │ order.events │
│   orderId: uuid,    │               └──────┬───────┘
│   customerId: uuid, │                      │
│   items: [...],     │                      │ subscribe
│   totalAmount: 150  │               ┌──────┴──────────────────┐
│ }                   │               │                          │
└─────────────────────┘        ┌──────┴──────┐         ┌───────┴─────┐
                                │ Inventory  │         │ Notification │
                                │ Service    │         │ Service      │
                                │ reserves   │         │ sends email  │
                                │ stock      │         │              │
                                └────────────┘         └─────────────┘
```

```java
// Publishing a domain event (Spring + Kafka)
@Service
@RequiredArgsConstructor
public class OrderService {

    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    @Transactional
    public Order createOrder(CreateOrderCommand cmd) {
        Order order = orderRepository.save(Order.from(cmd));

        // publish event after successful DB write
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .occurredAt(Instant.now())
            .orderId(order.getId().toString())
            .customerId(order.getCustomerId().toString())
            .items(order.getItems().stream().map(Item::toEventItem).toList())
            .totalAmount(order.getTotalAmount())
            .build();

        kafkaTemplate.send("order.events", order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish OrderCreatedEvent for order {}", order.getId(), ex);
                    // Consider outbox pattern for guaranteed delivery
                }
            });

        return order;
    }
}

// Consuming domain events
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final InventoryService inventoryService;

    @KafkaListener(topics = "order.events", groupId = "inventory-service")
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event, Acknowledgment ack) {
        try {
            inventoryService.reserveItems(event.getOrderId(), event.getItems());
            ack.acknowledge();
        } catch (InsufficientStockException e) {
            // publish OrderReservationFailed event → triggers saga compensation
            eventPublisher.publish(new OrderReservationFailedEvent(event.getOrderId(), e.getSku()));
            ack.acknowledge();  // ack anyway — don't reprocess indefinitely
        }
    }
}
```

### Saga Pattern for Distributed Transactions

```
Choreography-based Saga (event-driven):

Order Service          Inventory Service      Payment Service
     │                        │                      │
     │── OrderCreated ──────►│                      │
     │                        │── StockReserved ───►│
     │                        │                      │── PaymentProcessed ──►
     │                        │                      │                       (done)
     │
Compensation (rollback):
     │                        │                      │
     │                        │                      │── PaymentFailed ──────►
     │◄── OrderFailed ────────│◄── StockReleased ───│
     │                        │                      │

Orchestration-based Saga (explicit coordinator):

     ┌────────────────────────────────┐
     │       Order Saga Orchestrator  │
     │                                │
     │  1. reserve_inventory()        │
     │  2. process_payment()          │
     │  3. confirm_order()            │
     │  On failure: compensate all    │
     └────────────────────────────────┘
           │           │           │
     Inventory     Payment      Order
     Service       Service      Service
```

---

## 3. API Gateway

```
Client Request Flow:
                    ┌──────────────────────────────────┐
Request  ──────────►│            API Gateway           │
                    │                                  │
                    │  1. TLS termination              │
                    │  2. Authentication (JWT verify)  │
                    │  3. Authorization (RBAC check)   │
                    │  4. Rate limiting (per user/IP)  │
                    │  5. Request ID injection         │
                    │  6. Route matching               │
                    │  7. Load balancing               │
                    │  8. Response transformation      │
                    └──────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              User Service    Order Service   Product Service
```

**Gateway responsibilities — yes vs no:**

```
GATEWAY SHOULD:                    GATEWAY SHOULD NOT:
- Auth token verification          - Business logic
- Rate limiting                    - Data aggregation (BFF pattern instead)
- SSL termination                  - Complex transformations
- Request routing                  - Service orchestration
- CORS headers                     - Heavy processing
- Request/response logging
- API versioning redirect
```

---

## 4. Service Discovery

### Client-Side (Eureka)

```
┌──────────────┐    register    ┌──────────────┐
│ Order Service│ ─────────────► │    Eureka    │
│ 10.0.1.5:8080│               │    Server    │
└──────────────┘                │  (registry)  │
                                └──────────────┘
                                       ▲
┌──────────────┐   lookup instances    │
│ User Service │ ──────────────────────┘
│  (consumer)  │
│              │  ──► picks instance ──► calls directly
└──────────────┘
```

### Server-Side (Kubernetes Service + Ingress)

```
┌──────────────┐                ┌───────────────────┐
│  Client Pod  │ ─── DNS ─────► │  Kubernetes       │
│              │                │  Service (ClusterIP│
└──────────────┘                │  orders-svc:8080)  │
                                └──────────┬────────┘
                                           │ kube-proxy
                               ┌───────────┼───────────┐
                               ▼           ▼           ▼
                         Order Pod1   Order Pod2   Order Pod3

PREFER Kubernetes Service discovery:
- No client library needed
- Works with any language
- iptables/ipvs load balancing
- Health checks handled by Kubernetes
```

---

## 5. Distributed Tracing

```
Request flow with trace propagation:
                                          Trace ID: abc123
API Gateway ──────────────────────────────────────────────────►
   │  span_id: 001                                            │
   │  traceparent: 00-abc123-001-01                           │
   ▼                                                          │
Order Service                                                  │
   │  span_id: 002                                            │
   │  parent_span: 001                                        │
   │                                                          │
   ├── calls Inventory Service                                 │
   │     span_id: 003, parent: 002                            │
   │     (DB query span 004, parent: 003)                     │
   │                                                          │
   └── publishes Kafka event                                   │
         span_id: 005, parent: 002                            ▼
```

```java
// OpenTelemetry configuration (Spring Boot)
// application.yaml
management:
  tracing:
    sampling:
      probability: 0.1     # 10% sampling in production
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces

// Add custom spans
@Service
public class PaymentService {

    private final Tracer tracer;

    public PaymentResult processPayment(PaymentRequest request) {
        Span span = tracer.nextSpan()
            .name("payment.process")
            .tag("payment.provider", request.getProvider())
            .tag("payment.amount", request.getAmount().toString())
            .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            return paymentGateway.charge(request);
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}

// Propagate trace context with Kafka
@Bean
public ProducerFactory<String, Object> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    // ... kafka config ...
    return new DefaultKafkaProducerFactory<>(config,
        new StringSerializer(),
        new JsonSerializer<>());
    // Spring Kafka auto-propagates trace context headers
}
```

### Baggage (Cross-Service Context)

```java
// Pass correlation data across service boundaries
// In entry point (API Gateway or first service):
Baggage.current().toBuilder()
    .put("tenant.id", tenantId)
    .put("user.id", userId)
    .build()
    .makeCurrent();

// In any downstream service:
String tenantId = Baggage.current().getEntryValue("tenant.id");
// available without explicitly passing in every request
```

---

## 6. Service-to-Service Authentication

```
mTLS (Mutual TLS):
  Service A                              Service B
     │                                      │
     │── presents client cert ─────────────►│
     │◄─ presents server cert ──────────────│
     │    both verify each other's cert      │
     │── encrypted + authenticated ─────────►│

JWT Service Tokens:
  Service A                Identity Provider         Service B
     │── req service token ──────────────────►          │
     │◄─ JWT (exp 5min) ───────────────────────          │
     │── call + Bearer token ───────────────────────────►│
     │                                                   │
     │                                         validates JWT
     │                                         checks iss/aud

SPIFFE/SPIRE (production-grade):
  - Platform-agnostic workload identity
  - Automatic cert rotation
  - Works across K8s, VMs, bare metal
  - SVID (SPIFFE Verifiable Identity Document)
```

---

## 7. Data Ownership

```
RULE: Every service owns its data. No other service touches its database.

CORRECT:                              WRONG:
┌──────────────┐                    ┌──────────────┐   ┌──────────────┐
│ Order Service│                    │ Order Service│   │ User Service │
│              │                    │              │   │              │
│  [orders DB] │                    │  [orders DB] │   │              │
└──────────────┘                    └─────┬────────┘   └─────┬────────┘
                                          │                   │
                                          ▼                   ▼
                                    ┌───────────────────────────┐
                                    │       SHARED DATABASE     │  ← ANTI-PATTERN
                                    │   (couples all services)  │
                                    └───────────────────────────┘

Data replication approach:
- User Service owns user data
- Order Service needs user email for order confirmation
  → Subscribe to UserUpdatedEvent, maintain local read-only copy
  → Eventual consistency is acceptable here
  → Never call User Service synchronously to get email for every order display
```

### Eventual Consistency Patterns

```
Pattern: Read your own writes
Problem: After creating order, user reads it back — might get stale data

Solutions:
1. Write-through cache: update cache on write, serve from cache on read
2. Read from primary replica for 1-2s after write (sticky session to primary)
3. Optimistic UI: show write immediately in frontend, reconcile on refresh
4. Event sourcing: event log is the source of truth, views are projections

Pattern: Idempotent consumers
Every message consumer must handle duplicate delivery:

@KafkaListener(topics = "order.events")
public void handleOrderCreated(OrderCreatedEvent event) {
    if (processedEventRepo.exists(event.getEventId())) {
        log.warn("Duplicate event {}, skipping", event.getEventId());
        return;
    }
    // process...
    processedEventRepo.save(event.getEventId(), Instant.now());
}
```

---

## 8. Team Topology Alignment

```
Conway's Law:
"Organizations which design systems are constrained to produce designs
 which are copies of the communication structures of those organizations."

Implication: Mirror service boundaries to team structure

WRONG: Organize by technical layer
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Frontend Team│  │ Backend Team │  │  DB Team     │
│ (all UI)     │  │ (all APIs)   │  │ (all schemas)│
└──────────────┘  └──────────────┘  └──────────────┘
→ Every feature requires coordination between 3 teams

CORRECT: Organize by business capability (Inverse Conway Maneuver)
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Order Team   │  │  User Team   │  │ Catalog Team │
│ (full stack) │  │ (full stack) │  │ (full stack) │
│ owns order   │  │ owns user    │  │ owns catalog │
│ service + DB │  │ service + DB │  │ service + DB │
└──────────────┘  └──────────────┘  └──────────────┘
→ Team ships features independently
```

---

## 9. Deployment

### Independent Deployability

```
Release strategy:
1. Never require coordinated deploys across services
2. API changes must be backward-compatible
3. Use expand-contract (parallel run) for breaking changes

Expand-Contract for breaking API change:
Phase 1 (Expand):    Add new field/endpoint, keep old one
Phase 2 (Migrate):   Clients update to use new field/endpoint
Phase 3 (Contract):  Remove old field/endpoint

Example: rename "customerId" → "accountId"
  v1: return both {customerId: "x", accountId: "x"}  ← expansion
  v2: consumers updated
  v3: remove customerId                                ← contract
```

### Contract Testing (Pact)

```
Traditional integration testing:             Contract testing (Pact):
  Consumer ─── network ──► Provider            Consumer writes contract
  (fragile, slow, needs both up)               Provider verifies contract
                                               (no network, fast, independent)

Consumer (Order Service):
  - Defines: "I expect GET /users/{id} to return {id, email, name}"
  - Publishes contract to Pact Broker

Provider (User Service):
  - Downloads contract from Pact Broker
  - Runs verification: does our API satisfy this contract?
  - Breaks build if verification fails
```

```java
// Consumer test (Order Service)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service")
class UserServiceContractTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact getUserPact(PactDslWithProvider builder) {
        return builder
            .given("user with id abc123 exists")
            .uponReceiving("a request for user abc123")
                .method("GET")
                .path("/v1/users/abc123")
            .willRespondWith()
                .status(200)
                .body(newJsonBody(body -> {
                    body.stringValue("id", "abc123");
                    body.stringValue("email", "user@example.com");
                    body.stringValue("firstName", "John");
                }).build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getUserPact")
    void testGetUser(MockServer mockServer) {
        UserClient client = new UserClient(mockServer.getUrl());
        User user = client.getUser("abc123");
        assertThat(user.getEmail()).isEqualTo("user@example.com");
    }
}
```

---

## 10. Anti-Patterns

### Distributed Monolith

```
SYMPTOM: Microservices that must always deploy together
         or are tightly coupled via synchronous chains

┌───────────┐  sync  ┌───────────┐  sync  ┌───────────┐
│ Service A │ ──────► │ Service B │ ──────► │ Service C │
└───────────┘         └───────────┘         └───────────┘
(every A request blocks on B which blocks on C — latency adds up)

CAUSES:
- Shared database
- Synchronous call chains deeper than 2 hops
- Services deployed from the same pipeline

FIX:
- Introduce async events for non-critical paths
- Cache data locally instead of calling upstream
- Identify which calls truly need to be synchronous
```

### Chatty Services

```
SYMPTOM: 20 API calls to render one page

FIX 1: BFF (Backend for Frontend)
  - One endpoint per screen aggregates multiple service calls
  - Reduces client-facing round trips to 1

FIX 2: GraphQL Federation
  - Single GraphQL gateway, each service exposes a subgraph
  - Client fetches exactly what it needs in one request

FIX 3: Increase service granularity
  - Some services are too fine-grained; merge related capabilities
```

### Shared Database (MOST DANGEROUS)

```
WHAT IT LOOKS LIKE:
  Multiple services → same schema → same tables

PROBLEMS:
  - Schema change in one service breaks all others
  - No service can scale its DB independently
  - No clear ownership of data
  - Migrations require coordination across all teams

HOW IT STARTS:
  "We'll just share the users table" → slippery slope

FIX:
  - Each service gets its own schema (even in same DB engine at first)
  - Eventually: separate DB instances
  - Replicate needed data via events, not shared tables
```

### Synchronous Chain Anti-Pattern

```
PROBLEM: A → B → C → D (4-hop synchronous chain)
  Availability: 0.99^4 = 96% even if each service is 99% available
  Latency: adds up multiplicatively
  Coupling: D being slow makes A slow

SOLUTION:
  - Async: use events for non-blocking paths
  - Cache: B can cache D's data locally
  - Aggregate: if B always needs D's data, consider merging services
  - Timeout + circuit breaker on every hop
```

---

## Checklist

### Service Decomposition

- [ ] Service maps to a single bounded context
- [ ] Team can deploy service independently without coordinating with others
- [ ] Service owns its data — no shared tables with other services
- [ ] Service can answer its core questions without calling another service at runtime
- [ ] OpenAPI contract defined and published to contract registry

### Communication

- [ ] Timeout set on every synchronous call
- [ ] Circuit breaker on every outbound synchronous call
- [ ] Message consumers are idempotent (handle duplicate delivery)
- [ ] Domain events have a unique `eventId` for deduplication
- [ ] Kafka consumer group ID matches consuming service name

### Observability

- [ ] OpenTelemetry tracing configured with sampling
- [ ] Trace ID propagated in all outbound HTTP and Kafka headers
- [ ] Structured logging includes traceId, spanId, serviceId fields
- [ ] Health endpoint verified by readiness probe before traffic routing

### Security

- [ ] Service-to-service calls authenticated (mTLS or JWT service tokens)
- [ ] API Gateway validates auth before routing to services
- [ ] Services do not accept unauthenticated calls from within the cluster (zero-trust)

### Resilience

- [ ] Circuit breaker configured on all synchronous outbound calls
- [ ] Retry with exponential backoff + jitter on transient failures
- [ ] Bulkhead isolates critical from non-critical paths
- [ ] Graceful degradation defined when each dependency is unavailable
