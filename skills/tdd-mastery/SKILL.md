---
name: tdd-mastery
description: Apply comprehensive Test-Driven Development — Red-Green-Refactor, test doubles, testing pyramid, mutation testing, and TDD schools — when writing or reviewing tests in any language.
---

# TDD Mastery

## Core Philosophy

TDD is a design technique first, a testing technique second. Writing a test before the code forces you to think about the interface, the collaborators, and the expected behavior before you think about implementation. The tests become the first client of your code.

**Rule**: Never write production code without a failing test. Never write more test code than is sufficient to fail. Never write more production code than is sufficient to pass the failing test.

---

## The Red-Green-Refactor Cycle

```
         RED
       (write a
      failing test)
          |
          v
     +----------+
     |   TEST   |  <---- Failing test defines desired behavior
     |  FAILS   |
     +----------+
          |
          v
        GREEN
      (write just
    enough code to
       pass it)
          |
          v
     +----------+
     | TEST     |  <---- Minimal implementation — may be ugly
     | PASSES   |
     +----------+
          |
          v
       REFACTOR
    (clean up code
    without changing
      behavior)
          |
          v
     +----------+
     | ALL      |  <---- Tests still green after cleanup
     | TESTS    |
     | PASS     |
     +----------+
          |
          +-----> repeat
```

**Red phase rules:**
- The test must compile (unless you're doing strict outside-in and the class doesn't exist yet)
- The test must fail for the RIGHT reason — the assertion fails, not a setup error
- One failing test at a time; do not add more red tests while you have a failing one

**Green phase rules:**
- Write the simplest code that makes the test pass — even hardcoding is acceptable temporarily
- Resist the urge to generalize prematurely; let the next test force generalization
- If green requires touching many files, the design is telling you something

**Refactor phase rules:**
- All tests must remain green throughout refactoring
- Apply the Rule of Three: when a pattern appears three times, extract it
- Refactor both test code and production code; test code has the same quality bar
- Stop refactoring when tests are green and code communicates intent clearly

---

## Test Anatomy

### Arrange-Act-Assert (AAA)

```java
@Test
void should_apply_discount_when_customer_is_premium() {
    // ARRANGE
    var customer = Customer.premium("alice@example.com");
    var cart = Cart.withItems(List.of(Item.of("SKU-001", Money.of(100, "BRL"))));
    var pricer = new DiscountPricer();

    // ACT
    Money finalPrice = pricer.calculate(cart, customer);

    // ASSERT
    assertThat(finalPrice).isEqualTo(Money.of(85, "BRL")); // 15% discount
}
```

**Rules for AAA:**
- Each section must be visually separated by a blank line or comment
- A test should have exactly one ACT — if you feel compelled to act twice, write two tests
- Assertions should test *outcomes*, not *implementation steps*
- Never put logic (loops, conditionals) in a test — if you need it, extract a helper

### Given-When-Then (BDD-flavored)

Given-When-Then is the same structure with business-facing names. Prefer it when the test reads like a business rule:

```kotlin
@Test
fun `given overdue invoice, when late fee is applied, then penalty is 2 percent per day`() {
    // Given
    val invoice = Invoice.of(Money.of(1000, "BRL"), dueDate = LocalDate.of(2024, 1, 1))
    val today = LocalDate.of(2024, 1, 6)  // 5 days overdue

    // When
    val result = LateFeePolicy.apply(invoice, asOf = today)

    // Then
    assertThat(result.lateFee).isEqualTo(Money.of(100, "BRL"))  // 5 days * 2% * 1000
}
```

---

## Test Doubles

The term "mock" is overloaded. Use precise vocabulary:

```
+---------------+----------------------------------------------------------+
| Type          | Definition                                               |
+---------------+----------------------------------------------------------+
| Dummy         | Passed but never used. Satisfies a parameter requirement.|
| Stub          | Returns canned answers. Used to control indirect inputs. |
| Spy           | Records calls. Lets you verify interactions afterward.   |
| Mock          | Pre-programmed expectations. Fails if unexpected calls.  |
| Fake          | Working implementation with shortcuts (in-memory DB).   |
+---------------+----------------------------------------------------------+
```

### When to use each

