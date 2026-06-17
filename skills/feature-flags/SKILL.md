---
name: feature-flags
description: Invoked when the user needs to implement, architect, or clean up feature flags — including progressive rollout, A/B testing, kill switches, OpenFeature integration, Spring/Python SDKs, and flag lifecycle management.
---

# Feature Flags and Progressive Delivery

## Why Feature Flags

Feature flags **decouple deployment from release**. Code ships to production disabled. It is enabled for users deliberately, incrementally, and reversibly.

```
Without flags:                      With flags:
Deploy = Release                    Deploy != Release

[build] -> [test] -> [deploy]       [build] -> [test] -> [deploy] -> [enable for 1%]
                         |                                                  |
                    RISK: instant                                    [enable for 25%]
                    exposure to all                                         |
                    users                                           [enable for 100%]
                                                                           |
                                                                    [clean up flag]
```

Benefits:
- Instant rollback: flip flag off, no redeploy needed
- Trunk-based development: merge incomplete features behind a flag; no long-lived branches
- Canary releases: expose new code to 1% of users, watch metrics, then expand
- Kill switches: disable a feature instantly during an incident without a deploy

---

## Flag Types

```
+------------------+---------------------+---------------+-----------------------------+
| Type             | Purpose             | Lifespan      | Example                     |
+------------------+---------------------+---------------+-----------------------------+
| Release flag     | Hide incomplete     | Days to weeks | new-checkout-flow           |
|                  | feature             |               |                             |
+------------------+---------------------+---------------+-----------------------------+
| Experiment flag  | A/B testing;        | Days to weeks | checkout-cta-color          |
|                  | percentage rollout  | (until winner)|                             |
+------------------+---------------------+---------------+-----------------------------+
| Ops flag         | Kill switch;        | Long-lived    | rate-limiter-enabled        |
|                  | incident response   | (permanent)   |                             |
+------------------+---------------------+---------------+-----------------------------+
| Permission flag  | Per-user/tenant     | Permanent     | beta-dashboard-access       |
|                  | feature access      |               |                             |
+------------------+---------------------+---------------+-----------------------------+
```

---

## Flag Lifecycle

Flags that are never cleaned up are **technical debt**. Every flag needs an owner and an end date.

```
    CREATE           DEVELOP          TEST           ROLL OUT         CLEAN UP
      |                |               |                |                |
  Define flag      Code behind     Test both        1% -> 5%        Remove flag
  in system        the flag        code paths       -> 25%          Remove old
  Add to           Never merge     Use flag         -> 100%         code path
  backlog          without flag    overrides        (with metrics   Delete from
                                   in tests         gate)           flag system
                                                                    Delete tests
                                                                    for old path
```

A flag is not "done" when it reaches 100% rollout. It is done when the code, tests, and flag definition are all cleaned up.

---

## Targeting Rules

```
Targeting evaluation order (first match wins):
+-----------------------------------------+
| 1. Individual user overrides            |
|    userId == "user-123" -> TRUE         |
+-----------------------------------------+
| 2. Segment rules                        |
|    country == "US" AND plan == "pro"    |
|    -> TRUE                              |
+-----------------------------------------+
| 3. Percentage rollout                   |
|    Hash(userId) % 100 < 25             |
|    -> TRUE (25% of remaining users)    |
+-----------------------------------------+
| 4. Default value                        |
|    -> FALSE                             |
+-----------------------------------------+
```

Common targeting attributes:
- `userId`: stable ID for consistent assignment
- `email`: segment by domain (`@enterprise.com`)
- `country`: geographic rollout
- `plan`: free / pro / enterprise
- `tenantId`: per-tenant SaaS rollout
- `beta`: explicit opt-in flag

---

## Gradual Rollout Pattern

```
Week 1: 1%   -> Monitor error rate, latency, business metrics
              -> If metrics look good, proceed
Week 1: 5%   -> Monitor. Look for edge cases.
Week 2: 25%  -> Monitor. This is when you find most real-world issues.
Week 2: 50%  -> Compare A/B metrics: conversion, latency, errors
Week 3: 100% -> Full rollout
Week 3+:     -> CLEAN UP the flag (most teams skip this — don't)
```

