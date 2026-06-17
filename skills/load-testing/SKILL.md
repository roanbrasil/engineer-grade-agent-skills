---
name: load-testing
description: Invoked when the user needs to write, run, or interpret load tests with k6 or Gatling — including test type selection, SLA definition, script authoring, threshold configuration, and bottleneck identification.
---

# Load Testing with k6 and Gatling

## Test Type Selection

Choose the right test type before writing a single line of script:

```
Test Type    | VUs / Load         | Duration    | Goal
-------------|--------------------|--------------|-----------------------------------------
Smoke        | 1-2 VUs            | 1-2 min     | Verify script works; baseline
Load         | Expected peak VUs  | 5-60 min    | Verify SLAs at normal/peak load
Stress       | Ramp past capacity | 20-60 min   | Find breaking point, failure behavior
Soak         | Normal load        | 2-24 hours  | Find memory leaks, degradation over time
Spike        | 0 -> 10x -> 0      | 10-30 min   | Test autoscaling and recovery
Breakpoint   | Slow ramp to ∞     | Until break | Find exact capacity limit
```

### Visual: Load Shapes

```
Smoke:                Load:               Stress:
VUs                   VUs                 VUs
|                     |    ___________     |              _____
| *                   |   /           \    |             /
|  *                  |  /             \   |            /
|___________time       |_/_______________   |___________/________time
                               time

Soak:                 Spike:              Breakpoint:
VUs                   VUs                 VUs
|   __________        |    *              |              /
|  /          |       |   * *             |             /
| /           |       |  *   *            |            /
|_/____________       |_*_____*_________  |___________/________time
        time               time
```

---

## k6

### Script Structure

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Rate, Trend, Gauge } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const checkoutDuration = new Trend('checkout_duration');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 50 },   // ramp up to 50 VUs
    { duration: '5m', target: 50 },   // hold at 50 VUs
    { duration: '2m', target: 100 },  // ramp up to 100 VUs
    { duration: '5m', target: 100 },  // hold at 100 VUs
    { duration: '2m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],  // less than 1% error rate
    errors: ['rate<0.01'],
  },
};

// Runs once before VUs start (setup test data, get tokens, etc.)
export function setup() {
  const loginRes = http.post('https://api.example.com/login', {
    username: 'loadtest@example.com',
    password: 'secret',
  });
  return { token: loginRes.json('token') };
}

// The main VU function — runs in a loop per VU
export default function (data) {
  const params = {
    headers: {
      Authorization: `Bearer ${data.token}`,
      'Content-Type': 'application/json',
    },
  };

  const startTime = Date.now();

  // Step 1: Get products
  const productsRes = http.get('https://api.example.com/products', params);
  check(productsRes, {
    'products status is 200': (r) => r.status === 200,
    'products body has items': (r) => r.json('items').length > 0,
  });
  errorRate.add(productsRes.status !== 200);

  // Step 2: Add to cart
  const addRes = http.post(
    'https://api.example.com/cart/items',
    JSON.stringify({ productId: 123, quantity: 1 }),
    params
  );
  check(addRes, { 'add to cart 200': (r) => r.status === 200 });

  // Step 3: Checkout
  const checkoutRes = http.post('https://api.example.com/checkout', null, params);
  check(checkoutRes, { 'checkout 201': (r) => r.status === 201 });
  checkoutDuration.add(Date.now() - startTime);

  sleep(1); // think time between iterations
}

// Runs once after all VUs finish
export function teardown(data) {
  // Cleanup test data if needed
}
```

### VUs vs Iterations

```
VUs (Virtual Users):
- Concurrent simulated users
- Each VU runs the default function in a loop
- Set via options.vus or options.stages
- Good for: "simulate N concurrent users"

