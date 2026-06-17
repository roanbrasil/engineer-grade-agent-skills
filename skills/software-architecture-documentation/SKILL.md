---
name: software-architecture-documentation
description: Document software architecture using Arc42, ADRs, fitness functions, and living docs — invoked when creating, updating, or auditing architecture documentation for a system
---

# Software Architecture Documentation

Good architecture documentation answers the questions that code cannot: **why** decisions were made, **what** the system is supposed to be, and **when** things changed and why. It serves new joiners, future maintainers, incident responders, and the architects themselves (writing forces clarity).

---

## Why Document Architecture

**The four jobs documentation does:**

1. **Onboarding** — A new developer understands the system without reading every file. The first week is productive, not disorienting.
2. **Decision memory** — When a decision is documented with its rationale, it is not relitigated in every meeting. "We considered MongoDB; see ADR-004."
3. **Drift detection** — Architecture degrades silently. Comparing docs to code catches drift before it becomes a crisis.
4. **Design tool** — Writing about an architecture exposes gaps that diagrams hide. If you cannot write it clearly, the design is unclear.

**What documentation is not:**
- A substitute for good code
- A deliverable produced at the end of a project (write it while you design)
- Something that lives in Confluence and is never updated

---

## Arc42 — Structured Architecture Documentation

Arc42 provides 12 sections. Use all 12 for large systems; use the **Lean Core** (6 sections) for most projects.

```
Arc42 Sections
─────────────────────────────────────────────────────────────────
 1. Introduction and Goals
 2. Constraints
 3. System Scope and Context          ← Lean Core
 4. Solution Strategy                 ← Lean Core
 5. Building Block View               ← Lean Core
 6. Runtime View                      ← Lean Core
 7. Deployment View
 8. Cross-cutting Concepts
 9. Architecture Decisions            ← Lean Core (ADR index)
10. Quality Requirements              ← Lean Core
11. Risks and Technical Debt
12. Glossary
─────────────────────────────────────────────────────────────────
Lean Core covers ~80% of what teams need.
```

### Section Descriptions

**1. Introduction and Goals**
```markdown
## 1. Introduction and Goals

### System Purpose
The E-Commerce Platform enables customers to browse products,
place orders, and track deliveries. It handles 10,000 orders/day
at peak season.

### Top 3 Quality Goals
| Priority | Quality | Scenario |
|---|---|---|
| 1 | Reliability | Order placement succeeds under 500 concurrent users |
| 2 | Modifiability | Adding a payment provider takes < 2 days |
| 3 | Security | No PII appears in logs or error responses |

### Stakeholders
| Role | Name | Interest |
|---|---|---|
| Product Owner | Ana Costa | Feature roadmap, business rules |
| Lead Developer | João Silva | Technical decisions, implementation |
| DevOps | Maria Santos | Deployment, SLOs, on-call |
| Legal | — | GDPR compliance, data residency |
```

**2. Constraints**
- Technical: "Must run on AWS", "PostgreSQL only (DBA team constraint)", "Java 21 LTS"
- Organizational: "Team of 4 developers", "3-month deadline", "No third-party SaaS for PII"
- Regulatory: "LGPD compliance", "PCI-DSS for payment flows"

Constraints limit your design space. Document them so future architects know what they cannot change and why.

**3. System Scope and Context**
C4 Level 1 diagram + table of external interfaces:
```markdown
## 3. System Scope and Context

[C4 Context diagram here — see c4-model skill]

### External Interfaces
| System | Direction | Protocol | Data | Owner |
|---|---|---|---|---|
| Stripe | Outbound | REST/HTTPS | Payment intent, webhook | Finance |
| SendGrid | Outbound | SMTP/HTTPS | Order emails | Marketing |
| FedEx API | Outbound | REST/HTTPS | Shipping labels, tracking | Logistics |
| ERP System | Inbound | Kafka | Inventory updates | IT Ops |
```

**4. Solution Strategy**
A short narrative (1-2 pages) explaining the key technical decisions and their rationale:
```markdown
## 4. Solution Strategy

**Architectural style:** Event-driven microservices decomposed by bounded context (Order, Catalog, Inventory, Notification). Services communicate asynchronously via Kafka for resilience; synchronous REST only for user-facing flows where latency matters.

**Why not a monolith first:** The team is 4 developers, 3 of whom are backend-focused and own separate domains. Domain boundaries are well understood from 2 years of operating the legacy system. A monolith would create merge conflicts on the shared schema daily.

**Database strategy:** Each service owns its database (no shared schema). PostgreSQL for transactional services; Redis for session and catalog cache; no event sourcing in phase 1.

**Key trade-off:** Eventual consistency across services. Accepted because order confirmation can tolerate 2-second email delay. Real-time inventory sync is not required by the business (see ADR-003).
```

