---
name: java-excellence
description: Expert modern Java (17+) — records, sealed classes, virtual threads, streams, JVM tuning, and Effective Java principles
---

# Java Excellence

You are an expert modern Java engineer. Apply Java 17+ features, Effective Java principles, and idiomatic patterns. Prefer expressive, type-safe, minimal-boilerplate code over legacy verbose idioms.

---

## Modern Java Features (17+)

### Records

```java
// BEFORE (Java 14-): 40 lines of boilerplate
public final class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// AFTER (Java 16+): one line
record Point(int x, int y) {}

// Records can have custom constructors and methods
record Range(int min, int max) {
    Range {  // compact canonical constructor — validation
        if (min > max) throw new IllegalArgumentException("min > max");
    }
    int span() { return max - min; }
}
```

### Sealed Classes

```java
// Model a closed type hierarchy — exhaustive switch possible
public sealed interface Shape
    permits Circle, Rectangle, Triangle {}

record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}
record Triangle(double base, double height) implements Shape {}

// Exhaustive switch — compiler verifies all cases covered
double area(Shape s) {
    return switch (s) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t  -> 0.5 * t.base() * t.height();
        // no default needed — compiler knows all subtypes
    };
}
```

### Pattern Matching

```java
// instanceof pattern matching (Java 16+)
// BEFORE
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}
// AFTER
if (obj instanceof String s) {
    System.out.println(s.length());
}

// Switch pattern matching (Java 21)
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "negative int: " + i;
        case Integer i            -> "positive int: " + i;
        case String s             -> "string of length " + s.length();
        case null                 -> "null";
        default                   -> "other: " + obj.getClass().getSimpleName();
    };
}
```

### Text Blocks

```java
// BEFORE
String json = "{\n  \"name\": \"Alice\",\n  \"age\": 30\n}";

// AFTER (Java 15+)
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;  // closing """ sets the indentation baseline
```

---

## Functional Java

### Streams

```java
// Idiomatic stream pipeline
var result = employees.stream()
    .filter(e -> e.department().equals("Engineering"))
    .sorted(Comparator.comparing(Employee::salary).reversed())
    .limit(10)
    .map(Employee::name)
    .toList();  // Java 16+ — unmodifiable list, prefer over collect(toList())

// Grouping
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));

// Statistics
IntSummaryStatistics stats = employees.stream()
    .mapToInt(Employee::salary)
    .summaryStatistics();
```

**Stream anti-patterns**:
```java
// WRONG: modifying external state from stream (breaks parallelism, confusing)
List<String> names = new ArrayList<>();
employees.stream().forEach(e -> names.add(e.name())); // DON'T

// RIGHT
List<String> names = employees.stream().map(Employee::name).toList();

// WRONG: stream for simple iteration with no transformation
employees.stream().forEach(System.out::println); // pointless stream
// RIGHT
employees.forEach(System.out::println); // Iterable.forEach

// WRONG: parallel() without measuring
.parallelStream() // not always faster — overhead for small collections
```

### Optional

```java
// Optional is for RETURN TYPES only — signals "might be absent"
// NEVER use Optional as a field type or method parameter

// WRONG
public Optional<String> name = Optional.of("Alice"); // field — no
public void process(Optional<String> name) { ... }   // param — no

// RIGHT: return type
public Optional<User> findById(long id) {
    return users.stream().filter(u -> u.id() == id).findFirst();
}

// Chaining — avoid .get() and .isPresent()
String result = findById(42)
    .map(User::email)
    .filter(e -> e.endsWith("@company.com"))
    .orElse("unknown@company.com");

// orElseGet — lazy evaluation (use when default is expensive)
User user = findById(id).orElseGet(() -> createDefaultUser());

// orElseThrow — explicit failure
User user = findById(id).orElseThrow(() -> new NotFoundException(id));
```

### Method References

```java
// Four kinds
employees.forEach(System.out::println);           // static: Type::staticMethod
employees.stream().map(Employee::name)            // instance (unbound): Type::instanceMethod
         .map(String::toUpperCase);
employees.stream().filter(this::isActive);        // instance (bound): obj::instanceMethod
employees.stream().map(Employee::new)             // constructor: Type::new
         .collect(toList());
```

---

## Concurrency

### Virtual Threads (Project Loom, Java 21)

```
Platform threads:  1 OS thread per Java thread — expensive (~1MB stack each)
Virtual threads:   M:N mapping — millions of VTs on a few platform threads
                   Blocking a VT (I/O, sleep) does NOT block the platform thread

┌──────────────────────────────────────────────────────────────┐
│  Virtual Thread 1 ──► blocks on DB query                    │
│  Virtual Thread 2 ──► runs on carrier thread while VT1 waits│
│  Virtual Thread 3 ──► ...                                   │
│  ...millions...                                              │
└──────────────────────────────────────────────────────────────┘
```

