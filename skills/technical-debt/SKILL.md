---
name: technical-debt
description: Identify, quantify, prioritize, and pay down technical debt using structured taxonomies, fitness functions, ADRs, and paydown strategies
---

# Technical Debt Management

## Debt Taxonomy

### Ward Cunningham vs Martin Fowler Quadrant

```
                        DELIBERATE
                   (We knew what we were doing)
                            |
         +------------------+------------------+
         |                  |                  |
  "No time for        RECKLESS          "Ship now,        PRUDENT
   good design"                          refactor later"
         |                  |                  |
RECKLESS +------------------+------------------+ PRUDENT
         |                  |                  |
  "What's              INADVERTENT      "Now we know
   layering?"                            how we should
         |                  |             have done it"
         +------------------+------------------+
                            |
                       INADVERTENT
              (We didn't know we were doing it)
```

### The Four Quadrants Explained

| Quadrant | Label | Description | What to do |
|----------|-------|-------------|------------|
| Deliberate + Reckless | "No time for design" | Chose shortcuts knowingly, with no plan to fix them | Track immediately; treat as high-interest debt |
| Deliberate + Prudent | "Ship now, refactor later" | Conscious tradeoff with a concrete paydown plan | Only valid with a written plan and timeline |
| Inadvertent + Reckless | "What's layering?" | Team lacked knowledge of good practices | Training + architectural guardrails needed |
| Inadvertent + Prudent | "Now we know better" | Learned through experience | Most valuable: document the lesson in an ADR |

**Key insight:** Inadvertent + Prudent debt is not a failure — it's learning. The failure is not acting on it.

---

## Identifying Technical Debt

### Code-Level Signals

```
STATIC ANALYSIS RED FLAGS:
  Cyclomatic complexity > 10 per method      → Hard to test, risky to change
  Method length > 50 lines                   → Violates Single Responsibility
  Class length > 300 lines                   → God class smell
  Duplicate code > 5%                        → Shotgun surgery risk
  Test coverage < 70% on critical paths      → No safety net for refactoring
  TODO/FIXME comments (especially with dates)→ Known debt not tracked

DEPENDENCY RED FLAGS:
  Dependency on EOL library versions         → Security + support risk
  Circular dependencies between modules      → Tight coupling, hard to test
  > 20 transitive dependencies               → Supply chain exposure
  No dependency version pinning              → Non-reproducible builds
```

### Architecture-Level Signals

```
STRUCTURAL SMELLS:
  Big Ball of Mud                → No discernible structure
  Distributed Monolith           → Microservices with shared DB or sync calls everywhere
  Chatty Interfaces              → 30 API calls to render one page
  Spaghetti Integration          → Every service calls every other service directly
  Anemic Domain Model            → Domain objects with no behavior (just getters/setters)
  Missing Abstraction            → Business logic duplicated across services

PROCESS SIGNALS:
  Team velocity declining sprint over sprint
  Fear of changing certain files ("the scary module")
  New features require changes in 10+ files
  Bug fixes frequently introduce new bugs
  Onboarding takes months, not weeks
  "We can't refactor X — it touches everything"
```

### Fitness Functions (Automated Detection)

```java
// ArchUnit example — encode architecture rules as executable tests
@AnalyzeClasses(packages = "com.example")
class ArchitectureTests {

    @ArchTest
    static final ArchRule domainMustNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");

    @ArchTest
    static final ArchRule servicesMustNotCallEachOther =
        noClasses()
            .that().resideInAPackage("..service..")
            .should().dependOnClassesThat()
            .resideInAPackage("..service..");

    @ArchTest
    static final ArchRule cyclicDependenciesForbidden =
        slices()
            .matching("com.example.(*)..")
            .should().beFreeOfCycles();
}
```

```python
# Python equivalent using import-linter or custom pytest
# Check no business logic in API layer
def test_no_business_logic_in_routes():
    route_files = glob("src/api/routes/*.py")
    for file in route_files:
        content = Path(file).read_text()
        assert "if " not in content or content.count("if ") < 3, \
            f"{file} has too many conditionals — move logic to service layer"
```

---

## Quantifying Technical Debt

