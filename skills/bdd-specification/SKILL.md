---
name: bdd-specification
description: Apply Behaviour-Driven Development — Three Amigos, Gherkin, living documentation, step definitions, and BDD/TDD integration — when designing features collaboratively or writing acceptance specifications.
---

# BDD Specification

## Core Philosophy

BDD is a communication technique. It aligns business, development, and testing by expressing system behavior in a shared, structured language before a line of code is written. The executable specification is the contract; the passing scenario is the proof.

**Rule**: A scenario is a conversation between a business person and a machine. If a business person cannot read and validate it, it is not BDD — it is test automation with extra steps.

---

## The Three Amigos

Three perspectives that must align before any story is accepted for development:

```
        +------------------+
        |   BUSINESS       |
        |   ANALYST /      |
        |   PRODUCT OWNER  |
        |                  |
        |  "What value     |
        |   are we         |
        |   delivering?"   |
        +--------+---------+
                 |
                 | Discovery session
                 | (before sprint)
          +------+------+
          |             |
+---------+----+   +----+-----------+
|  DEVELOPER   |   |    TESTER      |
|              |   |                |
| "How will   |   | "How can this  |
|  we build   |   |  go wrong?"    |
|  this?"      |   |                |
+--------------+   +----------------+
```

**What each role contributes:**

| Role | Question | Contribution |
|------|----------|--------------|
| Business Analyst / PO | What outcome do we want? | Business rules, acceptance criteria |
| Developer | How will this work technically? | Constraints, edge cases, technical limits |
| Tester | What could go wrong? | Error scenarios, boundary conditions, unhappy paths |

**Three Amigos rules:**
- Meet before the story enters the sprint — discovery is upstream of development
- Use concrete examples, not abstract rules: "if the user has bought 3 items, does the discount apply?" not "discount applies to eligible users"
- The meeting is a success when all three can write the same scenario independently
- No more than 2–3 stories per session; depth over breadth

---

## Specification by Example

Transform business rules into concrete examples before writing scenarios.

**Business rule (abstract):** "Premium customers receive a loyalty discount."

**Questions to derive examples:**
- What qualifies as a premium customer? (>= 12 months, or >= 5 orders?)
- What discount? (percentage, fixed, or tiered?)
- Does it combine with promotional discounts?
- What happens on the exact boundary?

**Derived concrete examples:**

| Context | Action | Outcome |
|---------|--------|---------|
| Customer with 5+ orders | Checks out with full-price item | 15% discount applied |
| Customer with 4 orders | Checks out with full-price item | No discount |
| Premium customer + SALE item | Checks out | Discount not stacked |
| Premium customer | Checks out in different currency | Discount applies to that currency |

Each row in this table becomes a scenario. The examples reveal edge cases before any code is written.

---

## Gherkin Syntax

Gherkin is the structured natural language used to write executable specifications.

### Full syntax reference

```gherkin
Feature: Loyalty discount for premium customers
  As a premium customer
  I want to receive a loyalty discount on full-price items
  So that I am rewarded for my continued business

  Background:
    Given the product catalog contains:
      | sku     | name        | price  | on_sale |
      | SKU-001 | Dress Shirt | 200.00 | false   |
      | SKU-002 | Sale Jacket | 150.00 | true    |

  Scenario: Premium customer receives 15% discount
    Given I am a premium customer with 6 completed orders
    When I check out with item "SKU-001"
    Then my order total should be 170.00
    And the discount applied should be 30.00

  Scenario: Standard customer receives no discount
    Given I am a standard customer with 3 completed orders
    When I check out with item "SKU-001"
    Then my order total should be 200.00
    And no discount should be applied

  Scenario: Premium customer does not receive discount on sale items
    Given I am a premium customer with 6 completed orders
    When I check out with item "SKU-002"
    Then my order total should be 150.00
    And no discount should be applied

  Scenario Outline: Discount threshold is exactly 5 completed orders
    Given I am a customer with <order_count> completed orders
    When I check out with item "SKU-001"
    Then my order total should be <expected_total>

    Examples:
      | order_count | expected_total |
      | 4           | 200.00         |
      | 5           | 170.00         |
      | 6           | 170.00         |
      | 100         | 170.00         |
```

### Keyword semantics

