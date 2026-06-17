---
name: clean-code
description: Apply Clean Code principles when writing, reviewing, or refactoring code — invoked when the user asks to clean up code, improve readability, review naming, reduce complexity, or eliminate code smells.
---

# Clean Code

Clean Code is not about personal style. It is about professional discipline: writing code that communicates intent so clearly that the next reader — including yourself six months from now — can understand, modify, and extend it without fear.

This skill encodes the principles from Robert C. Martin's *Clean Code*, Refactoring by Martin Fowler, and decades of collective wisdom. Apply it ruthlessly but pragmatically.

---

## Core Philosophy

> "Clean code reads like well-written prose." — Grady Booch
> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." — Martin Fowler

Code is read far more often than it is written. The primary audience of code is humans, not compilers.

```
Write once, read many times:

  Writing time:  |====|
  Reading time:  |============================|

Optimize for reading, not writing.
```

---

## 1. Meaningful Names

Names are the single most visible quality signal in a codebase. Every name — variable, function, class, module, package — is an opportunity to communicate intent.

### Rules in Priority Order

**1. Reveal Intent**

The name must answer: what is this, why does it exist, how is it used?

```java
// BAD — what is d? days? distance? data?
int d = 7;
List<int[]> theList = new ArrayList<>();

// GOOD
int expirationDaysFromNow = 7;
List<Cell> flaggedCells = new ArrayList<>();
```

```python
# BAD
def calc(a, b, c):
    return (a * b) / c

# GOOD
def calculate_monthly_payment(principal, annual_rate, months):
    monthly_rate = annual_rate / 12
    return (principal * monthly_rate) / (1 - (1 + monthly_rate) ** -months)
```

**2. Avoid Disinformation**

Do not use names that carry false or misleading connotations.

```kotlin
// BAD — accountList is not a List, it's a Set; this is disinformation
val accountList = HashSet<Account>()

// GOOD
val accounts = HashSet<Account>()

// BAD — l and O look like 1 and 0
val l = 1
val O = 0

// BAD — redundant type encoding (Hungarian notation)
val strCustomerName: String = "Alice"
val arrItems: Array<Item> = arrayOf()

// GOOD
val customerName = "Alice"
val items: Array<Item> = arrayOf()
```

**3. Make Meaningful Distinctions**

Do not add noise words that add no information: `Info`, `Data`, `Manager`, `Handler`, `Helper`, `Util`, `Object`.

```rust
// BAD — what is the difference between these?
struct ProductInfo { ... }
struct ProductData { ... }
struct Product { ... }      // use this

// BAD
fn get_active_account() { ... }
fn get_active_account_info() { ... }    // noise

// BAD — number-series naming
fn process_item_a(item: &Item) { ... }
fn process_item_b(item: &Item) { ... }

// GOOD
fn validate_item(item: &Item) { ... }
fn persist_item(item: &Item) { ... }
```

**4. Use Pronounceable Names**

If you cannot pronounce it, you cannot discuss it.

```cpp
// BAD
struct DtaRcrd102 {
    int genymdhms;   // generation year/month/day/hour/minute/second
    int modymdhms;
    std::string pszqint;
};

// GOOD
struct Customer {
    std::chrono::time_point<std::chrono::system_clock> generationTimestamp;
    std::chrono::time_point<std::chrono::system_clock> modificationTimestamp;
    std::string recordId;
};
```

**5. Use Searchable Names**

Single-letter names and numeric constants cannot be grepped. The length of a name should correspond to the size of its scope.

```java
// BAD — magic numbers scattered everywhere
for (int j = 0; j < 34; j++) {
    s += t[j] * 4 / 5;
}

// GOOD
final int WORK_DAYS_PER_WEEK = 5;
final int NUMBER_OF_TASKS = 34;
final int DAILY_HOURS = 4;

int taskTotal = 0;
for (int taskIndex = 0; taskIndex < NUMBER_OF_TASKS; taskIndex++) {
    taskTotal += taskEstimates[taskIndex] * DAILY_HOURS / WORK_DAYS_PER_WEEK;
}
```

