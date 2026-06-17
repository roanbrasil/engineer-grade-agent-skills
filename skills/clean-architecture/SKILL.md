---
name: clean-architecture
description: Apply Clean Architecture (Uncle Bob) when designing or reviewing system structure — invoked when discussing layers, dependency direction, framework coupling, testability, or when the user asks how to structure a new feature or service.
---

# Clean Architecture

Clean Architecture is Robert C. Martin's synthesis of Hexagonal Architecture, Onion Architecture, and DCI into a single, opinionated structure. Its central claim: the architecture of a system should scream its purpose, not its technology.

A well-structured system should look like a library management system, a payroll processor, or an order fulfillment engine — not "a Spring Boot app" or "a Rails app."

---

## The Core Diagram

```
╔══════════════════════════════════════════════════════════════╗
║  Frameworks & Drivers                                        ║
║  (Web, DB, UI, External Services)                            ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │  Interface Adapters                                    │  ║
║  │  (Controllers, Presenters, Gateways, Repositories)    │  ║
║  │  ┌──────────────────────────────────────────────────┐ │  ║
║  │  │  Use Cases (Application Business Rules)          │ │  ║
║  │  │  ┌────────────────────────────────────────────┐  │ │  ║
║  │  │  │  Entities (Enterprise Business Rules)      │  │ │  ║
║  │  │  │  (Domain Objects, Value Objects)           │  │ │  ║
║  │  │  └────────────────────────────────────────────┘  │ │  ║
║  │  └──────────────────────────────────────────────────┘ │  ║
║  └────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════╝

        Dependencies flow INWARD only. →→→→→→→→→→→→
        Inner circles know NOTHING about outer circles.
```

---

## The Dependency Rule

**The single most important rule:** source code dependencies must always point inward — toward higher-level policy.

Nothing in an inner circle can know anything about something in an outer circle. In particular:
- Entities do not import use cases
- Use cases do not import controllers
- Use cases do not import database frameworks
- The business logic has zero imports from Spring, Django, Rails, SQLAlchemy, Hibernate

```
WRONG direction:                    CORRECT direction:
  Entity ──imports──> Controller      Controller ──imports──> UseCase
  UseCase ──imports──> Database       UseCase ──imports──> Entity
  Entity ──imports──> Framework       Repository (interface) lives in UseCase layer
```

When an inner circle needs to communicate outward — for example, a use case saving to a database — it defines an **interface** (port) that the outer circle implements. Dependency Inversion Principle (DIP) makes the dependency rule possible.

```java
// Use Case layer defines the interface (inner circle)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// Framework layer implements it (outer circle)
public class JpaOrderRepository implements OrderRepository {
    private final JpaOrderJpaRepository jpa;  // Spring Data — stays in outer circle

    @Override
    public void save(Order order) {
        jpa.save(OrderJpaEntity.from(order));
    }
}
```

---

## The Four Layers

### Layer 1: Entities (Enterprise Business Rules)

Entities encapsulate the most general and high-level rules of the business. They are the least likely to change when something external changes — a new web page, a new database, a new framework.

**What belongs here:**
- Domain objects with identity (Order, Customer, Product)
- Value objects (Money, EmailAddress, OrderId)
- Domain rules enforced through invariants
- No framework annotations, no persistence annotations

