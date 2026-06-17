---
name: design-patterns-mastery
description: Apply classic GoF and modern design patterns — Creational, Structural, Behavioral, Functional — with multi-language examples, ASCII UML diagrams, and anti-pattern warnings
---

# Design Patterns Mastery

## Overview

```
  Design patterns are proven solutions to recurring design problems.
  They are vocabulary, not recipes — communicate intent, not implementations.

  ┌─────────────────┬─────────────────────────────────────────────────────────┐
  │ Category        │ Patterns                                                │
  ├─────────────────┼─────────────────────────────────────────────────────────┤
  │ Creational      │ Factory Method, Abstract Factory, Builder,              │
  │                 │ Singleton, Prototype                                    │
  ├─────────────────┼─────────────────────────────────────────────────────────┤
  │ Structural      │ Adapter, Bridge, Composite, Decorator, Facade, Proxy    │
  ├─────────────────┼─────────────────────────────────────────────────────────┤
  │ Behavioral      │ Strategy, Observer, Command, Chain of Responsibility,   │
  │                 │ Template Method, State, Iterator                        │
  ├─────────────────┼─────────────────────────────────────────────────────────┤
  │ Modern/FP       │ Monad (Result/Either), Pipeline, Specification          │
  └─────────────────┴─────────────────────────────────────────────────────────┘
```

---

## Creational Patterns

### Factory Method

```
  Intent: Define an interface for creating an object, but let subclasses
          decide which class to instantiate.

  ┌──────────────────────────────────────────────────────────┐
  │                     Creator                              │
  │  + factoryMethod(): Product  ← abstract / overridden    │
  │  + someOperation()                                       │
  │       └── calls factoryMethod() internally              │
  └──────────────────┬───────────────────────────────────────┘
                     │ extends
        ┌────────────┴────────────┐
        │                         │
  ┌─────▼──────┐           ┌──────▼─────┐
  │ConcreteA   │           │ConcreteB   │
  │Creator     │           │Creator     │
  │+ factory() │           │+ factory() │
  │  → ProductA│           │  → ProductB│
  └────────────┘           └────────────┘
        │ creates                  │ creates
  ┌─────▼──────┐           ┌───────▼────┐
  │  ProductA  │           │  ProductB  │
  └────────────┘           └────────────┘

  When to use:
    → You don't know ahead of time what class to instantiate
    → You want subclasses to control the products they create
    → Framework/library code that others extend

  When NOT to use:
    → You have a fixed, known set of products → use direct instantiation
    → Simple cases where a plain function suffices
```

```java
// Java — Notification Factory
interface Notification {
    void send(String message, String recipient);
}

class EmailNotification implements Notification {
    public void send(String message, String recipient) {
        // send email...
    }
}

class SMSNotification implements Notification {
    public void send(String message, String recipient) {
        // send SMS...
    }
}

// Creator — the "factory method" is createNotification()
abstract class NotificationService {
    // Factory method — subclasses override this
    protected abstract Notification createNotification();

    // Business logic uses the factory method
    public void notify(String message, String recipient) {
        var notification = createNotification();  // ← polymorphic
        notification.send(message, recipient);
        auditLog(recipient, message);             // shared behavior
    }
}

class EmailNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new EmailNotification();
    }
}
```

```python
# Python — simpler with callables as factories
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> None: ...

class NotificationService(ABC):
    @abstractmethod
    def _create_notification(self) -> Notification: ...

    def notify(self, message: str, recipient: str) -> None:
        notification = self._create_notification()
        notification.send(message, recipient)

# In Python, often just pass a factory callable instead:
def create_notification_service(notification_factory: Callable[[], Notification]):
    def notify(message: str, recipient: str):
        notification_factory().send(message, recipient)
    return notify
```

---

### Abstract Factory

```
  Intent: Create families of related objects without specifying their
          concrete classes. Ensures products from same family work together.

  ┌────────────────────────────────────────────────────────────────────────┐
  │                      AbstractFactory                                   │
  │  + createButton(): Button                                              │
  │  + createCheckbox(): Checkbox                                          │
  └──────────────┬──────────────────────────────────────┬─────────────────┘
                 │ implements                            │ implements
  ┌──────────────▼───────────┐            ┌─────────────▼───────────────┐
  │    WindowsFactory        │            │       MacFactory             │
  │  + createButton()        │            │  + createButton()           │
  │      → WindowsButton     │            │      → MacButton            │
  │  + createCheckbox()      │            │  + createCheckbox()         │
  │      → WindowsCheckbox   │            │      → MacCheckbox          │
  └──────────────────────────┘            └─────────────────────────────┘

  Key point: WindowsButton and MacButton are related — they look consistent.
             You can't accidentally mix WindowsButton with MacCheckbox.

  When to use:
    → System must be independent of how its products are created
    → System needs multiple families of products (themes, OS, cloud providers)
    → Products in a family must work together

  When NOT to use:
    → You only have one product type → use Factory Method
    → Adding new products to the family is frequent → requires changing AbstractFactory
```

```kotlin
// Kotlin — Cloud Provider Abstract Factory
interface StorageClient {
    fun upload(key: String, data: ByteArray)
    fun download(key: String): ByteArray
}

interface QueueClient {
    fun publish(queue: String, message: String)
    fun consume(queue: String): String?
}

interface CloudFactory {
    fun createStorage(): StorageClient
    fun createQueue(): QueueClient
}

class AWSFactory : CloudFactory {
    override fun createStorage() = S3Client()
    override fun createQueue() = SQSClient()
}

class GCPFactory : CloudFactory {
    override fun createStorage() = GCSClient()
    override fun createQueue() = PubSubClient()
}

// Application code depends only on interfaces
class DataPipeline(private val cloud: CloudFactory) {
    private val storage = cloud.createStorage()
    private val queue = cloud.createQueue()

    fun process() {
        val msg = queue.consume("input")
        storage.upload("result/$msg", processData(msg!!))
    }
}

// Wire up:
val pipeline = DataPipeline(if (env == "aws") AWSFactory() else GCPFactory())
```

---

### Builder