Exception: loop counters `i`, `j`, `k` are acceptable when the scope is tiny (3-5 lines). Lambda parameters `x`, `e`, `n` are acceptable when the type is obvious from context.

**6. Class Names: Nouns; Method Names: Verbs**

```kotlin
// Classes: noun or noun phrase
class Customer { }
class WikiPage { }
class AddressParser { }

// Methods: verb or verb phrase
fun deletePage() { }
fun save() { }
fun isPosted(): Boolean { }   // accessors: get/is/has
fun setName(name: String) { } // mutators: set

// Constructors: use static factory methods with descriptive names
val fulcrumPoint = Complex.fromRealNumber(23.0)  // GOOD
val fulcrumPoint = Complex(23.0)                  // less clear
```

**7. Pick One Word Per Concept**

Choose `fetch`, `retrieve`, or `get` — not all three. Consistency is a form of documentation.

```python
# BAD — three words for the same concept
class UserRepository:
    def fetch_user(self, id): ...

class OrderRepository:
    def get_order(self, id): ...

class ProductRepository:
    def retrieve_product(self, id): ...

# GOOD — one convention across the codebase
class UserRepository:
    def find_by_id(self, id): ...

class OrderRepository:
    def find_by_id(self, id): ...

class ProductRepository:
    def find_by_id(self, id): ...
```

---

## 2. Functions

### The Prime Directive: Do One Thing

A function that does one thing cannot be meaningfully decomposed. A function that does three things is three functions.

```
Signal of multiple responsibilities:
  - "and" in the function name: saveAndValidate()
  - Multiple sections separated by blank lines
  - Multiple levels of abstraction in the body
  - Side effects hiding behind an innocent name
```

### Small Functions

Functions should be small. Very small. The ideal: 5–15 lines. Rarely over 20.

```java
// BAD — one giant function doing everything
public void processOrder(Order order) {
    // validate
    if (order.getCustomer() == null) throw new InvalidOrderException("...");
    if (order.getItems().isEmpty()) throw new InvalidOrderException("...");
    // calculate total
    double total = 0;
    for (Item item : order.getItems()) {
        total += item.getPrice() * item.getQuantity();
    }
    if (order.hasCoupon()) {
        total = total * (1 - order.getCoupon().getDiscountRate());
    }
    // persist
    orderRepository.save(order);
    // notify
    emailService.send(order.getCustomer().getEmail(), "Your order total: " + total);
}

// GOOD — composed small functions, each at one level of abstraction
public void processOrder(Order order) {
    validateOrder(order);
    double total = calculateOrderTotal(order);
    persistOrder(order);
    notifyCustomer(order, total);
}

private void validateOrder(Order order) {
    requireNonNull(order.getCustomer(), "Order must have a customer");
    require(!order.getItems().isEmpty(), "Order must have items");
}

private double calculateOrderTotal(Order order) {
    double subtotal = sumItemPrices(order.getItems());
    return applyCouponIfPresent(subtotal, order.getCoupon());
}
```

### One Level of Abstraction Per Function

Mix high-level and low-level in the same function and readers must context-switch constantly.

```python
# BAD — mixes high-level (render page) with low-level (string ops)
def render_page_with_setups_and_teardowns(page_data, is_suite):
    wiki_text = page_data.get_content()
    if is_test_page(page_data):
        # low-level string manipulation mixed with high-level concept
        suite_setup = find_inherited_page(page_data.wiki_page, "SuiteSetUp")
        if suite_setup:
            setup_path = construct_inherited_page_path(page_data.wiki_page, suite_setup)
            wiki_text = "!include -setup ." + setup_path + "\n" + wiki_text
    # ... more low-level string concatenation
    return wiki_text

# GOOD — each function at one level of abstraction
def render_page_with_setups_and_teardowns(page_data, is_suite):
    if is_test_page(page_data):
        include_setups(page_data, is_suite)
    page_data.render()
    if is_test_page(page_data):
        include_teardowns(page_data, is_suite)
    return page_data.get_html()
```

### Function Arguments

