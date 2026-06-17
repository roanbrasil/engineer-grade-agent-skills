---
name: ddd-tactical
description: Apply Domain-Driven Design tactical patterns — Entities, Value Objects, Aggregates, Domain Services, Repositories, Factories, and Domain Events — when modeling complex business domains in code.
---

# DDD Tactical Patterns

## Core Philosophy

The goal of DDD is to make the code a faithful model of the domain — to the point where a domain expert could read the code and confirm or deny its correctness. Tactical patterns are the building blocks. They are only valuable when they encode real domain concepts, not when applied mechanically.

**The central question for every design decision:** Does this belong in the domain, or is it an infrastructure or application concern?

---

## Ubiquitous Language

The shared vocabulary between domain experts and developers. Every concept that appears in a business conversation must appear in the code by the same name, and vice versa.

```
Domain Expert says:           Code must say:
"Invoice"                --> Invoice (class, not Bill or PaymentRequest)
"Overdue invoice"        --> Invoice.isOverdue() (not Invoice.status == 3)
"Apply late fee"         --> invoice.applyLateFee(LocalDate today)
"Void the invoice"       --> invoice.void() (not invoice.setStatus(VOIDED))
```

**Building the Ubiquitous Language:**
1. Listen for nouns: these are candidate Entities and Value Objects
2. Listen for verbs: these are candidate behaviors on those objects or Domain Services
3. Listen for rules: these are invariants on Aggregates
4. Create a domain glossary and keep it in the repository (`/docs/glossary.md`)
5. When the expert uses two words for the same thing — resolve it; ambiguity kills the model

---

## Bounded Contexts

A Bounded Context is a boundary within which a particular domain model applies. The same real-world concept can mean different things in different contexts.

```
+--------------------+        +----------------------+        +--------------------+
|   SALES CONTEXT    |        |  SHIPPING CONTEXT    |        | BILLING CONTEXT    |
|                    |        |                      |        |                    |
| Customer           |        | Customer             |        | Customer           |
|  - name            |        |  - delivery address  |        |  - billing address |
|  - email           |        |  - preferred carrier |        |  - payment method  |
|  - loyalty tier    |        |  - weight limit      |        |  - credit limit    |
|                    |        |                      |        |                    |
| Order              |        | Shipment             |        | Invoice            |
|  - line items      |        |  - packages          |        |  - line items      |
|  - discount        |        |  - tracking number   |        |  - tax lines       |
+--------------------+        +----------------------+        +--------------------+
```

"Customer" is three different things. Fighting to create one universal Customer model produces an object that serves no context well. Bounded Contexts let each model be optimal for its purpose.

---

## Context Mapping Patterns

The relationships between Bounded Contexts:

```
+----------+         +----------+
| UPSTREAM |         | DOWNSTREAM|
| (supplier)|------->|(customer) |
+----------+         +----------+
```

| Pattern | Description | When to use |
|---------|-------------|-------------|
| **Partnership** | Two teams succeed or fail together; plan jointly | High dependency, shared roadmap |
| **Shared Kernel** | Small shared model owned jointly | Strong coupling is acceptable |
| **Customer-Supplier** | Upstream provides what downstream needs; downstream negotiates | Different teams, aligned goals |
| **Conformist** | Downstream adopts upstream model as-is | Upstream won't negotiate; model is acceptable |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream model to its own | Upstream model is hostile or legacy |
| **Open Host Service** | Upstream publishes a protocol for all consumers | Many consumers; upstream wants stability |
| **Published Language** | Shared exchange format (e.g., JSON schema, Protobuf) | Cross-team/cross-org integration |
| **Separate Ways** | Contexts don't integrate at all | No meaningful overlap |

### Anti-Corruption Layer example