```
  Intent: Construct complex objects step by step. Separate construction from
          representation. Same construction process → different representations.

  ┌─────────────────────────────────────────────────────────────────────────┐
  │  Classic Builder (Director + Builder)                                   │
  │                                                                         │
  │  Director ──── uses ────► Builder (interface)                           │
  │  + construct()              + buildPartA()                              │
  │       │                     + buildPartB()                              │
  │       │                     + getResult(): Product                      │
  │       │                           △                                     │
  │       │                    ┌──────┴──────┐                              │
  │       │               ConcreteBuilderA  ConcreteBuilderB               │
  │       └──────────────► (injected)                                       │
  │                                                                         │
  │  Fluent Builder (modern, no Director):                                  │
  │  Product.builder()                                                      │
  │         .field1("value")                                                │
  │         .field2(42)                                                     │
  │         .build()            ← validates at build time                   │
  └─────────────────────────────────────────────────────────────────────────┘

  When to use:
    → Constructor has many optional parameters (> 4 is a code smell)
    → Object construction involves multiple steps or validation
    → Want to enforce invariants at construction time

  When NOT to use:
    → Simple objects with few fields → direct constructor or record/dataclass
    → Immutability not needed → just use setters
```

```java
// Java — Fluent Builder (Lombok @Builder does this automatically)
public final class HttpRequest {
    private final String method;
    private final URI uri;
    private final Map<String, String> headers;
    private final Duration timeout;
    private final int maxRetries;

    private HttpRequest(Builder builder) {
        this.method = Objects.requireNonNull(builder.method, "method required");
        this.uri = Objects.requireNonNull(builder.uri, "uri required");
        this.headers = Map.copyOf(builder.headers);
        this.timeout = builder.timeout;
        this.maxRetries = builder.maxRetries;
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String method;
        private URI uri;
        private final Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(30);
        private int maxRetries = 3;

        public Builder method(String method) { this.method = method; return this; }
        public Builder uri(URI uri) { this.uri = uri; return this; }
        public Builder header(String key, String value) {
            this.headers.put(key, value); return this;
        }
        public Builder timeout(Duration timeout) { this.timeout = timeout; return this; }
        public Builder maxRetries(int max) { this.maxRetries = max; return this; }

        public HttpRequest build() { return new HttpRequest(this); }
    }
}

// Usage:
var request = HttpRequest.builder()
    .method("POST")
    .uri(URI.create("https://api.example.com/orders"))
    .header("Authorization", "Bearer " + token)
    .timeout(Duration.ofSeconds(10))
    .build();
```

```python
# Python — dataclass with validation
from dataclasses import dataclass, field
from typing import Optional

@dataclass(frozen=True)  # immutable
class HttpRequest:
    method: str
    url: str
    headers: dict = field(default_factory=dict)
    timeout_seconds: int = 30
    max_retries: int = 3

    def __post_init__(self):
        if self.method not in ("GET", "POST", "PUT", "PATCH", "DELETE"):
            raise ValueError(f"Invalid method: {self.method}")
        if self.timeout_seconds <= 0:
            raise ValueError("timeout must be positive")

# Python doesn't need a Builder class — dataclasses with defaults
# and __post_init__ validation serve the same purpose.
```

```rust
// Rust — Builder with type-state (compile-time validation)
struct HttpRequest<State> {
    method: Option<String>,
    url: Option<String>,
    _state: std::marker::PhantomData<State>,
}

struct NoUrl;
struct HasUrl;

impl HttpRequest<NoUrl> {
    fn new() -> Self { Self { method: None, url: None, _state: PhantomData } }
    
    fn url(self, url: &str) -> HttpRequest<HasUrl> {
        HttpRequest { method: self.method, url: Some(url.to_string()), _state: PhantomData }
    }
}

impl HttpRequest<HasUrl> {
    fn build(self) -> Request {
        Request { method: self.method.unwrap_or("GET".into()), url: self.url.unwrap() }
    }
}

// Compile error if .url() not called before .build()
let req = HttpRequest::new().url("https://...").build();
```

---

### Singleton

```
  Intent: Ensure only one instance of a class exists and provide a
          global access point to it.

  ┌────────────────────────────────┐
  │          Singleton             │
  │  - instance: Singleton         │  ← static, private
  │  - Singleton()                 │  ← private constructor
  │  + getInstance(): Singleton    │  ← static, public
  │  + businessMethod()            │
  └────────────────────────────────┘

  When it's justified:
    → Logger (one per application)
    → Configuration holder (read at startup, immutable)
    → Connection pool manager (manages shared resources)
    → Registry / service locator
    → Hardware access (one instance controls the device)

  When it's an anti-pattern:
    → Encapsulates mutable shared state → global mutable state is evil
    → Makes testing hard (can't inject mocks)
    → Violates Single Responsibility (manages own lifecycle + does work)
    → In tests: state leaks between test cases

  Better alternative: Dependency Injection
    Register as singleton in DI container → injectable, mockable
    Spring @Bean + @Scope("singleton")
    Python: single instance at module level, injected via function args
```

```java
// Java — thread-safe Singleton with double-checked locking
public final class DatabasePool {
    private static volatile DatabasePool instance;  // volatile essential

    private final ConnectionPool pool;

    private DatabasePool() {
        this.pool = ConnectionPool.create(Config.load());
    }

    public static DatabasePool getInstance() {
        if (instance == null) {                        // first check (no lock)
            synchronized (DatabasePool.class) {
                if (instance == null) {                // second check (with lock)
                    instance = new DatabasePool();
                }
            }
        }
        return instance;
    }
}

// BETTER: Let Spring manage it
@Configuration
public class DatabaseConfig {
    @Bean  // Spring manages singleton lifecycle
    public ConnectionPool connectionPool() {
        return ConnectionPool.create(config());
    }
}
// Now it's injectable and mockable in tests
```

```kotlin
// Kotlin — object declaration (language-level singleton)
object DatabasePool {
    private val pool = ConnectionPool.create(Config.load())

    fun getConnection() = pool.borrow()
    fun releaseConnection(conn: Connection) = pool.release(conn)
}

// Usage: DatabasePool.getConnection() — thread-safe, lazy init
// For testability, still prefer injecting an interface
```

---

### Prototype

