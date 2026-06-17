# /stream — Event Streaming Pipeline Design & Debug

Design, implement, or debug an event streaming pipeline — Kafka, stream processors, messaging brokers.

## What this command does

1. Designs the event topology (topics, consumer groups, partitioning strategy)
2. Selects appropriate streaming semantics (at-least-once vs exactly-once)
3. Implements producers with correct acks and idempotency settings
4. Implements consumers with proper offset management and error handling
5. Designs schema contracts with evolution strategy
6. Identifies and fixes common streaming bugs (lag, rebalance loops, poison pills)

## Skills activated

- `kafka-mastery` — full producer/consumer/streams expertise
- `event-streaming` — windowing, stateful processing, stream/table duality
- `messaging-brokers` — broker selection, topology patterns
- `domain-events` — outbox pattern for reliable publishing
- `observability-excellence` — consumer lag monitoring, stream metrics
- `resilience-patterns` — DLT, retry topics, poison pill handling

## Usage

```
/stream <describe the streaming problem>
```

Examples:
- `/stream design a real-time order processing pipeline with Kafka`
- `/stream implement exactly-once payment processing`
- `/stream debug consumer lag in the inventory service`
- `/stream add a dead letter topic to the notification consumer`
- `/stream design a Kafka Streams aggregation for hourly sales totals`

## Pipeline Design Template

```
                    PRODUCER SIDE
  ┌──────────┐    ┌─────────────┐    ┌─────────────────┐
  │  Domain  │───▶│   Outbox    │───▶│  Kafka Topic    │
  │  Event   │    │  (DB table) │    │  (partitioned   │
  └──────────┘    └─────────────┘    │   by key)       │
                    Outbox Relay      └────────┬────────┘
                                              │
                    CONSUMER SIDE             ▼
  ┌──────────┐    ┌─────────────┐    ┌─────────────────┐
  │ Handler  │◀───│  Consumer   │◀───│  Consumer Group │
  │ (idempot)│    │  (manual    │    │  (sticky        │
  └──────────┘    │   commit)   │    │   rebalancer)   │
                  └──────┬──────┘    └─────────────────┘
                         │ on error
                         ▼
                  ┌─────────────┐
                  │  Retry      │
                  │  Topic      │─▶ DLT after N retries
                  └─────────────┘
```

## Rules

- Never use `enable.auto.commit=true` in production consumers
- Always use idempotent producers (`enable.idempotence=true`)
- Partition by business key that requires ordering (e.g., order ID, user ID)
- Schema changes must be backward-compatible (add optional fields only)
- Consumer lag is the primary health signal — alert on it
