---
name: observability-excellence
description: Design and implement production observability — logs, metrics, traces, alerting, dashboards, profiling, and error tracking using OpenTelemetry and modern tooling
---

# Observability Excellence

## The Three Pillars (and Why You Need All Three)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OBSERVABILITY TRIAD                                  │
│                                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐     │
│   │    LOGS      │    │   METRICS    │    │        TRACES            │     │
│   │              │    │              │    │                          │     │
│   │ What happened│    │ How much /   │    │ How did a request flow   │     │
│   │ at a specific│    │ how often /  │    │ across services?         │     │
│   │ time         │    │ how fast?    │    │                          │     │
│   │              │    │              │    │ TraceID: abc123          │     │
│   │ Discrete     │    │ Aggregated   │    │  └── Span: API Gateway   │     │
│   │ events       │    │ over time    │    │       └── Span: Service A│     │
│   │              │    │              │    │            └── Span: DB  │     │
│   └──────────────┘    └──────────────┘    └──────────────────────────┘     │
│                                                                             │
│   Tells you WHAT      Tells you HOW        Tells you WHERE                 │
│   happened            the system behaves   time was spent                  │
└─────────────────────────────────────────────────────────────────────────────┘

  None alone is sufficient:
  - Logs without metrics → can't see trends, high storage cost
  - Metrics without logs → you know something broke, not why
  - Traces without logs → missing business context inside spans
```

**The key insight:** Each pillar answers a different question. A 500 error in metrics tells you *something is broken*. Logs tell you *which request failed and why*. Traces tell you *which downstream service caused it and how long each hop took*.

---

## Pillar 1: Structured Logging

### JSON Logs — The Standard

```
BAD (unstructured):
  [2024-01-15 10:23:41] ERROR Failed to process order 12345 for user john@example.com

GOOD (structured JSON):
  {
    "timestamp": "2024-01-15T10:23:41.123Z",
    "level": "ERROR",
    "message": "Failed to process order",
    "service": "order-service",
    "version": "1.4.2",
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "span_id": "00f067aa0ba902b7",
    "order_id": "12345",
    "user_id": "usr_8x4k2m",          ← use opaque IDs, not PII
    "error_type": "InsufficientStock",
    "error_code": "STOCK_001",
    "duration_ms": 143
  }
```

### Log Levels — When to Use Each

```
┌───────────┬────────────────────────────────┬─────────────────────────────────┐
│   Level   │         Use When               │          Example                │
├───────────┼────────────────────────────────┼─────────────────────────────────┤
│   ERROR   │ Operation failed, needs        │ DB connection refused           │
│           │ immediate attention            │ Payment processor returned 500  │
├───────────┼────────────────────────────────┼─────────────────────────────────┤
│   WARN    │ Unexpected but recoverable;    │ Retry attempt 2/3               │
│           │ degraded behavior              │ Cache miss rate elevated        │
├───────────┼────────────────────────────────┼─────────────────────────────────┤
│   INFO    │ Normal operational events,     │ Order created successfully      │
│           │ business milestones            │ Service started on port 8080    │
├───────────┼────────────────────────────────┼─────────────────────────────────┤
│   DEBUG   │ Developer troubleshooting,     │ Entering method processOrder()  │
│           │ disabled in production         │ Cache key: order:12345          │
├───────────┼────────────────────────────────┼─────────────────────────────────┤
│   TRACE   │ Very fine-grained; loop        │ Processing item 3 of 47         │
│           │ iterations, SQL queries        │ SQL: SELECT * FROM orders...    │
└───────────┴────────────────────────────────┴─────────────────────────────────┘
```

### Correlation IDs and Request Context Propagation

```
Request enters → Generate/extract trace_id → Propagate via MDC/context

Java/Spring (SLF4J + MDC):
  Filter sets: MDC.put("trace_id", traceId);
               MDC.put("request_id", requestId);
               MDC.put("user_id", userId);
  Logback pattern: {"trace_id":"%X{trace_id}", "level":"%level", ...}
  ALL downstream log calls automatically include these fields

Python (structlog):
  structlog.contextvars.bind_contextvars(
      trace_id=trace_id,
      request_id=request_id,
  )
  # All subsequent log calls in this request include trace_id

Async propagation: use ThreadLocal → AsyncLocal → context vars
  Never lose the trace_id when crossing thread boundaries
