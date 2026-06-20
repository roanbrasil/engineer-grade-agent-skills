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

## Skills (90 total)

### Clean Engineering
| Skill | Description |
|-------|-------------|
| `clean-code` | Write code that communicates intent; eliminate noise |
| `clean-architecture` | Protect business logic from frameworks and infrastructure |
| `ports-and-adapters` | Hexagonal architecture: driving side vs driven side |
| `cqrs-event-sourcing` | Separate reads from writes; events as the source of truth |
| `domain-events` | Raise and handle domain events; outbox pattern |
| `refactoring-patterns` | Systematic refactoring: extract, move, strangler fig |
| `technical-debt` | Identify, quantify, and pay down technical debt strategically |

### Testing Excellence
| Skill | Description |
|-------|-------------|
| `tdd-mastery` | Red-Green-Refactor; test doubles; testing pyramid |
| `bdd-specification` | Three Amigos; Gherkin; living documentation |
| `property-based-testing` | Hypothesis-based testing; invariants; shrinking |
| `contract-testing` | Pact; consumer-driven contracts; API compatibility |
| `testcontainers-mastery` | Real databases and brokers in tests; no mocks |
| `load-testing` | k6, Gatling; performance regression; SLO validation |
| `static-analysis` | SpotBugs, ruff, clippy; SAST; SonarQube quality gates |

### Domain-Driven Design
| Skill | Description |
|-------|-------------|
| `ddd-tactical` | Entities, Value Objects, Aggregates, Repositories, Domain Services |
| `design-patterns-mastery` | GoF patterns + modern functional patterns |
| `saga-orchestration` | Distributed saga: orchestration vs choreography; compensating transactions |

### Language Expertise
| Skill | Description |
|-------|-------------|
| `rust-systems` | Ownership, async, idiomatic Rust |
| `java-excellence` | Modern Java 17+, virtual threads, streams |
| `kotlin-craft` | Coroutines, sealed classes, idiomatic Kotlin |
| `cpp-modern` | RAII, smart pointers, C++17/20 |
| `python-production` | Type hints, async, pydantic, production-ready Python |
| `concurrency-patterns` | Actor model, CSP, channels, thread safety across all languages |

### Frameworks
| Skill | Description |
|-------|-------------|
| `spring-boot-expert` | Spring Boot 3, WebFlux, Security, TestContainers |
| `fastapi-production` | FastAPI + Pydantic v2, async, dependency injection |
| `spring-ai` | Spring AI 1.x: ChatClient, Advisors, RAG, vector stores |
| `langchain-patterns` | LCEL, tools, memory, RAG chains |
| `langgraph-workflows` | StateGraph, human-in-the-loop, checkpointing, streaming |

### Event-Driven & Messaging
| Skill | Description |
|-------|-------------|
| `kafka-mastery` | Producers, consumers, Kafka Streams, schema registry |
| `event-streaming` | Stream processing patterns, windowing, stateful ops |
| `messaging-brokers` | RabbitMQ, SQS/SNS, NATS, broker selection |
| `websockets-realtime` | WebSocket, SSE, long polling; real-time patterns at scale |
| `iggy-streaming` | Iggy: high-performance Rust-native streaming; TCP/HTTP/QUIC |
| `robustmq` | RobustMQ: cloud-native Rust MQ; MQTT + Kafka + AMQP in one broker |
| `redpanda` | Redpanda: Kafka-compatible C++ streaming; no ZooKeeper; thread-per-core |
| `apache-pulsar` | Pulsar: multi-tenant, geo-replication, tiered storage, 4 subscription types |
| `nats-jetstream` | NATS + JetStream: lightweight messaging, KV store, object store, leaf nodes |
| `mqtt-patterns` | MQTT 3.1/5.0: IoT patterns, QoS, device shadow, shared subscriptions |

### Distributed Systems
| Skill | Description |
|-------|-------------|
| `distributed-systems-fundamentals` | CAP, consistency models, Raft/Paxos, vector clocks, failure detection |
| `microservices-excellence` | Service decomposition, inter-service communication, data ownership |
| `resilience-patterns` | Circuit breaker, retry, bulkhead, chaos engineering |
| `service-mesh` | Istio, Linkerd: mTLS, traffic management, observability |
| `financial-systems-patterns` | Ledger design, payment idempotency, double-entry bookkeeping |

### API Design
| Skill | Description |
|-------|-------------|
| `api-contract-design` | REST, gRPC, GraphQL; versioning; contract testing with Pact |
| `grpc-implementation` | Protobuf, streaming RPCs, interceptors, error handling |
| `graphql-schema-design` | Schema-first, DataLoader, Federation, N+1 prevention |
| `api-gateway-design` | Gateway patterns, BFF, rate limiting, auth at the edge |