Metrics gate between each step:
```
Decision: can we advance the rollout?
  YES if: error rate delta < 0.1%
          AND latency P95 delta < 10%
          AND no customer-reported regressions
          AND experiment metric moving in right direction
  NO if:  any of the above fails -> roll back to 0%, investigate
```

---

## SDK Patterns

### The Golden Rule

Flag evaluation should be simple, boolean, and free of business logic:

```python
# CORRECT: flag controls routing to implementation
if feature_flags.is_enabled("new-checkout", user_id=user.id):
    result = new_checkout_service.process(order)
else:
    result = legacy_checkout_service.process(order)

# WRONG: business logic inside flag check
if feature_flags.get_value("checkout-discount"):
    price = price * 0.8  # business logic should NOT be here
```

### Flag as Dependency Injection

Instead of scattered `if` blocks, inject the implementation:

```java
// WRONG: if blocks scattered everywhere
public OrderResult processOrder(Order order) {
    if (featureFlags.isEnabled("new-validation", userId)) {
        validateWithNewRules(order);
    } else {
        validateWithOldRules(order);
    }
    
    if (featureFlags.isEnabled("new-pricing", userId)) {
        applyNewPricing(order);
    } else {
        applyOldPricing(order);
    }
    // ...
}

// CORRECT: inject the right implementation at the entry point
public CheckoutService checkoutServiceFor(UserId userId) {
    if (featureFlags.isEnabled("new-checkout-service", userId)) {
        return newCheckoutService;
    }
    return legacyCheckoutService;
}

// Then the caller just uses checkoutService without knowing which one
```

---

## OpenFeature: Vendor-Neutral Standard

OpenFeature provides a standard SDK that works with any flag provider (LaunchDarkly, Unleash, Flipt, etc.).

```
+------------------+
|  Application     |   openfeature-sdk
|  Code            | <----------------> Provider Adapter <-> Flag Backend
+------------------+                   (LaunchDarkly,         (LaunchDarkly,
                                        Unleash, Flipt, etc.)  Unleash, etc.)
```

### Java / Spring

```java
// Dependency
// io.openfeature:sdk:1.x

// Register a provider (example: Flagd)
OpenFeatureAPI.getInstance().setProvider(new FlagdProvider());

// Get a client
FeatureClient client = OpenFeatureAPI.getInstance().getClient();

// Evaluate flags
boolean isEnabled = client.getBooleanValue("new-checkout", false, ctx);
String variant    = client.getStringValue("button-color", "blue", ctx);
int limit         = client.getIntegerValue("rate-limit", 100, ctx);
```

Evaluation context:
```java
ImmutableContext ctx = new ImmutableContext(
    userId,
    Map.of(
        "email",   Value.objectToValue(user.getEmail()),
        "plan",    Value.objectToValue(user.getPlan()),
        "country", Value.objectToValue(user.getCountry())
    )
);
```

### Spring Integration: FeatureFlagService Bean

```java
@Service
public class FeatureFlagService {

    private final FeatureClient client;

    public FeatureFlagService(FeatureClient client) {
        this.client = client;
    }

    public boolean isNewCheckoutEnabled(User user) {
        ImmutableContext ctx = buildContext(user);
        return client.getBooleanValue("new-checkout", false, ctx);
    }

    private ImmutableContext buildContext(User user) {
        return new ImmutableContext(user.getId(), Map.of(
            "plan", Value.objectToValue(user.getPlan().name()),
            "country", Value.objectToValue(user.getCountry())
        ));
    }
}
```

Spring conditional (for infrastructure-level flags):
```java
@ConditionalOnProperty(name = "features.new-cache", havingValue = "true")
@Bean
public CacheService newCacheService() {
    return new NewCacheService();
}
```