```kotlin
// Billing context consuming the legacy ERP's customer data
// The ERP model is hostile; we translate at the boundary

// ERP model (legacy, not our design)
data class ErpCustomerRecord(
    val custId: String,
    val custName: String,
    val addr1: String,
    val addr2: String,
    val payCode: Int,          // 1 = credit card, 2 = invoice, 3 = bank transfer
    val creditLimitCents: Long
)

// Our domain model (clean, expressive)
data class BillingCustomer(
    val id: CustomerId,
    val name: CustomerName,
    val billingAddress: Address,
    val paymentMethod: PaymentMethod,
    val creditLimit: Money
)

// ACL: lives at the boundary, belongs to the Billing context
class ErpCustomerTranslator {
    fun toDomain(erp: ErpCustomerRecord): BillingCustomer {
        return BillingCustomer(
            id = CustomerId.of(erp.custId),
            name = CustomerName.of(erp.custName),
            billingAddress = translateAddress(erp.addr1, erp.addr2),
            paymentMethod = translatePaymentMethod(erp.payCode),
            creditLimit = Money.ofCents(erp.creditLimitCents, Currency.BRL)
        )
    }

    private fun translatePaymentMethod(payCode: Int): PaymentMethod = when (payCode) {
        1 -> PaymentMethod.CREDIT_CARD
        2 -> PaymentMethod.INVOICE
        3 -> PaymentMethod.BANK_TRANSFER
        else -> throw DomainException("Unknown ERP pay code: $payCode")
    }
}
```

---

## Tactical Patterns

### Entities

An Entity has identity. Two entities with the same data are still different if they have different IDs. Identity persists through state changes over time.

```kotlin
class Order private constructor(
    val id: OrderId,                    // Identity — never changes
    private var status: OrderStatus,    // State — changes over time
    private val items: MutableList<OrderItem>,
    private val customerId: CustomerId
) {
    // Business behavior on the entity
    fun addItem(product: Product, quantity: Quantity) {
        check(status == OrderStatus.DRAFT) { "Cannot add items to a ${status} order" }
        items.add(OrderItem.of(product, quantity))
    }

    fun confirm() {
        check(status == OrderStatus.DRAFT) { "Order ${id} is already ${status}" }
        check(items.isNotEmpty()) { "Cannot confirm an empty order" }
        status = OrderStatus.CONFIRMED
    }

    // Equality based on identity, not state
    override fun equals(other: Any?) = other is Order && other.id == id
    override fun hashCode() = id.hashCode()

    companion object {
        fun create(customerId: CustomerId): Order =
            Order(OrderId.generate(), OrderStatus.DRAFT, mutableListOf(), customerId)
    }
}
```

**Entity rules:**
- Identity is assigned at creation and never changes
- State changes through explicit business methods — never through setters
- Equality is defined by identity
- Expose state as needed but protect it from external mutation

### Value Objects

A Value Object has no identity. Two Value Objects with the same data are interchangeable. Value Objects are immutable.

```kotlin
// Money: no identity, equality by value, immutable
data class Money private constructor(
    val amount: BigDecimal,
    val currency: Currency
) {
    init {
        require(amount >= BigDecimal.ZERO) { "Money amount cannot be negative: $amount" }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add $currency to ${other.currency}" }
        return copy(amount = amount + other.amount)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Cannot subtract ${other.currency} from $currency" }
        require(amount >= other.amount) { "Insufficient amount: $amount < ${other.amount}" }
        return copy(amount = amount - other.amount)
    }

    fun applyDiscount(pct: Percentage): Money =
        copy(amount = amount * (BigDecimal.ONE - pct.toDecimal()))

    companion object {
        fun of(amount: BigDecimal, currency: Currency) = Money(amount, currency)
        fun ofCents(cents: Long, currency: Currency) =
            Money(BigDecimal.valueOf(cents, 2), currency)
        val ZERO_BRL = Money(BigDecimal.ZERO, Currency.BRL)
    }
}

// Domain Primitive: a Value Object wrapping a primitive to enforce validity
@JvmInline
value class Email private constructor(val value: String) {
    companion object {
        private val PATTERN = Regex("^[^@]+@[^@]+\\.[^@]+$")
        fun of(raw: String): Email {
            require(raw.isNotBlank()) { "Email cannot be blank" }
            require(PATTERN.matches(raw)) { "Invalid email format: $raw" }
            return Email(raw.lowercase().trim())
        }
    }
}
```

**Value Object rules:**
- No identity; equality is structural (by all fields)
- Immutable — operations return new instances
- Self-validating in the constructor or factory method
- Replace primitives: use `Email` instead of `String`, `Money` instead of `BigDecimal`
- Can reference other Value Objects; never reference Entities