**Stub** — when the test needs a collaborator to return a specific value:
```java
// PaymentGateway is a stub: we don't care HOW it's called, just what it returns
PaymentGateway gateway = mock(PaymentGateway.class);
when(gateway.charge(any())).thenReturn(PaymentResult.success("TXN-001"));

var service = new OrderService(gateway);
Order order = service.placeOrder(cart, creditCard);

assertThat(order.status()).isEqualTo(OrderStatus.CONFIRMED);
```

**Spy** — when you need to verify a side effect occurred:
```java
// We care THAT the event was published, not what EmailService returns
EventPublisher publisher = spy(new InMemoryEventPublisher());

var service = new OrderService(gateway, publisher);
service.placeOrder(cart, creditCard);

verify(publisher).publish(argThat(event ->
    event instanceof OrderPlaced && ((OrderPlaced) event).orderId().equals(order.id())
));
```

**Mock** — when both the return value AND the interaction protocol matter:
```java
// Strict: must be called exactly once with these exact arguments
PaymentGateway gateway = mock(PaymentGateway.class, STRICT_STUBS);
when(gateway.charge(ChargeRequest.of(creditCard, Money.of(100, "BRL"))))
    .thenReturn(PaymentResult.success("TXN-001"));
```

**Fake** — when you need a realistic implementation for integration-level tests:
```java
// InMemoryOrderRepository is a fake: real logic, no database
var repository = new InMemoryOrderRepository();
var service = new OrderService(gateway, repository, publisher);
// Can now test multi-step workflows without a database
```

**Decision rule:** Prefer fakes and stubs. Use mocks sparingly — they tie tests to implementation, making refactoring painful. If you need many mocks for one test, the class has too many dependencies.

---

## The Testing Pyramid

```
                    /\
                   /  \
                  / E2E \         Few (5-10%)
                 /  Tests \       Slow, brittle, expensive
                /----------\      Test complete user journeys
               /            \
              / Integration   \   Some (15-25%)
             /     Tests       \  Test component wiring
            /                  \  Use real DB, real HTTP
           /--------------------\
          /                      \
         /     Unit Tests         \  Most (65-80%)
        /                          \ Fast (<1ms each)
       /____________________________\ Test single unit in isolation
```

**Unit tests** (the base):
- Test a single class/function in isolation
- All collaborators are test doubles
- Run in milliseconds; run on every save
- Cover all decision branches, edge cases, error paths

**Integration tests** (the middle):
- Test a slice of the application: service + repository + real database
- Use Testcontainers (Java/Kotlin) or similar for real infrastructure
- Cover the happy path and one or two error paths
- Run in CI on every push

**E2E tests** (the top):
- Test the system from the outside: HTTP call → database → HTTP response
- Cover critical user journeys only: "user can check out", "payment fails gracefully"
- Do not use test doubles; use a real (test) environment
- Run in CI before deployment to staging

**Anti-pattern — Ice Cream Cone**: many E2E, some integration, few unit. This is slow, expensive, and gives poor feedback on failures.

---

## TDD for Bug Fixes: The Prove-It Pattern

When a bug is reported, resist the impulse to fix it immediately. Instead:

```
1. REPRODUCE       Write a failing test that demonstrates the bug
                   If you can't write a test that fails, you don't understand the bug

2. VERIFY RED      Run the test — confirm it fails for the right reason

3. FIX             Write the minimal code to make the test pass

4. VERIFY GREEN    Run all tests — confirm only the new test changed state

5. DOCUMENT        The test IS the documentation of the bug; name it after the issue
```

```python
# Bug report: order total is wrong when applying multiple coupons
# Issue #4471

def test_multiple_coupons_do_not_compound_incorrectly():
    """Regression: issue #4471 — applying two coupons double-discounts the base price."""
    cart = Cart(items=[Item(sku="SKU-001", price=Decimal("200.00"))])
    coupon_10_pct = Coupon(code="SUMMER10", discount_pct=Decimal("10"))
    coupon_5_pct  = Coupon(code="VIP5",    discount_pct=Decimal("5"))

    total = cart.apply_coupons([coupon_10_pct, coupon_5_pct]).total()

    # Expected: 200 - 10% = 180; then 180 - 5% = 171
    # Bug was producing: 200 - 10% = 180; then 200 - 5% = 190; total = 170 (wrong base)
    assert total == Decimal("171.00")
```

