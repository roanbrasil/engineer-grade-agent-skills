---
name: resilience-patterns
description: Production resilience patterns — circuit breaker, retry, bulkhead, timeout, fallback, health checks, chaos engineering, and SLO/error budgets
---

# Resilience Patterns

You are an expert in production resilience engineering. Apply the right combination of patterns for each failure mode, configure them with production-grade parameters, and always define what the system does when things go wrong — before things go wrong.

---

## Failure Mode Taxonomy

```
┌────────────────────────────────────────────────────────────────────┐
│                     Failure Mode Taxonomy                          │
│                                                                    │
│  CRASH FAILURE      Process dies, pod restarts, OOM               │
│  ├── Detection: liveness probe, health endpoint                   │
│  └── Recovery: restart policy, readiness probe gates traffic      │
│                                                                    │
│  OMISSION FAILURE   Request sent, no response (lost packets,      │
│                     process frozen, queue full)                    │
│  ├── Detection: timeout                                           │
│  └── Recovery: retry + circuit breaker                            │
│                                                                    │
│  TIMING FAILURE     Response arrives, but too late                │
│  ├── Detection: timeout threshold                                 │
│  └── Recovery: timeout + fallback + hedged request               │
│                                                                    │
│  BYZANTINE FAILURE  Wrong/corrupted response (data corruption,    │
│                     partial writes, split brain)                  │
│  ├── Detection: response validation, checksums, consensus         │
│  └── Recovery: read repair, quorum reads, circuit breaker        │
└────────────────────────────────────────────────────────────────────┘
```

---

## 1. Circuit Breaker

### State Machine

```
                     failure_rate > threshold
         ┌─────────────────────────────────────────────┐
         │                                             │
         ▼                                             │
   ┌──────────┐         wait_duration_in_open_state    ▼
   │  CLOSED  │ ─────────────────────────────────► ┌──────┐
   │(normal)  │                                     │ OPEN │
   └──────────┘                                     │(fail │
         ▲                                          │ fast)│
         │                                          └──────┘
         │                                              │
         │  success_rate >= threshold                   │ after wait_duration
         │  in half-open                                ▼
         │                                        ┌──────────┐
         └───────────────────────────────────────│ HALF-OPEN │
                                                  │(probe)   │
                                                  └──────────┘
                                                       │
                                          failure in half-open
                                                       │
                                                       ▼
                                                  back to OPEN

CLOSED:    Normal operation. Count failures.
OPEN:      Fail fast (throw exception immediately, no network call).
           Allows recovery time for downstream.
HALF-OPEN: Allow limited test requests through.
           Success → CLOSED. Failure → OPEN again.
```

### Resilience4j Configuration

```yaml
# application.yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        # When to open
        failure-rate-threshold: 50              # open if 50%+ of calls fail
        slow-call-rate-threshold: 80            # open if 80%+ calls are slow
        slow-call-duration-threshold: 2000ms    # threshold for "slow call"
        minimum-number-of-calls: 10             # wait for 10 calls before evaluating

        # Sliding window
        sliding-window-type: COUNT_BASED        # or TIME_BASED
        sliding-window-size: 20                 # last 20 calls (COUNT_BASED)

        # Open state behavior
        wait-duration-in-open-state: 30s        # wait 30s before trying half-open
        automatic-transition-from-open-to-half-open-enabled: true

        # Half-open probing
        permitted-number-of-calls-in-half-open-state: 5

        # What counts as failure
        record-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - feign.RetryableException
        ignore-exceptions:
          - com.example.BusinessException  # don't open CB for business errors
```

