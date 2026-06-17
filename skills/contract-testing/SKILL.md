---
name: contract-testing
description: Invoked when the user needs to implement, debug, or architect consumer-driven contract testing with Pact between microservices — including broker setup, CI integration, and provider verification.
---

# Consumer-Driven Contract Testing with Pact

## The Problem with Integration Tests

```
Traditional integration test approach:
+----------+    +----------+    +----------+    +---------+
| Service A| -> | Service B| -> | Service C| -> |Database |
+----------+    +----------+    +----------+    +---------+
     All must be running. Slow. Brittle. Flaky.
     One service down = all tests fail.
     Hard to run locally. Hard to parallelize.
```

Contract testing decouples this:
```
Consumer test:   Consumer <-> Mock Provider  (fast, isolated)
Provider test:   Real Provider <-> Contract  (no consumer needed)
```

---

## Consumer-Driven Contracts: The Core Idea

The **consumer** defines exactly what it needs from the provider. The **provider** verifies it can fulfill those needs. The consumer is in control — not the provider.

```
+----------+  1. Write interaction  +----------+
| Consumer |----------------------> | Pact DSL |
+----------+                        +----------+
                                         |
                                  2. Generate contract
                                         |
                                         v
                                  +-------------+
                                  | pact.json   |  (contract file)
                                  +-------------+
                                         |
                                  3. Publish to broker
                                         |
                                         v
                                  +-------------+
                                  | Pact Broker |
                                  +-------------+
                                         |
                                  4. Provider pulls + verifies
                                         |
                                         v
                                  +----------+
                                  | Provider |  (real code, no consumer)
                                  +----------+
```

---

## Pact Workflow Step by Step

### Step 1: Consumer writes the interaction

The consumer writes a test that:
1. Defines what request it will make
2. Defines what response it expects (minimum, not maximum)
3. Runs the consumer code against a Pact mock server
4. Produces a `pact.json` contract file

### Step 2: Contract file (JSON) is generated

```json
{
  "consumer": { "name": "order-service" },
  "provider": { "name": "product-service" },
  "interactions": [
    {
      "description": "a request for product 123",
      "providerState": "product 123 exists",
      "request": {
        "method": "GET",
        "path": "/products/123"
      },
      "response": {
        "status": 200,
        "body": {
          "id": 123,
          "name": "Widget",
          "price": 9.99
        },
        "matchingRules": {
          "body": {
            "$.id": { "matchers": [{ "match": "type" }] },
            "$.price": { "matchers": [{ "match": "decimal" }] }
          }
        }
      }
    }
  ]
}
```

### Step 3: Publish to Pact Broker

```bash
pact-broker publish ./pacts \
  --consumer-app-version $(git rev-parse HEAD) \
  --broker-base-url https://broker.example.com \
  --tag main
```

### Step 4: Provider verifies

Provider loads the contract and replays each request against the real running code, verifying responses match the contract.

---

## Pact DSL

### `given` — Provider State

Sets up preconditions. Tells the provider what test data to create.

```
given("product 123 exists")
```

### `uponReceiving` — Interaction Description

Human-readable description of the interaction. Must be unique within a pact.

```
uponReceiving("a request for product 123")
```

### `willRespondWith` — Expected Response

What the consumer expects in return. Use matching rules, not exact values.

---

## Java Implementation

### Consumer test (JUnit 5)

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service")
class ProductServiceClientPactTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact productExists(PactDslWithProvider builder) {
        return builder
            .given("product 123 exists")
            .uponReceiving("a request for product 123")
                .path("/products/123")
                .method("GET")
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                    .integerType("id", 123)
                    .stringType("name", "Widget")
                    .decimalType("price", 9.99)
                )
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "productExists")
    void shouldFetchProduct(MockServer mockServer) {
        // Point your real client at the mock server
        ProductServiceClient client = new ProductServiceClient(mockServer.getUrl());
        
        Product product = client.getProduct(123L);
        
        assertThat(product.getId()).isEqualTo(123L);
        assertThat(product.getName()).isEqualTo("Widget");
    }
}
```

### Provider verification (Spring Boot)

```java
@Provider("product-service")
@PactBroker(
    url = "${PACT_BROKER_URL}",
    authentication = @PactBrokerAuth(token = "${PACT_BROKER_TOKEN}")
)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServiceProviderPactTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // Provider state handlers — set up test data
    @State("product 123 exists")
    void productExists() {
        // Insert test product into test DB or mock repository
        productRepository.save(new Product(123L, "Widget", BigDecimal.valueOf(9.99)));
    }

    @State("product 123 does not exist")
    void productDoesNotExist() {
        productRepository.deleteById(123L);
    }
}
```

---

## Python Implementation

### Consumer test (pact-python)

```python
import pytest
from pact import Consumer, Provider