---

## Outside-In TDD (London School) vs Inside-Out TDD (Detroit/Chicago School)

### London School (Outside-In / Mockist)

Start from the outermost layer (acceptance test or controller) and work inward, mocking collaborators that don't exist yet.

```
Acceptance Test (failing)
        |
        v
Controller Test --> mock Service
        |
        v
Service Test --> mock Repository
        |
        v
Repository Test --> real database (integration)
```

**Advantages:**
- Drives the design top-down from the user's perspective
- Forces explicit contracts between layers early
- Avoids building code that isn't needed

**Disadvantages:**
- Heavy use of mocks; tests coupled to implementation details
- Refactoring internal structure requires updating many mocks
- Can produce hollow tests that verify calls but not behavior

**When to use:** When building a new feature in an existing system; when the integration points are well-understood; when you practice BDD at the acceptance layer.

### Detroit School (Inside-Out / Classicist)

Start from the innermost domain logic and build outward. Use real objects; avoid mocks.

```
Domain Model Tests (real objects)
        |
        v
Application Service Tests (real domain, fake repository)
        |
        v
Integration Test (real everything)
```

**Advantages:**
- Tests verify real behavior, not call sequences
- Refactoring internals doesn't break tests
- Domain model emerges from concrete examples

**Disadvantages:**
- Can build the wrong abstraction if domain isn't well understood
- Tests may become slow if real collaborators are expensive
- Bottom-up design can miss how pieces connect until late

**When to use:** When doing DDD and the domain model is uncertain; when you want tests that survive refactoring; for greenfield domain code.

**Pragmatic answer:** Use Inside-Out for domain logic and Outside-In for application/adapter layers. Start with a failing integration test to define the walking skeleton, then switch to unit tests inside.

---

## Property-Based Testing

Instead of example-based tests (`input X → output Y`), generate hundreds of random inputs and verify *properties* that must always hold.

```python
# Example-based: tests one specific case
def test_sort_specific():
    assert sorted([3, 1, 2]) == [1, 2, 3]

# Property-based: tests invariants across many random inputs
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_properties(lst):
    result = sorted(lst)
    # Property 1: length is preserved
    assert len(result) == len(lst)
    # Property 2: result is ordered
    assert all(result[i] <= result[i+1] for i in range(len(result) - 1))
    # Property 3: same elements (no additions or removals)
    assert sorted(result) == sorted(lst)
```

```kotlin
// Kotlin with Kotest property testing
class MoneyProperties : StringSpec({
    "adding money is commutative" {
        checkAll(Arb.int(1..10_000), Arb.int(1..10_000)) { a, b ->
            val moneyA = Money.of(a, "BRL")
            val moneyB = Money.of(b, "BRL")
            moneyA + moneyB shouldBe moneyB + moneyA
        }
    }

    "discount never produces negative total" {
        checkAll(Arb.int(1..100_000), Arb.int(0..100)) { amount, pct ->
            val price = Money.of(amount, "BRL")
            val discount = Discount.percentage(pct)
            price.applyDiscount(discount).amount shouldBeGreaterThanOrEqualTo BigDecimal.ZERO
        }
    }
})
```

**Good properties to test:**
- Round-trip: `decode(encode(x)) == x`
- Idempotency: `f(f(x)) == f(x)`
- Commutativity: `f(a, b) == f(b, a)`
- Invariants that must always hold regardless of input
- Boundary conditions: empty, zero, max values

---

## Mutation Testing

Mutation testing answers: "Do my tests actually detect bugs?"

A mutation testing tool makes small changes (mutations) to your production code and re-runs your tests. If a mutation survives (tests still pass), your tests are not verifying that behavior.

```
Original code:         if (discount > 0.5) apply(discount);
Mutation 1 (survived): if (discount >= 0.5) apply(discount);   <-- tests don't catch boundary
Mutation 2 (killed):   if (discount > 0.5) skip(discount);     <-- tests catch this
```

**Common mutations:**
- Conditional boundary: `>` becomes `>=`
- Negate conditional: `if (a)` becomes `if (!a)`
- Return value: `return true` becomes `return false`
- Math operator: `+` becomes `-`
- Void method call removal: remove the call entirely

