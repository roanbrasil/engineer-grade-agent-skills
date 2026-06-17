# /verify — Test-Driven Verification

Prove the implementation works. Tests are proof — "seems right" is not done.

## What this command does

1. Writes unit tests for domain logic (fast, no I/O)
2. Writes integration tests for adapters (real DB/broker)
3. Writes acceptance tests for use cases (BDD scenarios if applicable)
4. Runs the full test suite and reports failures
5. Checks test coverage is meaningful (not just high)

## Skills activated

- `tdd-mastery` — Red-Green-Refactor, test doubles, test smells
- `bdd-specification` — Gherkin scenarios → executable specs
- `domain-events` — verify events are raised correctly
- `kafka-mastery` — embedded Kafka for consumer/producer tests
- `spring-boot-expert` — @DataJpaTest, TestContainers, @WebMvcTest
- `fastapi-production` — pytest, AsyncClient, fixture patterns

## Usage

```
/verify <what to test>
```

Examples:
- `/verify the PlaceOrder use case`
- `/verify the Kafka consumer handles poison messages correctly`
- `/verify the OrderRepository integration with PostgreSQL`
- `/verify the payment circuit breaker opens after 5 failures`

## Test Hierarchy

```
         ┌─────────────────┐
         │  Acceptance     │  ← BDD: business scenarios
         │  (slowest, few) │    Cucumber / Behave
         └────────┬────────┘
                  │
         ┌────────┴────────┐
         │  Integration    │  ← Adapters: DB, broker, HTTP
         │  (medium speed) │    TestContainers, embedded broker
         └────────┬────────┘
                  │
         ┌────────┴────────┐
         │  Unit           │  ← Domain logic: aggregates, use cases
         │  (fast, many)   │    Pure functions, mock adapters
         └─────────────────┘
```

## Rules

- A bug is not fixed until there is a test that would have caught it
- Never mock the database in integration tests
- Domain tests must not import Spring, FastAPI, or any framework
- A test that always passes proves nothing — see it fail first