```java
// Java usage with Resilience4j
@Service
public class PaymentClient {

    private final CircuitBreaker circuitBreaker;
    private final WebClient webClient;

    public PaymentClient(CircuitBreakerRegistry registry, WebClient webClient) {
        this.circuitBreaker = registry.circuitBreaker("payment-service");
        this.webClient = webClient;
    }

    public PaymentResult charge(ChargeRequest request) {
        return CircuitBreaker.decorateSupplier(circuitBreaker,
            () -> webClient.post()
                .uri("/charges")
                .bodyValue(request)
                .retrieve()
                .bodyToMono(PaymentResult.class)
                .block(Duration.ofSeconds(5))
        ).get();
    }
}

// Or annotation-based
@CircuitBreaker(name = "payment-service", fallbackMethod = "chargeFallback")
public PaymentResult charge(ChargeRequest request) {
    return paymentWebClient.post("/charges", request, PaymentResult.class);
}

private PaymentResult chargeFallback(ChargeRequest request, CallNotPermittedException ex) {
    log.warn("Circuit open for payment-service, using fallback");
    return PaymentResult.deferred(request.getOrderId());
}
```

```python
# Python with tenacity + custom circuit breaker
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30, expected_exception=Exception)
async def call_payment_service(request: dict) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        resp = await client.post("http://payment-service/charges", json=request)
        resp.raise_for_status()
        return resp.json()
```

---

## 2. Retry

### Exponential Backoff + Jitter

```
WITHOUT jitter (thundering herd):
T=0: 100 requests fail simultaneously
T=1: 100 retries hit at the SAME TIME → overload
T=2: 100 retries hit at the SAME TIME → overload again

WITH jitter (spread the load):
T=0:     100 requests fail simultaneously
T=0.5-2: retries spread randomly → no thundering herd

Backoff with full jitter:
  sleep = random_between(0, min(cap, base * 2^attempt))

  attempt 1: random(0, 1s)
  attempt 2: random(0, 2s)
  attempt 3: random(0, 4s)
  attempt 4: random(0, 8s)   ← never exceeds cap
  cap = 30s
```

```yaml
resilience4j:
  retry:
    instances:
      inventory-service:
        max-attempts: 4                          # 1 original + 3 retries
        wait-duration: 500ms                     # base delay
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2.0
        exponential-max-wait-duration: 10s       # cap at 10s
        retry-exceptions:
          - java.io.IOException
          - java.net.ConnectException
          - org.springframework.web.reactive.function.client.WebClientRequestException
        ignore-exceptions:
          - com.example.StockNotFoundException   # don't retry business errors
          - com.example.InvalidRequestException
```

```java
// Java retry with jitter (Resilience4j)
RetryConfig config = RetryConfig.custom()
    .maxAttempts(4)
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
        Duration.ofMillis(500),  // initial interval
        2.0,                     // multiplier
        0.5,                     // randomization factor (±50%)
        Duration.ofSeconds(30)   // max interval
    ))
    .retryOnException(ex ->
        ex instanceof IOException ||
        ex instanceof SocketTimeoutException)
    .build();

Retry retry = Retry.of("inventory-service", config);

// Compose with circuit breaker (retry runs INSIDE circuit breaker check)
Supplier<InventoryResult> decorated = Decorators.ofSupplier(this::callInventory)
    .withCircuitBreaker(circuitBreaker)
    .withRetry(retry)
    .decorate();
```

```python
# Python retry with tenacity
from tenacity import (
    retry, stop_after_attempt, wait_random_exponential,
    retry_if_exception_type, before_sleep_log
)
import logging

@retry(
    stop=stop_after_attempt(4),
    wait=wait_random_exponential(multiplier=0.5, max=30),  # jitter built in
    retry=retry_if_exception_type((httpx.NetworkError, httpx.TimeoutException)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
async def call_inventory_service(request: dict) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        resp = await client.post("http://inventory/reserve", json=request)
        resp.raise_for_status()
        return resp.json()
```

---

## 3. Bulkhead

### Thread Pool Isolation vs Semaphore Isolation

