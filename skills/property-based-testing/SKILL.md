---
name: property-based-testing
description: Apply property-based testing with Hypothesis (Python), jqwik (Java/Kotlin), or Kotest Property (Kotlin) to find edge cases in parsers, serialization, pure functions, and state machines
---

# Property-Based Testing

## What Is Property-Based Testing?

```
EXAMPLE-BASED (traditional unit test):
  def test_sort():
      assert sort([3, 1, 2]) == [1, 2, 3]   # tests ONE specific input

PROPERTY-BASED:
  @given(st.lists(st.integers()))
  def test_sort_is_idempotent(lst):
      assert sort(sort(lst)) == sort(lst)     # tests HUNDREDS of generated inputs
                                              # including [], [0], [-1, maxint], etc.
```

**Key shift:** Instead of "this input produces this output," ask "what must ALWAYS be true, regardless of input?"

The framework:
1. Generates hundreds of random examples
2. Runs your property assertion against each
3. When it finds a failure, **shrinks** to the minimal failing example
4. Reports: "Failed on `lst = [-1]`" not "Failed on `lst = [847, -231, 0, 99, -1, 3]`"

---

## Property Types

### 1. Invariants — Something Always True

```python
@given(st.lists(st.integers()))
def test_sort_length_preserved(lst):
    assert len(sort(lst)) == len(lst)           # length never changes

@given(st.integers(), st.integers())
def test_add_is_non_negative_for_non_negatives(a, b):
    assume(a >= 0 and b >= 0)                   # constrain inputs
    assert a + b >= 0
```

### 2. Idempotence — Applying Twice Equals Applying Once

```
f(f(x)) == f(x)

Examples:
  sort(sort(x)) == sort(x)
  normalize(normalize(x)) == normalize(x)
  set(set(x)) == set(x)
  strip(strip(s)) == strip(s)
  format(format(code)) == format(code)    ← great for code formatters!
```

```python
@given(st.text())
def test_strip_is_idempotent(s):
    assert s.strip().strip() == s.strip()

@given(st.text())
def test_formatter_is_idempotent(code):
    formatted_once = format_code(code)
    formatted_twice = format_code(formatted_once)
    assert formatted_once == formatted_twice
```

### 3. Round-Trip — Encode/Decode Returns Original

```
decode(encode(x)) == x

Examples:
  JSON:       json.loads(json.dumps(x)) == x
  Base64:     b64decode(b64encode(x)) == x
  Protobuf:   Message.FromString(message.SerializeToString()) == message
  Compression: decompress(compress(x)) == x
  Serializer: deserialize(serialize(obj)) == obj
```

```python
@given(st.dictionaries(st.text(), st.integers()))
def test_json_round_trip(data):
    assert json.loads(json.dumps(data)) == data

@given(st.binary())
def test_base64_round_trip(data):
    assert base64.b64decode(base64.b64encode(data)) == data
```

### 4. Commutativity — Order Doesn't Matter

```
f(a, b) == f(b, a)

Examples:
  a + b == b + a           (addition)
  union(A, B) == union(B, A)
  max(a, b) == max(b, a)
  sorted(A + B) == sorted(B + A)
```

```python
@given(st.sets(st.integers()), st.sets(st.integers()))
def test_set_union_is_commutative(a, b):
    assert a | b == b | a

@given(st.integers(), st.integers())
def test_addition_is_commutative(a, b):
    assert a + b == b + a
```

### 5. Monotonicity — More Input Means More (or Less) Output

```
len(list) increases → runtime of O(n) algorithm increases
discount rate increases → price decreases
compression ratio: more entropy → less compression
```

```python
@given(st.integers(min_value=1, max_value=100),
       st.integers(min_value=1, max_value=100))
def test_larger_list_takes_longer_to_sort(n1, n2):
    assume(n1 < n2)
    lst1 = list(range(n1))
    lst2 = list(range(n2))
    # This is a weak property — just checking correctness for now
    assert len(sort(lst1)) == n1
    assert len(sort(lst2)) == n2
```

### 6. Oracle — New vs Reference Implementation

