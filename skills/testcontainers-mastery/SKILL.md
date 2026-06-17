---
name: testcontainers-mastery
description: Write integration tests with real Docker containers — PostgreSQL, Kafka, Redis; Java @ServiceConnection; Python pytest fixtures; reuse mode; custom containers; CI configuration
---

# TestContainers Mastery — Expert Reference

## Why TestContainers Beats Mocks

```
┌──────────────────────────────────────────────────────────────────────┐
│                  Testing Strategy Comparison                          │
│                                                                        │
│  Unit tests + mocks:                                                   │
│    ┌────────────┐    mock    ┌───────────────┐                        │
│    │ App Code   │──────────►│ Fake Postgres  │  Fast, but behavioral  │
│    └────────────┘           └───────────────┘  divergence over time   │
│                                                                        │
│  TestContainers:                                                       │
│    ┌────────────┐  TCP/IP   ┌───────────────┐                        │
│    │ App Code   │──────────►│ Real Postgres │  Slower start, but     │
│    └────────────┘           │ in Docker     │  tests real behavior   │
│                             └───────────────┘                         │
│                                                                        │
│  What you get with real containers:                                    │
│    - Real SQL execution (constraints, triggers, generated columns)     │
│    - Real Kafka consumer group rebalancing                             │
│    - Real Redis eviction policies and TTL behavior                     │
│    - Real network timeouts and connection pool behavior                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Java — Core Setup

### Dependencies

```kotlin
// build.gradle.kts
testImplementation("org.testcontainers:testcontainers:1.19.7")
testImplementation("org.testcontainers:junit-jupiter:1.19.7")
testImplementation("org.testcontainers:postgresql:1.19.7")
testImplementation("org.testcontainers:kafka:1.19.7")

// Spring Boot 3.1+ integrations
testImplementation("org.springframework.boot:spring-boot-testcontainers")
testImplementation("org.testcontainers:junit-jupiter:1.19.7")
```

### Basic Container Pattern

```java
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class OrderRepositoryTest {

    // static = SHARED across all tests in this class (started once, stopped after class)
    // non-static = new container per test method (expensive, avoid)
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("orders_test")
        .withUsername("test")
        .withPassword("test");

    @Test
    void shouldSaveAndRetrieveOrder() {
        // Use the container's mapped port — it's a random host port
        String jdbcUrl = postgres.getJdbcUrl();  // jdbc:postgresql://localhost:RANDOM_PORT/orders_test
        DataSource ds = buildDataSource(jdbcUrl, postgres.getUsername(), postgres.getPassword());
        OrderRepository repo = new OrderRepository(ds);

        Order saved = repo.save(new Order("cust-1", OrderStatus.PENDING));
        assertThat(repo.findById(saved.getId())).isPresent();
    }
}
```

### Spring Boot 3.1+ — @ServiceConnection

```java
// Spring Boot 3.1+ auto-wires container directly to DataSource — no @DynamicPropertySource needed
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;

@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {

    @Container
    @ServiceConnection  // wires container URL/user/password into Spring's DataSource
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection  // wires bootstrap.servers into Spring's KafkaProperties
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.6.0")
    );

    @Autowired
    private OrderService orderService;  // uses real Postgres + real Kafka

    @Test
    void shouldPublishEventWhenOrderConfirmed() {
        // full integration: HTTP → service → DB + Kafka, no mocks
        Order order = orderService.createOrder(new CreateOrderRequest("cust-1"));
        orderService.confirmOrder(order.getId());

        // Assert Kafka message was published
        ConsumerRecord<String, String> record = KafkaTestUtils.getSingleRecord(
            consumer, "orders.confirmed", Duration.ofSeconds(10)
        );
        assertThat(record.value()).contains(order.getId());
    }
}
```

### Spring Boot 2.x / Older — @DynamicPropertySource

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("orders")
        .withUsername("test")
        .withPassword("test");

    // Inject container ports into Spring context before beans are created
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldPersistOrder() {
        // ...
    }
}
```

---

## Java — Database Testing

### PostgreSQL with Flyway/Liquibase

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withDatabaseName("orders")
    .withInitScript("db/init.sql");  // runs once at container start

// Flyway runs automatically in @DataJpaTest when on classpath
// Migrations in src/test/resources/db/migration/ run against the container