**Domain Primitives** (Value Objects wrapping a single primitive) are especially valuable:
- `CustomerId` instead of `UUID` or `String`
- `OrderQuantity` instead of `Int`
- `ProductSku` instead of `String`
- `Percentage` instead of `Double`

This eliminates an entire class of bugs: passing a price where a quantity is expected becomes a compile error.

### Aggregates

An Aggregate is a cluster of domain objects (Entities and Value Objects) that is treated as a single unit for the purposes of data changes. It enforces invariants (business rules that must always be true) and is the transaction boundary.

```
+-------------------------------------------------------+
|                   ORDER AGGREGATE                     |
|                                                       |
|  +----------+                                         |
|  |  Order   |  <-- Aggregate Root (entry point)       |
|  | (Entity) |                                         |
|  +----+-----+                                         |
|       |                                               |
|       | owns / contains                               |
|       |                                               |
|  +----v---------+   +------------------+              |
|  |  OrderItem   |   |  ShippingAddress  |             |
|  |  (Entity)    |   |  (Value Object)  |              |
|  +--------------+   +------------------+              |
|                                                       |
|  INVARIANTS enforced here:                            |
|  - Cannot confirm empty order                         |
|  - Total cannot exceed customer credit limit          |
|  - Cannot ship to country on embargo list             |
+-------------------------------------------------------+

External code can only interact through Order (the Root).
Direct access to OrderItem from outside is forbidden.
```

```kotlin
class Order private constructor(
    val id: OrderId,
    private var status: OrderStatus,
    private val _items: MutableList<OrderItem> = mutableListOf(),
    val customerId: CustomerId,
    private val creditLimit: Money
) {
    val items: List<OrderItem> get() = _items.toList()  // read-only view

    val total: Money get() = _items.fold(Money.ZERO_BRL) { acc, item ->
        acc + item.subtotal
    }

    fun addItem(product: Product, quantity: Quantity) {
        requireDraft()
        val newItem = OrderItem.of(product, quantity)
        val projectedTotal = total + newItem.subtotal
        // Invariant: total must not exceed credit limit
        require(projectedTotal <= creditLimit) {
            "Adding item would exceed credit limit of ${creditLimit}"
        }
        _items.add(newItem)
    }

    fun removeItem(sku: ProductSku) {
        requireDraft()
        _items.removeIf { it.sku == sku }
    }

    fun confirm(): OrderConfirmed {
        requireDraft()
        require(_items.isNotEmpty()) { "Cannot confirm empty order ${id}" }
        status = OrderStatus.CONFIRMED
        return OrderConfirmed(orderId = id, total = total, occurredAt = Instant.now())
    }

    fun cancel(reason: CancellationReason): OrderCancelled {
        require(status != OrderStatus.SHIPPED) { "Cannot cancel shipped order ${id}" }
        status = OrderStatus.CANCELLED
        return OrderCancelled(orderId = id, reason = reason, occurredAt = Instant.now())
    }

    private fun requireDraft() =
        require(status == OrderStatus.DRAFT) { "Order ${id} is ${status}, expected DRAFT" }
}
```

**Aggregate rules:**
1. **One Aggregate = One Transaction.** Never modify two aggregates in a single transaction.
2. **Only the Root is publicly accessible.** External objects hold references only to the root, never to internal entities.
3. **Design small aggregates.** If the aggregate is growing, a design smell is present. Most aggregates should have 2–5 objects.
4. **Reference other aggregates by ID only.** `Order` holds `customerId: CustomerId`, not `customer: Customer`.
5. **Aggregates enforce invariants.** Business rules that span multiple objects in the aggregate live on the root.

### Aggregate Root

The Aggregate Root is the Entity that is the single entry point into the aggregate. It is the gatekeeper for all invariant enforcement.

```java
// OrderItem is an entity INSIDE the aggregate — only accessible through Order
public class OrderItem {
    private final OrderItemId id;
    private final ProductSku sku;
    private final ProductName name;
    private final Money unitPrice;
    private Quantity quantity;

    // Package-private constructor: can only be created by Order (same package)
    OrderItem(ProductSku sku, ProductName name, Money unitPrice, Quantity quantity) {
        this.id = OrderItemId.generate();
        this.sku = sku;
        this.name = name;
        this.unitPrice = unitPrice;
        this.quantity = quantity;
    }

    public Money subtotal() {
        return unitPrice.multiply(quantity.value());
    }

    // No public setters — state change only through Order's methods
}
```