The ideal number of arguments: zero (niladic). Then one (monadic). Then two (dyadic). Three (triadic) requires very strong justification. More than three: wrap them in a parameter object.

```kotlin
// BAD — too many arguments (triad and beyond)
fun makeCircle(x: Double, y: Double, radius: Double): Circle
fun createUser(name: String, email: String, age: Int, role: String, active: Boolean): User

// GOOD — parameter objects reveal the concept
data class Point(val x: Double, val y: Double)
fun makeCircle(center: Point, radius: Double): Circle

data class UserRegistration(
    val name: String,
    val email: String,
    val age: Int,
    val role: Role,
    val active: Boolean = true
)
fun createUser(registration: UserRegistration): User
```

**Flag arguments are a design smell.** A boolean flag means the function does two things.

```java
// BAD
render(true);   // what does true mean?

// ALSO BAD — even named, it signals two functions
render(isSuite: true);

// GOOD — two functions, each with one clear purpose
renderForSuite();
renderForSingleTest();
```

### Command-Query Separation (CQS)

A function either does something (command) or answers something (query). Never both.

```rust
// BAD — sets a value AND returns a boolean; violates CQS
fn set_attribute(name: &str, value: &str) -> bool {
    if self.attributes.contains_key(name) {
        self.attributes.insert(name.to_string(), value.to_string());
        return true;
    }
    false
}
// leads to confusing usage:
if set_attribute("username", "bob") {  // wait, did this set or check?

// GOOD — separate the command from the query
fn has_attribute(&self, name: &str) -> bool {
    self.attributes.contains_key(name)
}

fn set_attribute(&mut self, name: &str, value: &str) {
    self.attributes.insert(name.to_string(), value.to_string());
}

// Usage is now unambiguous:
if self.has_attribute("username") {
    self.set_attribute("username", "bob");
}
```

---

## 3. Comments

**The best comment is no comment.** If you feel the urge to comment, first ask: can I rename or restructure to eliminate the need?

### When Comments Are Acceptable

**Legal comments** (copyright, license): required, keep them short.

**Intent comments** (WHY, not WHAT): document the decision, not the mechanism.

```cpp
// Using a thread-local cache here intentionally: this object is
// accessed from multiple threads but must not be shared because
// it holds connection state. Synchronizing would create a bottleneck.
thread_local Cache cache;
```

**Warning of consequences:**

```java
// Don't run this in production — it takes 45 minutes
// and locks the user table for the duration.
@Test
public void _slow_testWithRealDatabase() { ... }
```

**TODO comments:** acceptable if tracked; never as a permanent excuse.

**Amplification:** draw attention to something that seems unimportant but isn't.

### When Comments Are Not Acceptable

```java
// BAD — mumbling: says nothing useful
// Add all the modules
if (smodule.getDependSubsystems() != null) { ... }

// BAD — redundant: the code says this already
// Returns the day of the month
public int getDayOfMonth() { return dayOfMonth; }

// BAD — misleading: comment contradicts the code
// always returns when this.closed is true
public boolean waitForClose(long timeoutMillis) throws Exception {
    if (!closed) {
        wait(timeoutMillis);
        if (!closed) throw new Exception("MockResponseSender not closed");
    }
    // Returns even when closed is still false after timeout!
}

// BAD — mandated: don't add Javadoc to every method just because policy says so
/**
 * @param title The title of the CD
 * @param author The author of the CD
 * @param tracks The number of tracks on the CD
 */
public void addCD(String title, String author, int tracks) { ... }

// BAD — journal comments (use git log)
// Changes (from 11-Oct-2021):
// ----------------
// 11-Oct-2021: added conditions for null check
// 15-Oct-2021: refactored for performance

// BAD — commented-out code (use git, delete it)
// doSomethingOld();
// doSomethingElseOld();
doSomethingNew();
```

---

## 4. Error Handling

### Exceptions, Not Return Codes

Return codes force callers to check them — and they won't. Exceptions cannot be silently ignored.