### SonarQube / SQALE Method

```
SQALE Technical Debt Ratio = Remediation Cost / Development Cost

Ratings:
  A: 0-5%     (healthy)
  B: 6-10%    (some debt, manageable)
  C: 11-20%   (significant debt, slowing team)
  D: 21-50%   (high debt, risky changes)
  E: > 50%    (crisis — consider rewrite)
```

**Key SonarQube metrics to track:**
```
sqale_debt_ratio      → Overall technical debt ratio (target: < 10%)
complexity            → Cyclomatic complexity per file/method
duplicated_lines_density → Code duplication (target: < 3%)
coverage              → Test coverage (minimum: 70% on new code)
security_hotspots     → Security-sensitive code requiring review
```

### Story Point Estimation Method

```
For each debt item, estimate:
  EFFORT to fix (story points or hours)
  COST of NOT fixing (story points of slowdown per sprint)

Interest Rate = Cost of Not Fixing / Effort to Fix

  Interest Rate > 1x per sprint → High priority (paying more to carry it than fix it)
  Interest Rate 0.5x-1x        → Medium priority
  Interest Rate < 0.5x         → Low priority (low interest debt)

Example:
  Fixing legacy auth module: 8 story points
  Slowdown caused per sprint: 3 story points
  Interest rate: 3/8 = 0.375x per sprint
  Break-even: 8/3 = 2.7 sprints
```

---

## Tech Debt Register

Every tracked debt item needs these fields:

```markdown
| ID    | Description                         | Type         | Owner     | Effort | Interest Rate | Business Impact      | Priority | Status  |
|-------|-------------------------------------|--------------|-----------|--------|---------------|----------------------|----------|---------|
| TD-01 | Legacy auth service — no tests      | Inadvertent  | @team     | 8 SP   | High          | Blocks SSO feature   | P1       | Active  |
| TD-02 | Duplicated pricing logic (3 copies) | Deliberate   | @alice    | 5 SP   | Medium        | Bugs ship 3x         | P2       | Active  |
| TD-03 | Outdated ORM version (EOL Q2)       | External     | @bob      | 3 SP   | High          | Security risk        | P1       | Active  |
| TD-04 | God class: OrderService (1200 LOC)  | Inadvertent  | @carol    | 13 SP  | Medium        | New features slow    | P2       | Planned |
```

**Priority score formula:**
```
Priority Score = (Business Impact × 3) + (Interest Rate × 2) + (Security Risk × 5)

Business Impact: 1 (low) to 5 (blocks roadmap)
Interest Rate:   1 (< 0.25x/sprint) to 5 (> 2x/sprint)
Security Risk:   0 (no risk) to 1 (active risk)
```

---

## Prioritization: Interest Rate over Principal

```
KEY INSIGHT: A $10,000 debt at 30% monthly interest is more urgent than
             a $50,000 debt at 1% monthly interest.

Debt that BLOCKS new features:     highest interest rate → fix first
Debt that CAUSES recurring bugs:   compounding interest → fix next
Debt that SLOWS development:       medium interest → schedule
Debt that's MERELY ugly:           near-zero interest → low priority or Boy Scout
```

**Prioritization anti-pattern:** Fixing the largest, most intimidating debt first without assessing interest rate. Teams spend months on a cleanup that has low impact while high-interest items silently compound.

---

## Paydown Strategies

### 1. Boy Scout Rule (Default Strategy)

```
"Always leave the code cleaner than you found it."
— No dedicated sprints required
— Applied during normal feature work
— Small, continuous improvement
— Works for: naming, extract method, add test, remove duplication

HOW TO APPLY:
  When touching file X for feature Y:
    → Rename one unclear variable
    → Extract one long method
    → Add tests for one untested branch
    → Remove one dead code block

WHAT IT DOESN'T WORK FOR:
  → Structural refactors (cross-file changes)
  → Replacing a library or framework
  → Database schema changes
```

### 2. Strangler Fig Pattern