```
new_impl(x) == reference_impl(x)   for all x

When to use:
  - Rewriting a function for performance
  - Porting to a new language
  - Optimizing an algorithm
  - Refactoring with no spec other than "behaves the same"
```

```python
@given(st.text())
def test_new_tokenizer_matches_reference(text):
    expected = reference_tokenizer(text)     # slow but known-correct
    actual = new_fast_tokenizer(text)        # new implementation to verify
    assert actual == expected

@given(st.lists(st.integers()))
def test_optimized_median_matches_naive(lst):
    assume(len(lst) > 0)
    assert optimized_median(lst) == naive_median(lst)
```

---

## Python — Hypothesis

### Installation and Basic Usage

```bash
pip install hypothesis
```

```python
from hypothesis import given, assume, settings, HealthCheck
from hypothesis import strategies as st

@given(st.integers(), st.integers())
def test_addition_commutative(a, b):
    assert a + b == b + a   # runs ~100 examples by default
```

### Built-in Strategies

```python
# Primitives
st.integers()                          # all integers
st.integers(min_value=0, max_value=100)
st.floats()                            # includes NaN, inf, -inf
st.floats(allow_nan=False, allow_infinity=False)
st.booleans()
st.text()                              # unicode strings, including emoji, RTL, nulls
st.text(alphabet=st.characters(whitelist_categories=('Lu', 'Ll')))
st.binary()

# Collections
st.lists(st.integers())
st.lists(st.integers(), min_size=1, max_size=10)
st.sets(st.text())
st.dictionaries(st.text(), st.integers())
st.tuples(st.integers(), st.text(), st.booleans())

# Dates and times
st.datetimes()
st.dates()
st.timedeltas()

# Combinators
st.one_of(st.integers(), st.text())   # union of strategies
st.just(42)                           # always produces 42
st.none()                             # always produces None
st.sampled_from([1, 2, 3])           # pick from list
```

### Custom Strategies

```python
from dataclasses import dataclass
from hypothesis import strategies as st

@dataclass
class User:
    name: str
    age: int
    email: str

# st.builds: construct from component strategies
user_strategy = st.builds(
    User,
    name=st.text(min_size=1, max_size=50),
    age=st.integers(min_value=0, max_value=120),
    email=st.emails()
)

# @st.composite: complex multi-step construction
@st.composite
def valid_order(draw):
    items = draw(st.lists(st.integers(min_value=1, max_value=999), min_size=1))
    discount = draw(st.floats(min_value=0.0, max_value=0.5))
    subtotal = sum(items)
    assume(subtotal > 0)  # filter invalid combos
    return Order(items=items, discount=discount)

@given(valid_order())
def test_discount_never_exceeds_total(order):
    result = apply_discount(order)
    assert result.total >= 0
    assert result.total <= sum(order.items)
```

### Shrinking

```
When Hypothesis finds a failing case:
  Input: lst = [847, -231, 0, 99, -1, 3, 77, -500]

Hypothesis shrinks to the minimal failing example:
  Input: lst = [-1]   ← smallest input that still fails the property

WHY THIS MATTERS: You debug the simple case, not the complex one.
```

Shrinking is automatic — you get it for free with all built-in strategies. Custom `@st.composite` strategies that use `draw()` are also automatically shrinkable.

### Stateful Testing (State Machines)

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant, initialize

class QueueMachine(RuleBasedStateMachine):
    """Test that a Queue implementation behaves like a queue."""

    @initialize()
    def setup(self):
        self.model = []           # reference model (Python list)
        self.impl = MyQueue()     # implementation under test

    @rule(value=st.integers())
    def enqueue(self, value):
        self.model.append(value)
        self.impl.enqueue(value)

    @rule()
    def dequeue(self):
        if self.model:
            expected = self.model.pop(0)
            actual = self.impl.dequeue()
            assert actual == expected

    @invariant()
    def sizes_match(self):
        assert len(self.model) == self.impl.size()

TestQueue = QueueMachine.TestCase
```

### Database of Examples

```python
# Hypothesis remembers failing examples in .hypothesis/ directory
# On next run, it replays known failures FIRST before generating new ones

# To clear the database:
hypothesis database --help