```java
// BAD — return code error handling
DeviceResponse response = getHandle(DEV1);
if (response != DeviceController.INVALID) {
    DeviceRecord record = retrieveRecord(response);
    if (record.getStatus() != DEVICE_SUSPENDED) {
        pauseDevice(record);
        clearDeviceWorkQueue(record);
        closeDevice(record);
    } else {
        logger.log("Device suspended. Unable to shut down.");
    }
} else {
    logger.log("Invalid handle for: " + DEV1.toString());
}

// GOOD — exceptions let you separate the happy path
try {
    tryToShutDown();
} catch (DeviceShutDownError e) {
    logger.log(e);
}

private void tryToShutDown() throws DeviceShutDownError {
    DeviceRecord record = getHandle(DEV1);
    pauseDevice(record);
    clearDeviceWorkQueue(record);
    closeDevice(record);
}
```

### Don't Return Null

Returning null forces every caller to null-check. One missed check = NullPointerException in production.

```kotlin
// BAD
fun findEmployee(id: String): Employee? {
    // returns null if not found
}

// Every caller must remember to check:
val employee = findEmployee(id)
if (employee != null) {  // forget this once and you crash
    employee.pay()
}

// GOOD — use Optional or a Result type
fun findEmployee(id: String): Optional<Employee>
fun findEmployee(id: String): Result<Employee, EmployeeNotFound>

// Or return a Null Object (for collections: always return empty, never null)
fun findActiveEmployees(): List<Employee> = emptyList()  // never null
```

```python
# BAD
def find_user(user_id: str) -> User | None:
    ...

# GOOD — explicit Optional contract
from typing import Optional

def find_user(user_id: str) -> Optional[User]:  # at least honest about it
    ...

# BETTER — raise a domain exception for not-found
def find_user(user_id: str) -> User:
    user = self.repository.get(user_id)
    if user is None:
        raise UserNotFoundError(f"No user with id={user_id}")
    return user
```

### Don't Pass Null

Passing null into a function is asking for trouble. If a parameter is truly optional, make that explicit with an overload or a Null Object.

```rust
// BAD — what happens when name is None?
fn create_metric(name: Option<&str>, value: f64) -> Metric {
    Metric { name: name.unwrap(), value }  // panics at runtime
}

// GOOD — make nullability impossible at the type system level
fn create_metric(name: MetricName, value: f64) -> Metric
fn create_anonymous_metric(value: f64) -> Metric
```

---

## 5. Boundaries

When you use third-party libraries, APIs, or legacy code, wrap them. You own the boundary, not the external code.

### Wrapping Third-Party Code

```java
// BAD — Guava Cache leaked into domain logic everywhere
public class UserService {
    private final Cache<String, User> cache;  // Guava type in domain

    public User find(String id) {
        return cache.get(id, () -> repository.findById(id).orElseThrow());
    }
}

// GOOD — wrap Guava behind your own interface
public interface UserCache {
    Optional<User> get(String id);
    void put(String id, User user);
    void evict(String id);
}

public class GuavaUserCache implements UserCache {
    private final Cache<String, User> delegate;
    // Guava details are fully contained here
}
```

### Learning Tests

Before using a third-party API, write tests that probe its behavior. These tests:
1. Document your understanding
2. Catch breaking changes when you upgrade the library
3. Serve as executable examples

```python
# Learning tests for the `httpx` library
class TestHttpxBehavior:
    """These tests document how httpx works, so we know what to expect."""

    def test_404_raises_by_default_when_raise_for_status_called(self):
        with httpx.Client() as client:
            response = client.get("https://httpbin.org/status/404")
            with pytest.raises(httpx.HTTPStatusError):
                response.raise_for_status()

    def test_timeout_raises_connect_timeout(self):
        with pytest.raises(httpx.ConnectTimeout):
            httpx.get("https://httpbin.org/delay/10", timeout=0.001)
```

---

## 6. The Boy Scout Rule

> "Always leave the campground cleaner than you found it."

Every time you touch a file, improve it slightly. Rename one bad variable. Extract one long method. Remove one redundant comment. The compound interest of these micro-improvements is enormous.

