---
name: ports-and-adapters
description: Apply Hexagonal Architecture (Ports and Adapters) when designing application boundaries, defining interfaces between the domain and infrastructure, or reviewing how external dependencies are integrated — invoked when discussing ports, adapters, hexagonal architecture, or application isolation.
---

# Ports and Adapters (Hexagonal Architecture)

Hexagonal Architecture — coined by Alistair Cockburn — solves a fundamental problem: **the application core should not know whether it is being driven by a test, an HTTP request, a CLI command, or a message queue**. Symmetrically, **the application core should not know whether it stores data in PostgreSQL, MongoDB, an in-memory map, or a file**.

The solution: the application core defines *ports* (interfaces). External code provides *adapters* (implementations). The core is surrounded by adapters on all sides — hence "hexagonal."

---

## The Fundamental Asymmetry

Not all ports are equal. There are two fundamentally different sides:

```
                        ┌─────────────────────────────┐
  DRIVING SIDE          │                             │          DRIVEN SIDE
  (Primary)             │    APPLICATION HEXAGON      │          (Secondary)
                        │                             │
  Things that           │  ┌───────────────────────┐  │  Things the app
  drive the app  ──────►│  │                       │  │──────► drives
                        │  │   Domain Logic        │  │
  REST Controller       │  │   Use Cases           │  │  Database
  CLI Command           │  │   Domain Services     │  │  Email Service
  Message Consumer      │  │   Entities            │  │  Payment Gateway
  Test                  │  │                       │  │  File System
  Scheduled Job         │  └───────────────────────┘  │  Message Queue
                        │                             │
                        │  PRIMARY PORTS  SECONDARY PORTS
                        │  (left side)    (right side)
                        └─────────────────────────────┘

  Primary Adapters                          Secondary Adapters
  implement primary ports                   implement secondary ports
```

**Primary (Driving) Ports**: interfaces that external actors use to interact with the application core. The application exposes these.

**Secondary (Driven) Ports**: interfaces that the application uses to interact with external systems. The application defines these; infrastructure implements them.

---

## Primary Ports: What the Application Exposes

A primary port is a use case interface. It defines what the application can do, expressed in domain language — no HTTP, no CLI details.

```java
// Primary port — defined in the application core
public interface RegisterCustomerPort {
    CustomerId register(RegisterCustomerCommand command);
}

public interface QueryCustomerPort {
    CustomerView findById(CustomerId id);
    List<CustomerSummary> findAll(CustomerFilter filter);
}
```

```python
# Primary port — in the application package
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Protocol

class RegisterCustomerUseCase(Protocol):
    def execute(self, command: RegisterCustomerCommand) -> CustomerId: ...

class QueryCustomersUseCase(Protocol):
    def find_by_id(self, customer_id: CustomerId) -> CustomerView: ...
    def find_active(self) -> list[CustomerSummary]: ...
```

### Rules for Primary Ports

1. Defined **inside** the application core package
2. Use **domain types** only (no HTTP types, no framework types)
3. One interface per use case group (or even one interface per use case — open to debate)
4. The interface is the contract the primary adapter must satisfy

---

## Secondary Ports: What the Application Requires

A secondary port is a dependency interface. It represents something the application needs from the outside world, expressed in application terms — not in infrastructure terms.

```java
// Secondary ports — defined in the application core
public interface CustomerRepository {
    void save(Customer customer);
    Optional<Customer> findById(CustomerId id);
    List<Customer> findByStatus(CustomerStatus status);
    boolean existsByEmail(EmailAddress email);
}

public interface EmailNotificationPort {
    void sendWelcomeEmail(EmailAddress to, CustomerName name);
    void sendPasswordResetEmail(EmailAddress to, ResetToken token);
}

public interface PaymentPort {
    PaymentResult charge(PaymentMethod method, Money amount);
    void refund(PaymentReference reference, Money amount);
}
```