# In CI: commit .hypothesis/ to ensure failures are always replayed
```

### Settings

```python
from hypothesis import given, settings, HealthCheck
from hypothesis.database import DirectoryBasedExampleDatabase

@given(st.integers())
@settings(
    max_examples=500,           # more examples for thorough testing
    deadline=None,              # disable timing-out slow tests
    suppress_health_check=[HealthCheck.too_slow],
)
def test_my_function(n):
    ...

# Profile-based settings
settings.register_profile("ci", max_examples=1000)
settings.register_profile("dev", max_examples=50)
settings.load_profile("ci" if os.environ.get("CI") else "dev")
```

---

## Java/Kotlin — jqwik

### Setup

```gradle
// build.gradle.kts
dependencies {
    testImplementation("net.jqwik:jqwik:1.8.2")
    testImplementation("org.junit.platform:junit-platform-commons:1.10.0")
}

test {
    useJUnitPlatform()
}
```

### Basic Properties

```java
import net.jqwik.api.*;
import net.jqwik.api.constraints.*;

class StringPropertiesTest {

    @Property
    boolean reverseIsIdempotent(@ForAll String s) {
        return new StringBuilder(s).reverse().toString()
            .equals(new StringBuilder(
                new StringBuilder(s).reverse().toString()
            ).reverse().toString());
        // reverse(reverse(s)) == s
    }

    @Property
    void sortingPreservesLength(@ForAll List<@IntRange(min = -100, max = 100) Integer> list) {
        List<Integer> sorted = list.stream().sorted().collect(toList());
        assertThat(sorted).hasSize(list.size());
    }

    @Property
    void jsonRoundTrip(@ForAll @StringLength(min = 1, max = 100) @AlphaChars String input) {
        ObjectMapper mapper = new ObjectMapper();
        String json = mapper.writeValueAsString(input);
        String result = mapper.readValue(json, String.class);
        assertThat(result).isEqualTo(input);
    }
}
```

### Custom Arbitraries

```java
class OrderPropertiesTest {

    @Provide
    Arbitrary<Order> validOrders() {
        Arbitrary<List<Integer>> items = Arbitraries.integers()
            .between(1, 1000)
            .list()
            .ofMinSize(1)
            .ofMaxSize(20);

        Arbitrary<Double> discounts = Arbitraries.doubles()
            .between(0.0, 0.5);

        return Combinators.combine(items, discounts)
            .as((i, d) -> new Order(i, d));
    }

    @Property
    void discountNeverExceedsTotal(@ForAll("validOrders") Order order) {
        ProcessedOrder result = orderProcessor.process(order);
        assertThat(result.getTotal()).isGreaterThanOrEqualTo(0.0);
        assertThat(result.getTotal())
            .isLessThanOrEqualTo(order.getSubtotal());
    }
}
```

### Constraints

```java
@Property
void allConstraintsExample(
    @ForAll @IntRange(min = 1, max = 100) int age,
    @ForAll @StringLength(min = 1, max = 50) @AlphaChars String name,
    @ForAll @Positive double price,
    @ForAll @Size(min = 1, max = 10) List<String> tags
) {
    // guaranteed: 1 <= age <= 100, name is alpha 1-50 chars, etc.
}
```

### Stateful Testing with ActionSequence

```java
@Property
void queueBehavesCorrectly(
    @ForAll("queueActions") ActionSequence<MyQueue<Integer>> sequence
) {
    sequence.run(new MyQueue<>());
}

@Provide
Arbitrary<ActionSequence<MyQueue<Integer>>> queueActions() {
    return ActionSequence.sequences(
        Arbitraries.oneOf(enqueueAction(), dequeueAction())
    );
}

private Arbitrary<Action<MyQueue<Integer>>> enqueueAction() {
    return Arbitraries.integers().map(value -> queue -> {
        int sizeBefore = queue.size();
        queue.enqueue(value);
        assertThat(queue.size()).isEqualTo(sizeBefore + 1);
        return queue;
    });
}
```

---

## Kotlin — Kotest Property

### Setup

```gradle
dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-property:5.8.0")
}
```

### Basic Properties

```kotlin
import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe
import io.kotest.property.*
import io.kotest.property.arbitrary.*