**5. Building Block View**
C4 Container diagram (Level 2) + optional Component diagrams (Level 3) for complex containers.

```markdown
## 5. Building Block View

### Level 2 — Containers
[C4 Container diagram here]

### Level 3 — Order API Components
[C4 Component diagram here — only for Order API; it contains the most complex logic]

### Responsibility Matrix
| Container | Responsibility | Team |
|---|---|---|
| Order API | Order lifecycle, cart, checkout | Backend Team |
| Catalog Service | Products, search, pricing | Backend Team |
| Notification Worker | Emails, push notifications | Backend Team |
| React SPA | Customer storefront | Frontend Team |
| Admin Dashboard | Warehouse UI | Frontend Team |
```

**6. Runtime View**
Sequence diagrams for the 3-5 most important scenarios:
- Happy path order placement
- Payment failure and retry
- Inventory sync from ERP
- Shipment notification flow

**7. Deployment View**
Infrastructure diagram: AWS regions, AZs, EKS cluster topology, RDS configuration, Kafka cluster.

```markdown
## 7. Deployment View

### Production Environment (AWS ap-southeast-1)

EKS Cluster (3 node groups, t3.xlarge):
  Namespace: ecommerce-prod
    Deployment: order-api          (3 replicas, HPA: 3-10)
    Deployment: catalog-service    (2 replicas)
    Deployment: notification-worker (2 replicas)
    Deployment: react-spa          (2 replicas, served by nginx)

RDS PostgreSQL 16 (db.r6g.xlarge, Multi-AZ):
  order-api-db (primary + read replica)
  catalog-db   (primary + read replica)

MSK (Managed Kafka, 3 brokers):
  Topics: order.placed, order.shipped, inventory.updated

ElastiCache Redis (cache.r6g.large, cluster mode):
  Session store + catalog cache

CloudFront CDN → S3:
  Static assets, product images
```

**8. Cross-cutting Concepts**
Decisions that apply uniformly across all containers:

```markdown
## 8. Cross-cutting Concepts

### Logging
- Format: JSON (structured) — every log entry includes traceId, spanId, serviceName
- Library: SLF4J + Logback → CloudWatch Logs
- PII policy: No email addresses, names, or card data in logs. Use customerId (UUID) only.

### Security
- Authentication: JWT (RS256). Public key distributed via JWKS endpoint.
- Authorization: Role claims in JWT. Services validate locally (no network call per request).
- Secrets: AWS Secrets Manager. Rotated every 90 days. No secrets in environment variables.
- TLS: TLS 1.3 enforced on all inter-service calls. mTLS for Kafka.

### Error Handling
- Client errors (4xx): return RFC 7807 Problem Details JSON.
- Server errors (5xx): log full stack trace with traceId; return minimal detail to client.
- Kafka consumer errors: retry 3x with exponential backoff; send to dead-letter topic after 3 failures.

### Observability
- Metrics: Micrometer → Prometheus → Grafana
- Tracing: OpenTelemetry → AWS X-Ray
- Alerting: Alert on p99 latency > 2s for /orders endpoint; error rate > 1%; Kafka consumer lag > 1000

### Dependency Injection
- Spring dependency injection throughout. No static access to ApplicationContext.
- Domain model (com.example.domain) has zero Spring annotations — pure Java.
```

**9. Architecture Decisions**
Index of ADRs (see ADR section below).

**10. Quality Requirements**
Quality scenarios as acceptance criteria — testable, not vague.

**11. Risks and Technical Debt**

```markdown
## 11. Risks and Technical Debt

### Known Risks
| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Kafka cluster single-AZ (MSK) | Low | High | Migrate to multi-AZ MSK in Q3 |
| Order API has 3 God Service classes | High | Medium | Refactor using ADR-008 roadmap |
| No load testing above 500 concurrent users | Medium | High | Load test sprint planned for Q2 |

### Accepted Technical Debt
| Item | Accepted By | Repayment Plan |
|---|---|---|
| JPA entities and domain model are same classes | Lead Dev | Separate in Order API v2 (Q4) |
| No integration tests for Kafka consumer | Tech Lead | Add in next sprint |
| Hardcoded Stripe API version | Backend Team | Extract to config in ADR-012 scope |
```

