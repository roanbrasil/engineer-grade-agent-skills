---
name: refactoring-patterns
description: Expert refactoring guidance — Martin Fowler catalog with before/after examples in Java/Kotlin and Python, large-scale migration patterns, and safe refactoring discipline.
---

# Refactoring Patterns — Expert Reference

## The Refactoring Contract

> **Refactoring**: a behavior-preserving transformation of code structure.

If behavior changes, it is not a refactoring — it is a bug fix or a feature. The contract has two parts:

1. **Tests pass before you start.** If they don't, write tests first. Refactoring without tests is editing in the dark.
2. **Tests pass after every step.** Refactor in micro-steps. Each step must leave the system green.

```
Red → (fix bug) → Green → (refactor) → Green → (next feature) → Red → ...
```

### When to Refactor

- **Boy Scout Rule:** Leave the code cleaner than you found it. Small improvements every touch.
- **Before adding a feature:** Clean the host class first. Easier to add to clean code.
- **When fixing a bug:** The bug lived in bad structure. Fix the structure, not just the symptom.
- **During code review:** The reviewer sees it fresh. Best time to identify naming, cohesion issues.

### When NOT to Refactor

- **Tests don't exist.** Write characterization tests first, then refactor.
- **Under deadline pressure for the same feature.** Separate tickets: one to refactor, one to ship. Mixing them means neither is done carefully.
- **Rewriting is cheaper.** If the module has no tests, has been rewritten 3 times, and 80% of bugs come from it — rewrite. Refactoring a fundamentally broken design is expensive.

---

## Extract Patterns

### Extract Method / Function

**When:** A block of code can be named. The name is shorter to read than the block. Reduces need for inline comments.

```java
// BEFORE — Java
public void printBill(Order order) {
    System.out.println("Order: " + order.getId());
    // Calculate total with discount
    double total = 0;
    for (Item item : order.getItems()) {
        total += item.getPrice();
    }
    if (order.isLoyalCustomer()) {
        total *= 0.9;
    }
    System.out.println("Total: $" + total);
}

// AFTER — Java
public void printBill(Order order) {
    System.out.println("Order: " + order.getId());
    System.out.println("Total: $" + calculateTotal(order));
}

private double calculateTotal(Order order) {
    double subtotal = order.getItems().stream()
        .mapToDouble(Item::getPrice)
        .sum();
    return order.isLoyalCustomer() ? subtotal * 0.9 : subtotal;
}
```

```python
# BEFORE — Python
def print_bill(order):
    print(f"Order: {order.id}")
    # Calculate total with discount
    total = sum(item.price for item in order.items)
    if order.is_loyal_customer:
        total *= 0.9
    print(f"Total: ${total:.2f}")

# AFTER — Python
def print_bill(order):
    print(f"Order: {order.id}")
    print(f"Total: ${calculate_total(order):.2f}")

def calculate_total(order: Order) -> float:
    subtotal = sum(item.price for item in order.items)
    return subtotal * 0.9 if order.is_loyal_customer else subtotal
```

**Signal to extract:** You are writing a comment to explain a block. Name the method instead.

---

### Extract Class

**When:** A class has too many responsibilities. Identify a cohesive cluster of fields and methods that belong together.

```java
// BEFORE — Java: Person class knows too much about phone formatting
public class Person {
    private String name;
    private String officeAreaCode;
    private String officeNumber;

    public String getOfficePhone() {
        return "(" + officeAreaCode + ") " + officeNumber;
    }
}

// AFTER — Java: TelephoneNumber is a first-class concept
public class TelephoneNumber {
    private final String areaCode;
    private final String number;

    public TelephoneNumber(String areaCode, String number) {
        this.areaCode = areaCode;
        this.number = number;
    }

    public String format() {
        return "(" + areaCode + ") " + number;
    }
}

public class Person {
    private String name;
    private TelephoneNumber officePhone;

    public String getOfficePhone() {
        return officePhone.format();
    }
}
```

```python
# BEFORE — Python
class Person:
    def __init__(self, name, office_area_code, office_number):
        self.name = name
        self.office_area_code = office_area_code
        self.office_number = office_number

    def office_phone(self):
        return f"({self.office_area_code}) {self.office_number}"

# AFTER — Python
from dataclasses import dataclass

@dataclass(frozen=True)
class TelephoneNumber:
    area_code: str
    number: str

    def format(self) -> str:
        return f"({self.area_code}) {self.number}"

@dataclass
class Person:
    name: str
    office_phone: TelephoneNumber
```