```
Thread Pool Bulkhead (complete isolation):
┌─────────────────────────────────────────────────────┐
│                Application Thread Pool              │
│                                                     │
│  ┌────────────────┐    ┌────────────────┐           │
│  │ Payment Pool   │    │ Inventory Pool │           │
│  │  threads: 10   │    │  threads: 5    │           │
│  │  queue: 20     │    │  queue: 10     │           │
│  └────────────────┘    └────────────────┘           │
│                                                     │
│  Payment slow → only payment pool threads blocked   │
│  Inventory pool unaffected                          │
└─────────────────────────────────────────────────────┘

Semaphore Bulkhead (concurrent call limit, same thread):
  MAX 20 concurrent calls to payment service at any time
  If 21st call arrives: fail fast or wait in queue

USE Thread Pool when: calls are blocking, need timeout per call
USE Semaphore when: calls are async/reactive (no blocking)
```

```yaml
resilience4j:
  bulkhead:
    instances:
      payment-service:
        max-concurrent-calls: 20       # max parallel calls
        max-wait-duration: 100ms       # how long to wait if at limit

  thread-pool-bulkhead:
    instances:
      inventory-service:
        max-thread-pool-size: 10
        core-thread-pool-size: 5
        queue-capacity: 20
        keep-alive-duration: 20ms
```

---

## 4. Rate Limiter

### Token Bucket vs Sliding Window

```
Token Bucket:
  Bucket capacity: 100 tokens
  Refill rate: 10 tokens/second
  
  │████████████████████████│ ← 100 tokens (full)
  Request consumes 1 token
  Allows bursting up to bucket capacity
  
  Good for: allowing brief bursts above sustained rate

Sliding Window:
  Window: 60 seconds, max 100 requests
  Tracks timestamps of last 100 requests
  
  |-- 60s window --|
  [r r r r...100 r]  ← 101st request rejected until oldest exits window
  
  Good for: strict rate limiting with no burst allowance
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      external-api:
        limit-for-period: 100          # 100 calls per refresh period
        limit-refresh-period: 1s       # refresh every second
        timeout-duration: 100ms        # how long to wait for permission
```

```java
@RateLimiter(name = "external-api", fallbackMethod = "rateLimitFallback")
public ExternalData callExternalApi(String id) {
    return externalApiClient.getData(id);
}

private ExternalData rateLimitFallback(String id, RequestNotPermitted ex) {
    return cache.getIfPresent(id)
        .orElseThrow(() -> new ServiceUnavailableException("Rate limit exceeded"));
}
```

---

## 5. Timeout

### Always Set a Timeout

```
WHY INFINITE TIMEOUTS KILL SYSTEMS:
1. Downstream hangs (GC pause, deadlock, network partition)
2. Your thread blocks indefinitely
3. Thread pool exhausted
4. Your service now appears to hang too
5. YOUR upstream service now hangs
6. Cascading failure across entire system

Every call must have a timeout. No exceptions.

Timeout hierarchy:
  HTTP client timeout  <  Service SLA  <  Circuit breaker timeout
  (e.g., 5s)              (e.g., 7s)      (slow-call threshold 4s)
```

```java
// Spring WebClient — always set timeout
webClient.get()
    .uri("/inventory/{id}", id)
    .retrieve()
    .bodyToMono(InventoryItem.class)
    .timeout(Duration.ofSeconds(3))           // reactive timeout
    .onErrorMap(TimeoutException.class, ex ->
        new InventoryTimeoutException("Inventory service timed out after 3s"));

// HttpClient — connect + read timeout
HttpClient httpClient = HttpClient.create()
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)  // 2s connect
    .responseTimeout(Duration.ofSeconds(5));              // 5s read

// Feign client
@FeignClient(name = "inventory-service",
    configuration = InventoryFeignConfig.class)
public interface InventoryClient {
    @GetMapping("/inventory/{id}")
    InventoryItem getItem(@PathVariable String id);
}

@Configuration
class InventoryFeignConfig {
    @Bean
    Request.Options requestOptions() {
        return new Request.Options(
            2, TimeUnit.SECONDS,   // connect timeout
            5, TimeUnit.SECONDS,   // read timeout
            true
        );
    }
}
```

