# Skill Selection Guide

Use this guide to pick the right skill for your current task.

## "I need to design something"

```
What are you designing?
│
├── A domain model (business logic)
│   └── /architect → ddd-tactical, domain-events
│
├── A system architecture
│   ├── Monolith → clean-architecture, ports-and-adapters
│   ├── Microservices → microservices-excellence, resilience-patterns
│   └── Event-driven → event-streaming, kafka-mastery
│
├── An API
│   └── api-contract-design
│
└── A multi-agent system
    └── /agent-design → agent-driven-design, agents-integration-patterns
```

## "I need to implement something"

```
What language/framework?
│
├── Java → java-excellence + spring-boot-expert
├── Kotlin → kotlin-craft + spring-boot-expert
├── Python → python-production + fastapi-production
├── Rust → rust-systems
└── C++ → cpp-modern

What architecture pattern?
│
├── Hexagonal → ports-and-adapters
├── Clean Arch → clean-architecture
├── CQRS → cqrs-event-sourcing
└── Event-driven → domain-events + kafka-mastery
```

## "I need to test something"

```
Test type?
│
├── Business behavior → bdd-specification (Gherkin)
├── Domain logic unit tests → tdd-mastery
├── Kafka consumer/producer → kafka-mastery (TestContainers)
├── Spring Boot → spring-boot-expert (@DataJpaTest, @WebMvcTest)
└── FastAPI → fastapi-production (pytest + httpx)
```

## "Something is failing in production"

```
What kind of failure?
│
├── Service unavailable / timeouts → resilience-patterns
├── Message consumer stopped / lag spiking → kafka-mastery
├── Can't find the root cause → observability-excellence
├── Data inconsistency → cqrs-event-sourcing, domain-events
└── Performance regression → language-specific skill (java-excellence, rust-systems)
```

## "I need to review / refactor code"

```
What dimension?
│
├── Code quality → clean-code
├── Architecture adherence → clean-architecture / ports-and-adapters
├── Design patterns → design-patterns-mastery
├── Domain model correctness → ddd-tactical
└── Test quality → tdd-mastery (test smells section)
```