---

### Extract Variable

**When:** A complex expression is hard to read. The variable name is the documentation.

```kotlin
// BEFORE — Kotlin
if (order.items.sumOf { it.price } > 100.0 && customer.memberSince < LocalDate.now().minusYears(2)) {
    applyPremiumDiscount(order)
}

// AFTER — Kotlin
val orderTotal = order.items.sumOf { it.price }
val isHighValueOrder = orderTotal > 100.0
val isLoyalCustomer = customer.memberSince < LocalDate.now().minusYears(2)

if (isHighValueOrder && isLoyalCustomer) {
    applyPremiumDiscount(order)
}
```

---

### Extract Interface / Protocol

**When:** Multiple classes share behavior. Callers should depend on the abstraction, not the implementation.

```java
// BEFORE — Java: callers are coupled to concrete EmailNotifier
public class OrderService {
    private final EmailNotifier emailNotifier = new EmailNotifier();

    public void placeOrder(Order order) {
        processOrder(order);
        emailNotifier.send(order.getCustomerEmail(), "Order confirmed");
    }
}

// AFTER — Java: callers depend on Notifier interface
public interface Notifier {
    void notify(String recipient, String message);
}

public class EmailNotifier implements Notifier { ... }
public class SmsNotifier implements Notifier { ... }
public class NoOpNotifier implements Notifier { ... }  // for tests

public class OrderService {
    private final Notifier notifier;

    public OrderService(Notifier notifier) {  // injected
        this.notifier = notifier;
    }
}
```

```python
# Python — Protocol (structural subtyping, no inheritance needed)
from typing import Protocol

class Notifier(Protocol):
    def notify(self, recipient: str, message: str) -> None: ...

class EmailNotifier:
    def notify(self, recipient: str, message: str) -> None:
        send_email(recipient, message)

class OrderService:
    def __init__(self, notifier: Notifier) -> None:
        self.notifier = notifier
```

---

## Inline Patterns

### Inline Method

**When:** The method body is as clear as the method name. The indirection costs more than it gives.

```java
// BEFORE — Java: one-liner method adds no clarity
private boolean isOverdue(Task task) {
    return task.getDueDate().isBefore(LocalDate.now());
}

// Used only once — inline it
if (isOverdue(task)) { ... }

// AFTER — Java
if (task.getDueDate().isBefore(LocalDate.now())) { ... }
```

```python
# BEFORE
def is_overdue(task: Task) -> bool:
    return task.due_date < date.today()

# AFTER (when called in exactly one place and name adds nothing)
if task.due_date < date.today():
    ...
```

### Inline Variable

**When:** The variable name doesn't add meaning over the expression.

```kotlin
// BEFORE — Kotlin
val result = calculateDiscount(order)
return result

// AFTER
return calculateDiscount(order)
```

---

## Move Patterns

### Move Method / Function

**When:** A method uses more data from another class than from its own. Move it to the class it depends on most.

```java
// BEFORE — Account.overdraftCharge() uses BankAccount's fields
class Account {
    private BankAccount type;
    private int daysOverdrawn;

    double overdraftCharge() {
        if (type.isPremium()) {
            double result = 10;
            if (daysOverdrawn > 7) result += (daysOverdrawn - 7) * 0.85;
            return result;
        }
        return daysOverdrawn * 1.75;
    }
}

// AFTER — logic moves to BankAccount where it belongs
class BankAccount {
    boolean isPremium() { ... }

    double overdraftCharge(int daysOverdrawn) {
        if (isPremium()) {
            double result = 10;
            if (daysOverdrawn > 7) result += (daysOverdrawn - 7) * 0.85;
            return result;
        }
        return daysOverdrawn * 1.75;
    }
}

class Account {
    private BankAccount type;
    private int daysOverdrawn;

    double overdraftCharge() {
        return type.overdraftCharge(daysOverdrawn);
    }
}
```

---

## Rename Patterns

### Rename Variable / Method / Class

The single most impactful refactoring. A wrong name is a lie that every reader must mentally decode.

```python
# BEFORE — what is d? what is res?
def calc(d: list) -> float:
    res = 0
    for x in d:
        res += x.p * x.q
    return res

# AFTER
def calculate_invoice_total(line_items: list[LineItem]) -> float:
    total = 0.0
    for item in line_items:
        total += item.unit_price * item.quantity
    return total
```