@DataJpaTest   // starts minimal Spring context (JPA, H2 by default)
@AutoConfigureTestDatabase(replace = NONE)  // disable H2, use TestContainers instead
@Testcontainers
class OrderEntityTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void constraintViolation() {
        // Real DB constraints, not mocked
        assertThatThrownBy(() ->
            orderRepository.save(new Order(null, OrderStatus.PENDING))  // customerId required
        ).hasCauseInstanceOf(ConstraintViolationException.class);
    }
}
```

### Always Pin Image Versions

```java
// BAD — "latest" may break CI when a new Postgres major is released
new PostgreSQLContainer<>("postgres:latest")

// GOOD — pin to minor version; upgrade intentionally
new PostgreSQLContainer<>("postgres:16.2-alpine")
new PostgreSQLContainer<>("postgres:16-alpine")   // pins major, takes minor patches
```

---

## Java — Kafka Testing

```java
import org.testcontainers.kafka.KafkaContainer;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.springframework.kafka.test.utils.KafkaTestUtils;

@SpringBootTest
@Testcontainers
class OrderEventPublisherTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.6.0")
    );

    private KafkaConsumer<String, String> consumer;

    @BeforeEach
    void setUp() {
        // Create a test consumer pointing at the container
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
            kafka.getBootstrapServers(),  // "localhost:RANDOM_PORT"
            "test-group",
            "true"
        );
        consumer = new KafkaConsumer<>(consumerProps,
            new StringDeserializer(), new StringDeserializer());
        consumer.subscribe(List.of("orders.created"));
    }

    @Test
    void shouldPublishOrderCreatedEvent() {
        orderService.createOrder(new CreateOrderRequest("cust-1"));

        ConsumerRecord<String, String> record =
            KafkaTestUtils.getSingleRecord(consumer, "orders.created",
                Duration.ofSeconds(10));

        OrderCreatedEvent event = objectMapper.readValue(record.value(), OrderCreatedEvent.class);
        assertThat(event.getCustomerId()).isEqualTo("cust-1");
    }
}

// @EmbeddedKafka (Spring) vs TestContainers comparison:
//   @EmbeddedKafka:  starts in-process; faster; no Docker required
//                    limited to Spring Kafka features; may miss real Kafka quirks
//   KafkaContainer:  real Kafka binary; slower start (10-20s); requires Docker
//                    tests consumer group rebalancing, partition assignment, etc.
//   Rule: use @EmbeddedKafka for unit-style tests; KafkaContainer for integration tests
```

---

## Java — Custom Containers

```java
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.wait.strategy.Wait;

// Wrap any Docker image
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7.2-alpine")
    .withExposedPorts(6379)
    .withCommand("redis-server", "--requirepass", "testpassword");

@Container
static GenericContainer<?> myService = new GenericContainer<>("my-auth-service:latest")
    .withExposedPorts(8080)
    .withEnv("DATABASE_URL", "jdbc:postgresql://...")
    .withEnv("JWT_SECRET", "test-secret")
    // CRITICAL: wait for actual readiness, not just port open
    .waitingFor(Wait.forHttp("/actuator/health")
        .forStatusCode(200)
        .withStartupTimeout(Duration.ofSeconds(60)));

// After start: get the mapped port
String redisHost = redis.getHost();            // usually "localhost"
Integer redisPort = redis.getMappedPort(6379); // random host port

// Docker Compose for multi-service scenarios
@Container
static DockerComposeContainer<?> compose = new DockerComposeContainer<>(
    new File("src/test/resources/docker-compose.yml")
)
    .withExposedService("postgres", 5432, Wait.forListeningPort())
    .withExposedService("redis", 6379, Wait.forListeningPort())
    .withExposedService("kafka", 9092);
```

---

## Java — Performance Patterns

### Reuse Mode (fastest local dev)

```java
// Container survives between test runs — not stopped after JVM exits
// Ryuk daemon skips this container; same container reused next run

@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withReuse(true);   // keep alive between runs

// Required: create or edit ~/.testcontainers.properties
// testcontainers.reuse.enable=true
```

```properties
# ~/.testcontainers.properties
testcontainers.reuse.enable=true
```

```bash
# To clean up reused containers manually
docker ps --filter "label=org.testcontainers.sessionId" -q | xargs docker stop
```

### Shared Container Base Class

```java
// Shared single container across multiple test classes
// Declared once in a base class; all test classes extend it
public abstract class IntegrationTestBase {

    static final PostgreSQLContainer<?> POSTGRES;
    static final KafkaContainer KAFKA;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("orders")
            .withReuse(true);
        KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
            .withReuse(true);