```
  Intent: Create new objects by cloning existing ones.
          Avoid expensive initialization by copying a configured prototype.

  When to use:
    → Object creation is expensive (DB query, API call, heavy computation)
    → Objects differ only slightly (cloning + modifying is cheaper)
    → Building object graphs where structure matters more than class

  Java: implement Cloneable (shallow) or copy constructor (deep)
  Python: copy.copy() (shallow) / copy.deepcopy() (deep)
  Kotlin: data class .copy()

  Kotlin example:
    data class QueryConfig(
        val database: String,
        val timeout: Duration = Duration.ofSeconds(30),
        val maxRows: Int = 1000,
        val readOnly: Boolean = true
    )

    val baseQuery = QueryConfig(database = "orders_db")
    val longQuery = baseQuery.copy(timeout = Duration.ofMinutes(5), maxRows = 50000)
    val writeQuery = baseQuery.copy(readOnly = false)
    // Original unchanged — value semantics make this safe
```

---

## Structural Patterns

### Adapter

```
  Intent: Convert the interface of a class into another interface
          clients expect. Lets incompatible interfaces work together.

  Object Adapter (composition — preferred):
  ┌────────────┐     uses     ┌────────────┐    delegates   ┌──────────────┐
  │  Client    │ ──────────── │  Adapter   │ ─────────────► │  Adaptee     │
  │            │              │            │                 │  (legacy /   │
  │ expects:   │              │ implements │                 │   external)  │
  │ Target     │              │ Target     │                 │              │
  └────────────┘              │ wraps      │                 └──────────────┘
                              │ Adaptee    │
                              └────────────┘

  When to use:
    → Integrating a third-party library with a different interface
    → Wrapping legacy code behind a modern interface
    → Making classes work together that couldn't due to incompatible interfaces

  When NOT to use:
    → Interface is only slightly different — just rename
    → You own both sides — change the interface instead
```

```java
// Java — Adapt legacy payment processor to modern interface
interface PaymentGateway {
    PaymentResult charge(String customerId, Money amount, String currency);
}

// Legacy class we can't modify (third-party SDK)
class LegacyStripeClient {
    public StripeCharge createCharge(String tok, int amountCents, String cur) { ... }
}

// Adapter
class StripePaymentAdapter implements PaymentGateway {
    private final LegacyStripeClient stripe;

    StripePaymentAdapter(LegacyStripeClient stripe) {
        this.stripe = stripe;
    }

    @Override
    public PaymentResult charge(String customerId, Money amount, String currency) {
        var charge = stripe.createCharge(
            customerId,
            amount.toMinorUnits(),  // convert Money → int cents
            currency
        );
        return PaymentResult.fromStripeCharge(charge);  // convert result
    }
}
```

```python
# Python — Adapter using composition
class PaymentGateway(Protocol):
    def charge(self, customer_id: str, amount: Decimal, currency: str) -> PaymentResult: ...

class LegacyStripeClient:  # can't modify
    def create_charge(self, token: str, amount_cents: int, currency: str): ...

class StripePaymentAdapter:
    def __init__(self, stripe: LegacyStripeClient):
        self._stripe = stripe

    def charge(self, customer_id: str, amount: Decimal, currency: str) -> PaymentResult:
        charge = self._stripe.create_charge(
            token=customer_id,
            amount_cents=int(amount * 100),
            currency=currency
        )
        return PaymentResult(success=True, transaction_id=charge.id)
```

---

### Bridge

```
  Intent: Separate abstraction from implementation. Both can vary independently.

  ┌──────────────────────────────────────────────────────────────────┐
  │  WITHOUT Bridge: class explosion                                  │
  │    Shape × Color = classes needed                                 │
  │    Circle, Square, Triangle × Red, Green, Blue = 9 classes       │
  │                                                                  │
  │  WITH Bridge: composition                                         │
  │    Shape (abstraction) has a Color (implementation)              │
  │    3 Shapes + 3 Colors = 6 classes                               │
  └──────────────────────────────────────────────────────────────────┘

  ┌─────────────────────┐         ┌─────────────────────────┐
  │    Shape            │ has-a ► │    Renderer             │
  │  + draw()           │         │  + renderCircle(r)      │
  │  # renderer:Renderer│         │  + renderSquare(s)      │
  └──────────┬──────────┘         └────────────┬────────────┘
             △                                 △
      ┌──────┴──────┐                   ┌──────┴──────┐
   Circle        Square              VectorRenderer  RasterRenderer

  When to use:
    → Avoid permanent binding between abstraction and implementation
    → Both abstraction and implementation should be extensible via subclasses
    → Hide implementation from clients completely
    → Share implementation among multiple objects

  When NOT to use:
    → Simple hierarchy without multiple variation axes
    → Implementation won't change — adds indirection for no benefit
```

```kotlin
// Kotlin — Message + Delivery Channel Bridge
interface MessageChannel {
    fun send(content: String, recipient: String)
}

class EmailChannel : MessageChannel {
    override fun send(content: String, recipient: String) { /* send email */ }
}

class SlackChannel : MessageChannel {
    override fun send(content: String, recipient: String) { /* post to Slack */ }
}

abstract class Message(protected val channel: MessageChannel) {
    abstract fun deliver(recipient: String)
}

class AlertMessage(channel: MessageChannel, private val severity: String) : Message(channel) {
    override fun deliver(recipient: String) {
        channel.send("[${severity.uppercase()}] Alert triggered", recipient)
    }
}

class ReportMessage(channel: MessageChannel, private val period: String) : Message(channel) {
    override fun deliver(recipient: String) {
        channel.send("Weekly report for $period: ...", recipient)
    }
}

// Usage — mix and match independently
val alert = AlertMessage(SlackChannel(), "critical")
val report = ReportMessage(EmailChannel(), "2024-W03")
```

---

### Composite