@pytest.fixture(scope="session")
def pact():
    pact = Consumer("order-service").has_pact_with(
        Provider("product-service"),
        pact_dir="./pacts"
    )
    pact.start_service()
    yield pact
    pact.stop_service()


def test_get_product(pact):
    expected_body = {
        "id": 123,
        "name": Like("Widget"),       # type match, not value match
        "price": Like(9.99),
    }

    (pact
     .given("product 123 exists")
     .upon_receiving("a request for product 123")
     .with_request("GET", "/products/123")
     .will_respond_with(200, body=expected_body))

    with pact:
        client = ProductServiceClient(pact.uri)
        product = client.get_product(123)
        assert product["id"] == 123
```

---

## Kotlin Implementation (Spring Boot Provider)

```kotlin
@Provider("product-service")
@PactBroker
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductProviderPactTest {

    @LocalServerPort
    private var port: Int = 0

    @Autowired
    private lateinit var productRepository: ProductRepository

    @BeforeEach
    fun setUp(context: PactVerificationContext) {
        context.target = HttpTestTarget("localhost", port)
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider::class)
    fun verifyPact(context: PactVerificationContext) {
        context.verifyInteraction()
    }

    @State("product 123 exists")
    fun `product 123 exists`() {
        productRepository.save(Product(id = 123L, name = "Widget", price = BigDecimal("9.99")))
    }
}
```

---

## Matching Rules

Use matchers instead of exact values wherever possible. Exact values make contracts fragile.

```
+---------------+--------------------------------------+---------------------------+
| Matcher       | When to use                          | Example                   |
+---------------+--------------------------------------+---------------------------+
| Exact match   | IDs, enum values, status codes       | "status": "ACTIVE"        |
| Type match    | Any string, any number               | stringType("name", "any") |
| Regex match   | Formatted strings (dates, UUIDs)     | /^\d{4}-\d{2}-\d{2}$/    |
| each-like     | Arrays — verifies all items match    | eachLike({"id": 1})       |
| integer       | Must be an integer                   | integerType("count", 0)   |
| decimal       | Must be a number with decimal        | decimalType("price", 0.0) |
+---------------+--------------------------------------+---------------------------+
```

Java DSL examples:
```java
new PactDslJsonBody()
    .stringType("name")              // any string
    .integerType("count")            // any integer
    .decimalType("price")            // any decimal
    .stringMatcher("date", "\\d{4}-\\d{2}-\\d{2}", "2024-01-15")  // regex
    .minArrayLike("items", 1)        // array with at least 1 item
        .integerType("id")
        .stringType("name")
        .closeArray()
```

---

## Pact Broker: Publishing and Verification

```
+----------+   publish pact   +-------------+   pull pact    +----------+
| Consumer |----------------> | Pact Broker | <------------- | Provider |
+----------+   (on CI/CD)     |   (storage) |   (on CI/CD)  +----------+
                              +-------------+
                                    |
                              can-i-deploy
                              (gate before
                               deploy)
```

### can-i-deploy

This is the key CI gate. Run before deploying any service:

```bash
# Is order-service@abc123 compatible with what's currently deployed as product-service?
pact-broker can-i-deploy \
  --pacticipant order-service \
  --version $(git rev-parse HEAD) \
  --to-environment production \
  --broker-base-url https://broker.example.com

# Exit code 0 = can deploy; non-zero = contract broken, block deploy
```

### Webhook: trigger provider build on new contract

Configure in Pact Broker UI or API:
```json
{
  "events": [{ "name": "contract_content_changed" }],
  "request": {
    "method": "POST",
    "url": "https://ci.example.com/trigger/product-service-build",
    "headers": { "Content-Type": "application/json" }
  }
}
```

---

## Message Pact (Async / Kafka / SQS)

For event-driven systems where there's no HTTP request/response:

```java
// Consumer: verify it can process this message format
@Pact(consumer = "inventory-service")
public MessagePact orderPlacedEvent(MessagePactBuilder builder) {
    return builder
        .given("an order placed event")
        .expectsToReceive("an order placed event")
        .withContent(new PactDslJsonBody()
            .stringType("orderId")
            .integerType("customerId")
            .array("items")
                .object()
                    .integerType("productId")
                    .integerType("quantity")
                .closeObject()
            .closeArray()
        )
        .toPact();
}