        // Start in parallel
        Startables.deepStart(POSTGRES, KAFKA).join();
    }
}

// Each test class extends the base
@SpringBootTest
class OrderServiceTest extends IntegrationTestBase { /* ... */ }

@SpringBootTest
class PaymentServiceTest extends IntegrationTestBase { /* ... */ }
// Both classes share the same running containers
```

---

## Python — Core Patterns

### Installation

```bash
pip install testcontainers[postgres,kafka,redis]
# or
pip install testcontainers        # core + all extras
```

### pytest Fixtures

```python
# conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.kafka import KafkaContainer
from testcontainers.redis import RedisContainer

# scope="session" = one container for the entire test session (fastest)
# scope="module"  = one container per test module
# scope="function"= one container per test function (slowest, most isolated)

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def kafka_container():
    with KafkaContainer("confluentinc/cp-kafka:7.6.0") as kafka:
        yield kafka

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7.2-alpine") as redis:
        yield redis

# Use containers in fixtures that build clients/sessions
@pytest.fixture(scope="session")
def db_session(postgres_container):
    engine = create_engine(postgres_container.get_connection_url())
    Base.metadata.create_all(engine)    # run migrations
    with Session(engine) as session:
        yield session

@pytest.fixture(scope="session")
def kafka_producer(kafka_container):
    from kafka import KafkaProducer
    return KafkaProducer(
        bootstrap_servers=kafka_container.get_bootstrap_server()
    )
```

### Test Examples

```python
# test_order_repository.py
def test_save_and_retrieve_order(db_session):
    repo = OrderRepository(db_session)
    order = Order(customer_id="cust-1", status=OrderStatus.PENDING)
    saved = repo.save(order)

    retrieved = repo.find_by_id(saved.id)
    assert retrieved.customer_id == "cust-1"
    assert retrieved.status == OrderStatus.PENDING

def test_unique_constraint(db_session):
    repo = OrderRepository(db_session)
    # Real constraint violation — not mocked
    repo.save(Order(id="ord-1", customer_id="cust-1"))
    with pytest.raises(IntegrityError):
        repo.save(Order(id="ord-1", customer_id="cust-2"))  # duplicate pk

def test_kafka_event_published(kafka_container, order_service):
    from kafka import KafkaConsumer

    consumer = KafkaConsumer(
        "orders.created",
        bootstrap_servers=kafka_container.get_bootstrap_server(),
        auto_offset_reset="earliest",
        value_deserializer=lambda m: json.loads(m.decode("utf-8"))
    )

    order_service.create_order(customer_id="cust-1")

    message = next(consumer)  # blocks until message arrives
    assert message.value["customer_id"] == "cust-1"
```

### Custom Container in Python

```python
from testcontainers.core.container import DockerContainer
from testcontainers.core.waiting_utils import wait_for_logs

@pytest.fixture(scope="session")
def my_service():
    container = DockerContainer("my-service:latest")
    container.with_exposed_ports(8080)
    container.with_env("ENV", "test")
    container.with_bind_ports(8080, 18080)   # optional: pin host port

    with container:
        # Wait until the service logs "Started MyServiceApplication"
        wait_for_logs(container, "Started MyServiceApplication", timeout=60)
        yield container

def test_health(my_service):
    import requests
    port = my_service.get_exposed_port(8080)
    resp = requests.get(f"http://localhost:{port}/actuator/health")
    assert resp.status_code == 200
```

---

## Wait Strategies — Never Use sleep()

```java
// BAD: fixed sleep — fragile in CI, slow in dev
container.start();
Thread.sleep(5000);  // hope it's up by now

// GOOD: proper readiness strategies

// Wait for port to accept connections (TCP handshake)
.waitingFor(Wait.forListeningPort())

// Wait for HTTP endpoint to return specific status
.waitingFor(Wait.forHttp("/health").forStatusCode(200))

// Wait for HTTPS endpoint
.waitingFor(Wait.forHttps("/health").forStatusCode(200).allowInsecure())

// Wait for specific log message
.waitingFor(Wait.forLogMessage(".*database system is ready.*", 1))

// Wait for log + custom timeout
.waitingFor(Wait.forLogMessage(".*ready to accept connections.*", 1)
    .withStartupTimeout(Duration.ofSeconds(120)))

// Composite: multiple conditions
.waitingFor(new WaitAllStrategy()
    .withStrategy(Wait.forListeningPort())
    .withStrategy(Wait.forLogMessage(".*ready.*", 1)))