```java
// Entity — pure Java, no framework dependencies
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderLine> lines;
    private OrderStatus status;

    private Order(OrderId id, CustomerId customerId) {
        this.id = Objects.requireNonNull(id);
        this.customerId = Objects.requireNonNull(customerId);
        this.lines = new ArrayList<>();
        this.status = OrderStatus.DRAFT;
    }

    public static Order create(CustomerId customerId) {
        return new Order(OrderId.generate(), customerId);
    }

    public void addLine(ProductId product, Quantity qty, Money unitPrice) {
        if (status != OrderStatus.DRAFT) {
            throw new OrderNotModifiableException(id, status);
        }
        lines.add(new OrderLine(product, qty, unitPrice));
    }

    public void submit() {
        if (lines.isEmpty()) throw new EmptyOrderException(id);
        this.status = OrderStatus.SUBMITTED;
    }

    public Money total() {
        return lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

### Layer 2: Use Cases (Application Business Rules)

Use cases contain application-specific business rules. They orchestrate the flow of data to and from entities and direct those entities to use their enterprise-wide rules to achieve the goals of the use case.

**What belongs here:**
- One class per use case (or grouped by feature)
- Input ports (interfaces the use case exposes to controllers)
- Output ports (interfaces the use case requires from the outside world)
- No web framework types (HttpRequest, HttpResponse, etc.)
- No database framework types (EntityManager, Session, etc.)

```java
// Input port — what the use case accepts
public interface PlaceOrderUseCase {
    PlaceOrderResponse execute(PlaceOrderCommand command);
}

// Output port — what the use case requires from infrastructure
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentGateway {
    PaymentResult charge(CustomerId customer, Money amount);
}

// Use Case Implementation
public class PlaceOrderInteractor implements PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final CustomerRepository customerRepository;

    public PlaceOrderInteractor(
        OrderRepository orderRepository,
        PaymentGateway paymentGateway,
        CustomerRepository customerRepository
    ) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.customerRepository = customerRepository;
    }

    @Override
    public PlaceOrderResponse execute(PlaceOrderCommand command) {
        Customer customer = customerRepository.findById(command.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(command.customerId()));

        Order order = Order.create(customer.id());
        command.items().forEach(item ->
            order.addLine(item.productId(), item.quantity(), item.unitPrice())
        );
        order.submit();

        PaymentResult payment = paymentGateway.charge(customer.id(), order.total());
        if (!payment.successful()) {
            throw new PaymentDeclinedException(payment.reason());
        }

        orderRepository.save(order);
        return new PlaceOrderResponse(order.id(), order.total());
    }
}
```

### Layer 3: Interface Adapters

Adapters convert data from the format most convenient for use cases and entities to the format most convenient for external agencies (web, database, UI).

**What belongs here:**
- Controllers: convert HTTP request → command, call use case, convert response → HTTP response
- Presenters: format use case output for display
- Repository implementations: convert domain objects to/from database entities
- Data Transfer Objects (request/response models)

```kotlin
// Controller — in the Interface Adapter layer
@RestController
@RequestMapping("/orders")
class PlaceOrderController(
    private val placeOrder: PlaceOrderUseCase
) {
    @PostMapping
    fun placeOrder(@RequestBody request: PlaceOrderRequest): ResponseEntity<PlaceOrderApiResponse> {
        val command = PlaceOrderCommand(
            customerId = CustomerId(request.customerId),
            items = request.items.map { item ->
                OrderItemCommand(
                    productId = ProductId(item.productId),
                    quantity = Quantity(item.quantity),
                    unitPrice = Money(item.unitPrice, Currency.getInstance(item.currency))
                )
            }
        )

        val response = placeOrder.execute(command)

        return ResponseEntity.status(HttpStatus.CREATED)
            .body(PlaceOrderApiResponse(
                orderId = response.orderId.value,
                total = response.total.amount,
                currency = response.total.currency.currencyCode
            ))
    }
}
```

### Layer 4: Frameworks & Drivers

The outermost layer — the glue code that wires everything together. Spring Boot configuration, database configuration, message broker configuration. This layer is intentionally thin.

```java
// Wiring — in the Frameworks layer (Spring @Configuration)
@Configuration
public class OrderModuleConfiguration {

    @Bean
    public PlaceOrderUseCase placeOrderUseCase(
        OrderRepository orderRepository,
        PaymentGateway paymentGateway,
        CustomerRepository customerRepository
    ) {
        return new PlaceOrderInteractor(orderRepository, paymentGateway, customerRepository);
    }

    @Bean
    public OrderRepository orderRepository(JpaOrderRepository jpa) {
        return new JpaOrderRepositoryAdapter(jpa);
    }