### Domain Events

Things that happened in the domain, expressed in past tense, immutable.

```kotlin
// Domain events are raised by aggregates after enforcing invariants
sealed class OrderDomainEvent {
    abstract val orderId: OrderId
    abstract val occurredAt: Instant
}

data class OrderConfirmed(
    override val orderId: OrderId,
    val total: Money,
    override val occurredAt: Instant = Instant.now()
) : OrderDomainEvent()

data class OrderCancelled(
    override val orderId: OrderId,
    val reason: CancellationReason,
    override val occurredAt: Instant = Instant.now()
) : OrderDomainEvent()

data class ItemAddedToOrder(
    override val orderId: OrderId,
    val sku: ProductSku,
    val quantity: Quantity,
    val unitPrice: Money,
    override val occurredAt: Instant = Instant.now()
) : OrderDomainEvent()
```

See the `domain-events` skill for full event dispatch and outbox patterns.

### Repositories

A Repository provides collection-style access to aggregates. It abstracts the persistence mechanism from the domain.

```kotlin
// Domain defines the interface (in domain layer)
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun findByCustomer(customerId: CustomerId): List<Order>
    fun save(order: Order)
    fun delete(id: OrderId)
}

// Infrastructure implements it (in infrastructure layer)
class JpaOrderRepository(
    private val jpaRepo: SpringDataOrderJpaRepository,
    private val mapper: OrderMapper
) : OrderRepository {
    override fun findById(id: OrderId): Order? =
        jpaRepo.findById(id.value)
            .map(mapper::toDomain)
            .orElse(null)

    override fun save(order: Order) {
        val entity = mapper.toJpa(order)
        jpaRepo.save(entity)
    }

    // ... other methods
}

// In-memory fake for tests
class InMemoryOrderRepository : OrderRepository {
    private val store = mutableMapOf<OrderId, Order>()

    override fun findById(id: OrderId): Order? = store[id]
    override fun save(order: Order) { store[order.id] = order }
    override fun findByCustomer(customerId: CustomerId) =
        store.values.filter { it.customerId == customerId }
    override fun delete(id: OrderId) { store.remove(id) }
}
```

**Repository rules:**
- Only for Aggregates. Never a repository for an entity inside an aggregate.
- Domain owns the interface; infrastructure owns the implementation.
- Repository speaks domain language: `findOverdueOrders()` not `findByStatusAndDueDateBefore()`.
- Avoid query explosion: if you need many query methods, consider a separate ReadModel / Query side (CQRS).
- Repository handles one aggregate type. `OrderRepository`, not `EverythingRepository`.

### Domain Services

Stateless operations that involve domain logic but don't naturally belong on a single entity or value object.

```kotlin
// Domain Service: crosses multiple aggregates or requires domain knowledge
// that doesn't belong to any one aggregate
class LoyaltyDiscountPolicy {
    fun calculateDiscount(order: Order, customer: Customer): Money {
        // Discount depends on both Order (total) and Customer (tier)
        // Neither Order nor Customer should know about the other's internals
        if (customer.tier != CustomerTier.PREMIUM) return Money.ZERO_BRL
        if (order.containsOnlySaleItems()) return Money.ZERO_BRL

        return order.total.applyDiscount(Percentage.of(15))
            .let { order.total - it }
    }
}

// Fraud risk scoring — neither Order nor Customer is the right home
class FraudRiskScorer(
    private val velocityCheck: VelocityCheckService,
    private val blacklistService: BlacklistService
) {
    fun score(order: Order, customer: Customer, card: CreditCard): FraudRiskScore {
        val recentOrderCount = velocityCheck.countRecentOrders(customer.id, withinMinutes = 60)
        val isBlacklisted = blacklistService.isBlacklisted(card.bin)

        return FraudRiskScore.calculate(
            orderAmount = order.total,
            recentOrderCount = recentOrderCount,
            isBlacklisted = isBlacklisted
        )
    }
}
```

**Domain Service rules:**
- Stateless — no fields that represent state; inject collaborators through constructor
- Named after a domain concept: `PricingPolicy`, `FraudScorer`, `ShippingCalculator`
- Lives in the domain layer; does not depend on infrastructure
- Use when: an operation involves multiple aggregates, or requires external domain knowledge, or the logic doesn't fit on any single entity without making it bloated