**Tools:**
- Java: PIT (pitest.org) — `mvn test-compile org.pitest:pitest-maven:mutationCoverage`
- Python: mutmut — `mutmut run`
- Rust: cargo-mutants

**Interpreting results:**
- Mutation score = killed / total. Target 80%+ for critical domain logic
- Surviving mutations on trivial code (getters) are acceptable; surviving mutations on business rules are not
- Do not chase 100% — some mutations are semantically equivalent

---

## Test Smells

### Fragile Test (Change Detector)
A test that breaks on every refactoring without any behavioral change.

*Symptom:* Tests mock internal implementation details; testing private methods.

*Fix:* Test observable behavior, not implementation steps. Reduce mocks.

### Mystery Guest
Test depends on external state (file, database, global variable) set up somewhere else. When it fails, you have no idea why.

*Fix:* Make all setup explicit in the test or `@BeforeEach`. Tests must be self-contained.

### Eager Test
One test verifies many unrelated behaviors, has many assertions that test different things.

*Fix:* One assertion per test (or one *concern* per test). Split into focused tests.

### Chatty Test
Tests that print/log to console. Indicates the test is debugging, not verifying.

*Fix:* Remove all console output. Capture output and assert on it if needed.

### Slow Test
Unit test takes more than 100ms. Often caused by hitting real I/O in a "unit" test.

*Fix:* Introduce test doubles for I/O. Reserve real I/O for integration tests.

### Assertion Roulette
Multiple assertions with no messages; when one fails, unclear which one or why.

```java
// BAD
assertThat(result.status()).isEqualTo(CONFIRMED);
assertThat(result.total()).isEqualTo(Money.of(100, "BRL"));
assertThat(result.items()).hasSize(2);

// BETTER — use SoftAssertions or named assertions
assertSoftly(softly -> {
    softly.assertThat(result.status()).as("order status").isEqualTo(CONFIRMED);
    softly.assertThat(result.total()).as("order total").isEqualTo(Money.of(100, "BRL"));
    softly.assertThat(result.items()).as("item count").hasSize(2);
});
```

### Conditional Test Logic
Loops and if-statements in test code.

*Fix:* Use parameterized tests. If you need a condition, write two separate tests.

---

## Language Examples

### Java — JUnit 5 + Mockito + AssertJ

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentGateway paymentGateway;
    @Mock OrderRepository orderRepository;
    @InjectMocks OrderService orderService;

    @Test
    void should_confirm_order_when_payment_succeeds() {
        // Arrange
        var cart = Cart.withItem("SKU-1", Money.of(100, "BRL"));
        var card = CreditCard.of("4111111111111111", "12/26", "123");
        given(paymentGateway.charge(any()))
            .willReturn(PaymentResult.success("TXN-42"));

        // Act
        Order order = orderService.place(cart, card);

        // Assert
        assertThat(order.status()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(order.paymentId()).isEqualTo("TXN-42");
        then(orderRepository).should().save(order);
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "\t"})
    void should_reject_blank_customer_email(String email) {
        assertThatThrownBy(() -> Customer.of(email))
            .isInstanceOf(InvalidCustomerException.class)
            .hasMessageContaining("email");
    }
}
```

### Kotlin — Kotest

```kotlin
class DiscountPricerSpec : DescribeSpec({
    describe("DiscountPricer") {
        val pricer = DiscountPricer()

        context("when customer is premium") {
            val customer = Customer.premium("alice@example.com")

            it("applies 15 percent discount") {
                val cart = Cart(listOf(Item("SKU-1", Money(100, "BRL"))))
                pricer.calculate(cart, customer) shouldBe Money(85, "BRL")
            }

            it("does not apply discount to already-discounted items") {
                val cart = Cart(listOf(Item("SKU-SALE", Money(50, "BRL"), alreadyDiscounted = true)))
                pricer.calculate(cart, customer) shouldBe Money(50, "BRL")
            }
        }

        context("when customer is standard") {
            val customer = Customer.standard("bob@example.com")

            it("applies no discount") {
                val cart = Cart(listOf(Item("SKU-1", Money(100, "BRL"))))
                pricer.calculate(cart, customer) shouldBe Money(100, "BRL")
            }
        }
    }
})
```

### Python — pytest

```python
import pytest
from decimal import Decimal
from unittest.mock import MagicMock, call