```
  Intent: Compose objects into tree structures to represent part-whole
          hierarchies. Treat individual objects and compositions uniformly.

  ┌──────────────────────────────────────────────────────────────┐
  │   Component (interface)                                      │
  │   + operation()                                              │
  │   + add(Component)                                           │
  │   + remove(Component)                                        │
  └──────────────┬────────────────────────────┬─────────────────┘
                 △                            △
          ┌──────┴──────┐            ┌────────┴────────┐
          │    Leaf      │            │   Composite     │
          │ + operation()│            │ - children[]    │
          │              │            │ + operation()   │
          └──────────────┘            │   (iterates)    │
                                      └─────────────────┘

  Classic example: file system (File = Leaf, Directory = Composite)

  When to use:
    → Tree structures: file systems, org charts, UI component trees, DOM
    → Clients should ignore difference between compositions and individual objects

  When NOT to use:
    → Flat structure — just use a list
    → Operations differ significantly between leaf and composite
```

```java
// Java — UI Component tree
interface UIComponent {
    void render(int depth);
    int getWidth();
}

class Button implements UIComponent {
    private final String label;
    Button(String label) { this.label = label; }

    public void render(int depth) {
        System.out.println(" ".repeat(depth) + "[Button: " + label + "]");
    }
    public int getWidth() { return label.length() + 4; }
}

class Panel implements UIComponent {
    private final List<UIComponent> children = new ArrayList<>();

    public void add(UIComponent c) { children.add(c); }

    public void render(int depth) {
        System.out.println(" ".repeat(depth) + "[Panel]");
        children.forEach(c -> c.render(depth + 2));
    }

    public int getWidth() {
        return children.stream().mapToInt(UIComponent::getWidth).max().orElse(0);
    }
}

// Client treats Panel and Button identically
UIComponent root = new Panel();
root.add(new Button("Submit"));
UIComponent nested = new Panel();
nested.add(new Button("Cancel"));
nested.add(new Button("Back"));
root.add(nested);
root.render(0);
```

---

### Decorator

```
  Intent: Attach additional responsibilities to an object dynamically.
          Decorators provide a flexible alternative to subclassing for
          extending functionality.

  ┌─────────────────────────────────────────────────────────────────────┐
  │   Component (interface) ◄──────────────────────────────────────┐   │
  │   + operation(): String                                         │   │
  │         △                                                       │   │
  │    ┌────┴────┐                  ┌─────────────────────────┐    │   │
  │    │Concrete │                  │   BaseDecorator          │    │   │
  │    │Component│                  │   # wrapped: Component ──┘    │   │
  │    └─────────┘                  │   + operation() {             │   │
  │                                 │       wrapped.operation()     │   │
  │                                 │   }                           │   │
  │                                 └──────────────┬───────────────┘   │
  │                                                △                   │
  │                                    ┌───────────┴─────────┐         │
  │                               LoggingDecorator      CachingDecorator│
  └─────────────────────────────────────────────────────────────────────┘

  When to use:
    → Add behavior to individual objects without affecting others
    → Behavior can be withdrawn
    → Extension by subclassing is impractical (too many combinations)

  When NOT to use:
    → Order of decoration matters and is hard to get right
    → Too many thin decorator layers → debugging nightmare
    → Python: prefer Mixin or functools.wraps for simple cases
```

```java
// Java — HTTP Client decorators
interface HttpClient {
    Response execute(Request request);
}

class RealHttpClient implements HttpClient {
    public Response execute(Request request) { /* actual HTTP */ }
}

class LoggingHttpClient implements HttpClient {
    private final HttpClient wrapped;
    LoggingHttpClient(HttpClient wrapped) { this.wrapped = wrapped; }

    public Response execute(Request request) {
        log.info("HTTP {} {}", request.method(), request.uri());
        var start = System.currentTimeMillis();
        var response = wrapped.execute(request);
        log.info("HTTP {} {} → {} in {}ms",
            request.method(), request.uri(),
            response.status(), System.currentTimeMillis() - start);
        return response;
    }
}

class RetryingHttpClient implements HttpClient {
    private final HttpClient wrapped;
    private final int maxRetries;

    public Response execute(Request request) {
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return wrapped.execute(request);
            } catch (TransientException e) {
                if (attempt == maxRetries) throw e;
                Thread.sleep(exponentialBackoff(attempt));
            }
        }
        throw new IllegalStateException("unreachable");
    }
}

// Compose decorators:
HttpClient client = new LoggingHttpClient(
                        new RetryingHttpClient(
                            new RealHttpClient(), 3));
```

```python
# Python — Decorator pattern vs Python decorators
# Python's @decorator syntax is the Decorator pattern for functions

import functools
import time

def retry(max_attempts: int = 3, exceptions: tuple = (Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(2 ** attempt)
        return wrapper
    return decorator

def log_calls(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        logger.info(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        logger.info(f"{func.__name__} returned")
        return result
    return wrapper

# Stack decorators — order matters (outermost applies last)
@retry(max_attempts=3, exceptions=(ConnectionError,))
@log_calls
def call_external_api(endpoint: str) -> dict:
    ...
```

---

### Facade

```
  Intent: Provide a simplified interface to a complex subsystem.

  ┌────────────────────────────────────────────────────────────────┐
  │  Client                                                        │
  │    │                                                           │
  │    └──► Facade ──► SubsystemA ──► SubsystemB ──► SubsystemC  │
  │           (simple interface)    (complex internals hidden)     │
  └────────────────────────────────────────────────────────────────┘

  When to use:
    → Provide a simple interface to a complex subsystem
    → Layer your subsystem — facade defines entry point to each layer
    → Reduce dependencies clients have on internals

  Example: OrderFacade that coordinates Inventory, Payment, Fulfillment, Email
```

```java
// Java — Order placement facade
@Service
public class OrderPlacementFacade {
    // Subsystems — complex, with many methods
    private final InventoryService inventory;
    private final PaymentService payment;
    private final FulfillmentService fulfillment;
    private final NotificationService notification;

    public OrderConfirmation placeOrder(PlaceOrderCommand cmd) {
        // Simple interface hides the coordination complexity
        inventory.reserve(cmd.items());
        var charge = payment.charge(cmd.customerId(), cmd.total());
        var shipment = fulfillment.schedule(cmd.items(), cmd.address());
        notification.sendConfirmation(cmd.customerId(), shipment.trackingId());

        return OrderConfirmation.of(charge.id(), shipment.trackingId());
    }
}
// Client calls one method. Doesn't know about 4 subsystems.
```

---

### Proxy