class StringPropertiesTest : FunSpec({

    // forAll: property must return Boolean
    test("string length preserved by reverse") {
        forAll(Arb.string()) { s ->
            s.reversed().length == s.length
        }
    }

    // checkAll: property uses assertions (doesn't return Boolean)
    test("sort is idempotent") {
        checkAll(Arb.list(Arb.int())) { lst ->
            lst.sorted().sorted() shouldBe lst.sorted()
        }
    }

    test("addition is commutative") {
        forAll(Arb.int(), Arb.int()) { a, b ->
            a + b == b + a
        }
    }

    test("json round trip") {
        checkAll(Arb.map(Arb.string(1..20), Arb.int())) { map ->
            val json = objectMapper.writeValueAsString(map)
            val result = objectMapper.readValue<Map<String, Int>>(json)
            result shouldBe map
        }
    }
})
```

### Custom Arbitraries

```kotlin
data class User(val name: String, val age: Int, val email: String)

val arbUser: Arb<User> = arbitrary {
    User(
        name = Arb.string(1..50, Codepoint.alphanumeric()).bind(),
        age = Arb.int(0..120).bind(),
        email = Arb.email().bind()
    )
}

class UserPropertiesTest : FunSpec({
    test("user serialization round trip") {
        checkAll(arbUser) { user ->
            val serialized = serialize(user)
            val deserialized = deserialize<User>(serialized)
            deserialized shouldBe user
        }
    }
})
```

### Edge Case Coverage

```kotlin
// Kotest provides edge-case-aware arbitraries
Arb.int()        // includes Int.MIN_VALUE, Int.MAX_VALUE, 0, -1
Arb.long()       // includes Long.MIN_VALUE, Long.MAX_VALUE
Arb.double()     // includes NaN, Infinity, -Infinity, 0.0, -0.0
Arb.string()     // includes empty string, unicode, emoji

// Control iterations
checkAll(PropTestConfig(iterations = 1000), Arb.int()) { n ->
    // runs 1000 times instead of default 1000
}
```

---

## When PBT Shines

```
EXCELLENT FIT:
+---------------------------+------------------------------------------------+
| Domain                    | Example property to test                       |
+---------------------------+------------------------------------------------+
| Parsers                   | parse(unparse(x)) == x                         |
| Serialization / codecs    | decode(encode(x)) == x                         |
| Data transformations      | transform preserves invariants                 |
| Pure functions            | commutativity, associativity, monotonicity     |
| Sorting / searching       | sorted output has correct length and ordering  |
| Compression               | decompress(compress(x)) == x                   |
| Encryption                | decrypt(encrypt(x, key), key) == x             |
| Numeric algorithms        | results within epsilon of reference impl       |
| State machines            | invariants hold across all action sequences    |
| Protocol implementations  | request/response round trips                   |
| Data validation           | valid inputs always accepted; schema preserved |
+---------------------------+------------------------------------------------+

POOR FIT:
  - Tests with external I/O side effects (use example-based with mocks)
  - Tests that verify specific business rules for specific inputs
    ("if age < 18, reject" — better as an example test)
  - Performance tests (PBT overhead is significant)
  - UI/UX behavior (hard to define properties)
```

---

## Combining PBT with Unit Tests

```
PRINCIPLE: The two testing styles are complementary, not competing.

UNIT TESTS are best for:
  - Documenting specific behaviors ("empty cart returns 0")
  - Verifying known edge cases found by PBT (lock in regressions)
  - Business rules with specific expected outputs
  - Error message wording

PROPERTY-BASED TESTS are best for:
  - Finding unknown edge cases
  - Verifying structural invariants
  - Testing across a whole domain, not just sampled points
  - Refactoring safety (oracle properties against old impl)

WORKFLOW:
  1. Write PBT for structural properties
  2. PBT discovers a failing case: e.g., decode(encode("")) throws
  3. Add a unit test: test_encode_decode_empty_string()
     (this locks in the regression and documents the bug)
  4. Fix the bug
  5. Both tests pass going forward