    @Bean
    public PaymentGateway paymentGateway(StripeClient stripe) {
        return new StripePaymentGateway(stripe);
    }
}
```

---

## Screaming Architecture

Look at the top-level package structure. What does it scream?

```
BAD — screams "Spring MVC app":
  com.example
  ├── controllers/
  │   ├── UserController.java
  │   └── OrderController.java
  ├── services/
  │   ├── UserService.java
  │   └── OrderService.java
  ├── repositories/
  │   ├── UserRepository.java
  │   └── OrderRepository.java
  └── models/
      ├── User.java
      └── Order.java

GOOD — screams "Order Management System":
  com.example
  ├── orders/
  │   ├── domain/
  │   │   ├── Order.java
  │   │   ├── OrderLine.java
  │   │   └── Money.java
  │   ├── application/
  │   │   ├── PlaceOrderUseCase.java
  │   │   ├── PlaceOrderInteractor.java
  │   │   └── OrderRepository.java          ← interface, lives in application
  │   └── infrastructure/
  │       ├── JpaOrderRepositoryAdapter.java ← implementation, outer layer
  │       └── PlaceOrderController.java
  ├── customers/
  │   └── ...
  └── payments/
      └── ...
```

The web framework is a plugin. The database is a plugin. The business rules are the center.

---

## The Humble Object Pattern

When code is hard to test because it interacts with external systems (UI, database, network), split the behavior into:

1. **The Humble Object** — does the hard-to-test interaction, contains almost no logic
2. **The Testable Object** — contains all the logic, easy to test in isolation

```java
// Without Humble Object — hard to test because logic is mixed with HTTP
public class OrderController {
    public HttpResponse handle(HttpRequest request) {
        String id = request.getParam("id");
        Order order = repository.findById(id);
        if (order == null) {
            return HttpResponse.notFound("Order " + id + " not found");
        }
        double tax = order.getTotal() * 0.1;  // tax logic buried in controller!
        return HttpResponse.ok(order.getTotal() + tax);
    }
}

// With Humble Object — the controller is humble (no logic), logic is testable
// Presenter (testable — no HTTP dependency)
public class OrderPresenter {
    public OrderViewModel present(Order order) {
        Money tax = order.total().multiply(0.1);
        return new OrderViewModel(order.id(), order.total(), tax, order.total().add(tax));
    }
}

// Controller (humble — no logic, just wiring)
public class OrderController {
    public HttpResponse handle(HttpRequest request) {
        return orderRepository.findById(request.getParam("id"))
            .map(order -> HttpResponse.ok(presenter.present(order)))
            .orElse(HttpResponse.notFound("Order not found"));
    }
}
```

---

## Testing Strategy Per Layer

```
Layer                  Test Type           Dependencies          Speed
──────────────────────────────────────────────────────────────────────
Entities               Unit tests          None                  Fast
Use Cases              Unit tests          Mock output ports     Fast
Interface Adapters     Integration tests   Real DB / real HTTP   Medium
Frameworks & Drivers   E2E / Smoke tests   Full stack            Slow

Testing pyramid:
  △     E2E (few)
 ▲▲▲    Integration (some)
▲▲▲▲▲  Unit (many — entities + use cases)
```

### Testing Use Cases with Mock Adapters

```java
class PlaceOrderInteractorTest {

    private final OrderRepository orderRepository = mock(OrderRepository.class);
    private final PaymentGateway paymentGateway = mock(PaymentGateway.class);
    private final CustomerRepository customerRepository = mock(CustomerRepository.class);

    private final PlaceOrderInteractor sut = new PlaceOrderInteractor(
        orderRepository, paymentGateway, customerRepository
    );