### Infrastructure & Platform
| Skill | Description |
|-------|-------------|
| `kubernetes-production` | HPA, VPA, PDB, RBAC, network policies, production hardening |
| `helm-charts` | Chart design, templating, OCI registry, chart testing |
| `terraform-patterns` | Modules, state, workspaces, CI/CD for infra |
| `ci-cd-pipelines` | GitHub Actions, pipeline design, deployment strategies |
| `cloud-native-design` | 12-Factor, GitOps, distroless containers, graceful shutdown |
| `platform-engineering` | IDP, Backstage, golden paths, self-service infra |
| `local-dev-environment` | Docker Compose, devcontainers, Tilt, Taskfile, direnv |

### Data & Storage
| Skill | Description |
|-------|-------------|
| `database-design` | Schema design, normalization, indexes, migrations |
| `nosql-patterns` | MongoDB, DynamoDB, Cassandra, Redis — selection and patterns |
| `redis-mastery` | Data structures, caching patterns, streams, Lua, cluster |
| `vector-databases` | pgvector, Chroma, Weaviate, Pinecone, Qdrant |

### Agent Systems
| Skill | Description |
|-------|-------------|
| `agents-integration-patterns` | MCP, A2A, coordination/messaging/routing/resilience patterns |
| `agent-driven-design` | ADD framework: Model + Harness; topology patterns; production concerns |
| `mcp-server-development` | Build MCP servers: tools, resources, prompts |
| `a2a-protocol` | Agent Cards, task lifecycle, capability discovery, A2A servers |

### AI Engineering
| Skill | Description |
|-------|-------------|
| `rag-production` | Chunking, embedding, hybrid retrieval, reranking, eval |
| `llm-evals` | RAGAS, LLM-as-judge, PromptFoo, DeepEval |
| `llm-observability` | Tracing LLM calls, cost tracking, quality monitoring |
| `llm-cost-optimization` | Caching, prompt compression, model routing, batching |
| `prompt-engineering-advanced` | Chain-of-thought, few-shot, structured outputs, adversarial robustness |
| `fine-tuning-llms` | LoRA, QLoRA, PEFT, dataset prep, evaluation, deployment |
| `ai-safety-guardrails` | Prompt injection defense, output validation, PII protection, red-teaming |

### Observability
| Skill | Description |
|-------|-------------|
| `observability-excellence` | Logs, metrics, traces; SLO-based alerting; RED/USE method |
| `opentelemetry-deep-dive` | OTel SDK, instrumentation, collectors, backends |
| `ebpf-observability` | Pixie, Cilium/Hubble, Pyroscope continuous profiling |
| `debugging-production` | JVM diagnostics, py-spy, K8s debugging, distributed trace analysis |

### Architecture Documentation
| Skill | Description |
|-------|-------------|
| `software-architecture-documentation` | Arc42, ADRs, fitness functions, ArchUnit |
| `c4-model` | Context, Container, Component, Code diagrams |
| `uml-diagrams` | Class, sequence, state machine, activity, use case |

### Security & Quality
| Skill | Description |
|-------|-------------|
| `security-hardening` | OWASP Top 10, auth patterns, secrets management, threat modeling |
| `code-review-excellence` | Giving and receiving reviews; PR design; review culture |
| `git-workflow` | Branching strategies, conventional commits, PR hygiene |
| `incident-management` | On-call, runbooks, post-mortems, blameless culture |
| `feature-flags` | LaunchDarkly, Unleash; flag lifecycle; kill switches |
| `performance-profiling` | CPU/memory profiling; flamegraphs; database query analysis |

### SaaS & Business Patterns
| Skill | Description |
|-------|-------------|
| `saas-architecture` | Multi-tenancy models, tenant isolation, billing, entitlements |
| `financial-systems-patterns` | Ledger design, payment idempotency, double-entry bookkeeping |

### Frontend & Web
| Skill | Description |
|-------|-------------|
| `css-mastery` | Flexbox, Grid, modern CSS, responsive design, performance |
| `design-systems` | Design tokens, component APIs, Storybook, theming |
| `react-patterns` | Hooks patterns, state management, performance, compound components |
| `web-performance` | Core Web Vitals, critical rendering path, caching, streaming SSR |
| `accessibility` | WCAG 2.1 AA, ARIA, keyboard navigation, screen reader testing |
| `frontend-architecture` | Rendering strategies, micro-frontends, state architecture, API layer |

### Emerging Technologies
| Skill | Description |
|-------|-------------|
| `wasm-patterns` | WebAssembly for browser, WASI, Spin, Wasmtime plugin systems |

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