### Application Services

Application Services are the use case layer. They orchestrate the domain, manage transactions, and coordinate infrastructure.

```kotlin
// Application Service: thin orchestrator
// Owns the transaction boundary
// Does NOT contain business logic — that lives in the domain
@Transactional
class PlaceOrderUseCase(
    private val orderRepository: OrderRepository,
    private val customerRepository: CustomerRepository,
    private val discountPolicy: LoyaltyDiscountPolicy,
    private val eventPublisher: DomainEventPublisher
) {
    fun execute(command: PlaceOrderCommand): OrderId {
        // 1. Load aggregates
        val customer = customerRepository.findById(command.customerId)
            ?: throw CustomerNotFoundException(command.customerId)

        // 2. Create new aggregate
        val order = Order.create(customer.id, customer.creditLimit)

        // 3. Apply domain logic
        command.items.forEach { item ->
            order.addItem(item.product, item.quantity)
        }

        // 4. Apply domain service
        val discount = discountPolicy.calculateDiscount(order, customer)
        order.applyDiscount(discount)

        // 5. Domain operation that raises event
        val event = order.confirm()

        // 6. Persist
        orderRepository.save(order)

        // 7. Publish events (after commit in production — see domain-events skill)
        eventPublisher.publish(event)

        return order.id
    }
}
```

**Application Service rules:**
- One Application Service per use case (or cohesive group of related use cases)
- Owns the transaction boundary — not the domain, not the repository
- Translates between DTOs/commands and domain objects
- Contains no business logic — delegates everything to domain objects
- Depends on domain interfaces (Repository, Domain Service) not implementations

### Factories

Factories handle complex creation logic that doesn't belong in a constructor or a simple factory method.

```kotlin
// Simple factory method on the aggregate: preferred when possible
class Order private constructor(...) {
    companion object {
        fun create(customerId: CustomerId, creditLimit: Money): Order =
            Order(OrderId.generate(), OrderStatus.DRAFT, customerId, creditLimit)
    }
}

// Factory class: when creation requires external dependencies or complex rules
class OrderFactory(
    private val customerRepository: CustomerRepository,
    private val inventoryService: InventoryService
) {
    fun createFromQuote(quote: SalesQuote): Order {
        val customer = customerRepository.findById(quote.customerId)
            ?: throw CustomerNotFoundException(quote.customerId)

        // Validate inventory before creating order
        quote.lineItems.forEach { line ->
            require(inventoryService.isAvailable(line.sku, line.quantity)) {
                "Item ${line.sku} not available in quantity ${line.quantity}"
            }
        }

        val order = Order.create(customer.id, customer.creditLimit)
        quote.lineItems.forEach { line ->
            order.addItem(line.product, line.quantity)
        }
        return order
    }
}
```

---

## Anti-Patterns

### Anemic Domain Model

The most common DDD failure. Entities and Value Objects are just data containers. All logic lives in Services.

```java
// ANEMIC — Order is just a struct
public class Order {
    public String id;
    public String status;
    public List<OrderItem> items;
    public BigDecimal total;
    // Only getters and setters
}

// All logic in a service — tells, doesn't ask
public class OrderService {
    public void confirm(Order order) {
        if (order.getStatus().equals("DRAFT") && !order.getItems().isEmpty()) {
            order.setStatus("CONFIRMED");
            order.setConfirmedAt(Instant.now());
        }
    }
}
```

```kotlin
// RICH — Order knows its own rules
class Order(...) {
    fun confirm(): OrderConfirmed {
        require(status == OrderStatus.DRAFT) { "..." }
        require(items.isNotEmpty()) { "..." }
        status = OrderStatus.CONFIRMED
        return OrderConfirmed(id, total, Instant.now())
    }
}
```

### Bloated Entities

An entity that knows everything and does everything. God object in DDD clothing.

Signs: Order that calculates shipping, applies promotions, sends emails, generates PDF invoices.

Fix: Extract to Value Objects (immutable calculations), Domain Services (cross-aggregate logic), and delegate notifications to event handlers.

### Cross-Aggregate References

```kotlin
// BAD: Order holds a reference to Customer
class Order(val customer: Customer, ...)

// GOOD: Order holds only the CustomerId
class Order(val customerId: CustomerId, ...)
```