```python
# Python — always explicit timeout
async with httpx.AsyncClient(timeout=httpx.Timeout(
    connect=2.0,   # seconds to establish connection
    read=5.0,      # seconds to receive response
    write=2.0,     # seconds to send request body
    pool=1.0,      # seconds to acquire connection from pool
)) as client:
    resp = await client.get(f"http://inventory/{id}")
```

---

## 6. Fallback

### Fallback Hierarchy (ordered by preference)

```
1. Return cached (stale) data
   └── Best: data exists and staleness is acceptable

2. Return degraded/partial response
   └── Good: can serve something useful without dependency

3. Return empty collection
   └── Acceptable: UI can handle "no items" gracefully

4. Return default/static response
   └── Use when: known safe default exists

5. Fail fast with meaningful error
   └── Use when: cannot safely degrade, better to surface error

NEVER: silently swallow errors or return wrong data
```

```java
@Service
public class RecommendationService {

    @CircuitBreaker(name = "ml-service", fallbackMethod = "getDefaultRecs")
    @Cacheable(value = "recommendations", key = "#userId")
    public List<Product> getPersonalizedRecs(String userId) {
        return mlServiceClient.getRecommendations(userId);
    }

    // Fallback 1: stale cache (Caffeine or Redis)
    private List<Product> getDefaultRecs(String userId, Exception ex) {
        log.warn("ML service unavailable for user {}, using fallback: {}", userId, ex.getMessage());

        // Try stale cache first
        List<Product> stale = cacheManager.getCache("recommendations-stale").get(userId, List.class);
        if (stale != null && !stale.isEmpty()) {
            return stale;
        }

        // Fallback 2: popular items (always available from local DB)
        return productRepository.findTopSellingProducts(20);
    }
}
```

```python
# Python fallback with cache
async def get_recommendations(user_id: str) -> list[dict]:
    try:
        async with httpx.AsyncClient(timeout=3.0) as client:
            resp = await client.get(f"http://ml-service/recs/{user_id}")
            resp.raise_for_status()
            recs = resp.json()
            # Update stale cache on success
            await cache.set(f"recs:stale:{user_id}", recs, ttl=3600)
            return recs
    except (httpx.RequestError, httpx.HTTPStatusError) as e:
        logger.warning("ML service unavailable, using fallback", extra={"error": str(e)})
        # Try stale cache
        stale = await cache.get(f"recs:stale:{user_id}")
        if stale:
            return stale
        # Final fallback: popular items
        return await get_popular_items(limit=20)
```

---

## 7. Hedging

### Issue Duplicate Requests, Take First Response

```
WHEN TO USE:
  - Latency tail is the problem (p99 >> p50)
  - Backend has multiple instances
  - Idempotent read operations (safe to call twice)

HOW IT WORKS:
  T=0ms:   Send request to Instance A
  T=100ms: (A hasn't responded) Send to Instance B
  T=150ms: Instance B responds first → use B's response
  T=200ms: Instance A responds → cancel/ignore
  
  p99 latency ≈ p50 of (A, B) → dramatic tail latency reduction

NEVER USE FOR:
  - Non-idempotent operations (POST that creates a resource)
  - Expensive operations (would double backend load)
```

```java
// Reactive hedging with Project Reactor
public Mono<ProductData> getProductWithHedging(String productId) {
    Mono<ProductData> primary = callProductService("primary", productId);
    Mono<ProductData> secondary = callProductService("secondary", productId)
        .delaySubscription(Duration.ofMillis(100));  // hedge after 100ms

    return Mono.firstWithValue(primary, secondary)
        .timeout(Duration.ofSeconds(3));
}

private Mono<ProductData> callProductService(String instance, String productId) {
    return webClients.get(instance)
        .get()
        .uri("/products/{id}", productId)
        .retrieve()
        .bodyToMono(ProductData.class);
}
```