```
GOAL: Replace legacy system incrementally while keeping it alive and running.

         BEFORE                          DURING                          AFTER
  +----------------+           +------------------+              +----------------+
  |   LEGACY SVC   |           |  FACADE / PROXY  |              |   NEW SVC      |
  | (monolith/old) |    →      |   routes traffic  |    →         | (clean impl)   |
  +----------------+           +--+----------+----+              +----------------+
                                  |          |
                            NEW SVC    LEGACY SVC
                            (new paths)  (old paths)

STEPS:
  1. Put a routing layer (facade/proxy) in front of legacy
  2. Implement NEW behavior in the new service
  3. Route NEW traffic to new service; legacy handles the rest
  4. Migrate paths one-by-one from legacy to new
  5. Remove legacy when 0% traffic reaches it
```

```python
# Strangler Fig in code — feature flag routing
class PaymentProcessor:
    def process(self, payment: Payment) -> Result:
        if feature_flags.is_enabled("new-payment-engine", payment.user_id):
            return self.new_engine.process(payment)  # new implementation
        return self.legacy_engine.process(payment)   # old implementation
```

### 3. Branch by Abstraction

```
GOAL: Replace an implementation that many callers depend on,
      without a big-bang rewrite.

STEPS:

  Step 1: Create abstraction over existing behavior
  +------------+          +-------------+          +----------+
  | Callers    |  ----→   | Abstraction |  ----→   | Legacy   |
  +------------+          +-------------+          +----------+

  Step 2: Write new implementation behind abstraction
  +------------+          +-------------+    →   +----------+
  | Callers    |  ----→   | Abstraction |  ----→ | Legacy   |
  +------------+          +-------------+    →   +----------+
                                             +-------+
                                             |  NEW  |
                                             +-------+

  Step 3: Migrate callers to new implementation (one by one)
  Step 4: Remove legacy implementation and clean up abstraction
```

```java
// Step 1: Introduce abstraction
interface UserRepository {
    Optional<User> findById(String id);
    void save(User user);
}

// Step 2: Legacy implementation wrapped
class LegacyUserRepository implements UserRepository { ... }

// Step 3: New implementation
class PostgresUserRepository implements UserRepository { ... }

// Step 4: Switch via DI, remove legacy when all tests pass
```

### 4. Scheduled Paydown

```
RULE: Reserve 20% of sprint capacity for debt paydown.
      Never drop below 10% — below this, debt compounds faster than features ship.

SPRINT ALLOCATION:
  70% → Feature delivery
  20% → Technical debt paydown
  10% → Bugs and incidents

HOW TO PROTECT THE BUDGET:
  - Put debt items in the sprint like any other story (with story points)
  - Track debt velocity separately from feature velocity
  - Report debt ratio to stakeholders — make it visible
  - Never "borrow" from debt budget for feature crunch (negotiate scope instead)
```

---

## Architecture Decision Records (ADRs)

```
PURPOSE: Record WHY a decision was made, not just WHAT was decided.
         Future engineers can understand context instead of undoing decisions blindly.
```

### ADR Template

```markdown
# ADR-[number]: [Short title]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-[N]
**Deciders:** [list of people involved]

## Context
[What situation or problem drove this decision? What forces are at play?]

## Decision
[The specific choice made. State it clearly: "We will use X."]

## Rationale
[Why this option over the alternatives? What trade-offs were made?]

## Alternatives Considered
| Option | Pros | Cons | Why rejected |
|--------|------|------|--------------|
| A      | ...  | ...  | ...          |
| B      | ...  | ...  | ...          |

## Consequences
**Positive:** [What becomes easier or better?]
**Negative:** [What debt are we taking on? What becomes harder?]
**Risks:** [What could go wrong? What triggers a revisit?]

## Revisit Trigger
[Condition under which this decision should be re-evaluated]
```

**ADR storage:** Commit to the repository at `docs/decisions/ADR-NNNN-title.md`. Treat ADRs as code — they are reviewed, approved, and versioned.

---

## Architecture Fitness Functions

```
DEFINITION: An automated test that encodes an architectural rule.
            If the rule is violated, the build breaks.

PRINCIPLE:  "If it's not a test, it's not enforced."
```

### Examples by Category