```
Touch file → leave it better:
  ✓ Found: confusing variable name → Renamed it
  ✓ Found: 80-line method → Extracted 3 private methods
  ✓ Found: comment explaining WHAT → Deleted comment, renamed to explain itself
  ✓ Found: duplicated validation logic → Extracted shared method
  ✗ Found: whole module needs redesign → Scope this as a separate task, don't do it now
```

---

## 7. Code Smells Reference

```
SMELL                        SIGNAL                                  REFACTORING
─────────────────────────────────────────────────────────────────────────────────
Long Method                  > 20 lines, multiple blank sections     Extract Method
Large Class                  > 200 lines, many fields/methods        Extract Class
Long Parameter List          > 3 parameters                          Introduce Parameter Object
Divergent Change             One class changes for many reasons      Extract Class
Shotgun Surgery              One change touches many classes         Move Method/Field
Feature Envy                 Method uses another class more         Move Method
Data Clumps                  Same 3-4 fields appear together        Extract Class
Primitive Obsession          String/int for domain concepts          Replace with Value Object
Switch Statements            switch on type code repeated            Replace with Polymorphism
Parallel Inheritance         Two hierarchies must stay in sync       Collapse Hierarchy
Lazy Class                   Class does too little to justify it    Inline Class
Speculative Generality       "We might need this someday"           Remove Parameter / Inline
Temporary Field              Field only set in some paths            Introduce Null Object
Message Chains               a.getB().getC().getD()                  Hide Delegate
Middle Man                   Class delegates most of its work        Remove Middle Man
Inappropriate Intimacy       Classes know each other's internals     Move Method, Extract Class
Alternative Classes           Same interface, different names         Rename Method / Merge
Data Class                   Only getters/setters, no behavior       Move Method into class
Refused Bequest              Subclass ignores parent methods         Replace Inheritance with Delegation
Comments                     Comment explains confusing code         Rename / Extract Method
```

### Detailed Smell Examples

**Feature Envy**
```java
// BAD — Order method is envious of Customer's data
public class Order {
    public String generateShippingLabel() {
        return customer.getName() + "\n"
             + customer.getAddress().getStreet() + "\n"
             + customer.getAddress().getCity() + ", "
             + customer.getAddress().getState() + " "
             + customer.getAddress().getZip();
    }
}

// GOOD — the behavior belongs where the data is
public class Address {
    public String toShippingLabel(String recipientName) {
        return recipientName + "\n" + street + "\n" + city + ", " + state + " " + zip;
    }
}
```

**Primitive Obsession**
```kotlin
// BAD — primitives for domain concepts
fun createOrder(customerId: String, totalAmount: Double, currency: String) { }

// GOOD — value objects
fun createOrder(customerId: CustomerId, total: Money) { }

@JvmInline
value class CustomerId(val value: String) {
    init { require(value.isNotBlank()) { "CustomerId cannot be blank" } }
}

data class Money(val amount: BigDecimal, val currency: Currency) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
}
```

**Data Clumps**
```python
# BAD — these three always appear together
def send_email(smtp_host: str, smtp_port: int, smtp_user: str, subject: str, body: str):
    ...

# They're a clump — extract them
@dataclass
class SmtpConfig:
    host: str
    port: int
    user: str

def send_email(smtp: SmtpConfig, subject: str, body: str):
    ...
```

**Message Chains (Law of Demeter)**
```cpp
// BAD — violates Law of Demeter; you know too much about object internals
double tax = order.getCustomer().getAddress().getRegion().getTaxRate();

// GOOD — ask the object to compute it
double tax = order.getTaxRate();  // Order delegates internally

// The Law of Demeter: a method should only call methods on:
//   - itself
//   - its parameters
//   - objects it creates
//   - its direct components (fields)
```

---

## 8. Multi-Language Examples

### Rust: Value Objects and Exhaustive Error Handling