---

## 8. Health Checks

### Kubernetes Probes

```yaml
# Kubernetes deployment probes
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: my-service
          livenessProbe:
            # Is the process alive? If not, restart the pod.
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30      # wait before first check
            periodSeconds: 10            # check every 10s
            timeoutSeconds: 5
            failureThreshold: 3          # restart after 3 consecutive failures
            successThreshold: 1

          readinessProbe:
            # Is the pod ready to serve traffic? If not, remove from load balancer.
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3          # remove from LB after 3 failures
            successThreshold: 1          # re-add after 1 success

          startupProbe:
            # Is the app still starting up? Disables liveness during startup.
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30         # allow up to 30 * 10s = 300s to start
            periodSeconds: 10
```

```
Probe semantics:
LIVENESS:  Am I healthy enough to continue running?
           Failure → Kubernetes RESTARTS the pod
           Check: is process deadlocked? Is JVM responsive?
           DO NOT: check external dependencies here (DB down ≠ pod should restart)

READINESS: Am I ready to serve requests?
           Failure → Kubernetes REMOVES pod from Service endpoints
           Recovery → pod re-added automatically
           Check: DB connection, cache connection, dependent services

STARTUP:   Am I done starting up?
           Disables liveness probe until startup succeeds
           Use for: slow-starting apps (legacy, large init data loads)
```

```java
// Spring Boot — separate liveness from readiness
@Component
public class DatabaseReadinessIndicator implements HealthIndicator {

    private final DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            conn.prepareStatement("SELECT 1").execute();
            return Health.up()
                .withDetail("database", "reachable")
                .build();
        } catch (SQLException e) {
            return Health.down()
                .withDetail("database", "unreachable")
                .withException(e)
                .build();
        }
    }
}
// Spring Boot automatically groups this under readiness, not liveness
```

---

## 9. Graceful Degradation

```
Define: What does each service do when each dependency is unavailable?

Service Dependency Matrix:
┌──────────────┬─────────────────────────────────────────────────┐
│ Dependency   │ Degraded Behavior                               │
├──────────────┼─────────────────────────────────────────────────┤
│ DB down      │ Serve from read-through cache, accept writes    │
│              │ to durable queue, mark service not-ready        │
├──────────────┼─────────────────────────────────────────────────┤
│ Cache down   │ Serve directly from DB (slower but correct)    │
│              │ log warning, alert, continue                    │
├──────────────┼─────────────────────────────────────────────────┤
│ ML service   │ Return popular items (static fallback)         │
│ down         │                                                 │
├──────────────┼─────────────────────────────────────────────────┤
│ Email down   │ Queue to retry queue, user gets no email       │
│              │ (eventually consistent)                         │
├──────────────┼─────────────────────────────────────────────────┤
│ Auth service │ Short-circuit: reject all requests with 503    │
│ down         │ (cannot safely degrade auth)                   │
└──────────────┴─────────────────────────────────────────────────┘
```

---

## 10. Load Shedding

```
PROBLEM: System receives more requests than it can handle
         Without shedding: all requests slow down, latency spikes, OOM, crash

SOLUTION: Explicitly reject excess requests BEFORE they consume resources

Queue depth check:
  if current_queue_depth > MAX_QUEUE_DEPTH:
      return 503 Service Unavailable immediately
  else:
      enqueue and process

Priority-based shedding:
  ACCEPT always: health checks, admin, critical payments
  SHED first:    analytics, recommendations, reporting
```