```

```python
# Example: unit test + PBT together
class TestJsonSerializer:

    # Unit tests: specific behaviors, documentation
    def test_empty_dict_serializes_to_braces(self):
        assert serialize({}) == "{}"

    def test_nested_dict_round_trips(self):
        data = {"a": {"b": {"c": 1}}}
        assert deserialize(serialize(data)) == data

    # Property-based: structural invariants across all inputs
    @given(st.dictionaries(
        st.text(min_size=1),
        st.one_of(st.integers(), st.text(), st.booleans(), st.none())
    ))
    def test_round_trip_all_json_types(self, data):
        assert deserialize(serialize(data)) == data

    @given(st.dictionaries(st.text(), st.integers()))
    def test_serialized_length_positive(self, data):
        assert len(serialize(data)) > 0
```

---

## Anti-Patterns

```
+----------------------------------------+------------------------------------------+
| Anti-Pattern                           | Why It Fails                             |
+----------------------------------------+------------------------------------------+
| Testing implementation details         | Property breaks on refactoring even when |
| ("output contains substring X")        | behavior is correct                      |
+----------------------------------------+------------------------------------------+
| Overly constrained generators          | assume() filters too many inputs;        |
| ("assume all of these conditions")     | framework warns; edge cases missed       |
+----------------------------------------+------------------------------------------+
| Ignoring shrunk counterexamples        | The shrunk example IS the bug diagnosis; |
|                                        | ignoring it means fixing the wrong thing |
+----------------------------------------+------------------------------------------+
| Properties that always pass            | "result is not None" — trivially true;   |
|                                        | provides no value                        |
+----------------------------------------+------------------------------------------+
| Replacing all unit tests with PBT      | Unit tests document specific behaviors;  |
|                                        | PBT is additive, not a replacement       |
+----------------------------------------+------------------------------------------+
| Not committing .hypothesis/ directory  | Known failures not replayed in CI;       |
| in version control                     | regressions can silently reappear        |
+----------------------------------------+------------------------------------------+
| Slow strategies without deadline=None  | Tests time out before finding failures;  |
|                                        | set deadline=None for slow code          |
+----------------------------------------+------------------------------------------+
| Testing with max_examples=10           | Too few examples; misses rare edge cases;|
|                                        | use at least 100 (default), 500 for CI   |
+----------------------------------------+------------------------------------------+
```

---

## Quick Reference Checklist

### Choosing What to Test with PBT

- [ ] Does my function have a round-trip property? (serialize/deserialize, encode/decode)
- [ ] Is there an existing "reference" implementation I can oracle-test against?
- [ ] Are there structural invariants that must hold for all inputs? (length, ordering, non-negativity)
- [ ] Is this a pure function with mathematical properties? (commutativity, idempotence)
- [ ] Does this involve a state machine? (use stateful testing)

### Hypothesis (Python) Checklist

- [ ] Use `@given` with appropriate strategies from `hypothesis.strategies`
- [ ] Use `assume()` sparingly — prefer constrained strategies over filters
- [ ] Use `@st.composite` for complex multi-field objects
- [ ] Set `max_examples=500` in CI profile
- [ ] Commit `.hypothesis/` directory to version control
- [ ] For slow code: add `@settings(deadline=None)`
- [ ] Check for `HealthCheck` warnings — they indicate strategy problems

### jqwik (Java/Kotlin) Checklist

- [ ] Annotate test method with `@Property`, parameters with `@ForAll`
- [ ] Use `@Provide` for custom `Arbitrary` implementations
- [ ] Use constraints (`@IntRange`, `@StringLength`) to shape input domain
- [ ] For stateful tests: use `ActionSequence`
- [ ] Run with `./gradlew test` — jqwik integrates with JUnit Platform

### Kotest Property (Kotlin) Checklist

- [ ] Use `forAll` for Boolean-returning properties
- [ ] Use `checkAll` for assertion-based properties
- [ ] Use `arbitrary { }` builder for custom domain objects
- [ ] Set `iterations` in `PropTestConfig` for thoroughness
- [ ] Use `Arb.bind()` inside `arbitrary {}` to compose strategies