```
  Intent: Provide a surrogate or placeholder for another object to
          control access to it.

  ┌─────────────────────────────────────────────────────────────┐
  │  Proxy types:                                               │
  │                                                             │
  │  Virtual Proxy   → lazy initialization of expensive object  │
  │  Remote Proxy    → local representative for remote object   │
  │                    (gRPC stub, REST client)                 │
  │  Protection Proxy → access control / authorization check   │
  │  Caching Proxy   → cache results of expensive operations    │
  │  Logging Proxy   → log calls before forwarding             │
  └─────────────────────────────────────────────────────────────┘

  Same interface as real subject:
  ┌────────────┐    ┌──────────────────────────────┐
  │  Client    │    │  Proxy                        │
  │            │    │  - realSubject: Subject        │
  │ uses:      │    │  + operation() {               │
  │ Subject    │───►│      // pre-processing         │
  │ interface  │    │      realSubject.operation()   │
  └────────────┘    │      // post-processing        │
                    │  }                             │
                    └──────────────────────────────┘

  When to use:
    → Lazy initialization (virtual proxy)
    → Access control (protection proxy)
    → Local execution of remote service (remote proxy)
    → Logging and caching without modifying real subject

  When NOT to use:
    → Response time matters — every proxy adds a layer
    → Proxy logic becomes more complex than real subject
```

```python
# Python — Caching Proxy
class ImageLoader:
    def load(self, path: str) -> bytes:
        # Expensive disk/network operation
        with open(path, 'rb') as f:
            return f.read()

class CachingImageProxy:
    def __init__(self, loader: ImageLoader):
        self._loader = loader
        self._cache: dict[str, bytes] = {}

    def load(self, path: str) -> bytes:
        if path not in self._cache:
            self._cache[path] = self._loader.load(path)
        return self._cache[path]

# Python's @property and __getattr__ enable transparent proxies
```

---

## Behavioral Patterns

### Strategy

```
  Intent: Define a family of algorithms, encapsulate each one, and
          make them interchangeable. Strategy lets the algorithm vary
          independently from clients that use it.

  ┌─────────────────────────────────────────────────────────────┐
  │  Context ──────────────────────────────► Strategy           │
  │  - strategy: Strategy                   + execute()         │
  │  + setStrategy(s)                            △              │
  │  + executeStrategy()                ┌────────┴────────┐     │
  │      └── strategy.execute()      StrategyA         StrategyB│
  └─────────────────────────────────────────────────────────────┘

  When to use:
    → Multiple related classes differ only in behavior
    → Need different variants of an algorithm
    → Algorithm uses data the client shouldn't know about
    → Class defines many behaviors via conditionals — extract each branch

  When NOT to use:
    → Few strategies that never change — just use if/switch
    → Language has first-class functions — just pass a function
```

```java
// Java — Sorting strategy
interface SortStrategy<T> {
    List<T> sort(List<T> items, Comparator<T> comparator);
}

class MergeSortStrategy<T> implements SortStrategy<T> {
    public List<T> sort(List<T> items, Comparator<T> comparator) { /* merge sort */ }
}

class QuickSortStrategy<T> implements SortStrategy<T> {
    public List<T> sort(List<T> items, Comparator<T> comparator) { /* quicksort */ }
}

class Sorter<T> {
    private SortStrategy<T> strategy;

    public Sorter(SortStrategy<T> strategy) { this.strategy = strategy; }
    public void setStrategy(SortStrategy<T> s) { this.strategy = s; }

    public List<T> sort(List<T> items, Comparator<T> cmp) {
        return strategy.sort(items, cmp);
    }
}
```

```python
# Python — Strategy with callables (simpler than classes)
from typing import Callable, TypeVar
T = TypeVar('T')

# Strategy IS just a function — no wrapper class needed
def sort_with(
    items: list[T],
    comparator: Callable[[T, T], int],
    strategy: Callable[[list[T], Callable], list[T]]  # ← the strategy
) -> list[T]:
    return strategy(items, comparator)

# Strategies are plain functions
def merge_sort(items, comparator): ...
def quick_sort(items, comparator): ...

result = sort_with(orders, compare_by_date, merge_sort)
# Swap strategy by passing different function
result = sort_with(orders, compare_by_date, quick_sort)
```

---

### Observer

```
  Intent: Define one-to-many dependency so that when one object changes
          state, all its dependents are notified and updated automatically.

  ┌──────────────────────────────────────────────────────────────────┐
  │  Subject (Observable)          Observer                          │
  │  - observers: List             + update(event)                   │
  │  + subscribe(Observer)               △                           │
  │  + unsubscribe(Observer)   ┌─────────┴──────────┐               │
  │  + notify(event)      Observer1            Observer2             │
  └──────────────────────────────────────────────────────────────────┘

  When to use:
    → Object needs to notify unknown number of other objects
    → Change in one object requires changing others; don't know how many
    → Event-driven systems; UI frameworks

  When NOT to use:
    → Observers form a chain → unexpected cascades
    → Order of notification matters and is not guaranteed
    → Memory leaks: observers not removed → subject holds reference
```

```kotlin
// Kotlin — Event bus (simplified Observer)
data class OrderCreatedEvent(val orderId: String, val customerId: String)

interface EventHandler<T> {
    fun handle(event: T)
}

class EventBus {
    private val handlers = mutableMapOf<Class<*>, MutableList<EventHandler<*>>>()

    fun <T : Any> subscribe(eventType: Class<T>, handler: EventHandler<T>) {
        handlers.getOrPut(eventType) { mutableListOf() }.add(handler)
    }

    @Suppress("UNCHECKED_CAST")
    fun <T : Any> publish(event: T) {
        handlers[event::class.java]?.forEach { handler ->
            (handler as EventHandler<T>).handle(event)
        }
    }
}

// Usage
val bus = EventBus()
bus.subscribe(OrderCreatedEvent::class.java) { event ->
    emailService.sendConfirmation(event.customerId)
}
bus.subscribe(OrderCreatedEvent::class.java) { event ->
    analyticsService.track("order_created", event.orderId)
}

bus.publish(OrderCreatedEvent("ord_123", "cust_456"))
// Both handlers called
```

---

### Command