Iterations:
- Total number of times the default function runs (across all VUs)
- Set via options.iterations
- Good for: "run this test exactly 1000 times total"
- Use options.vus + options.iterations for fixed total load
```

```javascript
// Fixed iteration count example
export const options = {
  vus: 10,
  iterations: 1000,  // 1000 total iterations, 10 VUs each doing 100
};
```

### Stages: Ramp Up, Hold, Ramp Down

```javascript
export const options = {
  stages: [
    // Smoke test first
    { duration: '30s', target: 2 },
    // Ramp up
    { duration: '5m', target: 100 },
    // Steady state — collect metrics here
    { duration: '10m', target: 100 },
    // Ramp down gracefully
    { duration: '2m', target: 0 },
  ],
};
```

### Thresholds: Define SLA Failure Conditions

The test fails (exit code non-zero) if any threshold is breached:

```javascript
export const options = {
  thresholds: {
    // HTTP request duration
    http_req_duration: [
      'p(50)<100',   // P50 under 100ms
      'p(95)<500',   // P95 under 500ms
      'p(99)<1000',  // P99 under 1 second
    ],
    // Error rate
    http_req_failed: ['rate<0.01'],  // <1% failures

    // Tag-specific thresholds (only checkout endpoint)
    'http_req_duration{name:checkout}': ['p(95)<800'],

    // Custom metrics
    errors: ['rate<0.05'],
    checkout_duration: ['p(95)<2000'],
  },
};
```

Tag requests for per-endpoint thresholds:
```javascript
const checkoutRes = http.post(url, body, {
  tags: { name: 'checkout' },
});
```

### Checks: Per-Request Assertions

Checks do NOT fail the test — they report pass/fail rates. Use thresholds to fail the test.

```javascript
check(response, {
  'status is 200': (r) => r.status === 200,
  'response time OK': (r) => r.timings.duration < 500,
  'body has id': (r) => r.json('id') !== undefined,
  'content-type is JSON': (r) =>
    r.headers['Content-Type'].includes('application/json'),
});
```

### Custom Metrics

```javascript
import { Counter, Gauge, Rate, Trend } from 'k6/metrics';

// Counter: monotonically increasing count
const totalOrders = new Counter('total_orders');
totalOrders.add(1);  // increment

// Gauge: current value (can go up and down)
const activeUsers = new Gauge('active_users');
activeUsers.add(1);

// Rate: percentage of values that are non-zero/truthy
const successRate = new Rate('success_rate');
successRate.add(response.status === 200);

// Trend: statistical analysis of a value
const responseTime = new Trend('custom_response_time');
responseTime.add(response.timings.duration);
```

### Output: InfluxDB + Grafana

```bash
# Run with InfluxDB output
k6 run --out influxdb=http://localhost:8086/k6 script.js

# Prometheus remote write
k6 run --out experimental-prometheus-rw script.js

# JSON output for post-processing
k6 run --out json=results.json script.js
```

### k6 Browser Module (Playwright-based)

```javascript
import { browser } from 'k6/experimental/browser';

export const options = {
  scenarios: {
    browser: {
      executor: 'shared-iterations',
      options: { browser: { type: 'chromium' } },
    },
  },
};

export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto('https://example.com');
    await page.locator('button#submit').click();
    await page.waitForNavigation();
    // measure Core Web Vitals
  } finally {
    await page.close();
  }
}
```

### k6 Cloud vs Self-Hosted Distributed

```bash
# Self-hosted: run from multiple machines with k6 operator (Kubernetes)
# or k6 run on each machine

# k6 Cloud: distributed from Grafana Cloud
k6 cloud script.js