**Rename process (safe):**
1. If the IDE supports it, use "Rename Symbol" — it finds all references.
2. For public API methods: add the new name as an alias first, deprecate the old name, migrate callers, remove old name (Expand-Contract).
3. Run tests after rename. A failing test often reveals a missed reference.

---

## Structural Patterns

### Replace Conditional with Polymorphism

**When:** Long `switch`/`if-else` block dispatches on a type field. Adding a new type requires modifying the switch.

```java
// BEFORE — Java: switch on type string
public double calculatePay(Employee employee) {
    switch (employee.getType()) {
        case "ENGINEER": return calculateEngineerPay(employee);
        case "MANAGER": return calculateManagerPay(employee);
        case "SALESPERSON": return calculateSalespersonPay(employee);
        default: throw new IllegalArgumentException("Unknown type");
    }
}

// AFTER — Java: polymorphism replaces the switch
public abstract class Employee {
    public abstract double calculatePay();
}

public class Engineer extends Employee {
    @Override public double calculatePay() { ... }
}

public class Manager extends Employee {
    @Override public double calculatePay() { ... }
}

// Caller
employee.calculatePay();  // dispatch is gone
```

```python
# BEFORE — Python
def calculate_pay(employee):
    if employee.type == "engineer":
        return calculate_engineer_pay(employee)
    elif employee.type == "manager":
        return calculate_manager_pay(employee)
    raise ValueError(f"Unknown type: {employee.type}")

# AFTER — Python
from abc import ABC, abstractmethod

class Employee(ABC):
    @abstractmethod
    def calculate_pay(self) -> float: ...

class Engineer(Employee):
    def calculate_pay(self) -> float: ...

class Manager(Employee):
    def calculate_pay(self) -> float: ...

# Caller
employee.calculate_pay()
```

---

### Replace Primitive with Value Object

**When:** A primitive (string, float, int) represents a domain concept. Domain rules belong in the type, not scattered in callers.

```java
// BEFORE — Java: money as primitive float
public class Order {
    private double amount;       // what currency? negative allowed?
    private String currency;     // separate field, must stay in sync
}

// AFTER — Java: Money encapsulates its invariants
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new CurrencyMismatchException();
        return new Money(amount.add(other.amount), currency);
    }
}
```

```python
# BEFORE — Python: email as str, validation scattered everywhere
def register_user(email: str) -> None:
    if "@" not in email:  # repeated everywhere
        raise ValueError("Invalid email")
    ...

# AFTER — Python: Email validates once at construction
from dataclasses import dataclass
import re

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if not re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", self.value):
            raise ValueError(f"Invalid email: {self.value}")

    def __str__(self) -> str:
        return self.value
```

**Common candidates for value objects:** Money, Email, PhoneNumber, Address, Percentage, DateRange, Coordinates, URL.

---

### Introduce Parameter Object

**When:** Multiple related parameters always travel together. Group them.

```kotlin
// BEFORE — Kotlin
fun generateReport(
    startDate: LocalDate,
    endDate: LocalDate,
    region: String,
    currency: Currency
): Report { ... }

// AFTER — Kotlin
data class ReportCriteria(
    val dateRange: DateRange,
    val region: String,
    val currency: Currency
)

fun generateReport(criteria: ReportCriteria): Report { ... }
```

---

### Separate Query from Modifier (CQS)

**When:** A method both returns a value AND changes state. Callers can't call it for observation without causing mutation.

```java
// BEFORE — Java: getTotalPrice() has a side effect
public class Cart {
    public double getTotalPrice() {
        double total = items.stream().mapToDouble(Item::getPrice).sum();
        lastCalculatedTotal = total;  // SIDE EFFECT
        auditLog.record("price calculated: " + total);  // SIDE EFFECT
        return total;
    }
}

// AFTER — Java: query and modifier are separate
public class Cart {
    public double getTotalPrice() {  // pure query, no side effects
        return items.stream().mapToDouble(Item::getPrice).sum();
    }

    public void recordPriceCalculation() {  // modifier, no return value
        double total = getTotalPrice();
        lastCalculatedTotal = total;
        auditLog.record("price calculated: " + total);
    }
}
```

---

## Large-Scale Patterns

### Strangler Fig

Incrementally replace a legacy system. New code grows around the old; old code is strangled as it's replaced.