```
  Intent: Encapsulate a request as an object, thereby letting you
          parameterize clients with different requests, queue them,
          log them, or support undo/redo.

  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  Invoker ──────────────────► Command (interface)                  │
  │  - command: Command            + execute()                        │
  │  + setCommand(c)               + undo()                           │
  │  + executeCommand()                 △                             │
  │                           ┌────────┴──────────┐                  │
  │                      ConcreteCmd1         ConcreteCmd2            │
  │                      - receiver: R         state for undo         │
  │                      + execute()                                  │
  │                      + undo()                                     │
  └────────────────────────────────────────────────────────────────────┘

  When to use:
    → Parameterize objects with operations
    → Queue, log, and execute requests at different times
    → Implement undo/redo
    → Transaction-like bundles of operations

  When NOT to use:
    → Simple direct calls suffice
    → No need for undo/redo, queuing, or logging
```

```java
// Java — Command with undo for text editor
interface TextCommand {
    void execute();
    void undo();
}

class InsertTextCommand implements TextCommand {
    private final TextBuffer buffer;
    private final int position;
    private final String text;

    InsertTextCommand(TextBuffer buffer, int position, String text) {
        this.buffer = buffer;
        this.position = position;
        this.text = text;
    }

    public void execute() { buffer.insert(position, text); }
    public void undo()    { buffer.delete(position, position + text.length()); }
}

class CommandHistory {
    private final Deque<TextCommand> history = new ArrayDeque<>();
    private final Deque<TextCommand> undone = new ArrayDeque<>();

    public void execute(TextCommand cmd) {
        cmd.execute();
        history.push(cmd);
        undone.clear();  // new command clears redo stack
    }

    public void undo() {
        if (!history.isEmpty()) {
            var cmd = history.pop();
            cmd.undo();
            undone.push(cmd);
        }
    }

    public void redo() {
        if (!undone.isEmpty()) {
            var cmd = undone.pop();
            cmd.execute();
            history.push(cmd);
        }
    }
}
```

---

### Chain of Responsibility

```
  Intent: Avoid coupling sender of request to receiver. Give multiple
          objects a chance to handle the request. Chain the objects and
          pass request along chain until handled.

  ┌──────────────────────────────────────────────────────────────────┐
  │  Request ──► Handler1 ──► Handler2 ──► Handler3 ──► (unhandled) │
  │              handles?      handles?     handles?                  │
  │              yes: stop     no: pass on  yes: stop                 │
  └──────────────────────────────────────────────────────────────────┘

  Modern form: middleware pipeline (Express, Spring interceptors, Django middleware)

  When to use:
    → More than one object may handle a request, handler determined at runtime
    → Set of handlers should be dynamic
    → Middleware pipelines: auth, logging, rate limiting, caching

  When NOT to use:
    → Request must be handled — no guarantee with pure chain
    → Performance critical: chain traversal overhead
```

```python
# Python — Middleware pipeline (Flask/FastAPI style)
from typing import Callable

Handler = Callable[[dict], dict]  # request → response
Middleware = Callable[[dict, Handler], dict]

def auth_middleware(request: dict, next_handler: Handler) -> dict:
    if not request.get("headers", {}).get("Authorization"):
        return {"status": 401, "body": "Unauthorized"}
    return next_handler(request)

def logging_middleware(request: dict, next_handler: Handler) -> dict:
    print(f"→ {request['method']} {request['path']}")
    response = next_handler(request)
    print(f"← {response['status']}")
    return response

def rate_limit_middleware(request: dict, next_handler: Handler) -> dict:
    if is_rate_limited(request):
        return {"status": 429, "body": "Too Many Requests"}
    return next_handler(request)

def compose_middleware(*middlewares: Middleware) -> Callable[[Handler], Handler]:
    def decorator(handler: Handler) -> Handler:
        for middleware in reversed(middlewares):
            handler = lambda req, mw=middleware, h=handler: mw(req, h)
        return handler
    return decorator

# Assemble the pipeline
pipeline = compose_middleware(
    logging_middleware,
    auth_middleware,
    rate_limit_middleware,
)(actual_handler)

response = pipeline(request)
```

---

### Template Method

```
  Intent: Define the skeleton of an algorithm in a base class,
          deferring some steps to subclasses.

  ┌────────────────────────────────────────────────────────────┐
  │  AbstractClass                                             │
  │  + templateMethod() {   ← final, defines algorithm order  │
  │      step1()            ← implemented in base             │
  │      step2()            ← abstract, subclass implements   │
  │      step3()            ← hook (empty default)            │
  │  }                                                         │
  │  # step2(): abstract                                       │
  │  # step3() {}  ← hook                                     │
  └────────────────────────────────────────────────────────────┘

  When to use:
    → Multiple classes share the same algorithm structure
    → Only specific steps differ between variants
    → Framework providing extension points for customization

  When NOT to use:
    → Strategy is more flexible — prefer composition over inheritance
    → Too many abstract steps → complex class hierarchy

  Python alternative: pass hook functions as arguments (avoid inheritance)
```

```java
// Java — Data export template
abstract class DataExporter {
    // Template method — final: don't override the algorithm
    public final void export(String destination) {
        var data = fetchData();        // implemented here (common)
        var validated = validate(data); // abstract — subclass decides
        var formatted = format(validated); // abstract — CSV or JSON
        write(formatted, destination); // implemented here (common)
        onComplete(destination);       // hook — optional override
    }

    protected abstract List<Record> validate(List<Record> data);
    protected abstract String format(List<Record> data);

    private List<Record> fetchData() { return repository.findAll(); }
    private void write(String content, String dest) { Files.write(dest, content); }
    protected void onComplete(String dest) {}  // hook: no-op default
}

class CsvExporter extends DataExporter {
    protected List<Record> validate(List<Record> data) {
        return data.stream().filter(r -> r.isValid()).toList();
    }
    protected String format(List<Record> data) {
        return CsvSerializer.serialize(data);
    }
    @Override
    protected void onComplete(String dest) {
        log.info("CSV export complete: {}", dest);
    }
}
```

---

### State