```python
# Secondary ports — in the application core
from typing import Protocol, Optional

class CustomerRepository(Protocol):
    def save(self, customer: Customer) -> None: ...
    def find_by_id(self, customer_id: CustomerId) -> Optional[Customer]: ...
    def find_by_email(self, email: EmailAddress) -> Optional[Customer]: ...

class NotificationPort(Protocol):
    def send_welcome(self, customer: Customer) -> None: ...

class PaymentPort(Protocol):
    def charge(self, method: PaymentMethod, amount: Money) -> PaymentResult: ...
```

### Rules for Secondary Ports

1. Defined **inside** the application core (not in infrastructure)
2. Named in **application terms**, not in infrastructure terms:
   - NOT: `JpaCustomerRepository`, `PostgresCustomerStore`, `StripePaymentAdapter`
   - YES: `CustomerRepository`, `PaymentPort`, `NotificationPort`
3. Methods reflect what the application needs, not what the database provides
4. No SQL, no HTTP, no infrastructure concepts in the interface

---

## Adapters: Connecting the Outside World

Adapters translate between the external world and the application's ports.

### Primary Adapters (Driving Adapters)

A primary adapter translates an external protocol into a port call.

```kotlin
// REST adapter — drives the application via the primary port
@RestController
@RequestMapping("/customers")
class CustomerRestAdapter(
    private val registerCustomer: RegisterCustomerUseCase,
    private val queryCustomers: QueryCustomersUseCase
) {
    @PostMapping
    fun register(@RequestBody request: RegisterCustomerRequest): ResponseEntity<CustomerResponse> {
        val command = RegisterCustomerCommand(
            name = CustomerName(request.name),
            email = EmailAddress(request.email)
        )
        val customerId = registerCustomer.execute(command)
        return ResponseEntity.created(URI("/customers/${customerId.value}"))
            .body(CustomerResponse(customerId.value))
    }

    @GetMapping("/{id}")
    fun getById(@PathVariable id: String): CustomerView {
        return queryCustomers.findById(CustomerId(id))
    }
}

// CLI adapter — same ports, different protocol
class CustomerCliAdapter(
    private val registerCustomer: RegisterCustomerUseCase
) {
    fun run(args: Array<String>) {
        val command = RegisterCustomerCommand(
            name = CustomerName(args[0]),
            email = EmailAddress(args[1])
        )
        val id = registerCustomer.execute(command)
        println("Customer registered: ${id.value}")
    }
}

// Message consumer adapter — same ports, different protocol
@Component
class CustomerRegistrationConsumer(
    private val registerCustomer: RegisterCustomerUseCase
) : MessageListener {
    override fun onMessage(message: Message) {
        val event = deserialize(message, CustomerRegistrationRequestedEvent::class)
        val command = RegisterCustomerCommand(
            name = CustomerName(event.name),
            email = EmailAddress(event.email)
        )
        registerCustomer.execute(command)
    }
}
```

### Secondary Adapters (Driven Adapters)

A secondary adapter implements a secondary port using a specific infrastructure technology.

```java
// Database adapter — implements the CustomerRepository port
@Repository
public class JpaCustomerAdapter implements CustomerRepository {
    private final JpaCustomerRepository jpa;
    private final CustomerMapper mapper;

    @Override
    public void save(Customer customer) {
        jpa.save(mapper.toJpa(customer));
    }

    @Override
    public Optional<Customer> findById(CustomerId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public Optional<Customer> findByEmail(EmailAddress email) {
        return jpa.findByEmail(email.value()).map(mapper::toDomain);
    }
}

// Email adapter — implements the NotificationPort
@Service
public class SendGridEmailAdapter implements NotificationPort {
    private final SendGridClient sendGrid;

    @Override
    public void sendWelcome(Customer customer) {
        var email = Email.builder()
            .to(customer.email().value())
            .subject("Welcome to our platform!")
            .htmlContent(WelcomeTemplate.render(customer.name()))
            .build();
        sendGrid.send(email);
    }
}

// In-memory adapter — for testing or local development
public class InMemoryCustomerAdapter implements CustomerRepository {
    private final Map<CustomerId, Customer> store = new ConcurrentHashMap<>();

    @Override
    public void save(Customer customer) { store.put(customer.id(), customer); }

    @Override
    public Optional<Customer> findById(CustomerId id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public Optional<Customer> findByEmail(EmailAddress email) {
        return store.values().stream()
            .filter(c -> c.email().equals(email))
            .findFirst();
    }
}
```