Note: `@ConditionalOnProperty` is evaluated at startup. For runtime-dynamic flags, use `FeatureFlagService` directly.

### Python Integration

```python
# pip install openfeature-sdk

from openfeature import api
from openfeature.evaluation_context import EvaluationContext

# Configure provider (example: Flagd)
from openfeature.provider.flagd.provider import FlagdProvider
api.set_provider(FlagdProvider())

client = api.get_client()

# Evaluate
ctx = EvaluationContext(
    targeting_key=user.id,
    attributes={
        "plan": user.plan,
        "country": user.country,
        "email": user.email,
    }
)

is_enabled = client.get_boolean_value("new-checkout", False, ctx)
variant     = client.get_string_value("algorithm-variant", "v1", ctx)
```

Python with Unleash:
```python
# pip install UnleashClient

from UnleashClient import UnleashClient

client = UnleashClient(
    url="https://unleash.example.com/api",
    app_name="my-service",
    custom_headers={"Authorization": "Bearer <token>"}
)
client.initialize_client()

# Check flag (with context)
context = {"userId": str(user.id), "properties": {"plan": user.plan}}
if client.is_enabled("new-checkout", context):
    return new_checkout_service.process(order)
```

---

## Tool Comparison

```
+------------------+-------------+----------+------------------+-------------------+
| Tool             | Type        | Cost     | Key strength     | Notes             |
+------------------+-------------+----------+------------------+-------------------+
| LaunchDarkly     | SaaS        | $$$      | Full-featured,   | Industry standard |
|                  |             |          | best SDKs        | for large teams   |
+------------------+-------------+----------+------------------+-------------------+
| Unleash          | Open-source | Free     | Self-hosted,     | Best open-source  |
|                  | (+ SaaS)    | (+ SaaS) | GDPR-friendly    | option            |
+------------------+-------------+----------+------------------+-------------------+
| Flipt            | Open-source | Free     | GitOps-native,   | Good for K8s-    |
|                  |             |          | gRPC API         | native teams      |
+------------------+-------------+----------+------------------+-------------------+
| Flagsmith        | Open-source | Free     | Remote config    | Good for mobile  |
|                  | (+ SaaS)    | (+ SaaS) | + flags          |                   |
+------------------+-------------+----------+------------------+-------------------+
| OpenFeature      | SDK std.    | Free     | Vendor neutral   | Use with any of   |
|                  |             |          |                  | the above         |
+------------------+-------------+----------+------------------+-------------------+
```

---

## Metrics and Experimentation

### Track per Variant

For every flag rollout, define the metrics to watch before enabling:

```
Flag: new-checkout-flow
Variants: control (old), treatment (new)
Metrics to track:
  Primary:   checkout_completion_rate  (want: treatment >= control)
  Guardrail: checkout_error_rate       (want: treatment <= control + 0.1%)
  Guardrail: p95_checkout_latency_ms   (want: treatment <= control * 1.1)
  Secondary: average_order_value       (observe, don't gate on)
```

### Statistical Significance

Do not conclude an experiment early because it looks like it's winning:

```
Rule: Run experiment for minimum 2 weeks (captures weekly seasonality)
      Reach 95% statistical significance before declaring winner
      Correct for multiple hypothesis testing if tracking many metrics

Bad: "After 2 days the new flow has 2% higher conversion — ship it!"
Good: "After 14 days, 95% confidence interval for conversion delta: [+1.2%, +3.1%]
       — the new flow is better; ship and clean up flag."
```

---

## Testing with Flags

### Test Both Code Paths

```java
@Test
void newCheckoutFlow_whenFlagEnabled() {
    // Override flag in test
    featureFlagService.override("new-checkout", true);
    
    OrderResult result = checkoutService.process(testOrder);
    
    assertThat(result.getVersion()).isEqualTo("v2");
}

@Test
void legacyCheckoutFlow_whenFlagDisabled() {
    featureFlagService.override("new-checkout", false);
    
    OrderResult result = checkoutService.process(testOrder);
    
    assertThat(result.getVersion()).isEqualTo("v1");
}

// Also test the removal: after flag is cleaned up, there should be
// no test referencing the old code path.
```

