# /architect — Domain & System Design

Design the domain model and system architecture before writing a single line of implementation code.

## What this command does

1. Identifies bounded contexts from the problem description
2. Models the core domain using DDD tactical patterns (aggregates, entities, value objects, domain events)
3. Selects the right architectural style (Hexagonal, Clean, CQRS+ES, or simpler)
4. Maps service boundaries and integration points
5. Produces a lightweight Architecture Decision Record (ADR)

## Skills activated

- `ddd-tactical` — ubiquitous language, aggregate design, context mapping
- `clean-architecture` — dependency rule, layer responsibilities
- `ports-and-adapters` — driving vs driven ports, adapter boundaries
- `domain-events` — what events the domain raises and why
- `cqrs-event-sourcing` — when read/write separation adds value
- `microservices-excellence` — when to split, when to stay monolithic
- `agents-integration-patterns` — when the system involves AI agents

## Usage

```
/architect <describe what you're building>
```

Examples:
- `/architect an order management system for an e-commerce platform`
- `/architect a real-time fraud detection pipeline using Kafka`
- `/architect a multi-agent system for automated code review`

## Output

```
BOUNDED CONTEXTS
  ├── Ordering (core domain)
  ├── Inventory (supporting)
  └── Notifications (generic)

AGGREGATE: Order
  Root: Order
  Entities: OrderLine
  Value Objects: Money, Address, SKU
  Domain Events: OrderPlaced, OrderShipped, OrderCancelled

ARCHITECTURE: Hexagonal
  Primary Ports: PlaceOrderUseCase, CancelOrderUseCase
  Secondary Ports: OrderRepository, PaymentGateway, EventPublisher
  Adapters (primary): REST controller, CLI, message consumer
  Adapters (secondary): PostgreSQL repo, Stripe client, Kafka producer

INTEGRATION POINTS
  → Payment service (sync, REST)
  → Inventory service (async, domain events)
  → Notification service (async, domain events)

ADR-001: Use hexagonal architecture to isolate domain from Spring Boot
```