---

## The Application Hexagon: Dependency Inversion in Action

The hexagon (application core) depends only on its own ports. Infrastructure depends on the hexagon. This is Dependency Inversion applied architecturally.

```
Package structure that enforces the rule:

  com.example.customers
  ├── domain/                       ← Entities, Value Objects
  │   ├── Customer.java
  │   ├── CustomerId.java
  │   └── EmailAddress.java
  │
  ├── application/                  ← Use Cases + Port definitions
  │   ├── ports/
  │   │   ├── in/                   ← Primary ports (what we expose)
  │   │   │   ├── RegisterCustomerUseCase.java
  │   │   │   └── QueryCustomersUseCase.java
  │   │   └── out/                  ← Secondary ports (what we require)
  │   │       ├── CustomerRepository.java
  │   │       └── NotificationPort.java
  │   └── service/
  │       └── RegisterCustomerService.java  ← implements in-port, uses out-port
  │
  └── adapters/                     ← Adapters (outer layer)
      ├── in/
      │   ├── rest/
      │   │   └── CustomerRestAdapter.java
      │   └── messaging/
      │       └── CustomerRegistrationConsumer.java
      └── out/
          ├── persistence/
          │   └── JpaCustomerAdapter.java
          └── notification/
              └── SendGridEmailAdapter.java
```

```
Import rules (enforced by ArchUnit or package-by-layer conventions):

  domain         → imports nothing from application or adapters
  application    → imports domain only; never imports adapters
  adapters/in    → imports application.ports.in and domain
  adapters/out   → imports application.ports.out and domain
```

---

## Testing Strategy

### Unit Test Use Cases (No Infrastructure)

```java
class RegisterCustomerServiceTest {

    // Secondary port replaced by in-memory adapter — no database needed
    private final CustomerRepository customerRepository = new InMemoryCustomerAdapter();
    // Notification port replaced by spy/mock
    private final NotificationPort notifications = mock(NotificationPort.class);

    private final RegisterCustomerService sut = new RegisterCustomerService(
        customerRepository, notifications
    );

    @Test
    void registers_new_customer_and_sends_welcome() {
        var command = new RegisterCustomerCommand(
            new CustomerName("Alice"),
            new EmailAddress("alice@example.com")
        );

        var customerId = sut.execute(command);

        assertThat(customerRepository.findById(customerId)).isPresent();
        verify(notifications).sendWelcome(argThat(c ->
            c.email().equals(new EmailAddress("alice@example.com"))
        ));
    }

    @Test
    void rejects_duplicate_email() {
        var email = new EmailAddress("alice@example.com");
        customerRepository.save(Customer.create(new CustomerName("Alice"), email));

        var command = new RegisterCustomerCommand(new CustomerName("Alice2"), email);

        assertThatThrownBy(() -> sut.execute(command))
            .isInstanceOf(DuplicateEmailException.class);
    }
}
```

### Integration Test Adapters in Isolation

```python
# Test the JPA adapter in isolation — no use case involved
class TestJpaCustomerAdapter:
    @pytest.fixture
    def adapter(self, db_session):
        return JpaCustomerAdapter(db_session)

    def test_saves_and_retrieves_customer(self, adapter):
        customer = Customer.create(
            name=CustomerName("Bob"),
            email=EmailAddress("bob@example.com")
        )
        adapter.save(customer)

        found = adapter.find_by_id(customer.id)

        assert found is not None
        assert found.name == CustomerName("Bob")

    def test_finds_by_email(self, adapter):
        customer = Customer.create(CustomerName("Bob"), EmailAddress("bob@example.com"))
        adapter.save(customer)

        found = adapter.find_by_email(EmailAddress("bob@example.com"))
        assert found.id == customer.id

    def test_returns_none_for_unknown_email(self, adapter):
        result = adapter.find_by_email(EmailAddress("unknown@example.com"))
        assert result is None
```