```

### What NOT to Log

```
┌─────────────────────────────────────────────────────────────────┐
│  NEVER LOG                        WHY                           │
├─────────────────────────────────────────────────────────────────┤
│  Passwords, tokens, API keys      Secrets exposure              │
│  Credit card numbers, CVV         PCI-DSS violation             │
│  Social security numbers          GDPR/privacy violation        │
│  Email addresses, full names      PII — log opaque user_id      │
│  Health data, DOB                 HIPAA violation               │
│  Full request/response bodies     May contain any of the above  │
│  Stack traces in user responses   Information disclosure        │
└─────────────────────────────────────────────────────────────────┘

  Rule: Log IDs and codes, not values
  Log: user_id="usr_8x4k2m"      NOT: email="john@company.com"
  Log: payment_method_id="pm_3"  NOT: card_number="4111..."
```

### Code Example: Structured Logging in Java/Spring

```java
// logback-spring.xml — JSON output
// Add dependency: net.logstash.logback:logstash-logback-encoder

@Component
@Slf4j
public class OrderService {

    public Order processOrder(String orderId, String userId) {
        // Bind context once at entry point
        MDC.put("order_id", orderId);
        MDC.put("user_id", userId);  // opaque ID, not email

        log.info("Processing order",
            StructuredArguments.keyValue("order_id", orderId),
            StructuredArguments.keyValue("status", "started"));

        try {
            var result = inventoryClient.reserve(orderId);

            log.info("Order processed successfully",
                StructuredArguments.keyValue("duration_ms", timer.elapsed()),
                StructuredArguments.keyValue("items_reserved", result.itemCount()));

            return result;

        } catch (InsufficientStockException e) {
            // WARN not ERROR — business exception, not system failure
            log.warn("Insufficient stock for order",
                StructuredArguments.keyValue("sku", e.getSku()),
                StructuredArguments.keyValue("requested", e.getRequested()),
                StructuredArguments.keyValue("available", e.getAvailable()));
            throw e;
        } catch (Exception e) {
            // ERROR — system failure
            log.error("Unexpected error processing order",
                StructuredArguments.keyValue("error_type", e.getClass().getSimpleName()),
                e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

### Code Example: Structured Logging in Python/FastAPI

```python
# pip install structlog
import structlog
import structlog.contextvars

logger = structlog.get_logger()

# Configure once at startup
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ]
)

# FastAPI middleware
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        trace_id=request.headers.get("X-Trace-ID", generate_id()),
        path=request.url.path,
        method=request.method,
    )
    response = await call_next(request)
    logger.info("request_completed", status_code=response.status_code)
    return response

# In service layer — context is automatically included
async def process_order(order_id: str, user_id: str):
    structlog.contextvars.bind_contextvars(order_id=order_id)
    log = logger.bind(user_id=user_id)

    log.info("processing_order_started")
    try:
        result = await inventory.reserve(order_id)
        log.info("order_processed", items_reserved=result.item_count)
        return result
    except InsufficientStockError as e:
        log.warning("insufficient_stock", sku=e.sku, available=e.available)
        raise
    except Exception as e:
        log.error("order_processing_failed", error=str(e), exc_info=True)
        raise
```

---

## Pillar 2: Metrics

### Metric Types

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  COUNTER         │ Monotonically increasing value. Rate of events.           │
│                  │ Examples: http_requests_total, errors_total               │
│                  │ Query: rate(http_requests_total[5m]) → req/sec            │
├──────────────────┼──────────────────────────────────────────────────────────┤
│  GAUGE           │ Instantaneous value, can go up or down.                   │
│                  │ Examples: active_connections, queue_depth, memory_bytes   │
│                  │ Query: process_resident_memory_bytes → current value      │
├──────────────────┼──────────────────────────────────────────────────────────┤
│  HISTOGRAM       │ Distribution of values in configurable buckets.           │
│  (preferred)     │ Examples: request_duration_seconds, payload_size_bytes    │
│                  │ Allows: p50, p95, p99 latency; configurable on server     │
│                  │ Query: histogram_quantile(0.99, rate(duration_bucket[5m]))│
├──────────────────┼──────────────────────────────────────────────────────────┤
│  SUMMARY         │ Pre-calculated quantiles, computed on client side.        │
│  (avoid for      │ Less flexible — quantiles fixed at instrumentation time   │
│   distributed)   │ Cannot aggregate across instances                         │
└──────────────────┴──────────────────────────────────────────────────────────┘
```

### RED Method — For Services

```
  Rate     → How many requests per second is the service handling?
  Errors   → What fraction of requests are failing?
  Duration → How long does a typical / p99 request take?

  This is your default starting dashboard for any HTTP/RPC service.

  Prometheus queries:
    Rate:     rate(http_requests_total{service="order-service"}[5m])
    Errors:   rate(http_requests_total{status=~"5.."}[5m])
              / rate(http_requests_total[5m])
    Duration: histogram_quantile(0.99,
                rate(http_request_duration_seconds_bucket[5m]))
```

### USE Method — For Resources

```
  Utilization → What % of time is the resource busy?
  Saturation  → How much work is queued / waiting?
  Errors      → Are there error events for this resource?

  Apply to: CPUs, memory, disks, network interfaces, thread pools

  Examples:
    CPU:          utilization=cpu_usage_percent, saturation=load_average
    Database:     utilization=connection_pool_used/max,
                  saturation=query_queue_depth
    Thread Pool:  utilization=active_threads/max_threads,
                  saturation=task_queue_size
```

### Labeling Strategy — Cardinality Warning

```
  GOOD labels (low cardinality, bounded set):
    method="GET"           ← ~7 values
    status_code="200"      ← ~10-20 values
    service="order-api"    ← ~dozens of services
    region="us-east-1"     ← ~handful of regions

  BAD labels (high cardinality, unbounded set):
    user_id="usr_8x4k2m"   ← millions of users → KILLS Prometheus
    order_id="ord_12345"   ← each request → metrics DB explosion
    url="/api/orders/12345" ← use "/api/orders/{id}" instead

  Rule: Labels should have < ~100 distinct values.
        Use trace_id in LOGS and TRACES, not in METRICS.

  Cardinality explosion symptoms:
    → Prometheus OOMKills
    → Scrape timeouts
    → Dashboards take minutes to load
```

### Code Example: Metrics with Micrometer (Spring Boot)

```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Counter orderErrorCounter;
    private final Timer orderDuration;
    private final Gauge queueDepth;

    public OrderService(MeterRegistry registry, OrderQueue queue) {
        this.orderCounter = Counter.builder("orders.processed.total")
            .description("Total orders processed")
            .tag("service", "order-service")
            .register(registry);

        this.orderErrorCounter = Counter.builder("orders.errors.total")
            .description("Total order processing errors")
            .tag("service", "order-service")
            .register(registry);

        this.orderDuration = Timer.builder("orders.duration")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        // Gauge observes live value — do NOT increment/decrement manually
        Gauge.builder("orders.queue.depth", queue, OrderQueue::size)
            .description("Current order queue depth")
            .register(registry);
    }

    public Order processOrder(String orderId) {
        return orderDuration.record(() -> {
            try {
                var result = doProcess(orderId);
                orderCounter.increment(1, "status", "success");
                return result;
            } catch (Exception e) {
                orderErrorCounter.increment(1, "error_type",
                    e.getClass().getSimpleName());
                throw e;
            }
        });
    }
}
```

---

## Pillar 3: Distributed Tracing

### Core Concepts

```
  TRACE: End-to-end journey of a single request through the system
  SPAN:  A single unit of work (one service call, one DB query, one method)
  PARENT SPAN: The span that caused this span to start

  Trace abc123:
  │
  ├── [0ms]  Span: API Gateway         (duration: 245ms)
  │     │
  │     ├── [2ms]   Span: Auth Service     (duration: 12ms)
  │     │
  │     └── [15ms]  Span: Order Service    (duration: 228ms)
  │                   │
  │                   ├── [16ms]  Span: Inventory DB query  (duration: 8ms)
  │                   │
  │                   ├── [25ms]  Span: Payment Service     (duration: 195ms)
  │                   │            │
  │                   │            └── [30ms]  Span: Stripe API   (duration: 180ms)
  │                   │
  │                   └── [220ms] Span: Publish OrderCreated event (duration: 5ms)

  This shows: payment → Stripe is where 73% of latency lives
```

### Baggage — Cross-Service Context

```
  Baggage: key-value pairs propagated with the trace context
  Use for: tenant ID, feature flags, A/B test variant, deployment version

  W3C TraceContext headers (standard):
    traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
    tracestate:  vendor-specific metadata
    baggage:     tenant=acme,ab_variant=B

  Caution: baggage adds overhead to every request hop.
           Keep it small — IDs and flags only, never large payloads.
```

### Sampling Strategies

```
  HEAD SAMPLING (decide at trace start):
    + Simple, low overhead
    - May drop the very traces you need (errors, slow requests)

    Fixed rate:    sample 1% of all traces
    Rate limiting: sample up to N traces/sec per service

  TAIL SAMPLING (decide after trace completes):
    + Can prioritize: always keep errors, slow p99, low-volume paths
    - Requires buffering; more complex infrastructure (OTel Collector)

    Typical policy:
      - 100% of traces with errors
      - 100% of traces > p99 latency threshold
      - 1% of healthy/fast traces (baseline visibility)
      - 100% of low-traffic endpoints

  Head sampling for volume control → Tail sampling for quality
```

---

## OpenTelemetry — The Standard

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPENTELEMETRY PIPELINE                                   │
│                                                                             │
│  ┌──────────────┐    OTLP/gRPC    ┌──────────────────┐                     │
│  │  Your App    │ ─────────────── │  OTel Collector  │                     │
│  │              │                 │                  │                     │
│  │  SDK:        │                 │  Processors:     │                     │
│  │  - Traces    │                 │  - Batch         │──── Jaeger          │
│  │  - Metrics   │                 │  - Filter        │──── Prometheus      │
│  │  - Logs      │                 │  - Tail sample   │──── DataDog         │
│  │              │                 │  - Enrich        │──── Grafana Cloud   │
│  └──────────────┘                 └──────────────────┘                     │
│                                                                             │
│  Key benefit: change backend without changing application code              │
└─────────────────────────────────────────────────────────────────────────────┘

  Components:
    SDK        → instrument your code
    API        → stable interface (don't take SDK dependency in libraries)
    Exporters  → OTLP, Jaeger, Zipkin, Prometheus
    Collector  → agent/gateway mode; fanout, enrichment, sampling
    OTLP       → OpenTelemetry Protocol — the wire format
```

### OTel Setup: Java/Spring Boot

```java
// build.gradle
implementation 'io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter'

// application.yml
otel:
  service:
    name: order-service
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  traces:
    sampler: parentbased_traceidratio
    sampler.arg: "0.1"  # 10% head sampling
  metrics:
    export:
      interval: 30s

// Manual instrumentation for business spans
@Service
public class OrderService {

    private final Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");

    public Order processOrder(String orderId) {
        Span span = tracer.spanBuilder("processOrder")
            .setAttribute("order.id", orderId)
            .setAttribute("order.source", "api")
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            var order = repository.findById(orderId);
            span.setAttribute("order.customer_tier", order.getCustomerTier());

            var result = doProcess(order);

            span.setStatus(StatusCode.OK);
            return result;

        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### OTel Setup: Python/FastAPI

```python
# pip install opentelemetry-sdk opentelemetry-instrumentation-fastapi
#             opentelemetry-exporter-otlp-proto-grpc

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

# Setup — run once at startup
def setup_telemetry():
    provider = TracerProvider(
        resource=Resource.create({
            SERVICE_NAME: "order-service",
            SERVICE_VERSION: "1.4.2",
            DEPLOYMENT_ENVIRONMENT: os.environ["ENV"],
        })
    )
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(
            endpoint="http://otel-collector:4317"
        ))
    )
    trace.set_tracer_provider(provider)