```
  Intent: Allow an object to alter its behavior when its internal state
          changes. The object will appear to change its class.

  ┌────────────────────────────────────────────────────────────────────┐
  │  Context ──────────────────────────────► State (interface)        │
  │  - state: State                          + handle(context)        │
  │  + setState(s)                                △                   │
  │  + request() {                    ┌───────────┼───────────┐       │
  │      state.handle(this)         Pending   Confirmed   Shipped     │
  │  }                                                                 │
  └────────────────────────────────────────────────────────────────────┘

  When to use:
    → Object's behavior depends on its state
    → Operations have large multi-part conditionals on object state
    → State transitions are complex and numerous

  When NOT to use:
    → Few states with simple transitions — use enum + switch
    → States share most behavior — excessive duplication
```

```kotlin
// Kotlin — Order state machine
interface OrderState {
    fun confirm(order: Order)
    fun ship(order: Order)
    fun cancel(order: Order)
    fun getStatus(): String
}

class PendingState : OrderState {
    override fun confirm(order: Order) {
        // validate, charge, transition
        order.setState(ConfirmedState())
    }
    override fun ship(order: Order) = error("Cannot ship unconfirmed order")
    override fun cancel(order: Order) { order.setState(CancelledState()) }
    override fun getStatus() = "PENDING"
}

class ConfirmedState : OrderState {
    override fun confirm(order: Order) = error("Already confirmed")
    override fun ship(order: Order) {
        fulfillmentService.dispatch(order)
        order.setState(ShippedState())
    }
    override fun cancel(order: Order) {
        paymentService.refund(order)
        order.setState(CancelledState())
    }
    override fun getStatus() = "CONFIRMED"
}

class Order(private var state: OrderState = PendingState()) {
    fun setState(s: OrderState) { state = s }
    fun confirm() = state.confirm(this)
    fun ship() = state.ship(this)
    fun cancel() = state.cancel(this)
    fun getStatus() = state.getStatus()
}
```

---

### Iterator

```
  Intent: Provide a way to sequentially access elements of an aggregate
          object without exposing its underlying representation.

  Most modern languages have this built-in:
    Java: Iterable<T> / Iterator<T> / for-each
    Python: __iter__ / __next__ / yield
    Kotlin: Sequence<T>, operator fun iterator()
    Rust: Iterator trait

  When to implement custom Iterator:
    → Traverse complex data structures (trees, graphs)
    → Lazy evaluation of large datasets
    → Multiple simultaneous traversals of same collection
```

```python
# Python — Lazy tree traversal iterator
class TreeNode:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []

class DepthFirstIterator:
    def __init__(self, root: TreeNode):
        self._stack = [root]

    def __iter__(self):
        return self

    def __next__(self):
        if not self._stack:
            raise StopIteration
        node = self._stack.pop()
        self._stack.extend(reversed(node.children))
        return node.value

# Generator version (simpler, Pythonic)
def depth_first(node: TreeNode):
    yield node.value
    for child in node.children:
        yield from depth_first(child)

# Usage — same interface
for value in depth_first(root):
    process(value)
```

---

## Modern / Functional Patterns

### Monad: Result/Either for Error Handling

```
  Problem: exceptions break referential transparency, mix error and value,
           require try/catch at every level.

  Solution: Result<T, E> type — value OR error, never both, never exception

  ┌─────────────────────────────────────────────────────────────────┐
  │  Result<T, E>                                                   │
  │    Ok(value: T)     — success path                              │
  │    Err(error: E)    — failure path                              │
  │                                                                 │
  │  map(fn):      transforms Ok value, passes Err through          │
  │  flatMap(fn):  chains Result-returning operations               │
  │  mapError(fn): transforms Err, passes Ok through               │
  └─────────────────────────────────────────────────────────────────┘
```

```kotlin
// Kotlin — Result monad (stdlib + custom)
sealed class Result<out T, out E> {
    data class Ok<T>(val value: T) : Result<T, Nothing>()
    data class Err<E>(val error: E) : Result<Nothing, E>()

    fun <R> map(transform: (T) -> R): Result<R, E> = when (this) {
        is Ok -> Ok(transform(value))
        is Err -> this
    }

    fun <R> flatMap(transform: (T) -> Result<R, E>): Result<R, E> = when (this) {
        is Ok -> transform(value)
        is Err -> this
    }
}

// Domain errors as sealed class
sealed class OrderError {
    object InsufficientStock : OrderError()
    object PaymentFailed : OrderError()
    data class ValidationError(val field: String, val message: String) : OrderError()
}

// No exceptions — errors are values
fun createOrder(cmd: CreateOrderCommand): Result<Order, OrderError> {
    val validated = validate(cmd)
        .flatMap { v -> inventory.reserve(v.items) }
        .flatMap { reserved -> payment.charge(reserved.total) }
        .map { charge -> Order.create(cmd, charge) }

    return validated
}

// Caller handles errors explicitly
when (val result = createOrder(cmd)) {
    is Result.Ok -> respond(201, result.value)
    is Result.Err -> when (result.error) {
        is OrderError.InsufficientStock -> respond(422, "Out of stock")
        is OrderError.PaymentFailed -> respond(402, "Payment failed")
        is OrderError.ValidationError -> respond(400, result.error.message)
    }
}
```

```rust
// Rust — Result is the language idiom
fn process_order(cmd: CreateOrderCommand) -> Result<Order, OrderError> {
    let validated = validate(&cmd)?;         // ? = early return Err
    let reserved = inventory.reserve(validated.items)?;
    let charge = payment.charge(reserved.total)?;
    Ok(Order::create(cmd, charge))
}

// Combinators
let result = process_order(cmd)
    .map(|order| order.with_tracking_number(generate_tracking()))
    .map_err(|e| format!("Order failed: {:?}", e));
```

---

### Pipeline

```
  Intent: Compose a sequence of transformations. Each step receives
          the output of the previous step. Unix pipe philosophy in code.

  data → step1 → step2 → step3 → result

  When to use:
    → Data transformation / ETL
    → Request/response middleware
    → Build systems, data validation chains
    → Composable business rules
```