### Test the Primary Adapter (REST) with Mock Use Case

```kotlin
@WebMvcTest(CustomerRestAdapter::class)
class CustomerRestAdapterTest {

    @MockBean
    private lateinit var registerCustomer: RegisterCustomerUseCase

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Test
    fun `POST customers returns 201 with location header`() {
        val customerId = CustomerId.generate()
        whenever(registerCustomer.execute(any())).thenReturn(customerId)

        mockMvc.post("/customers") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"name": "Alice", "email": "alice@example.com"}"""
        }.andExpect {
            status { isCreated() }
            header { string("Location", containsString(customerId.value)) }
        }
    }

    @Test
    fun `POST customers returns 422 when email already registered`() {
        whenever(registerCustomer.execute(any()))
            .thenThrow(DuplicateEmailException(EmailAddress("alice@example.com")))

        mockMvc.post("/customers") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"name": "Alice", "email": "alice@example.com"}"""
        }.andExpect {
            status { isUnprocessableEntity() }
        }
    }
}
```

---

## Enforcing the Architecture with ArchUnit

```java
// Architecture fitness function — fails the build if rules are violated
class HexagonalArchitectureTest {

    private static final JavaClasses CLASSES = new ClassFileImporter()
        .importPackages("com.example");

    @Test
    void domain_does_not_depend_on_application_or_adapters() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..application..", "..adapters..")
            .check(CLASSES);
    }

    @Test
    void application_does_not_depend_on_adapters() {
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("..adapters..")
            .check(CLASSES);
    }

    @Test
    void adapters_out_only_implement_out_ports() {
        classes()
            .that().resideInAPackage("..adapters.out..")
            .and().implement(JavaClass.Predicates.assignableTo(Object.class))
            .should().implement(
                JavaClass.Predicates.resideInAPackage("..ports.out..")
            )
            .check(CLASSES);
    }
}
```

---

## Anti-Patterns

### 1. Leaking Infrastructure Types into the Domain

```java
// BAD — domain entity knows about JPA
import javax.persistence.*;  // LEAK: JPA in the domain!

@Entity
@Table(name = "customers")
public class Customer {
    @Id @GeneratedValue
    private Long id;    // Long ID is a JPA choice, not a domain choice

    @Column(unique = true)
    private String email;
}

// GOOD — clean domain entity
public class Customer {
    private final CustomerId id;      // domain type
    private EmailAddress email;       // domain value object
    // no JPA imports
}
```

### 2. Defining Ports in the Wrong Place

```java
// BAD — CustomerRepository defined in the infrastructure package
// This means the application core must import from infrastructure!
package com.example.infrastructure.persistence;
public interface CustomerRepository { ... }

// The application service then imports infrastructure:
import com.example.infrastructure.persistence.CustomerRepository;  // WRONG DIRECTION!

// GOOD — CustomerRepository defined in the application core
package com.example.application.ports.out;
public interface CustomerRepository { ... }

// Infrastructure adapter imports from application (correct direction):
import com.example.application.ports.out.CustomerRepository;  // CORRECT
```

### 3. Bypassing the Port

```python
# BAD — use case directly imports the adapter (bypasses the port)
from com.example.adapters.out.persistence.jpa_customer_adapter import JpaCustomerAdapter

class RegisterCustomerService:
    def __init__(self):
        self.repository = JpaCustomerAdapter()  # direct dependency on adapter!

# GOOD — use case depends on the port (injected at wiring time)
class RegisterCustomerService:
    def __init__(self, repository: CustomerRepository):  # port, not adapter
        self.repository = repository
```