| Keyword | Meaning |
|---------|---------|
| `Feature` | The capability being specified. One .feature file per feature. |
| `Background` | Steps run before every scenario in the file. Use sparingly. |
| `Scenario` | A single concrete example of behavior. |
| `Scenario Outline` | A template with multiple examples (a data table). |
| `Examples` | The data table for `Scenario Outline`. |
| `Given` | Pre-condition / system state before the action. |
| `When` | The action the user or system takes. One per scenario. |
| `Then` | The observable outcome. What we assert. |
| `And` / `But` | Continuation of the previous keyword (same semantics). |
| `*` | Bullet-style — use when Given/When/Then would feel forced (setup lists). |

---

## Living Documentation

The relationship between scenarios and documents:

```
Business requirement
        |
        v
  Three Amigos session
        |
        v
  Concrete examples
        |
        v
  Gherkin .feature files  <---- Version controlled alongside code
        |
        v
  Step definitions        <---- Bind Gherkin to production code
        |
        v
  Automated test run
        |
        v
  Published report        <---- Human-readable, always current
  (Cucumber Reports,
   Allure, Serenity)
```

**Living documentation rules:**
- Feature files live in the same repository as the code
- Scenarios must stay green at all times; a red scenario is a broken contract
- Non-technical stakeholders can read the published report and understand what the system does
- Never update a scenario to make it pass — fix the code or renegotiate the business rule

---

## Step Definitions

Step definitions bind Gherkin phrases to code. They are the glue layer.

### Java (Cucumber-JVM)

```java
// src/test/java/steps/LoyaltyDiscountSteps.java

public class LoyaltyDiscountSteps {

    private Customer customer;
    private Order order;
    private final ProductCatalog catalog = TestCatalog.load();
    private final OrderService orderService = new OrderService(
        new InMemoryOrderRepository(),
        new LoyaltyDiscountPolicy()
    );

    @Given("I am a premium customer with {int} completed orders")
    public void iAmAPremiumCustomerWithOrders(int orderCount) {
        customer = CustomerFixture.withCompletedOrders(orderCount);
        // 5+ orders = premium tier
    }

    @Given("I am a standard customer with {int} completed orders")
    public void iAmAStandardCustomerWithOrders(int orderCount) {
        customer = CustomerFixture.withCompletedOrders(orderCount);
    }

    @When("I check out with item {string}")
    public void iCheckOutWithItem(String sku) {
        Product product = catalog.findBySku(sku);
        Cart cart = Cart.withItem(product);
        order = orderService.checkout(cart, customer);
    }

    @Then("my order total should be {bigdecimal}")
    public void myOrderTotalShouldBe(BigDecimal expected) {
        assertThat(order.total().amount())
            .as("order total")
            .isEqualByComparingTo(expected);
    }

    @Then("the discount applied should be {bigdecimal}")
    public void theDiscountAppliedShouldBe(BigDecimal expected) {
        assertThat(order.discountAmount().amount())
            .as("discount amount")
            .isEqualByComparingTo(expected);
    }

    @Then("no discount should be applied")
    public void noDiscountShouldBeApplied() {
        assertThat(order.discountAmount().amount())
            .as("discount amount")
            .isEqualByComparingTo(BigDecimal.ZERO);
    }
}
```

### Python (Behave)

```python
# features/steps/loyalty_discount_steps.py

from behave import given, when, then
from decimal import Decimal
from myapp.domain import Customer, Cart, Product
from myapp.services import OrderService
from myapp.policies import LoyaltyDiscountPolicy
from tests.fixtures import CustomerFixture, TestCatalog

@given("I am a premium customer with {order_count:d} completed orders")
def step_premium_customer(context, order_count):
    context.customer = CustomerFixture.with_completed_orders(order_count)

@given("I am a standard customer with {order_count:d} completed orders")
def step_standard_customer(context, order_count):
    context.customer = CustomerFixture.with_completed_orders(order_count)

@when('I check out with item "{sku}"')
def step_checkout_item(context, sku):
    product = TestCatalog.find_by_sku(sku)
    cart = Cart(items=[product])
    service = OrderService(policy=LoyaltyDiscountPolicy())
    context.order = service.checkout(cart, context.customer)

@then("my order total should be {expected}")
def step_order_total(context, expected):
    assert context.order.total == Decimal(expected), (
        f"Expected total {expected}, got {context.order.total}"
    )

@then("no discount should be applied")
def step_no_discount(context):
    assert context.order.discount == Decimal("0.00"), (
        f"Expected no discount, got {context.order.discount}"
    )
```