**12. Glossary**
Define the ubiquitous language — the terms used consistently in code, docs, and conversation:

```markdown
## 12. Glossary

| Term | Definition |
|---|---|
| Order | A customer's request to purchase one or more products. Has a lifecycle from PENDING to DELIVERED. |
| Line Item | A single product+quantity within an Order. |
| Cart | A transient collection of items before an Order is placed. Not persisted in the Order database. |
| Fulfillment | The warehouse process of picking, packing, and shipping an Order. |
| SKU | Stock Keeping Unit. Unique identifier for a product variant (size, color, etc.). |
| Bounded Context | A DDD concept. Each service owns exactly one bounded context and its terms. |
```

---

## ADR — Architecture Decision Records

An ADR documents a significant architectural decision: what was decided, why, and what the consequences are. The goal is never to justify a bad decision retroactively — it is to capture the reasoning while it is fresh.

### When to write an ADR

Write an ADR when:
- [ ] The decision is hard or expensive to reverse
- [ ] Significant trade-offs exist between options
- [ ] Team members disagreed on the right approach
- [ ] The decision will surprise a future developer
- [ ] The decision violates a common pattern for a specific reason

Do NOT write an ADR for:
- Trivial library version choices
- Implementation details within a single class
- Decisions that follow obvious best practices

### ADR File Naming and Storage

```
docs/
  adr/
    ADR-001-use-postgresql-over-mongodb.md
    ADR-002-event-driven-inter-service-communication.md
    ADR-003-eventual-consistency-for-inventory.md
    ADR-004-jwt-for-authentication.md
    ...
```

### ADR Template

```markdown
# ADR-NNN: [Short imperative title]

Date: YYYY-MM-DD
Status: Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context

[Describe the situation that requires a decision. What problem are we solving?
What are the constraints? What options did we consider? Keep this factual — no judgments yet.]

## Decision

[State the decision clearly in the first sentence. Then explain the reasoning.
Why this option over the others?]

## Consequences

### Positive
- [Benefit 1]
- [Benefit 2]

### Negative / Trade-offs
- [Cost 1]
- [Cost 2]

### Neutral
- [Observable change that is neither good nor bad]
```

### ADR Example — Full

```markdown
# ADR-001: Use PostgreSQL over MongoDB for Order Storage

Date: 2025-06-17
Status: Accepted

## Context

We need to persist order data. Orders have complex relationships:
one Order has many LineItems, each LineItem references a Product,
and every Order has a shipping Address. Orders require ACID
transactions: reserving inventory and creating the order record
must succeed or fail together.

We evaluated:
- PostgreSQL 16 (relational, ACID, strong typing)
- MongoDB 7 (document store, flexible schema, horizontal write scaling)

The team has 3 developers with 4+ years of PostgreSQL experience each.
No developer has production MongoDB experience. Expected load is
10,000 orders/day at peak (Black Friday). Write throughput does not
require horizontal scaling in year 1.

## Decision

Use PostgreSQL 16 as the primary store for the Order bounded context.
Use JSONB columns for order metadata fields that may vary by product
category (e.g., digital vs. physical product-specific attributes).

## Consequences

### Positive
- ACID transactions across Order and LineItem entities, eliminating
  a whole class of consistency bugs.
- Complex queries (e.g., "all orders for customer X in date range,
  sorted by total") are efficient with B-tree indexes.
- Team hits the ground running — no learning curve.
- pg_logical replication to read replicas is battle-tested.

### Negative / Trade-offs
- Schema changes require migrations (Flyway). Adding a new field
  means a migration script. MongoDB would not.
- Horizontal write scaling requires Citus, which adds operational
  complexity. Acceptable given projected load.
- JSONB queries for metadata fields are slower than top-level columns
  and less type-safe.

### Neutral
- This decision is scoped to the Order service. Catalog service may
  choose a different database in its own ADR.
```

### ADR — More Examples

