# /implement — Polyglot Feature Implementation

Build a feature or component with the right language, framework, and patterns applied from the start.

## What this command does

1. Selects the right language + framework idioms
2. Applies the architecture agreed in `/architect` (or infers from context)
3. Writes production-ready code, not scaffolding
4. Integrates messaging/streaming if the feature requires it
5. Leaves entry points for tests (no tangled dependencies)

## Skills activated

Based on context, activates the relevant subset of:

**Language skills** (auto-detected from project):
- `rust-systems` | `java-excellence` | `kotlin-craft` | `python-production` | `cpp-modern`

**Framework skills**:
- `spring-boot-expert` — Java/Kotlin + Spring Boot 3
- `fastapi-production` — Python + FastAPI + Pydantic v2

**Architecture skills**:
- `ports-and-adapters` — keeps implementation inside the hexagon
- `clean-code` — naming, function size, single responsibility

**Messaging** (when relevant):
- `kafka-mastery` — Kafka producers/consumers/streams
- `messaging-brokers` — RabbitMQ, SQS, NATS

## Usage

```
/implement <what to build>
```

Examples:
- `/implement the PlaceOrder use case with Kafka event publishing`
- `/implement a Kafka consumer for inventory reservations in Kotlin`
- `/implement a FastAPI endpoint with JWT auth and Pydantic validation`
- `/implement a circuit breaker around the payment service call`

## The Implementation Cycle

```
  Understand       Define Port      Implement       Write Tests
  the use case  →  interface     →  use case     →  (see /verify)
                   (no infra!)      logic

  Implement        Wire Adapter     Integration
  Adapter       →  in DI config  →  Test Adapter
  (infra code)
```

## Rules

- Never import framework types into the application core
- Always return domain types, never persistence types
- Every public method has a test entry point
- No `new` in business logic (use dependency injection / factory)