```java
// Simple: one virtual thread per request (replaces thread pools for I/O)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            // Blocking I/O here is fine — virtual thread yields, not OS thread
            return httpClient.send(request, BodyHandlers.ofString());
        });
    }
}

// Direct creation
Thread.ofVirtual().start(() -> System.out.println("virtual!"));
```

**Virtual thread pitfalls**:
- Avoid `synchronized` blocks (pin the carrier thread) — use `ReentrantLock` instead
- Not beneficial for CPU-bound work — only helps I/O-bound concurrency

### CompletableFuture Composition

```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))          // runs in ForkJoinPool
    .thenApply(user -> user.email())               // transform (same thread)
    .thenCompose(email -> sendEmail(email))        // chain another async op
    .exceptionally(ex -> {                         // error recovery
        log.error("Failed", ex);
        return "fallback@example.com";
    });

// Parallel composition
var userFuture  = CompletableFuture.supplyAsync(() -> fetchUser(id));
var orderFuture = CompletableFuture.supplyAsync(() -> fetchOrders(id));

CompletableFuture.allOf(userFuture, orderFuture).thenRun(() -> {
    User user     = userFuture.join();
    List<?> orders = orderFuture.join();
    // both done now
});
```

### Java Memory Model — Key Rules

```
Happens-before (HB) relationships guarantee visibility:
  1. Thread start: actions before t.start() HB all actions in t
  2. Monitor unlock HB subsequent lock of same monitor
  3. volatile write HB subsequent volatile read of same variable
  4. Thread termination: all actions in t HB t.join() returns

volatile: guarantees visibility, NOT atomicity
  volatile int x;   // reads/writes are atomic and visible
  x++;              // NOT atomic — read, increment, write are 3 ops

Use AtomicInteger for compound operations:
  AtomicInteger x = new AtomicInteger(0);
  x.incrementAndGet(); // atomic
  x.compareAndSet(expected, newValue); // CAS
```

---

## Collections

```
Choosing the right collection:
  Need fast lookup by key?     → HashMap (O(1) avg)
  Need sorted by key?          → TreeMap (O(log n))
  Need insertion order?        → LinkedHashMap
  Need thread-safe map?        → ConcurrentHashMap (never Hashtable/synchronizedMap)
  Need unique elements?        → HashSet / TreeSet / LinkedHashSet
  Need ordered by priority?    → PriorityQueue
  Need thread-safe queue?      → ConcurrentLinkedQueue / ArrayBlockingQueue
  Need immutable list?         → List.of(...) / List.copyOf(...)
```

```java
// Immutable collections (Java 9+)
List<String>        list = List.of("a", "b", "c");
Set<String>         set  = Set.of("x", "y");
Map<String, Integer> map = Map.of("one", 1, "two", 2);

// Mutable copy of immutable
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));

// ConcurrentHashMap — never use Collections.synchronizedMap
ConcurrentHashMap<String, Long> counts = new ConcurrentHashMap<>();
counts.merge(key, 1L, Long::sum); // atomic compute
```

---

## JVM Tuning for Containers

```bash
# Always set -XX:+UseContainerSupport (default in JDK 11+)
# JVM reads cgroup limits, not host memory

# G1GC (default Java 9+): good for most workloads
java -XX:+UseG1GC \
     -Xms512m -Xmx2g \
     -XX:MaxGCPauseMillis=200 \
     -jar app.jar

# ZGC: ultra-low pause (<1ms), good for latency-sensitive services
java -XX:+UseZGC \
     -Xmx4g \
     -jar app.jar

# Shenandoah: concurrent compaction, similar to ZGC
java -XX:+UseShenandoahGC -jar app.jar

# Container-friendly flags
java -XX:+UseContainerSupport \
     -XX:MaxRAMPercentage=75.0 \    # use 75% of container memory
     -XX:+ExitOnOutOfMemoryError \  # fail fast, let k8s restart
     -jar app.jar
```

**GC selection guide**:
```
Throughput-focused batch jobs   → G1GC or ParallelGC
Latency-sensitive services      → ZGC or Shenandoah
Interactive/low-memory apps     → SerialGC (single-core containers)
```

---

## Effective Java Principles (Applied)

### Prefer Composition over Inheritance

```java
// WRONG: inheriting just to reuse methods
class Stack<E> extends ArrayList<E> {  // Stack IS-A ArrayList? No.
    public void push(E e) { add(e); }
    public E pop() { return remove(size() - 1); }
}

// RIGHT: composition
class Stack<E> {
    private final Deque<E> delegate = new ArrayDeque<>();
    public void push(E e) { delegate.push(e); }
    public E pop() { return delegate.pop(); }
    public boolean isEmpty() { return delegate.isEmpty(); }
    // only expose what Stack should expose
}
```

### Minimize Mutability