# k6 Cloud with local config
k6 cloud --vus 10000 --duration 5m script.js
```

---

## Gatling

### Scenario DSL

```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class ProductApiSimulation extends Simulation {

  val httpProtocol = http
    .baseUrl("https://api.example.com")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")

  // Feeder: inject dynamic data
  val productFeeder = csv("products.csv").circular
  // products.csv:
  // productId,productName
  // 123,Widget
  // 456,Gadget

  val browseScenario = scenario("Browse Products")
    .feed(productFeeder)
    .exec(
      http("Get Product List")
        .get("/products")
        .check(status.is(200))
        .check(jsonPath("$.items[*].id").findAll.saveAs("productIds"))
    )
    .pause(1, 3)  // think time: 1-3 seconds random
    .exec(
      http("Get Product Detail")
        .get("/products/#{productId}")
        .check(status.is(200))
        .check(jsonPath("$.name").is("#{productName}"))
    )
    .pause(2)

  val checkoutScenario = scenario("Checkout Flow")
    .exec(session => session.set("cartId", java.util.UUID.randomUUID().toString))
    .exec(
      http("Add to Cart")
        .post("/cart/items")
        .body(StringBody("""{"productId": 123, "quantity": 1}"""))
        .check(status.is(200))
    )
    .exec(
      http("Checkout")
        .post("/checkout")
        .body(StringBody("""{"cartId": "#{cartId}"}"""))
        .check(status.is(201))
        .check(jsonPath("$.orderId").saveAs("orderId"))
    )

  setUp(
    browseScenario.inject(
      nothingFor(5.seconds),
      atOnceUsers(10),
      rampUsers(50).during(2.minutes),
      constantUsersPerSec(20).during(5.minutes)
    ),
    checkoutScenario.inject(
      rampUsers(20).during(2.minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile(95).lt(500),
     global.successfulRequests.percent.gt(99),
     forAll.failedRequests.percent.lt(1)
   )
}
```

### Feeders: Dynamic Data Injection

```scala
// CSV feeder
val csvFeeder = csv("data/users.csv").random

// JSON feeder
val jsonFeeder = jsonFile("data/products.json").circular

// Programmatic feeder
val customFeeder = Iterator.continually(Map(
  "userId" -> java.util.UUID.randomUUID().toString,
  "timestamp" -> System.currentTimeMillis()
))

// Use in scenario
scenario("My scenario")
  .feed(csvFeeder)
  .exec(http("request").get("/users/#{userId}"))
```

### Assertions

```scala
setUp(...).assertions(
  // Global assertions
  global.responseTime.mean.lt(200),
  global.responseTime.percentile(95).lt(500),
  global.responseTime.percentile(99).lt(1000),
  global.successfulRequests.percent.gt(99.0),
  global.requestsPerSec.gt(100.0),

  // Per-request assertions
  details("Checkout").responseTime.percentile(95).lt(800),
  details("Get Product List").failedRequests.count.is(0)
)
```

### Running Gatling

```bash
# Maven
mvn gatling:test

# Gradle
gradle gatlingRun

# Standalone
./bin/gatling.sh

# CI: run specific simulation
mvn gatling:test -Dgatling.simulationClass=com.example.ProductApiSimulation
```

HTML report generated at: `target/gatling/results/productapisimulation-*/index.html`

---

## SLA Definition

Define SLAs before writing tests. They should come from business requirements.

```
Metric          | Target           | How to measure
----------------|------------------|----------------------------------
P50 latency     | < 100ms          | k6: p(50)<100, Gatling: mean.lt(100)
P95 latency     | < 500ms          | k6: p(95)<500
P99 latency     | < 1000ms         | k6: p(99)<1000
Error rate      | < 0.1%           | k6: rate<0.001
Throughput      | > 500 RPS        | k6: http_reqs rate
Availability    | > 99.9%          | (1 - error rate) * 100
```

---

## Bottleneck Identification

### CPU-Bound vs Memory-Bound vs I/O-Bound

```
Symptom                              | Likely Bottleneck
-------------------------------------|--------------------------------
CPU at 100%, latency increases       | CPU-bound: optimize algorithms,
                                     | add instances, use async
Response time high, CPU low          | I/O-bound: DB queries, network
                                     | calls, disk reads
OOM errors, GC pauses               | Memory-bound: memory leak,
                                     | insufficient heap, cache too large
Latency spikes at concurrency peaks  | Lock contention, connection pool
                                     | exhaustion, thread pool saturation
```

### Reading k6 Output

```
          ✓ status is 200
          ✗ response time OK
           ↳  87% — 4350 / 5000

     checks.........................: 93.50% ✓ 9350 ✗ 650
     data_received..................: 15 MB  250 kB/s
     data_sent......................: 2.4 MB 40 kB/s
     http_req_blocked...............: avg=1.23ms  min=1µs    med=3µs    max=1.23s   p(90)=5µs   p(95)=6µs
     http_req_connecting............: avg=521µs   min=0s     med=0s     max=1.23s   p(90)=0s    p(95)=0s
   ✓ http_req_duration..............: avg=132ms   min=55ms   med=110ms  max=2.5s    p(90)=210ms p(95)=350ms
   ✗ http_req_duration{name:checkout}: avg=850ms   ...                              p(95)=1.2s
     http_req_failed................: 2.00%  ✓ 98 ✗ 4902
     http_reqs......................: 5000   83.33/s
     vus............................: 50     min=50     max=50
```

Key metrics to read:
- `http_req_duration p(95)` — primary latency SLA metric
- `http_req_failed rate` — error rate
- `http_req_blocked` high = connection pool exhausted, DNS issues
- `http_req_connecting` high = too many new connections (missing keep-alive or pool)
- High difference between `med` and `p(99)` = high variance, investigate tail

### Latency Distribution Shapes

```
Normal (healthy):          Bimodal (two code paths):    Long tail (occasional slow):
     *                         *         *                    ***
    ***                       ***       ***                  *****
   *****                     *****     *****                *******....
__|_____|__time            __|___|_____|__|__              __|_____|________time
  fast  slow                fast      slow                fast    slow outliers
```

---

## Load Test Environment Requirements

```
CRITICAL: Never run load tests against production (unless canary/shadow traffic setup).

Checklist for load test environment:
[ ] Production-like data volume (same table sizes, similar data distribution)
[ ] Same network topology (load balancer, VPN, service mesh as production)
[ ] Same hardware class (same instance types, same CPU/RAM ratio)
[ ] External dependencies mocked or stubbed at expected latency
[ ] Monitoring and APM active (you need to see what breaks)
[ ] Alerts silenced for non-critical (or at least labeled as load test)
[ ] DB is seeded with realistic record counts
[ ] Cache is warm (run a warm-up load test first, then measure)
[ ] Load generator is NOT the bottleneck (k6 machine must have spare CPU/network)
```

---

## Results Interpretation

### Percentiles vs Averages

```
Request times (ms): 50, 51, 52, 53, 54, 55, 56, 57, 58, 5000

Average: 544ms  <- completely misleading; 9 out of 10 users < 58ms
P50:      53ms  <- half of requests are faster than this
P95:     720ms  <- interpolated; 95% of requests faster than this
P99:    4505ms  <- the slow tail; 1% of users experience this

5000ms request = one user's experience
Average hides it. P99 reveals it.
```

### Interpreting Latency Distribution

```
Good distribution (stable system):
  100% |
   95% |.....
   50% |.....
    0% |_____|______|_________ latency
          fast  medium

Bad: High variance (unstable, resource contention):
   99% |                           .
   95% |                     .....
   50% |.....
    0% |_____|_______________|_____ latency
          fast              slow

Bad: Bimodal (two populations, likely cache hit vs miss):
   99% |          .      .
   50% |.    .
    0% |_____|_____| _____|_____ latency
          fast        slow
```

### Soak Test Analysis

```
Memory over time (healthy):                Memory over time (leak):
MB                                         MB
| ----flat line----                        |              /
|                                          |            /
|                                          |          /
|________________________________ time     |________/_____________ time

Latency over time (healthy):               Latency over time (degradation):
ms  ----flat line----                      ms
|                                          |              /
|________________________________ time     |____________/________ time
```

---

## Anti-Patterns

```
WRONG: Running load tests against production without isolation.
RIGHT: Use a dedicated load test environment. Or shadow traffic in production only after
       careful validation.

WRONG: Measuring average latency and calling it an SLA.
RIGHT: Define SLAs using P95 and P99. Report percentiles always.

WRONG: Starting with maximum load immediately (no ramp-up).
RIGHT: Always ramp up. Instant full load doesn't represent real user behavior and can
       cause cache warming effects that skew results.

WRONG: Not warming up before collecting data.
RIGHT: Include warm-up phase in every test. JIT, connection pools, caches need to
       stabilize. Start measuring only after the system reaches steady state.

WRONG: Load testing with unrealistic data (1 user, 3 products in DB).
RIGHT: Seed with production-like data volume. Index behavior, query planner, and
       cache behavior all change with data size.

WRONG: Ignoring the load generator as a bottleneck.
RIGHT: Monitor k6/Gatling machine CPU and network. If load gen CPU > 80%, results
       are invalid — the tool is the bottleneck, not the system under test.

WRONG: Running load tests once and declaring the system "passes."
RIGHT: Run load tests in CI on every significant change. Regressions happen silently.

WRONG: Spike test = just suddenly sending lots of traffic.
RIGHT: Spike test must also measure *recovery*: how long until latency returns to normal?
```

---

## Quick Reference: k6 Recipes

```javascript
// Scenario: Constant RPS (not VUs) — use executor
export const options = {
  scenarios: {
    constant_rps: {
      executor: 'constant-arrival-rate',
      rate: 100,           // 100 iterations per second
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50, // initial VU pool
      maxVUs: 200,         // allow scaling up
    },
  },
};

// Scenario: Multiple user types (browsing vs buying)
export const options = {
  scenarios: {
    browsers: {
      executor: 'constant-vus',
      vus: 80,
      duration: '10m',
      exec: 'browse',
    },
    shoppers: {
      executor: 'constant-vus',
      vus: 20,
      duration: '10m',
      exec: 'checkout',
    },
  },
};

export function browse() { /* browse scenario */ }
export function checkout() { /* checkout scenario */ }
```