    @Test
    void places_order_successfully_when_payment_approved() {
        // Arrange
        var customer = TestCustomers.active();
        when(customerRepository.findById(customer.id())).thenReturn(Optional.of(customer));
        when(paymentGateway.charge(any(), any())).thenReturn(PaymentResult.approved());

        var command = new PlaceOrderCommand(customer.id(), List.of(
            new OrderItemCommand(PRODUCT_ID, Quantity.of(2), Money.of(50, USD))
        ));

        // Act
        var response = sut.execute(command);

        // Assert
        assertThat(response.total()).isEqualTo(Money.of(100, USD));
        verify(orderRepository).save(argThat(order ->
            order.status() == OrderStatus.SUBMITTED
        ));
    }

    @Test
    void throws_when_payment_declined() {
        // Arrange
        var customer = TestCustomers.active();
        when(customerRepository.findById(customer.id())).thenReturn(Optional.of(customer));
        when(paymentGateway.charge(any(), any()))
            .thenReturn(PaymentResult.declined("Insufficient funds"));

        // Act & Assert
        assertThatThrownBy(() -> sut.execute(validCommand(customer.id())))
            .isInstanceOf(PaymentDeclinedException.class)
            .hasMessageContaining("Insufficient funds");

        verify(orderRepository, never()).save(any());
    }
}
```

---

## Anti-Patterns

### 1. Framework Coupling in the Domain

```java
// BAD — JPA annotations on the Entity (framework leaks into inner circle)
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderLine> lines;
}

// GOOD — separate JPA entity from domain entity
// Domain entity (inner circle — no JPA)
public class Order { ... }

// JPA entity (outer circle — all JPA details here)
@Entity
@Table(name = "orders")
class OrderJpaEntity {
    @Id private String id;
    @OneToMany(cascade = CascadeType.ALL) private List<OrderLineJpaEntity> lines;

    public static OrderJpaEntity from(Order order) { ... }
    public Order toDomain() { ... }
}
```

### 2. Business Logic in Controllers

```python
# BAD — business logic in the controller
class OrderController:
    def create_order(self, request):
        # Validation that belongs in the domain
        if not request.items:
            return Response({"error": "Order must have items"}, status=400)

        # Calculation that belongs in the domain
        total = sum(item['price'] * item['quantity'] for item in request.items)
        if total > request.user.credit_limit:
            return Response({"error": "Exceeds credit limit"}, status=422)

        # Persistence mixed with business rules
        order = Order.objects.create(user=request.user, total=total)
        for item in request.items:
            OrderItem.objects.create(order=order, **item)

        return Response({"order_id": order.id}, status=201)

# GOOD — controller is thin, use case has the logic
class OrderController:
    def create_order(self, request):
        command = PlaceOrderCommand(
            customer_id=request.user.id,
            items=[OrderItemDto(**item) for item in request.data['items']]
        )
        result = self.place_order_use_case.execute(command)
        return Response({"order_id": str(result.order_id)}, status=201)
```

### 3. Anemic Domain Model

```java
// BAD — Order has no behavior, logic is scattered in services
public class Order {
    private String id;
    private String status;
    private List<OrderLine> lines;
    // Only getters and setters
}

// Logic scattered everywhere:
public class OrderService {
    public void submitOrder(Order order) { order.setStatus("SUBMITTED"); }
    public Money calcTotal(Order order) { ... }
    public void validateOrder(Order order) { ... }
}

// GOOD — rich domain model, behavior lives with data
public class Order {
    public void submit() {
        if (lines.isEmpty()) throw new EmptyOrderException(id);
        this.status = OrderStatus.SUBMITTED;
    }
    public Money total() { ... }
}
```

### 4. Dependency Direction Violations

```
VIOLATION MAP — any arrow pointing outward is a bug:

  Entity ──→ UseCase        ✗ Entities must not know about use cases
  Entity ──→ Framework      ✗ No JPA/Spring in entities
  UseCase ──→ Controller    ✗ Use cases must not know about HTTP
  UseCase ──→ Database      ✗ Use cases must not import JPA/SQLAlchemy
  UseCase ──→ Framework     ✗ No Spring annotations in use cases
```

---

## Concrete Feature Walkthrough

**Feature: "Customer can submit a product review"**

```
Step 1: Entity — ProductReview
  - Has invariants: rating 1-5, comment not empty, not blank
  - Domain events: ReviewSubmitted