```

---

## CI Configuration

```yaml
# GitHub Actions
name: Integration Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest   # Docker available by default
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }

      - name: Run integration tests
        run: ./gradlew integrationTest
        env:
          TESTCONTAINERS_RYUK_DISABLED: "false"   # let Ryuk clean up
          # DOCKER_HOST is set automatically on ubuntu-latest runners

      - name: Run tests (Docker-in-Docker)
        # If using a container-based runner that needs DinD:
        run: ./gradlew integrationTest
        env:
          DOCKER_HOST: tcp://docker:2375
          TESTCONTAINERS_HOST_OVERRIDE: docker  # tell TC to use "docker" hostname
```

```yaml
# GitLab CI with Docker-in-Docker
integration-tests:
  image: eclipse-temurin:21
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    TESTCONTAINERS_HOST_OVERRIDE: docker
  script:
    - ./gradlew integrationTest
```

---

## Production Checklist

```
Container setup:
  [ ] Image version pinned (postgres:16-alpine, not postgres:latest)
  [ ] @Container field is static (shared across tests in class, not per-method)
  [ ] waitingFor() configured — no Thread.sleep() or fixed delays
  [ ] Resource limits set for containers in CI (avoid OOM in constrained runners)

Java specific:
  [ ] @Testcontainers on every test class using @Container
  [ ] Spring Boot 3.1+: use @ServiceConnection
  [ ] Spring Boot <3.1: use @DynamicPropertySource
  [ ] @DataJpaTest + @AutoConfigureTestDatabase(replace=NONE) for JPA slice tests
  [ ] Flyway/Liquibase migrations run in test container (not H2 dialect)

Python specific:
  [ ] scope="session" on all container fixtures (one container per test run)
  [ ] conftest.py at project root for shared fixtures
  [ ] wait_for_logs or HTTP health check before yielding container

Performance:
  [ ] withReuse(true) + testcontainers.reuse.enable=true for local dev
  [ ] Parallel container start: Startables.deepStart(c1, c2).join()
  [ ] One container per test suite (not per test method)
  [ ] CI: reuse disabled (TESTCONTAINERS_RYUK_DISABLED=false keeps cleanup working)

CI:
  [ ] Docker available (ubuntu-latest in GitHub Actions works out of the box)
  [ ] DinD configured when using container-based CI runners
  [ ] TESTCONTAINERS_HOST_OVERRIDE set when host is not localhost
```

---

## Anti-Patterns

```
WRONG: New container per test method
  @Test void test1() { try (var pg = new PostgreSQLContainer<>(...)) { ... } }
  @Test void test2() { try (var pg = new PostgreSQLContainer<>(...)) { ... } }
  // Docker container start per test = 5-10s overhead each = test suite takes minutes
RIGHT: static @Container or session-scoped fixture — start once, share across all tests.

WRONG: Using latest tag
  new PostgreSQLContainer<>("postgres:latest")
  // postgres:latest changed from 15 to 16 and your JSON column syntax broke
RIGHT: new PostgreSQLContainer<>("postgres:16-alpine") — upgrade intentionally.

WRONG: Fixed sleep instead of waitingFor()
  container.start();
  Thread.sleep(3000);  // works on dev laptop; fails in slow CI
RIGHT: .waitingFor(Wait.forLogMessage(".*ready.*", 1))

WRONG: H2 in-memory DB for JPA tests
  @DataJpaTest  // default: replaces datasource with H2
  // H2 has dialect differences; constraints may not match Postgres
RIGHT: @AutoConfigureTestDatabase(replace = NONE) + PostgreSQL TestContainer.

WRONG: Sharing container across unrelated test classes via global state without reuse
  // Container may already be stopped by Ryuk when second class starts
RIGHT: Use base class with static block + Startables.deepStart(), or reuse mode.

WRONG: Ryuk disabled in CI permanently
  TESTCONTAINERS_RYUK_DISABLED=true in CI
  // Containers accumulate; runner runs out of resources
RIGHT: Only disable Ryuk locally with reuse mode; keep it enabled in CI.

WRONG: Not setting TESTCONTAINERS_HOST_OVERRIDE in DinD CI
  Container starts on "docker" host but code connects to "localhost"
  // Connection refused
RIGHT: TESTCONTAINERS_HOST_OVERRIDE=docker tells TestContainers to use that hostname.
```