```java
// Prefer immutable classes: all fields final, no setters
public final class Money {
    private final long amount;
    private final Currency currency;

    public Money(long amount, Currency currency) {
        this.amount = amount;
        this.currency = Objects.requireNonNull(currency);
    }

    // Return new instance instead of mutating
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount + other.amount, this.currency);
    }
}
```

### Null Handling

```java
// WRONG: return null
public User findUser(long id) {
    return null; // caller must know to check
}

// RIGHT: Optional for nullable returns
public Optional<User> findUser(long id) { ... }

// RIGHT: Objects.requireNonNull for defensive checks
public void process(String input) {
    this.input = Objects.requireNonNull(input, "input must not be null");
}

// RIGHT: Annotations for documentation
public void send(@NonNull String message, @Nullable String header) { ... }
```

---

## Testing

```java
// JUnit 5 — idiomatic test class
@DisplayName("OrderService")
class OrderServiceTest {

    @Nested
    @DisplayName("when creating an order")
    class CreateOrder {

        @Test
        @DisplayName("should reject empty cart")
        void rejectEmptyCart() {
            var service = new OrderService(mock(PaymentGateway.class));
            assertThatThrownBy(() -> service.create(Cart.empty()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("empty");
        }

        @ParameterizedTest
        @ValueSource(ints = {-1, 0, Integer.MIN_VALUE})
        @DisplayName("should reject non-positive quantities")
        void rejectNonPositiveQuantity(int qty) {
            assertThatThrownBy(() -> new CartItem(PRODUCT, qty))
                .isInstanceOf(IllegalArgumentException.class);
        }
    }

    @Test
    void processPayment_delegatesToGateway() {
        var gateway = mock(PaymentGateway.class);
        when(gateway.charge(any(), anyLong())).thenReturn(PaymentResult.SUCCESS);

        var service = new OrderService(gateway);
        service.create(validCart());

        verify(gateway).charge(eq(validCart().total()), anyLong());
        verifyNoMoreInteractions(gateway);
    }
}
```

**AssertJ over JUnit assertions**:
```java
// JUnit (weak error messages)
assertEquals(expected, actual);
assertTrue(list.contains("foo"));

// AssertJ (descriptive failures, fluent API)
assertThat(actual).isEqualTo(expected);
assertThat(list).contains("foo").hasSize(3).doesNotContain("bar");
assertThat(map).containsEntry("key", "value");
```

---

## Build Tools

```xml
<!-- Maven: explicit dependency management -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

```kotlin
// Gradle (Kotlin DSL): concise, programmable
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.junit.jupiter:junit-jupiter")
}
tasks.test { useJUnitPlatform() }
```

**Maven vs Gradle**:
```
Maven:   Predictable, verbose XML, better IDE support for legacy projects
Gradle:  Faster (incremental builds, build cache), concise, flexible — prefer for new projects
```

---

## Common Pitfalls

| Pitfall                                 | Fix                                                   |
|-----------------------------------------|-------------------------------------------------------|
| `new ArrayList()` raw types             | `new ArrayList<String>()` — always use generics       |
| `==` for String comparison              | `.equals()` always                                    |
| `catch (Exception e) {}` swallowing     | Log + rethrow or handle specifically                  |
| Checked exceptions in lambdas           | Wrap in unchecked, or use a functional interface      |
| `static` mutable state                  | Inject dependencies, avoid global state               |
| `SimpleDateFormat` in multithreading    | Use `DateTimeFormatter` (thread-safe)                 |
| `HashMap.get()` without null check      | `getOrDefault()` or `containsKey()` first             |
| Overusing checked exceptions            | Prefer unchecked for programming errors               |
| `for (int i=0; i<list.size(); i++)`    | Enhanced for-each or streams                          |
| `Collections.synchronizedList`          | `CopyOnWriteArrayList` or rethink concurrency model   |

---

## Checklist

- [ ] Java 17+ features used where appropriate (records, sealed, pattern matching)
- [ ] `Optional` only as return type, never field/parameter
- [ ] Streams used for transformation pipelines, not for side effects
- [ ] Collections are immutable by default (`List.of`, `Map.of`)
- [ ] Virtual threads considered for I/O-bound concurrent code (Java 21)
- [ ] `CompletableFuture` chains have `.exceptionally()` or `.handle()`
- [ ] `volatile` only for visibility, `AtomicXxx` for compound operations
- [ ] `ConcurrentHashMap` for thread-safe maps, never `synchronizedMap`
- [ ] JVM container flags set (`UseContainerSupport`, `MaxRAMPercentage`)
- [ ] Tests use JUnit5 + AssertJ, not legacy JUnit4 assertions
- [ ] No raw types, no unchecked casts without comment
- [ ] `try-with-resources` for all `Closeable` resources
- [ ] Effective Java: composition over inheritance, minimize mutability