Step 2: Use Case — SubmitProductReviewUseCase
  - Input port: SubmitReviewCommand(customerId, productId, rating, comment)
  - Output port: ReviewRepository (interface defined here)
  - Output port: ProductRepository (to verify product exists)
  - Logic: verify customer bought product, verify not already reviewed, create review

Step 3: Interface Adapters
  - ReviewController: maps HTTP POST /products/{id}/reviews → command
  - JpaReviewRepository: implements ReviewRepository using Spring Data JPA
  - ReviewPresenter: maps domain response → API JSON response

Step 4: Framework
  - Spring @Configuration wires everything
  - JPA entity ReviewJpaEntity handles persistence mapping
```

```kotlin
// Entity
data class ProductReview(
    val id: ReviewId,
    val customerId: CustomerId,
    val productId: ProductId,
    val rating: Rating,           // value object: enforces 1-5
    val comment: ReviewComment,   // value object: enforces non-blank
    val submittedAt: Instant
) {
    companion object {
        fun submit(
            customerId: CustomerId,
            productId: ProductId,
            rating: Rating,
            comment: ReviewComment
        ): ProductReview = ProductReview(
            id = ReviewId.generate(),
            customerId = customerId,
            productId = productId,
            rating = rating,
            comment = comment,
            submittedAt = Instant.now()
        )
    }
}

// Use Case
class SubmitProductReviewInteractor(
    private val reviewRepository: ReviewRepository,
    private val purchaseRepository: PurchaseRepository
) : SubmitProductReviewUseCase {

    override fun execute(command: SubmitReviewCommand): ReviewId {
        purchaseRepository.findByCustomerAndProduct(command.customerId, command.productId)
            ?: throw CustomerDidNotPurchaseProductException(command.customerId, command.productId)

        if (reviewRepository.existsByCustomerAndProduct(command.customerId, command.productId)) {
            throw DuplicateReviewException(command.customerId, command.productId)
        }

        val review = ProductReview.submit(
            customerId = command.customerId,
            productId = command.productId,
            rating = Rating(command.rating),
            comment = ReviewComment(command.comment)
        )

        reviewRepository.save(review)
        return review.id
    }
}
```

---

## When to Use Clean Architecture

- Systems expected to last years
- Codebases with multiple delivery mechanisms (REST API + CLI + message consumer)
- High business logic complexity that needs testability
- Teams that need to swap infrastructure without touching the domain
- When the domain should be understandable without reading framework docs

## When NOT to Use Clean Architecture

- Simple CRUD apps with minimal business logic — overhead is not justified
- Prototypes or MVPs where you need to validate the idea first
- Microservices so small that one layer = one class anyway
- Scripts, tools, utilities
- When the team does not understand it — a cargo-culted clean architecture is worse than no architecture

---

## Checklist

### Structure
- [ ] The top-level package structure reveals the business domain, not the framework
- [ ] Entities have no framework imports (no JPA, no Spring, no Django ORM)
- [ ] Use cases have no HTTP types, no database framework types
- [ ] Repository interfaces are defined in the use case layer, not in infrastructure

### Dependency Rule
- [ ] No import in an inner circle references an outer circle
- [ ] Use cases depend on abstractions (interfaces), not concrete implementations
- [ ] Dependency Inversion is used wherever inner circle needs outer circle behavior

### Behavior Distribution
- [ ] Business rules live in entities and use cases, not in controllers or repositories
- [ ] Controllers only convert data and delegate to use cases
- [ ] Domain model is rich — entities have behavior, not just data

### Testing
- [ ] Entities are unit-testable with no mocks at all
- [ ] Use cases are unit-testable with mock output ports
- [ ] Repository implementations have integration tests against a real database
- [ ] No E2E tests required to verify core business logic

### Anti-Patterns Avoided
- [ ] No anemic domain model
- [ ] No business logic in controllers
- [ ] No framework annotations in domain entities
- [ ] No direct database calls in use cases