@Test
@PactTestFor(pactMethod = "orderPlacedEvent", providerType = ProviderType.ASYNCH)
void shouldProcessOrderPlacedEvent(List<Message> messages) {
    OrderPlacedEvent event = objectMapper.readValue(
        messages.get(0).contentsAsString(), OrderPlacedEvent.class
    );
    // assert handler can process it
    inventoryHandler.handle(event);
}
```

```java
// Provider: verify it publishes this format
@State("an order placed event")
public Map<String, Object> orderPlacedState() {
    // Return the actual message body the provider would publish
    return Map.of(
        "orderId", "order-456",
        "customerId", 789,
        "items", List.of(Map.of("productId", 123, "quantity", 2))
    );
}
```

---

## Pact vs OpenAPI

These are **complementary**, not alternatives:

```
+------------------+----------------------------------+----------------------------------+
|                  | Pact                             | OpenAPI                          |
+------------------+----------------------------------+----------------------------------+
| What it tests    | Behavior (specific interactions) | Shape (all possible endpoints)   |
| Who defines it   | Consumer                         | Provider                         |
| Test execution   | Automated CI gate                | Lint/validate tools              |
| Runtime check    | Yes (verified against real code) | No (documentation only)          |
| Versioning       | Per consumer-provider pair       | Single document                  |
| Use for          | CI contract gate                 | API documentation, codegen       |
+------------------+----------------------------------+----------------------------------+
```

Use both: OpenAPI for documentation and schema validation, Pact for behavioral CI gates.

---

## PactFlow: Bi-Directional Contract Testing

When you can't run provider verification tests (e.g., third-party APIs):

```
Consumer publishes Pact contract  --+
                                    +--> PactFlow compares  --> can-i-deploy
Provider publishes OpenAPI spec   --+
```

The provider uploads their OpenAPI spec; PactFlow validates that the spec satisfies the consumer contracts without running code.

---

## CI/CD Integration

```yaml
# GitHub Actions example
jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew test  # runs Pact consumer tests, generates pacts/
      - run: |
          pact-broker publish ./pacts \
            --consumer-app-version ${{ github.sha }} \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --tag ${{ github.ref_name }}

  can-i-deploy:
    needs: consumer-tests
    runs-on: ubuntu-latest
    steps:
      - run: |
          pact-broker can-i-deploy \
            --pacticipant order-service \
            --version ${{ github.sha }} \
            --to-environment production \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }}
```

---

## Anti-Patterns

```
WRONG: Testing business logic in contracts.
       "given user has premium subscription, when checkout, then apply 20% discount"
RIGHT: Contracts test HTTP interaction format, not domain rules.
       "given a valid cart, when POST /orders, then return 201 with order ID"

WRONG: Too many provider states that mirror all business scenarios.
RIGHT: Minimum states needed for consumer interactions. States are setup only, not logic.

WRONG: Hardcoding values instead of using matchers.
       .body("{\"name\": \"Widget\"}")  <- breaks if provider test data differs
RIGHT: .body(new PactDslJsonBody().stringType("name", "Widget"))

WRONG: Consumer defines what the provider *should* return (wish list).
RIGHT: Consumer defines the minimum it *needs* to function (subset).

WRONG: Never running can-i-deploy before deploying.
RIGHT: can-i-deploy is a hard gate in every deployment pipeline.

WRONG: Keeping contracts forever without cleaning up old interactions.
RIGHT: Review and prune contracts when consumer code no longer needs an interaction.

WRONG: Using Pact to test internal services that share a codebase.
RIGHT: Pact is for services with separate deployment cycles and separate teams.
```

---

## Checklist: Setting Up Pact from Scratch

```
Consumer side:
[ ] Add pact-jvm (or pact-python, pact-js) dependency
[ ] Write at least one consumer test with @Pact annotation
[ ] Confirm pact JSON is generated in ./pacts/ on test run
[ ] Add pact publish step in CI after consumer tests pass
[ ] Tag pacts with branch name and git SHA

Provider side:
[ ] Add pact-jvm provider dependency
[ ] Write @State methods for every provider state in consumer pacts
[ ] Configure PactBroker URL in provider test
[ ] Confirm provider verification passes locally
[ ] Add provider verification step in CI

CI/CD gates:
[ ] can-i-deploy runs before every deployment (consumer and provider)
[ ] Broker webhooks trigger provider builds when consumer contracts change
[ ] Deploy only if can-i-deploy exits 0

Broker:
[ ] Pact Broker is running (self-hosted or PactFlow)
[ ] Authentication configured (token or basic auth)
[ ] Environments registered (development, staging, production)
[ ] Record deployments when services are deployed: pact-broker record-deployment
```