```java
@Component
public class LoadSheddingFilter implements Filter {

    private final Semaphore semaphore;

    public LoadSheddingFilter(@Value("${app.max-concurrent-requests:200}") int maxConcurrent) {
        this.semaphore = new Semaphore(maxConcurrent, true);
    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) req;

        // Never shed health checks
        if (httpReq.getRequestURI().startsWith("/actuator/health")) {
            chain.doFilter(req, res);
            return;
        }

        if (!semaphore.tryAcquire()) {
            HttpServletResponse httpRes = (HttpServletResponse) res;
            httpRes.setStatus(503);
            httpRes.setHeader("Retry-After", "5");
            httpRes.getWriter().write("{\"error\": \"Service temporarily overloaded\"}");
            return;
        }
        try {
            chain.doFilter(req, res);
        } finally {
            semaphore.release();
        }
    }
}
```

---

## 11. Chaos Engineering

### Hypothesis-Driven Chaos

```
Chaos Engineering Process:
  ┌─────────────────────────────────────┐
  │  1. DEFINE STEADY STATE             │
  │     Metric: orders/second = 1000    │
  │     Error rate < 0.1%               │
  │     p99 latency < 500ms             │
  └──────────────────┬──────────────────┘
                     │
  ┌──────────────────▼──────────────────┐
  │  2. FORM HYPOTHESIS                 │
  │     "Even if payment service is     │
  │      slow (5s latency), order       │
  │      service maintains throughput   │
  │      via async + circuit breaker"   │
  └──────────────────┬──────────────────┘
                     │
  ┌──────────────────▼──────────────────┐
  │  3. INJECT FAILURE                  │
  │     Increase payment svc latency    │
  │     to 5s using toxiproxy           │
  └──────────────────┬──────────────────┘
                     │
  ┌──────────────────▼──────────────────┐
  │  4. OBSERVE                         │
  │     Dashboard: orders/s maintained? │
  │     Circuit opened after 30s?       │
  │     Alerts fired correctly?         │
  └──────────────────┬──────────────────┘
                     │
  ┌──────────────────▼──────────────────┐
  │  5. LEARN + FIX                     │
  │     Findings → backlog              │
  │     Fix weaknesses found            │
  │     Increase experiment scope       │
  └─────────────────────────────────────┘
```

```bash
# Chaos Mesh (Kubernetes-native)
# Inject network latency on payment service pods
cat <<EOF | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-latency
  namespace: production
spec:
  action: delay
  mode: percentage
  value: "50"                    # affect 50% of pods
  selector:
    labelSelectors:
      "app": "payment-service"
  delay:
    latency: "3000ms"
    jitter: "500ms"
    correlation: "100"
  duration: "5m"
EOF

# Kill pods randomly (like Netflix Chaos Monkey)
cat <<EOF | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: ["production"]
    labelSelectors:
      "tier": "non-critical"
  schedule: "@every 10m"
  duration: "1m"
EOF
```

```python
# Toxiproxy for integration test chaos
import toxiproxy

# In pytest fixtures
@pytest.fixture
def slow_payment_service():
    proxy = toxiproxy.Toxiproxy()
    service = proxy.create(name="payment", upstream="payment-service:8080", listen="0.0.0.0:18080")
    toxic = service.add_toxic("latency", type="latency", attributes={"latency": 5000})
    yield "http://localhost:18080"
    toxic.destroy()
    service.destroy()

def test_circuit_opens_when_payment_is_slow(slow_payment_service):
    client.configure_payment_url(slow_payment_service)
    # make 20 calls
    for _ in range(20):
        result = order_service.create_order(test_order())
        assert result.payment_status in ("pending", "deferred")
    # circuit should now be open
    assert metrics.circuit_breaker_state("payment-service") == "OPEN"
```

---

## 12. SLO / Error Budgets

### Defining SLOs

```
SLI (Service Level Indicator): Measurement (e.g., success rate)
SLO (Service Level Objective): Target (e.g., 99.9% success)
SLA (Service Level Agreement): Contract with penalty for breach

SLI → SLO → Error Budget

SLO: 99.9% of requests succeed over 30 days
Error budget: 0.1% of requests can fail
  = 0.001 × 30 × 24 × 60 = 43.2 minutes of downtime
  or at 100 req/s: 2,592 errors allowed per month

WHEN ERROR BUDGET IS EXHAUSTED:
  → Freeze new feature releases
  → Focus 100% on reliability
  → Post-mortem required
```