```python
# Python — Functional pipeline with composition
from functools import reduce
from typing import Callable, TypeVar

T = TypeVar('T')

def pipe(*functions: Callable) -> Callable:
    """Compose functions left to right: pipe(f, g, h)(x) == h(g(f(x)))"""
    return reduce(lambda f, g: lambda x: g(f(x)), functions)

# Each step is a pure function
def parse_order(raw: dict) -> Order: ...
def validate_order(order: Order) -> Order: ...
def enrich_with_prices(order: Order) -> Order: ...
def apply_discounts(order: Order) -> Order: ...

process_order = pipe(
    parse_order,
    validate_order,
    enrich_with_prices,
    apply_discounts,
)

result = process_order(raw_input)

# With Result monad for error handling
def safe_pipe(*functions):
    def apply(value):
        result = Result.Ok(value)
        for fn in functions:
            result = result.flatMap(lambda v, f=fn: safe_call(f, v))
        return result
    return apply
```

```java
// Java — Stream pipeline (built-in)
orders.stream()
    .filter(o -> o.getStatus() == PENDING)
    .map(this::enrichWithCustomer)
    .map(this::applyDiscounts)
    .filter(o -> o.getTotal().isGreaterThan(Money.of(100, "USD")))
    .collect(toList());

// Java — CompletableFuture pipeline (async)
CompletableFuture.supplyAsync(() -> fetchOrder(orderId))
    .thenApply(this::validate)
    .thenCompose(order -> inventoryService.reserve(order.items()))
    .thenCompose(reserved -> paymentService.charge(reserved.total()))
    .thenApply(charge -> Order.create(charge))
    .exceptionally(e -> handleError(e));
```

---

### Specification

```
  Intent: Encapsulate business rules as composable predicates.
          Combine with AND, OR, NOT. Rules are first-class objects.

  When to use:
    → Complex, reusable business rules
    → Rules need to be combined in different ways
    → Rules need to be stored, serialized, or displayed
    → Domain-driven design: query specifications for repositories
```

```java
// Java — Order eligibility specifications
interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate)
                         && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate)
                         || other.isSatisfiedBy(candidate);
    }

    default Specification<T> not() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }
}

class PremiumCustomerSpec implements Specification<Customer> {
    public boolean isSatisfiedBy(Customer c) {
        return c.getTier() == CustomerTier.PREMIUM;
    }
}

class LargeOrderSpec implements Specification<Order> {
    private final Money threshold;
    public boolean isSatisfiedBy(Order o) {
        return o.getTotal().isGreaterThan(threshold);
    }
}

class HighValueOrderSpec implements Specification<Order> {
    private final CustomerRepository customers;

    public boolean isSatisfiedBy(Order order) {
        var customer = customers.findById(order.getCustomerId());
        return new PremiumCustomerSpec().isSatisfiedBy(customer)
            || new LargeOrderSpec(Money.of(1000, "USD")).isSatisfiedBy(order);
    }
}

// Compose specs
var eligibleForDiscount = new HighValueOrderSpec(customers)
    .and(new LargeOrderSpec(Money.of(500, "USD")).not().not()); // readable business rules
```

---

## When Patterns Become Anti-Patterns

### Pattern-itis

```
  Signs of over-engineering:
    → Factory for a class with one concrete implementation
    → Singleton that wraps a stateless utility class
    → Strategy with one strategy
    → Observer with one observer that's always the same
    → Builder for a class with two fields
    → Abstract Factory where all products are identical
    → Command pattern for operations that don't need undo/queue

  The Pattern Imposition Test:
    Ask: "What problem is this pattern solving here?"
    If answer is vague ("flexibility", "extensibility") → red flag.
    Patterns should solve a concrete, present problem.

  GoF in the age of FP and modern type systems:
    Many GoF patterns exist to work around OOP limitations.

    Command     → lambda / first-class function
    Strategy    → function parameter
    Template Method → higher-order function
    Iterator    → lazy sequence / generator
    Factory     → factory function
    Observer    → event streams / reactive

    In Python, Kotlin, Rust, Scala: prefer functional style.
    In Java pre-8: GoF patterns were necessary.
    In Java 17+: often a lambda suffices.
```

```java
// BEFORE (Pattern for pattern's sake)
interface SortStrategy {
    List<Integer> sort(List<Integer> list);
}

class BubbleSortStrategy implements SortStrategy {
    public List<Integer> sort(List<Integer> list) { ... }
}

class Sorter {
    private SortStrategy strategy;
    public void setStrategy(SortStrategy s) { this.strategy = s; }
    public List<Integer> sort(List<Integer> list) { return strategy.sort(list); }
}

Sorter sorter = new Sorter();
sorter.setStrategy(new BubbleSortStrategy());
sorter.sort(list);

// AFTER (Java 8+, cleaner)
Function<List<Integer>, List<Integer>> bubbleSort = list -> { ... };
Function<List<Integer>, List<Integer>> quickSort = list -> { ... };

List<Integer> sorted = bubbleSort.apply(list);  // done
```

---

## Practical Checklist

### Choosing a Pattern

- [ ] Identify the recurring problem — don't impose a pattern, discover it
- [ ] Can a simpler solution (function, data class, if/else) solve it?
- [ ] Does the pattern's intent exactly match the problem?
- [ ] Will future teammates recognize this pattern without explanation?

### Applying a Pattern

- [ ] Program to interfaces, not implementations
- [ ] Prefer composition over inheritance (especially for Decorator, Bridge)
- [ ] Each pattern participant has a single responsibility
- [ ] Parameterize behavior, not structure (favor Strategy over Template Method)
- [ ] Use the simplest form that works (fluent Builder over Director+Builder)

### Code Quality

- [ ] Pattern names used in class/interface names (OrderState, PaymentStrategy)
- [ ] No surprise behavior in pattern participants (Liskov Substitution)
- [ ] Test each participant independently (Adapter, Strategy, Command)
- [ ] Document non-obvious patterns with diagram comment

### Anti-Patterns to Avoid

- [ ] NEVER: Singleton for testable logic — use DI
- [ ] NEVER: Abstract Factory when one factory covers all cases
- [ ] NEVER: Template Method when a function parameter suffices
- [ ] NEVER: Command pattern with no need for undo/queue/log
- [ ] NEVER: Observer without cleanup mechanism (memory leaks)
- [ ] NEVER: Specification pattern for rules simpler than `x > 100`
- [ ] NEVER: Composite where all objects are always Leaf (no tree structure)
```