```rust
use std::fmt;

// Value object with invariant enforcement
#[derive(Debug, Clone, PartialEq)]
pub struct EmailAddress(String);

impl EmailAddress {
    pub fn new(value: &str) -> Result<Self, InvalidEmail> {
        if value.contains('@') && value.len() > 3 {
            Ok(EmailAddress(value.to_lowercase()))
        } else {
            Err(InvalidEmail(value.to_string()))
        }
    }

    pub fn value(&self) -> &str {
        &self.0
    }
}

#[derive(Debug)]
pub struct InvalidEmail(String);

impl fmt::Display for InvalidEmail {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "'{}' is not a valid email address", self.0)
    }
}

// Exhaustive error handling with ? operator — no null, no hidden panics
pub fn register_user(email: &str, name: &str) -> Result<User, RegistrationError> {
    let email = EmailAddress::new(email).map_err(RegistrationError::InvalidEmail)?;
    let name = UserName::new(name).map_err(RegistrationError::InvalidName)?;
    Ok(User::new(email, name))
}
```

### Python: Command-Query Separation with Dataclasses

```python
from dataclasses import dataclass, field
from typing import List
from uuid import UUID, uuid4

@dataclass
class ShoppingCart:
    id: UUID = field(default_factory=uuid4)
    _items: List['CartItem'] = field(default_factory=list, repr=False)

    # Queries — return values, no side effects
    @property
    def items(self) -> List['CartItem']:
        return list(self._items)  # defensive copy

    @property
    def total(self) -> 'Money':
        return sum((item.subtotal for item in self._items), Money.zero())

    def contains(self, product_id: 'ProductId') -> bool:
        return any(item.product_id == product_id for item in self._items)

    # Commands — side effects, return None
    def add_item(self, product_id: 'ProductId', quantity: int, unit_price: 'Money') -> None:
        if self.contains(product_id):
            raise DuplicateItemError(product_id)
        self._items.append(CartItem(product_id, quantity, unit_price))

    def remove_item(self, product_id: 'ProductId') -> None:
        self._items = [i for i in self._items if i.product_id != product_id]
```

---

## When to Apply This Skill

- You are writing new code (apply proactively)
- You are reviewing code (apply as checklist)
- You are asked to refactor code (identify smells first, then apply targeted refactorings)
- A function is > 20 lines
- A class has > 5 public methods or > 200 lines
- A variable name requires a comment to explain it
- You feel the urge to write a comment explaining WHAT the code does

## When NOT to Apply Dogmatically

- Performance-critical hot paths where clarity must be traded for speed (document the trade-off)
- Generated code (don't clean what will be regenerated)
- Prototype/throwaway code explicitly labeled as such
- When the team has an established different convention — consistency beats personal style

---

## Checklist

### Naming
- [ ] Every name reveals intent — no abbreviations, no noise words
- [ ] Classes are nouns; methods are verbs
- [ ] One word per concept across the codebase
- [ ] No disinformation (no false type hints, no misleading names)
- [ ] Names are pronounceable and searchable
- [ ] No magic numbers — all constants are named

### Functions
- [ ] Each function does exactly one thing
- [ ] No function exceeds ~20 lines
- [ ] One level of abstraction per function
- [ ] No more than 2-3 parameters (use parameter objects otherwise)
- [ ] No flag arguments
- [ ] Command-Query Separation respected

### Comments
- [ ] No comments explain WHAT the code does
- [ ] Comments explain WHY (decisions, trade-offs, warnings)
- [ ] No commented-out code
- [ ] No journal/changelog comments

### Error Handling
- [ ] No return codes for errors — use exceptions or Result types
- [ ] No null returns — use Optional, empty collections, or domain exceptions
- [ ] No null parameters — use overloads or Null Objects

### Smells Eliminated
- [ ] No Long Methods (> 20 lines)
- [ ] No Large Classes (> 200 lines / too many responsibilities)
- [ ] No Feature Envy
- [ ] No Primitive Obsession for domain concepts
- [ ] No Data Clumps
- [ ] No Message Chains (Law of Demeter respected)
- [ ] No Shotgun Surgery risk
- [ ] No Speculative Generality ("YAGNI")

### Boy Scout
- [ ] The code is measurably cleaner than when you found it