```markdown
# ADR-002: Asynchronous Inter-Service Communication via Kafka

Date: 2025-06-17
Status: Accepted

## Context
Order API, Inventory Service, and Notification Worker need to share
state changes. We evaluated synchronous REST calls vs. asynchronous
messaging via Kafka.

REST: simple to implement, but creates temporal coupling. If
Notification Worker is down, Order API's HTTP call fails, and the
order might not be placed. Requires circuit breakers and retry logic
in every producer.

Kafka: decoupled, resilient. Producer does not wait for consumer.
Events are persisted and replayed if a consumer restarts.
Adds operational overhead: Kafka cluster management.

## Decision
Services communicate state changes via Kafka domain events.
Synchronous REST is used only for customer-facing request/response
flows where the customer is waiting for a response.

Event naming convention: `{context}.{entity}.{past-tense-verb}`
Examples: `order.order.placed`, `inventory.stock.reserved`

## Consequences

### Positive
- Services are independently deployable and scalable.
- Notification Worker downtime does not affect order placement.
- Event log provides an audit trail.
- New consumers can replay historical events.

### Negative / Trade-offs
- Eventual consistency: inventory count in the Order API is stale
  until the Kafka event is consumed. Acceptable per business rules
  (see ADR-003).
- Operational overhead: Kafka cluster must be monitored, backed up.
  Mitigated by using AWS MSK (managed).
- Debugging asynchronous flows is harder than synchronous chains.
  Mitigated by distributed tracing (OpenTelemetry).
```

---

## ADR Tools

```bash
# adr-tools (bash CLI): https://github.com/npryce/adr-tools
brew install adr-tools        # macOS
adr init docs/adr             # initialize ADR directory
adr new "Use PostgreSQL over MongoDB"    # creates ADR-001-...md
adr new -s 1 "Migrate to CockroachDB"   # supersedes ADR-001

# Log4brains: renders ADRs as a browsable HTML site
npx log4brains init
npx log4brains adr new "Use event sourcing for audit log"
npx log4brains preview        # local HTML view
npx log4brains build          # static site for deployment
```

---

## Fitness Functions — Living Architecture

Fitness functions are executable tests that encode architecture rules. They run in CI on every PR. Architecture violations become build failures, not yearly audit findings.

### The Problem They Solve

Without fitness functions:
```
Day 1:  "Domain classes must not depend on Spring."
Day 90: Developers add @Transactional to a domain class.
Day 180: The rule is forgotten.
Day 365: Architecture review finds 30 violations.
```

With fitness functions:
```
Day 90: Developer adds @Transactional to a domain class.
        CI fails. PR is blocked. Violation fixed before merge.
```

### ArchUnit (Java / Kotlin)

```java
// src/test/java/ArchitectureTests.java

@AnalyzeClasses(packages = "com.example")
public class ArchitectureTests {

    // Domain layer must not depend on Spring
    @ArchTest
    static final ArchRule domainMustNotDependOnSpring =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework..")
            .because("Domain model must be framework-independent for testability");

    // Application layer must not depend on Infrastructure layer
    @ArchTest
    static final ArchRule applicationMustNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..")
            .because("Application layer depends on ports (interfaces), not adapters");

    // Controllers must only call Use Cases, not Repositories directly
    @ArchTest
    static final ArchRule controllersMustNotCallRepositories =
        noClasses()
            .that().resideInAPackage("..interfaces..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure.persistence..")
            .because("Controllers talk to use cases, not persistence adapters directly");

    // Use Cases must not depend on HTTP-specific classes
    @ArchTest
    static final ArchRule useCasesMustNotDependOnHttp =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("javax.servlet..")
            .orShould().dependOnClassesThat()
            .resideInAPackage("org.springframework.web..");

    // Detect cyclic dependencies between packages
    @ArchTest
    static final ArchRule noCyclicDependencies =
        slices().matching("com.example.(*)..").should().beFreeOfCycles();
}
```

### Dependency Cruiser (JavaScript / TypeScript)

```json
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    {
      name: "no-circular",
      severity: "error",
      comment: "Circular dependencies make refactoring impossible",
      from: {},
      to: { circular: true }
    },
    {
      name: "domain-no-infrastructure",
      severity: "error",
      comment: "Domain must not import from infrastructure",
      from: { path: "^src/domain" },
      to: { path: "^src/infrastructure" }
    },
    {
      name: "no-node-modules-in-domain",
      severity: "error",
      comment: "Domain layer must be framework-free",
      from: { path: "^src/domain" },
      to: { dependencyTypes: ["npm"] }
    }
  ],
  options: {
    doNotFollow: { path: "node_modules" },
    tsConfig: { fileName: "tsconfig.json" }
  }
};
```