```yaml
# Prometheus SLO recording rules
groups:
  - name: slo_rules
    interval: 1m
    rules:
      # Success rate SLI
      - record: job:http_request_success_rate:5m
        expr: |
          sum(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))

      # Latency SLI: p99 < 500ms
      - record: job:http_request_p99_latency:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
          )

      # Error budget burn rate alert
      # Alert if burning budget 14.4x faster than allowed (will exhaust in 2h)
      - alert: ErrorBudgetFastBurn
        expr: |
          (1 - job:http_request_success_rate:5m) > 14.4 * (1 - 0.999)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning fast — will exhaust in ~2h"
          runbook: "https://runbooks.internal/error-budget-burn"
```

```
Error budget burn rate multipliers:
  14.4x burn rate → exhausts in 2 hours   → page immediately
  6x burn rate    → exhausts in 5 hours   → page within 15 min
  3x burn rate    → exhausts in 10 hours  → ticket, investigate today
  1x burn rate    → exhausts in 30 days   → monitor
```

---

## Pattern Composition

```
RIGHT ORDER TO COMPOSE PATTERNS:
(outermost to innermost)

┌─────────────────────────────────────────────────────┐
│ Rate Limiter (protect self from too many requests)  │
│  └─ Circuit Breaker (fail fast if downstream sick)  │
│      └─ Bulkhead (isolate thread pool)              │
│          └─ Timeout (bound the call duration)       │
│              └─ Retry (retry transient failures)    │
│                  └─ Actual Network Call             │
│                                                     │
│ + Fallback (wraps Circuit Breaker, catches open)    │
└─────────────────────────────────────────────────────┘

Resilience4j decorator order:
Decorators.ofSupplier(call)
    .withCircuitBreaker(cb)     ← inner
    .withRetry(retry)           ← runs inside CB
    .withBulkhead(bulkhead)     ← bounds concurrency
    .withRateLimiter(rl)        ← outermost
    .decorate()
```

---

## Checklist

### Per Outbound Call

- [ ] Timeout configured (connect + read, never infinite)
- [ ] Circuit breaker wrapping the call
- [ ] Retry with exponential backoff + jitter (on transient errors only)
- [ ] Retry does NOT retry on 4xx (business errors — retrying won't help)
- [ ] Fallback defined: what does the caller return when this call fails?
- [ ] Metrics: track failure rate, latency percentiles per dependency

### Service-Level

- [ ] Liveness probe: checks only process health, never external deps
- [ ] Readiness probe: checks DB, cache, critical dependencies
- [ ] Startup probe: configured if startup takes > 30s
- [ ] Load shedding: 503 with Retry-After when overloaded
- [ ] Graceful shutdown: drain in-flight requests before terminating

### Resilience4j Configuration

- [ ] `minimum-number-of-calls` set to statistically meaningful number (>= 10)
- [ ] `failure-rate-threshold` tuned to real failure patterns (not 50% default blindly)
- [ ] `wait-duration-in-open-state` long enough for dependency to recover (30s+)
- [ ] `ignore-exceptions` excludes business exceptions (4xx, validation errors)
- [ ] Circuit breaker events monitored in Grafana dashboard

### Chaos Engineering

- [ ] Runbook exists for each dependency failure scenario
- [ ] Chaos experiments documented with hypothesis + expected result
- [ ] Chaos experiments run in staging before production
- [ ] Error budget dashboards visible to team

### SLO / Error Budget

- [ ] SLO defined (success rate %, p99 latency target)
- [ ] Error budget burn rate alerts configured (fast burn + slow burn)
- [ ] Runbook linked in alert annotation
- [ ] Post-mortem process defined for SLO breaches