```
Phase 1: Route new traffic to new implementation
┌─────────┐     ┌──────────┐     ┌─────────────────┐
│ Clients │────►│  Router  │────►│  Legacy System  │
└─────────┘     └──────────┘     └─────────────────┘

Phase 2: New implementation handles specific paths
┌─────────┐     ┌──────────┐     ┌─────────────────┐
│ Clients │────►│  Router  │────►│  Legacy System  │
└─────────┘     └──────────┘     └─────────────────┘
                     │
                     └──────────►┌─────────────────┐
                      (new paths)│  New System     │
                                 └─────────────────┘

Phase 3: Legacy is strangled, decommissioned
┌─────────┐     ┌──────────┐     ┌─────────────────┐
│ Clients │────►│  Router  │────►│  New System     │
└─────────┘     └──────────┘     └─────────────────┘
                                 (legacy removed)
```

**Implementation:**
```python
# Router directs by feature flag or path
def handle_request(request):
    if new_system_handles(request.path):
        return new_system.handle(request)
    return legacy_system.handle(request)
```

---

### Branch by Abstraction

Introduce an abstraction layer, make old code implement it, build new implementation behind it, switch over, remove old.

```
Step 1: Introduce abstraction
  PaymentGateway (interface)
        │
        └──► StripeV1 (current implementation)

Step 2: Build new implementation
  PaymentGateway (interface)
        │
        ├──► StripeV1 (old)
        └──► StripeV2 (new, under test)

Step 3: Switch the wiring
  PaymentGateway (interface)
        │
        └──► StripeV2 (active)
             StripeV1 (dormant — can toggle back)

Step 4: Remove old implementation
  PaymentGateway (interface)
        │
        └──► StripeV2
```

```java
// Step 1 — define abstraction
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
}

// Step 2 — new implementation alongside old
public class StripeV2Gateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Money amount, PaymentMethod method) {
        // new implementation
    }
}

// Step 3 — switch via dependency injection / configuration
@Bean
public PaymentGateway paymentGateway(StripeV2Gateway gateway) {
    return gateway;  // previously returned StripeV1Gateway
}
```

---

### Parallel Change (Expand-Contract)

Safe approach for changing a method signature or API without breaking callers.

```
Phase 1 — Expand: add new signature alongside old
public class UserService {
    /** @deprecated use findByEmail(Email) */
    public User findUser(String email) { ... }

    public User findByEmail(Email email) { ... }  // NEW
}

Phase 2 — Migrate: update all callers to new signature
// Replace all findUser("x@y.com") with findByEmail(new Email("x@y.com"))

Phase 3 — Contract: remove old signature
public class UserService {
    public User findByEmail(Email email) { ... }
}
```

```python
# Python parallel change
class UserService:
    def find_user(self, email: str) -> User:
        """Deprecated: use find_by_email(Email) instead."""
        import warnings
        warnings.warn("find_user is deprecated", DeprecationWarning, stacklevel=2)
        return self.find_by_email(Email(email))

    def find_by_email(self, email: Email) -> User:  # new canonical method
        ...
```

---

## Refactoring Checklist

### Before Starting
```
[ ] All existing tests pass
[ ] You understand what the code does (read it, don't guess)
[ ] Scope is defined: what will you refactor, what is out of scope
[ ] Branch created; you can abort safely
```

### During Refactoring
```
[ ] One refactoring at a time — don't mix Extract + Rename + Move
[ ] Run tests after each micro-step
[ ] Commit frequently with descriptive messages: "refactor: extract calculateTotal method"
[ ] No behavior changes (not even "just this small fix")
```

### After Refactoring
```
[ ] All tests pass
[ ] Code coverage has not decreased
[ ] PR description explains the structural improvement, not just "cleanup"
[ ] No TODO comments left behind without tickets
[ ] Public API is unchanged (or Expand-Contract applied)
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Refactor + Feature in same PR | Reviewer can't tell what's behavior change vs structural | Separate PRs |
| Refactoring without tests | You don't know if you broke something | Write characterization tests first |
| "Big Bang" refactor | One giant PR, impossible to review or revert | Incremental, ship each step |
| Rename to "better" naming style (team disagreement) | Style debate, not clarity improvement | Agree on conventions first |
| Pulling in scope creep | "While I'm here..." grows to 3000-line PR | Commit the todo, open a ticket |
| Changing public API as part of refactor | Breaks callers | Expand-Contract; keep old signature |
| Unused variable/method removal mid-refactor | Hard to distinguish from intentional removal | Separate cleanup commit |