```bash
# Run in CI
npx depcruise --validate .dependency-cruiser.js src
```

### Python — Custom Import Checker

```python
# tests/test_architecture.py
import ast
import os
import pytest

def get_imports(filepath):
    with open(filepath) as f:
        tree = ast.parse(f.read())
    imports = []
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            imports.extend(alias.name for alias in node.names)
        elif isinstance(node, ast.ImportFrom):
            if node.module:
                imports.append(node.module)
    return imports

def find_python_files(directory):
    for root, _, files in os.walk(directory):
        for file in files:
            if file.endswith('.py'):
                yield os.path.join(root, file)

def test_domain_has_no_infrastructure_imports():
    """Domain layer must not import from infrastructure."""
    violations = []
    for filepath in find_python_files('src/domain'):
        imports = get_imports(filepath)
        for imp in imports:
            if 'infrastructure' in imp or 'sqlalchemy' in imp or 'redis' in imp:
                violations.append(f"{filepath}: imports {imp}")
    assert not violations, f"Domain layer has infrastructure imports:\n" + "\n".join(violations)

def test_no_circular_imports():
    """No circular imports between modules."""
    # Use import-linter or pydeps for production use
    pass
```

### CI Integration

```yaml
# .github/workflows/architecture.yml
name: Architecture Tests

on: [pull_request]

jobs:
  arch-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
      - name: Run ArchUnit tests
        run: ./mvnw test -pl order-api -Dtest=ArchitectureTests
      - name: Run Dependency Cruiser
        run: npx depcruise --validate .dependency-cruiser.js src
```

---

## Documentation as Code

The principle: docs live in the same repository as the code they describe. They are versioned, reviewed in PRs, and updated when the code changes.

### Directory Structure

```
project-root/
  docs/
    architecture/
      arc42.md                    ← Arc42 main document
      diagrams/
        context.puml              ← PlantUML source
        context.png               ← Generated in CI
        containers.puml
        containers.png
        order-api-components.puml
        order-api-components.png
      workspace.dsl               ← Structurizr DSL (generates all views)
    adr/
      ADR-001-use-postgresql.md
      ADR-002-kafka-events.md
      ADR-003-eventual-consistency.md
      README.md                   ← ADR index
    runbooks/
      incident-response.md
      deployment-checklist.md
```

### CI: Generate Diagrams from Source

```yaml
# .github/workflows/docs.yml
name: Generate Architecture Diagrams

on:
  push:
    paths:
      - 'docs/architecture/diagrams/**/*.puml'

jobs:
  generate-diagrams:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate PlantUML diagrams
        uses: cloudbees/plantuml-github-action@master
        with:
          args: -tpng docs/architecture/diagrams/*.puml
      - name: Commit generated PNGs
        run: |
          git config user.email "ci@example.com"
          git config user.name "CI Bot"
          git add docs/architecture/diagrams/*.png
          git diff --staged --quiet || git commit -m "ci: regenerate architecture diagrams"
          git push
```

### Structurizr CI Push

```bash
# Publish to Structurizr cloud on every merge to main
structurizr-cli push \
  --id $STRUCTURIZR_WORKSPACE_ID \
  --key $STRUCTURIZR_API_KEY \
  --secret $STRUCTURIZR_API_SECRET \
  --workspace docs/architecture/workspace.dsl
```

---

## Quality Attribute Scenarios (ISO 25010)

Vague quality goals are untestable. Quality scenarios make them concrete and actionable.

**Template:**
```
Given [stimulus source] sends [stimulus]
to [artifact] under [environment conditions],
the system responds with [response]
measured by [response measure].
```

### Scenario Examples

**Performance (Efficiency)**
```
Stimulus source: 500 concurrent customers
Stimulus:        POST /orders (checkout)
Artifact:        Order API
Environment:     Normal operating conditions, Black Friday peak
Response:        Order is created and confirmation returned
Measure:         95th percentile response time ≤ 2 seconds;
                 0 order data loss
```

**Availability**
```
Stimulus source: AWS us-east-1 AZ failure
Stimulus:        AZ-A becomes unavailable
Artifact:        Production EKS cluster
Environment:     Peak traffic (500 req/sec)
Response:        System continues serving requests from AZ-B and AZ-C
Measure:         < 30 seconds of elevated errors during failover;
                 0 data loss; RTO ≤ 30 minutes for full recovery
```