Cross-aggregate references violate aggregate boundaries. When Order is loaded, Customer gets loaded too (and everything Customer references). Transaction boundaries become unclear. Separate aggregates must be loaded separately and referenced by ID.

### Repository for Non-Aggregates

```kotlin
// BAD: OrderItemRepository doesn't make sense
interface OrderItemRepository {
    fun findById(id: OrderItemId): OrderItem?
    fun save(item: OrderItem)
}

// OrderItem is part of the Order aggregate — it is accessed through Order
val order = orderRepository.findById(orderId)
val item = order.items.find { it.id == itemId }
```

### Putting Domain Logic in Application Services

```kotlin
// BAD: Business rule in Application Service
class PlaceOrderUseCase {
    fun execute(command: PlaceOrderCommand) {
        // This is domain logic — belongs on Order or LoyaltyDiscountPolicy
        val discount = if (customer.orderCount >= 5) total * 0.15 else BigDecimal.ZERO
        order.setDiscount(discount)
    }
}

// GOOD: Application Service delegates to domain
class PlaceOrderUseCase {
    fun execute(command: PlaceOrderCommand) {
        val discount = discountPolicy.calculateDiscount(order, customer)  // domain service
        order.applyDiscount(discount)  // domain method
    }
}
```

---

## Layer Architecture

```
+-----------------------------------------------------------------------+
|  INTERFACES / ADAPTERS                                                |
|  REST Controllers, gRPC, CLI, Event Consumers                        |
|  Translate external requests into Commands/Queries                   |
+-----------------------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------------------+
|  APPLICATION LAYER                                                    |
|  Application Services (Use Cases)                                    |
|  Command/Query handlers                                               |
|  Transaction management, event publishing                            |
+-----------------------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------------------+
|  DOMAIN LAYER                                                         |
|  Entities, Value Objects, Aggregates, Aggregate Roots                |
|  Domain Services, Domain Events                                      |
|  Repository interfaces, Factory interfaces                           |
|  NO external dependencies (no Spring, no JPA, no HTTP)              |
+-----------------------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------------------+
|  INFRASTRUCTURE LAYER                                                 |
|  Repository implementations (JPA, MongoDB, Redis)                   |
|  Event publisher implementations (Kafka, RabbitMQ, SQS)            |
|  External service clients                                            |
|  Database schema, migrations                                         |
+-----------------------------------------------------------------------+
```

**Dependency rule:** Dependencies point inward. Domain has no dependencies. Application depends on Domain. Infrastructure depends on Application and Domain. Interfaces depend on Application.

---

## Practical Checklist

### Ubiquitous Language
- [ ] Every class and method name comes from the domain glossary
- [ ] Domain expert can read the code and confirm its accuracy
- [ ] No technical terms (DTO, Manager, Helper, Data) in the domain layer

### Value Objects
- [ ] All primitive obsession eliminated? (no bare String for email, UUID for ID, etc.)
- [ ] Value Objects are immutable?
- [ ] Equality is structural, not by reference?
- [ ] Self-validating in constructor or factory method?

### Entities
- [ ] Has a stable identity that never changes?
- [ ] State changes only through explicit business methods?
- [ ] Equality is by identity, not by state?

### Aggregates
- [ ] Is the boundary justified by invariants that must be enforced together?
- [ ] Is the aggregate small (2-5 objects)?
- [ ] Does external code only reference the Root?
- [ ] Are other aggregates referenced by ID only?
- [ ] Does one aggregate change per transaction?

### Domain Services
- [ ] Is the service stateless?
- [ ] Named after a domain concept?
- [ ] Has no infrastructure dependencies?
- [ ] Could not be moved onto an Entity without making that entity bloated?

### Repositories
- [ ] Defined in domain layer (interface only)?
- [ ] Only for Aggregates?
- [ ] Methods speak domain language?

### Application Services
- [ ] Contains no business logic?
- [ ] Owns the transaction boundary?
- [ ] Is thin (under 30-40 lines per method)?

### Anti-patterns check
- [ ] No anemic entities? (Entities have behavior, not just getters/setters)
- [ ] No God entity? (Order doesn't send emails or calculate shipping)
- [ ] No cross-aggregate direct references?
- [ ] No repository for internal entities?