### In-Memory Test Double

```java
@TestConfiguration
public class TestFeatureFlagConfig {

    @Bean
    @Primary
    public FeatureFlagService testFeatureFlagService() {
        return new InMemoryFeatureFlagService(); // all flags default to false
    }
}

// Or use a map-backed implementation
public class InMemoryFeatureFlagService implements FeatureFlagService {
    private final Map<String, Boolean> overrides = new HashMap<>();
    
    public void override(String flag, boolean value) {
        overrides.put(flag, value);
    }
    
    @Override
    public boolean isEnabled(String flag, String userId) {
        return overrides.getOrDefault(flag, false);
    }
}
```

```python
# pytest fixture
@pytest.fixture
def feature_flags(monkeypatch):
    flags = {}
    
    def mock_is_enabled(flag_name, default=False, ctx=None):
        return flags.get(flag_name, default)
    
    monkeypatch.setattr(feature_flag_client, "get_boolean_value", mock_is_enabled)
    return flags  # tests can set flags["new-checkout"] = True

def test_new_checkout(feature_flags):
    feature_flags["new-checkout"] = True
    result = checkout_service.process(order)
    assert result["version"] == "v2"
```

---

## Anti-Patterns

```
WRONG: Flag controls business logic, not routing.
       if flags.is_enabled("premium-discount"):
           price = price * 0.8  # business logic in flag check
RIGHT: Flag selects which pricing strategy to use.
       pricing = premium_pricing if flags.is_enabled("premium-pricing") else standard_pricing
       price = pricing.calculate(order)

WRONG: Permanent flags with no owner and no cleanup date.
RIGHT: Every flag has an owner (team/person) and a planned removal date in the backlog.

WRONG: Testing only the enabled code path.
RIGHT: Write tests for both paths. CI must pass with flag ON and OFF.

WRONG: Skipping flag removal after 100% rollout.
RIGHT: Add a "clean up flag X" ticket to the sprint immediately after 100% rollout.
       Flag cleanup = delete the flag from the flag system + remove the if/else code
       + delete the dead code path + delete associated tests.

WRONG: Using one flag to control multiple behaviors.
       if flags.is_enabled("new-system"):
           use_new_db()
           use_new_cache()
           use_new_api()
RIGHT: One flag per behavior. Granular flags allow partial rollback.

WRONG: Checking the flag in every function throughout the call stack.
RIGHT: Check the flag once at the entry point, inject the right implementation.

WRONG: Not tracking which version of code a user got.
RIGHT: Log flag values alongside request traces. Essential for debugging incidents.

WRONG: Letting developers keep flags enabled in local dev without documentation.
RIGHT: Document default values for local development. Use a dev-defaults config.
```

---

## Checklist: Rolling Out a New Feature

```
Before development:
[ ] Create flag in flag system with description and owner
[ ] Define targeting rules for rollout stages
[ ] Define success and guardrail metrics
[ ] Agree on rollout timeline and criteria to advance

During development:
[ ] Code is behind flag (flag check at entry point, not scattered)
[ ] Both code paths are tested (flag ON and flag OFF)
[ ] Flag override works in local dev environment

Before production enablement:
[ ] Smoke test with 0% real users (internal team override)
[ ] Verify metrics instrumentation is working
[ ] Create rollback runbook (just: "set flag to 0%")

Rollout:
[ ] 1%: monitor for 24h, check all guardrail metrics
[ ] 5%: monitor for 24h
[ ] 25%: monitor for 48h, check statistical significance if experiment
[ ] 50%: monitor for 48h, make go/no-go decision
[ ] 100%: full rollout

Cleanup (within 1-2 weeks of 100%):
[ ] Remove flag check from code
[ ] Delete old code path
[ ] Delete tests for old code path
[ ] Delete flag from flag system
[ ] Close the flag ticket in backlog
```