### 4. Anemic Ports (Too Many Fine-Grained Methods)

```java
// BAD — port that mirrors the database API
public interface CustomerRepository {
    void insert(Customer c);
    void update(Customer c);
    void delete(CustomerId id);
    Customer selectById(CustomerId id);
    List<Customer> selectAll();
    List<Customer> selectByStatus(String status);
    int count();
    boolean existsByEmail(String email);
    // ... 20 more methods
}

// GOOD — port expresses what the application actually needs
public interface CustomerRepository {
    void save(Customer customer);   // handles insert or update
    Optional<Customer> findById(CustomerId id);
    boolean existsByEmail(EmailAddress email);
}
// Only add methods when a use case actually needs them
```

### 5. Fat Primary Adapters

```kotlin
// BAD — REST adapter contains business logic
@RestController
class CustomerController(private val repository: CustomerRepository) {
    @PostMapping("/customers")
    fun create(@RequestBody request: CreateCustomerRequest): ResponseEntity<*> {
        // Business logic in the adapter!
        if (repository.existsByEmail(EmailAddress(request.email))) {
            return ResponseEntity.status(422).body(mapOf("error" to "Email taken"))
        }
        val customer = Customer.create(CustomerName(request.name), EmailAddress(request.email))
        repository.save(customer)
        return ResponseEntity.status(201).body(mapOf("id" to customer.id.value))
    }
}

// GOOD — adapter is thin, logic is in the use case
@RestController
class CustomerController(private val registerCustomer: RegisterCustomerUseCase) {
    @PostMapping("/customers")
    fun create(@RequestBody request: CreateCustomerRequest): ResponseEntity<*> {
        val id = registerCustomer.execute(RegisterCustomerCommand(
            name = CustomerName(request.name),
            email = EmailAddress(request.email)
        ))
        return ResponseEntity.status(201).body(mapOf("id" to id.value))
    }
}
```

---

## When to Use Hexagonal Architecture

- Applications with complex business logic worth isolating
- Systems with multiple delivery mechanisms (REST + CLI + event consumer)
- Teams doing TDD — the isolation makes unit testing effortless
- When infrastructure is likely to change (switch databases, switch messaging)
- Long-lived systems where maintainability is a priority

## When NOT to Use Hexagonal Architecture

- Simple CRUD applications with little to no business logic
- Prototypes — the overhead kills speed of exploration
- Microservices so small they are essentially one use case with one database call
- Teams not familiar with DI and interface-based design — the pattern requires understanding before it helps

---

## Checklist

### Ports
- [ ] Primary ports (in-ports) are defined inside the application package
- [ ] Secondary ports (out-ports) are defined inside the application package
- [ ] All port interfaces use only domain types — no infrastructure types
- [ ] Ports reflect application needs, not infrastructure capabilities

### Adapters
- [ ] Primary adapters are thin — they translate protocol to command and delegate
- [ ] Primary adapters contain no business logic
- [ ] Secondary adapters implement out-ports and contain no business logic
- [ ] In-memory adapters exist for all secondary ports (for testing)

### Application Core
- [ ] The application core has zero imports from adapter packages
- [ ] The application core has zero imports from framework packages (Spring, Django, etc.)
- [ ] Use cases depend on injected port interfaces, not on concrete adapter classes

### Testing
- [ ] Use cases are unit-tested with in-memory or mock secondary adapters
- [ ] Secondary adapters have integration tests against real infrastructure
- [ ] Primary adapters have tests with mock use case implementations
- [ ] No test requires the full stack to verify a business rule

### Architecture
- [ ] Package structure reflects the hexagonal boundaries
- [ ] Architecture fitness functions (ArchUnit or similar) enforce the rules
- [ ] Adding a new delivery mechanism requires no changes to the application core
- [ ] Swapping a database adapter requires no changes to the application core