### Kotlin (Cucumber-JVM with Kotlin DSL)

```kotlin
// src/test/kotlin/steps/LoyaltyDiscountSteps.kt

class LoyaltyDiscountSteps : En {
    private lateinit var customer: Customer
    private lateinit var order: Order
    private val catalog = TestCatalog.load()
    private val service = OrderService(LoyaltyDiscountPolicy())

    init {
        Given("I am a premium customer with {int} completed orders") { count: Int ->
            customer = CustomerFixture.withCompletedOrders(count)
        }

        When("I check out with item {string}") { sku: String ->
            val product = catalog.findBySku(sku)
            val cart = Cart(listOf(product))
            order = service.checkout(cart, customer)
        }

        Then("my order total should be {bigdecimal}") { expected: BigDecimal ->
            order.total.amount shouldBe expected
        }

        Then("no discount should be applied") {
            order.discountAmount.amount shouldBe BigDecimal.ZERO
        }
    }
}
```

---

## Domain Language in Scenarios

Scenarios must use the Ubiquitous Language of the domain. If the domain model uses the word "Premium", the scenario must say "Premium". If the code calls something else, the code is wrong.

**Bad — technical language leaks into scenarios:**
```gherkin
Scenario: Database record updated when discount flag is true
  Given the customer row has orders_count = 5 and is_premium = true
  When the checkout endpoint receives POST /api/orders with sku="SKU-001"
  Then the response body contains {"discount": 30.00}
```

**Good — domain language throughout:**
```gherkin
Scenario: Premium customer receives loyalty discount at checkout
  Given Alice is a premium customer
  When Alice checks out with a full-price shirt at 200.00
  Then her order total is 170.00
  And the 15% loyalty discount is displayed on her receipt
```

**Rules:**
- Use role names, not technical roles: "the customer", "the warehouse operator", "the fraud analyst"
- Use domain terms: "checkout" not "POST /api/orders"; "cancel order" not "set status to CANCELLED"
- Quantities and values from the domain: "full-price item" not "item with no discount_flag"
- Past tense for history: "has placed 5 orders" not "has orders_count = 5"

---

## When BDD Adds Value vs When It's Overhead

### BDD adds the most value when:
- The business rules are complex and likely to be misunderstood
- Multiple stakeholders need to agree on behavior before development
- The feature will be maintained for years by changing teams
- Regulatory requirements must be provably satisfied
- There are many edge cases that business needs to explicitly sign off on

### BDD adds little value when:
- The team is one developer and no business stakeholders exist
- The feature is purely technical (caching, logging, infrastructure)
- The behavior is trivial and unambiguous
- The scenarios would just be translations of unit tests into Gherkin

### Signal that BDD is being misused:
- Scenarios written by developers alone, after the code
- Scenarios that only technical people can read
- More than 10 scenarios per feature file
- Scenarios that describe HTTP requests and JSON responses
- Step definitions that duplicate what unit tests already verify

---

## Integration with TDD Cycle

BDD and TDD operate at different levels. They compose:

```
BDD (Acceptance Level)
┌─────────────────────────────────────┐
│  Scenario: Premium customer discount │  <-- Failing (RED at acceptance level)
│  [Gherkin + Step Definitions]       │
└──────────────┬──────────────────────┘
               │
               │ Step definitions call application code
               │ Application code calls domain objects
               │
               v
TDD (Unit Level)
┌─────────────────────────────────────┐
│  test_premium_customer_15_pct()     │  <-- RED
│  LoyaltyDiscountPolicy class        │  <-- GREEN
│  Refactor policy                    │  <-- REFACTOR
│  test_boundary_at_5_orders()        │  <-- RED again
│  ...                                │
└─────────────────────────────────────┘
               │
               │ All unit tests green
               │
               v
BDD (Acceptance Level)
┌─────────────────────────────────────┐
│  Scenario: Premium customer discount │  <-- Now GREEN
└─────────────────────────────────────┘
```

**Practical workflow:**
1. Three Amigos session → write Feature file and Scenarios (they are RED)
2. Run BDD test → get a clear failure: step not implemented or class missing
3. Switch to TDD at unit level to build the domain logic
4. Step definitions call real application code; BDD scenario turns GREEN
5. Add more scenarios for edge cases discovered in step 3

---

## Tools