```
DEPENDENCY DIRECTION (ArchUnit/jQAssistant):
  domain must not depend on infrastructure
  API layer must not contain business logic
  Circular dependencies between modules → build fails

PERFORMANCE:
  No SQL query may scan a table > 1M rows without an index
  p99 response time of critical endpoints < 500ms (contract test)

SECURITY:
  No hardcoded credentials (regex scan in CI)
  All external inputs validated before reaching service layer
  No plaintext passwords in logs

SIZE LIMITS:
  No class > 400 lines
  No method > 30 lines
  No file with cyclomatic complexity > 15

COUPLING:
  Service A may not call Service B synchronously (only via events)
  No shared database tables between service boundaries
```

---

## Rewrite vs Refactor Decision Framework

```
                    START
                      |
        Is the codebase testable?
         /                    \
        YES                    NO
         |                      |
  Is the team                Can it be made testable
  productive?                incrementally?
   /      \                   /           \
  YES      NO               YES            NO
   |        |                |              |
Refactor  Is tech          Make it        Is tech EOL
 incre-    EOL?           testable, then   AND team
 mentally   |             refactor        near zero
           YES                           productivity?
            |                             /        \
         Rewrite                        YES         NO
                                         |           |
                                      Rewrite    Refactor
                                                (hard, but
                                                 possible)
```

### Rewrite Criteria (ALL must be true)

- [ ] Codebase is fundamentally untestable (cannot add tests without rewriting logic)
- [ ] Technology is end-of-life with no migration path
- [ ] Team productivity is near zero (velocity declining for 6+ months)
- [ ] The new implementation can be proven correct before legacy is removed
- [ ] Business can fund and tolerate a 6-18 month parallel investment

**Warning:** Most rewrites take 3-5x longer than estimated and recreate the same problems unless the architecture and team practices change.

---

## Anti-Patterns

```
+----------------------------+------------------------------------------------------+
| Anti-Pattern               | Why It Fails                                         |
+----------------------------+------------------------------------------------------+
| "Debt Sprint"              | One sprint cannot fix years of compounding debt;     |
|                            | creates illusion of progress, then reverts           |
+----------------------------+------------------------------------------------------+
| Zero-Tolerance Debt Policy | Debt is a tool; some debt is rational; zero-         |
|                            | tolerance causes paralysis and misses ship windows   |
+----------------------------+------------------------------------------------------+
| No Tracking                | Invisible debt grows exponentially; team "forgets"  |
|                            | known problems until they become crises              |
+----------------------------+------------------------------------------------------+
| Big-Bang Refactor          | Long-running branch diverges from main;             |
|                            | merge conflict nightmare; risk of full revert        |
+----------------------------+------------------------------------------------------+
| Refactoring without tests  | Refactoring changes structure without changing       |
|                            | behavior — impossible to verify without tests        |
+----------------------------+------------------------------------------------------+
| "We'll clean it up later"  | "Later" never comes without a specific date,         |
| (without a date)           | owner, and allocated capacity                        |
+----------------------------+------------------------------------------------------+
| Debt tracked in documents  | Items not in the issue tracker don't get done;      |
| not issue trackers         | register must be actionable                          |
+----------------------------+------------------------------------------------------+
```

---

## Quick Reference Checklist

### Identifying Debt

- [ ] Run static analysis (SonarQube, ESLint, Pylint) on the codebase
- [ ] Check cyclomatic complexity — flag methods > 10
- [ ] Check test coverage — flag paths below 70%
- [ ] Run dependency audit (`npm audit`, `pip-audit`, `gradle dependencyCheckAnalyze`)
- [ ] Run architecture fitness functions — fix violations before merge
- [ ] Review team velocity trend — declining? Investigate
- [ ] Survey team: "What file scares you most to change?"

### Managing Debt

- [ ] All debt items in the issue tracker with owner and estimated effort
- [ ] Priority score calculated for each item (impact × interest rate)
- [ ] 20% sprint capacity reserved for debt paydown
- [ ] ADR written for every significant architectural decision
- [ ] Architecture fitness functions running in CI pipeline
- [ ] Quarterly debt review: add new items, close resolved items, update priorities
- [ ] Debt ratio reported to stakeholders (make it visible, not shameful)