    # Auto-instrument FastAPI, HTTPX, SQLAlchemy, etc.
    FastAPIInstrumentor.instrument_app(app)
    HTTPXClientInstrumentor().instrument()

tracer = trace.get_tracer("order-service")

# Manual span for business operations
async def process_order(order_id: str) -> Order:
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)

        try:
            order = await db.get_order(order_id)
            span.set_attribute("order.customer_tier", order.customer_tier)

            result = await do_process(order)
            span.set_status(trace.StatusCode.OK)
            return result

        except BusinessException as e:
            # Business exceptions — record but don't set ERROR status
            span.add_event("business_exception",
                          {"exception.type": type(e).__name__,
                           "exception.message": str(e)})
            raise
        except Exception as e:
            span.set_status(trace.StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

---

## Alerting

### Alert on Symptoms, Not Causes

```
  BAD (cause-based):
    "CPU > 90% for 5 minutes"
    → CPU could be 90% and everything is fine (batch job)
    → CPU could be 50% and users are suffering (thread deadlock)

  GOOD (symptom-based):
    "Error rate > 1% for 5 minutes"
    "p99 latency > 2s for 5 minutes"
    "Order success rate < 99.5% for 10 minutes"

  The Four Golden Signals → the right things to alert on:
    1. Latency   → p99 request duration crossing SLO threshold
    2. Traffic   → sudden drops (may indicate cascading failure)
    3. Errors    → error rate above threshold
    4. Saturation → approaching resource limits BEFORE they break things
```

### SLO-Based Alerting

```
  SLI (Service Level Indicator): measurable metric — error rate, latency
  SLO (Service Level Objective): target — "99.9% of requests < 500ms"
  SLA (Service Level Agreement): contractual SLO with consequences

  Error Budget: how much you can violate your SLO in a period
    99.9% uptime → 0.1% failure → ~43 min/month error budget

  SLO Alert (burn rate alerting):
    Alert when error budget is burning faster than sustainable

    Fast burn (page someone NOW):
      burn_rate > 14x over 1 hour window → will exhaust budget in 3 days
    Slow burn (ticket):
      burn_rate > 3x over 6 hour window → will exhaust budget in 10 days

  Prometheus SLO alert example:
    - alert: OrderServiceSLOBurnRateFast
      expr: |
        (
          rate(http_requests_total{service="order-service",status=~"5.."}[1h])
          / rate(http_requests_total{service="order-service"}[1h])
        ) > (14 * 0.001)   # 14x the 0.1% error rate budget
      for: 2m
      annotations:
        summary: "Order service burning error budget 14x too fast"
        runbook: "https://wiki/runbooks/order-service-slo"
```

### Fighting Alert Fatigue

```
  Symptoms of alert fatigue:
    → Team ignores pages
    → Alerts acknowledged immediately and closed without action
    → "Expected noise" alerts normalized
    → On-call rotation dreaded

  Fixes:
    1. Every alert must be actionable — if you can't do anything, remove it
    2. Every alert must have a runbook link
    3. Route severity correctly:
         P1 → wake someone up (true customer impact)
         P2 → notify in Slack (degraded but recoverable)
         P3 → create a ticket (investigate next business day)
    4. Track alert frequency — alert > 5x/week without action = remove/fix
    5. Post-incident: "was this alert useful?" review
    6. Dead alerts: disable anything not triggered in 90 days
```

---

## Dashboards

### The Four Golden Signals Layout

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Service: Order Service          [5m] [1h] [24h] [7d]    Env: [prod ▼] │
├────────────────────┬─────────────────────┬───────────────────────────────┤
│    LATENCY         │       TRAFFIC        │           ERRORS              │
│                    │                      │                               │
│  p50: 45ms         │  Req/sec: 1,247      │  Error rate: 0.03%            │
│  p95: 120ms        │                      │                               │
│  p99: 340ms  ▲warn │  [graph: req/s over  │  [graph: error% over time]    │
│                    │   time with deploy   │                               │
│  [latency graph    │   annotations ▼]     │  5xx: 3/min                   │
│   with SLO line]   │                      │  4xx: 18/min                  │
├────────────────────┴─────────────────────┴───────────────────────────────┤
│    SATURATION                                                             │
│                                                                           │
│  DB connections: 67/100  ████████████████░░░░░  67%                      │
│  Thread pool:    45/200  ██████████░░░░░░░░░░░  23%                      │
│  JVM heap:       2.1/4GB ██████████░░░░░░░░░░░  53%                      │
│  CPU:            31%     ██████░░░░░░░░░░░░░░░  31%                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### Annotating Deployments

```
  Every deploy → annotate all dashboards automatically

  Grafana annotation API (call from CI/CD pipeline):
    curl -X POST \
      -H "Authorization: Bearer $GRAFANA_TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
        "dashboardId": 42,
        "time": '"$(date +%s000)"',
        "tags": ["deploy", "order-service"],
        "text": "Deploy v1.4.2 by github-actions (PR #789)"
      }' \
      https://grafana.company.com/api/annotations

  This makes correlating "latency spike" → "recent deploy" trivial.
```

---

## Profiling

### Continuous Profiling

```
  Traditional profiling: run in dev, on demand, not in production
  Continuous profiling: always-on, low overhead (<1%), in production

  Tools:
    pprof (Go):        built-in, expose /debug/pprof endpoint
    Py-Spy (Python):   sampling profiler, attach to running process
    async-profiler:    Java, low overhead, JVM-aware
    Pyroscope:         open source continuous profiling platform
    Grafana Pyroscope: hosted version, integrates with traces

  Flame Graph reading:
    Width  = time spent (absolute or % of total)
    Height = call stack depth
    Top    = leaf function (where CPU actually spent)
    Wide frames at top = hot spots to optimize

  pprof in Go:
    import _ "net/http/pprof"
    go tool pprof -http :8080 http://myservice/debug/pprof/profile?seconds=30

  Linking traces to profiles (Grafana):
    When a trace shows slow span → click "profile" → see CPU flame graph
    for that exact request's timeframe
```

---

## Error Tracking

### Sentry / Error Monitoring Essentials

```
  Error tracking is NOT the same as logging:
    Logs → every event, including errors, stored in log aggregator
    Error tracking → deduplicated, grouped by fingerprint, with context

  Grouping / Fingerprinting:
    Exception type + stack trace → fingerprint → group into one "issue"
    1,000 users hitting same bug = 1 issue with 1,000 occurrences
    Without grouping → impossible to prioritize

  Context enrichment:
    User: user_id, plan, region (no PII without consent)
    Request: URL, method, headers (scrub auth tokens)
    Release: version, commit SHA
    Environment: prod, staging
    Breadcrumbs: last 10 log events leading up to the error

  Error flow:
    New error → alert team
    Assign → acknowledge (stops repeat alerts)
    Fix → mark resolved with release version
    Regression → auto-reopen if same error occurs in newer release

  Integration with OTel:
    Sentry SDK captures trace_id → link error to distributed trace
    Click error in Sentry → see full trace in Jaeger
```

---

## Production Debugging Playbook

### How to Debug a Slow Endpoint

```
  Scenario: p99 latency for POST /orders spiked from 200ms to 2000ms

  Step 1 — METRICS (zoom out, understand scope):
    □ When did it start? Check deploy annotations on dashboard
    □ All instances or one? (check by pod/instance label)
    □ All customers or segment? (check by region/tier label)
    □ Is error rate also elevated? (separate problem or cascade?)
    □ Saturation? DB connections, thread pool, external service timeouts?

  Step 2 — TRACES (find the slow path):
    □ Filter traces for POST /orders where duration > 1s
    □ Look at trace waterfall — which span is wide?
    □ Is it DB query? External API? Internal method?
    □ Is one span retrying? (same child span repeated multiple times)
    □ Compare slow trace vs fast trace — what's different?

  Step 3 — LOGS (understand why):
    □ Filter logs by trace_id of a slow trace
    □ Look for WARN/ERROR messages in the span's timeframe
    □ Check for retry logs, timeout logs, slow query logs
    □ Look for context: was this a specific customer? Order size?

  Step 4 — PROFILE (if it's CPU-bound):
    □ Pull CPU profile for the affected time window
    □ Flame graph → find hot method

  Common findings:
    N+1 query → single request causes 100s of DB queries
    Lock contention → threads waiting, saturation metric elevated
    External service degraded → third-party API span is wide
    GC pause → JVM heap saturation before GC kick in
    Cold start → first request after deploy, cache empty
```

---

## Practical Checklist

### Instrumentation

- [ ] Structured JSON logs with correlation IDs in every service
- [ ] Log levels used correctly; DEBUG disabled in production
- [ ] No PII or secrets in logs
- [ ] Metrics: RED method (rate, errors, duration) for every HTTP endpoint
- [ ] Metrics: USE method for DB connection pools, thread pools
- [ ] Labels have bounded cardinality (< 100 distinct values each)
- [ ] OTel SDK installed and configured with OTLP exporter
- [ ] Collector deployed in agent mode per node or gateway mode
- [ ] Sampling: 100% errors + tail sampling for slow traces
- [ ] Auto-instrumentation for frameworks (HTTP, DB, messaging)

### Alerting

- [ ] Alerts on symptoms (error rate, latency) not causes (CPU, memory)
- [ ] Every alert has a runbook link
- [ ] SLOs defined for all customer-facing services
- [ ] SLO burn rate alerting (fast burn + slow burn)
- [ ] Alert routing: P1 → page, P2 → Slack, P3 → ticket
- [ ] Alert review process: weekly triage of noisy alerts

### Dashboards

- [ ] Four Golden Signals dashboard per service
- [ ] Deploy annotations automated via CI/CD
- [ ] SLO dashboard showing error budget remaining
- [ ] On-call runbook linked from dashboard

### Anti-Patterns to Avoid

- [ ] Logging everything at INFO → noise, high storage cost, no signal
- [ ] Metrics with user_id or request_id labels → cardinality explosion
- [ ] Alerting on every exception → alert fatigue
- [ ] Missing trace context in async code → broken trace trees
- [ ] Custom metrics reinventing what OTel provides → maintenance burden
- [ ] No sampling → storing 100% of traces is expensive and unnecessary
```