### Cucumber (Java/Kotlin)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
```

```java
// src/test/java/CucumberRunner.java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "steps")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, value = "pretty, html:target/cucumber-report.html")
public class CucumberRunner {}
```

### Behave (Python)

```
# Install
pip install behave

# Project structure
features/
  loyalty_discount.feature
  steps/
    loyalty_discount_steps.py
  environment.py   # before/after hooks

# Run
behave
behave --tags @smoke
behave features/loyalty_discount.feature
```

### SpecFlow (.NET)

```xml
<!-- .csproj -->
<PackageReference Include="SpecFlow.NUnit" Version="3.9.74" />
<PackageReference Include="SpecFlow.Plus.LivingDocPlugin" Version="3.9.57" />
```

```csharp
[Binding]
public class LoyaltyDiscountSteps
{
    private Customer _customer;
    private Order _order;
    private readonly OrderService _service = new OrderService(new LoyaltyDiscountPolicy());

    [Given(@"I am a premium customer with (\d+) completed orders")]
    public void GivenIAmAPremiumCustomerWith(int orderCount)
    {
        _customer = CustomerFixture.WithCompletedOrders(orderCount);
    }

    [When(@"I check out with item ""(.*)""")]
    public void WhenICheckOutWithItem(string sku)
    {
        var cart = Cart.WithItem(TestCatalog.FindBySku(sku));
        _order = _service.Checkout(cart, _customer);
    }

    [Then(@"my order total should be (.*)")]
    public void ThenMyOrderTotalShouldBe(decimal expected)
    {
        _order.Total.Amount.Should().Be(expected);
    }
}
```

---

## Anti-Patterns

### UI-Level BDD

**Bad:**
```gherkin
Scenario: User logs in and applies discount
  Given I open the browser at "http://shop.example.com"
  When I click on the "Login" button
  And I fill in "email" with "alice@example.com"
  And I fill in "password" with "secret"
  And I click "Submit"
  And I navigate to "http://shop.example.com/cart"
  And I click "Apply Coupon"
  Then I should see "Discount applied: 15%"
```

This is a Selenium script in Gherkin clothing. It tests UI mechanics, not behavior. A button rename breaks it.

**Good:** Test at the service/domain level. If you need E2E UI tests, keep them in a separate suite, not as BDD scenarios.

### Technical Steps

**Bad:**
```gherkin
Given the database contains a row in the customers table with premium_tier = 1
When I send a POST request to /api/v1/orders with body {"sku": "SKU-001"}
Then the response status code is 200
And the response body JSON path $.discount equals 30.00
```

Step definitions exist; scenarios don't communicate to business people.

### Too Many Scenarios

If a feature has more than 8-10 scenarios, you are either:
- Missing a Scenario Outline (repeated structure with different data)
- Covering unit-level cases that belong in unit tests
- Describing multiple features in one file

### Scenarios Written After Code

Scenarios written after the code is working are not specifications — they are post-hoc documentation that often drift from reality. BDD value comes from the conversations, not the syntax.

### Skipping the Three Amigos

Developers writing scenarios alone produces Gherkin that only developers understand. The business analyst is the audience of the scenario; without them in the room, you're writing the wrong thing.

---

## Practical Checklist

### Before the Three Amigos session
- [ ] User story has a clear role, action, and value statement
- [ ] Acceptance criteria are written (even if abstract)
- [ ] Domain expert and tester are scheduled

### During the Three Amigos session
- [ ] We derived concrete examples, not just rules
- [ ] We found at least one edge case from the tester's perspective
- [ ] All three roles can read and agree on each scenario
- [ ] We identified what is out of scope (to avoid scope creep)

### Writing the Feature file
- [ ] Feature title names the capability, not the ticket number
- [ ] Each scenario has exactly one `When` step
- [ ] Scenarios use domain language, not technical language
- [ ] No HTTP, JSON, SQL, or UI mechanics in scenario steps
- [ ] Background is used only for setup common to ALL scenarios in the file
- [ ] Scenario Outline used for repeated structure with different data

### Step definitions
- [ ] Each step phrase matches its regex exactly and has one implementation
- [ ] No business logic in step definitions — only call application code
- [ ] Steps are composable and reusable across scenarios
- [ ] Test state is stored in the scenario context object, not static fields

### After implementation
- [ ] All scenarios are green in CI
- [ ] Report is published and accessible to business stakeholders
- [ ] Feature file is committed alongside the code that implements it
- [ ] No scenarios are tagged `@ignore` or `@wip` without a ticket tracking completion
