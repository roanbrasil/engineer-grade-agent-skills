# Engineer-Grade Agent Skills

**Production-quality engineering skills for AI coding agents — polyglot, event-driven, architecture-first.**

```
  ARCHITECT        CODE           TEST          REVIEW         SHIP
 ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
 │  Design  │→ │  Build   │→ │  Verify  │→ │  Harden  │→ │ Release  │
 │  Domain  │  │Polyglot  │  │ TDD/BDD  │  │ Patterns │  │Resilient │
 └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
  /architect     /implement    /verify        /harden        /release
```

---

## What makes this different

Most agent skill collections focus on the development lifecycle. This one goes deeper:

| Dimension | What's here |
|-----------|-------------|
| **Polyglot** | Rust, Java, Kotlin, C++, Python — idiomatic patterns per language |
| **Event-driven** | Kafka mastery, stream processing, messaging broker expertise |
| **Architecture-first** | Clean Architecture, Hexagonal/Ports-Adapters, DDD, CQRS+ES |
| **Agent-native** | Patterns for multi-agent systems (ADD, integration patterns) |
| **Quality trilogy** | TDD, BDD, and Domain Events woven through every skill |

---

## Quick Start

```bash
# Install via Claude Code plugin marketplace
/plugin marketplace add roanbrasil/engineer-grade-agent-skills
/plugin install engineer-grade-agent-skills

# Or clone locally
git clone https://github.com/roanbrasil/engineer-grade-agent-skills
cd engineer-grade-agent-skills
```

---

## Commands

| Command | What it does |
|---------|-------------|
| `/architect` | Design the domain model: bounded contexts, aggregates, events |
| `/implement` | Build a feature: choose language, framework, patterns |
| `/verify` | Test the implementation: TDD cycle, BDD scenarios, contract tests |
| `/harden` | Resilience, security, observability review |
| `/release` | API contract check, backward compatibility, deployment readiness |
| `/stream` | Design or debug an event streaming pipeline |
| `/agent-design` | Design a multi-agent system using ADD + integration patterns |

---

## Skills

### Clean Engineering
| Skill | Description |
|-------|-------------|
| `clean-code` | Write code that communicates intent; eliminate noise |
| `clean-architecture` | Protect business logic from frameworks and infrastructure |
| `ports-and-adapters` | Hexagonal architecture: driving side vs driven side |
| `cqrs-event-sourcing` | Separate reads from writes; events as the source of truth |
| `domain-events` | Raise and handle domain events; outbox pattern |

### Testing Excellence
| Skill | Description |
|-------|-------------|
| `tdd-mastery` | Red-Green-Refactor; test doubles; testing pyramid |
| `bdd-specification` | Three Amigos; Gherkin; living documentation |

### Domain-Driven Design
| Skill | Description |
|-------|-------------|
| `ddd-tactical` | Entities, Value Objects, Aggregates, Repositories, Domain Services |
| `design-patterns-mastery` | GoF patterns + modern functional patterns |

### Language Expertise
| Skill | Description |
|-------|-------------|
| `rust-systems` | Ownership, async, idiomatic Rust |
| `java-excellence` | Modern Java 17+, virtual threads, streams |
| `kotlin-craft` | Coroutines, sealed classes, idiomatic Kotlin |
| `cpp-modern` | RAII, smart pointers, C++17/20 |
| `python-production` | Type hints, async, pydantic, production-ready Python |

### Frameworks
| Skill | Description |
|-------|-------------|
| `spring-boot-expert` | Spring Boot 3, WebFlux, Security, TestContainers |
| `fastapi-production` | FastAPI + Pydantic v2, async, dependency injection |

### Event-Driven & Messaging
| Skill | Description |
|-------|-------------|
| `kafka-mastery` | Producers, consumers, Kafka Streams, schema registry |
| `event-streaming` | Stream processing patterns, windowing, stateful ops |
| `messaging-brokers` | RabbitMQ, SQS/SNS, NATS, broker selection |

### Microservices
| Skill | Description |
|-------|-------------|
| `microservices-excellence` | Service decomposition, inter-service communication, data ownership |
| `resilience-patterns` | Circuit breaker, retry, bulkhead, chaos engineering |

### Agent Systems
| Skill | Description |
|-------|-------------|
| `agents-integration-patterns` | MCP, A2A, coordination/messaging/routing/resilience patterns |
| `agent-driven-design` | ADD framework: Model + Harness; topology patterns; production concerns |

### Infrastructure & API
| Skill | Description |
|-------|-------------|
| `observability-excellence` | Logs, metrics, traces; OpenTelemetry; SLO-based alerting |
| `api-contract-design` | REST, gRPC, GraphQL; versioning; contract testing with Pact |

---

## Philosophy

> **Skills encode the judgment of senior engineers, not just their knowledge.**

Each skill answers: *when* to apply a pattern, *why* it matters, and *what goes wrong* when you get it wrong. Code examples are multi-language and idiomatic. Anti-patterns are first-class citizens — knowing what NOT to do is half the skill.

### The Quality Triad

Every feature produced with these skills passes through three lenses:

```
       Domain Correctness
            (DDD)
           /      \
          /        \
   Behavioral    Structural
   Correctness   Correctness
     (TDD/BDD)   (Clean Arch)
```

---

## Inspired by

- [Addy Osmani's agent-skills](https://github.com/addyosmani/agent-skills) — lifecycle-oriented skills
- [roanbrasil/agents-integration-patterns](https://github.com/roanbrasil/agents-integration-patterns) — multi-agent vocabulary
- [roanbrasil/agent-driven-design](https://github.com/roanbrasil/agent-driven-design) — ADD framework
- *Enterprise Integration Patterns* (Hohpe & Woolf, 2003)
- *Clean Architecture* (Martin, 2017)
- *Domain-Driven Design* (Evans, 2003)
- *Designing Data-Intensive Applications* (Kleppmann, 2017)

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Each skill follows the same structure:
- Frontmatter: `name`, `description`
- When to use / When NOT to use
- Core concepts with ASCII diagrams
- Multi-language code examples
- Anti-patterns
- Checklist