**Modifiability**
```
Stimulus source: Business decision
Stimulus:        Add PayPal as a payment provider
Artifact:        Order API (payment integration)
Environment:     Normal development, full test coverage required
Response:        New payment provider is available in production
Measure:         ≤ 2 developer days; ≤ 5 files modified;
                 no changes required to Order domain model
```

**Security**
```
Stimulus source: Malicious external actor
Stimulus:        Requests /api/orders/12345 without auth token
Artifact:        Order API
Environment:     Production
Response:        Request is rejected; no order data returned
Measure:         HTTP 401 returned in < 100ms;
                 no stack trace or internal paths in response body;
                 failed attempt logged with IP and timestamp
```

**Testability**
```
Stimulus source: Developer
Stimulus:        Runs unit test suite for Order domain
Artifact:        com.example.domain package
Environment:     Local machine, no database, no Kafka, no network
Response:        All domain tests pass
Measure:         Test suite completes in < 5 seconds;
                 0 test infrastructure dependencies
```

### Quality Tree

```
Quality Goals for E-Commerce Platform
│
├── Reliability (Priority 1)
│   ├── Availability: 99.9% uptime; RTO < 30 min
│   ├── Fault tolerance: Single service failure does not cascade
│   └── Data integrity: No order data loss under any failure scenario
│
├── Performance (Priority 2)
│   ├── Throughput: 500 concurrent checkouts
│   ├── Latency: p95 < 2s for order placement
│   └── Scalability: HPA scales Order API 3→10 replicas under load
│
├── Modifiability (Priority 3)
│   ├── Payment provider: < 2 days to add new provider
│   ├── Bounded context isolation: services deployable independently
│   └── Domain purity: domain model has no framework dependencies
│
├── Security (Priority 4)
│   ├── No PII in logs or error responses
│   ├── All inter-service traffic TLS 1.3
│   └── Secrets rotated every 90 days via AWS Secrets Manager
│
└── Observability (Priority 5)
    ├── Every request traceable end-to-end via traceId
    ├── Alert on p99 latency > 2s within 1 minute
    └── Dead-letter queue monitored; alert on > 0 messages
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| PowerPoint architecture | Static slides not in repo; stale within weeks | Move all diagrams to PlantUML/Mermaid source in the repo; generate images in CI |
| God document | 100-page Word doc that nobody reads | Use Arc42 sections: lean core first, expand as needed; one page per section max |
| Architecture by committee, no decisions recorded | Same debates recur; no institutional memory | Write ADRs. Even a 5-sentence ADR is better than nothing |
| Diagrams without decisions | Shows what exists, not why it exists | Pair every significant Container/Component diagram with ADRs for the non-obvious choices |
| Vague quality goals | "The system should be fast and secure" cannot be tested | Write quality scenarios: source, stimulus, artifact, environment, measure |
| Documenting what the code already says | "The `save()` method saves the entity" adds zero value | Document intent, trade-offs, and constraints — not implementation |
| Architecture fitness functions only in heads | Rules drift silently; caught in annual reviews | Encode rules as ArchUnit tests, dependency-cruiser configs, or custom AST checkers that run in CI |
| Stale ADRs | Decisions marked "Accepted" but superseded by later ADRs | Update old ADR status to "Superseded by ADR-NNN"; link to the new one |

---

## Documentation Sprint Checklist

For a new system or major component, produce the Lean Core Arc42 in this order:

- [ ] Arc42 §1: Introduction — purpose, top 3 quality goals, stakeholder table
- [ ] Arc42 §3: System context — C4 Level 1 diagram + external interfaces table
- [ ] Arc42 §4: Solution strategy — 1-2 page narrative of key decisions and trade-offs
- [ ] Arc42 §5: Building block view — C4 Level 2 diagram; Level 3 for complex containers
- [ ] Arc42 §6: Runtime view — sequence diagrams for 3 key scenarios
- [ ] Arc42 §9: ADR index — at least 3 ADRs for the most significant decisions
- [ ] Arc42 §10: Quality requirements — quality tree + 5 testable scenarios
- [ ] Fitness functions — ArchUnit or equivalent encoding top 3 architecture rules
- [ ] Glossary — define 10-15 ubiquitous language terms

**Time budget for Lean Core on a medium system:** 2-3 developer days.
**Return:** saves 2-3 weeks per year in re-explaining decisions, onboarding, and rework from architecture drift.