class TestOrderService:
    def test_confirms_order_when_payment_succeeds(self):
        # Arrange
        gateway = MagicMock()
        gateway.charge.return_value = PaymentResult(success=True, transaction_id="TXN-42")
        repo = InMemoryOrderRepository()
        service = OrderService(gateway=gateway, repository=repo)
        cart = Cart(items=[Item(sku="SKU-1", price=Decimal("100.00"))])

        # Act
        order = service.place(cart, credit_card=VALID_CARD)

        # Assert
        assert order.status == OrderStatus.CONFIRMED
        assert order.payment_id == "TXN-42"
        assert repo.find_by_id(order.id) == order

    def test_raises_when_payment_fails(self):
        gateway = MagicMock()
        gateway.charge.side_effect = PaymentDeclinedError("Insufficient funds")
        service = OrderService(gateway=gateway, repository=InMemoryOrderRepository())

        with pytest.raises(PaymentDeclinedError, match="Insufficient funds"):
            service.place(Cart.empty(), VALID_CARD)

    @pytest.mark.parametrize("amount,pct,expected", [
        (Decimal("100"), 10, Decimal("90")),
        (Decimal("200"), 50, Decimal("100")),
        (Decimal("150"), 0,  Decimal("150")),
    ])
    def test_discount_calculation(self, amount, pct, expected):
        assert Discount(pct).apply_to(amount) == expected
```

### Rust — Built-in test framework

```rust
// src/domain/discount.rs

pub struct Discount {
    percentage: u8,
}

impl Discount {
    pub fn new(percentage: u8) -> Result<Self, DomainError> {
        if percentage > 100 {
            return Err(DomainError::InvalidDiscount(percentage));
        }
        Ok(Self { percentage })
    }

    pub fn apply(&self, amount: u64) -> u64 {
        amount - (amount * self.percentage as u64 / 100)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn applies_percentage_discount() {
        let discount = Discount::new(15).unwrap();
        assert_eq!(discount.apply(200), 170);
    }

    #[test]
    fn zero_percent_leaves_amount_unchanged() {
        let discount = Discount::new(0).unwrap();
        assert_eq!(discount.apply(500), 500);
    }

    #[test]
    fn rejects_discount_above_100_percent() {
        let result = Discount::new(101);
        assert!(matches!(result, Err(DomainError::InvalidDiscount(101))));
    }

    // Property-based with proptest crate
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn discounted_amount_never_exceeds_original(
            amount in 0u64..1_000_000,
            pct in 0u8..=100
        ) {
            let discount = Discount::new(pct).unwrap();
            prop_assert!(discount.apply(amount) <= amount);
        }
    }
}
```

---

## Practical Checklist

### Before writing any code
- [ ] Can I write a failing test that describes the desired behavior?
- [ ] Is the test name a sentence describing what the system does?
- [ ] Does the test fail for the right reason (assertion, not setup)?

### During Green phase
- [ ] Did I write only enough code to pass the failing test?
- [ ] Am I resisting the urge to generalize before the next test forces it?

### During Refactor phase
- [ ] Do all tests still pass?
- [ ] Does the code communicate its intent without comments?
- [ ] Have I removed duplication in both production and test code?

### Test quality
- [ ] Is each test independent (can run in any order)?
- [ ] Is the test self-contained (all setup visible in the test)?
- [ ] Does it test behavior, not implementation?
- [ ] Does it have a clear Arrange / Act / Assert structure?
- [ ] Does it run fast enough to be in the unit test suite?

### Coverage
- [ ] Happy path covered?
- [ ] All error paths covered?
- [ ] Boundary values tested?
- [ ] Have I run mutation tests on critical logic?

---

## When TDD Adds Most Value

- Domain logic with complex rules and many branches
- Bug fixes — the Prove-It Pattern is non-negotiable
- Refactoring existing code — tests are the safety net
- API/library design — tests expose usability problems early

## When TDD is Harder (but still worth it)

- UI code — use it for the logic behind the UI, not pixel positions
- Exploratory/spike code — write tests after discovery; delete the spike
- Highly stateful systems — use fakes to manage state complexity

## When to Skip TDD

- Throwaway scripts with a 1-day lifespan
- Configuration files
- Generated code (protobuf, jOOQ, OpenAPI clients)
