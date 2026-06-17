---
name: opentelemetry-deep-dive
description: Instrument services with OpenTelemetry — traces, metrics, logs, Java auto/manual instrumentation, Python auto-instrumentation, OTel Collector pipelines, sampling strategies, and semantic conventions
---

# OpenTelemetry Deep Dive — Expert Reference

## OTel Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OpenTelemetry Architecture                            │
│                                                                           │
│  Service A              Service B               Service C                │
│  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐          │
│  │ App Code    │        │ App Code    │        │ App Code    │          │
│  ├─────────────┤        ├─────────────┤        ├─────────────┤          │
│  │ OTel SDK    │        │ OTel SDK    │        │ OTel SDK    │          │
│  │ (API impl)  │        │ (API impl)  │        │ (API impl)  │          │
│  └──────┬──────┘        └──────┬──────┘        └──────┬──────┘          │
│         │ OTLP gRPC             │ OTLP gRPC             │ OTLP gRPC       │
│         └──────────────────────┼──────────────────────┘                 │
│                                ▼                                          │
│                    ┌───────────────────────┐                             │
│                    │   OTel Collector      │                             │
│                    │  ┌──────────────────┐ │                             │
│                    │  │ Receivers        │ │  OTLP, Jaeger, Zipkin,     │
│                    │  │ (OTLP, Prometheus│ │  Prometheus scrape, etc.   │
│                    │  ├──────────────────┤ │                             │
│                    │  │ Processors       │ │  batch, memory_limiter,    │
│                    │  │ (batch, filter,  │ │  resource, attributes      │
│                    │  │  resource)       │ │                             │
│                    │  ├──────────────────┤ │                             │
│                    │  │ Exporters        │ │  Jaeger, Tempo, Prometheus │
│                    │  │ (Jaeger, Tempo,  │ │  Datadog, Honeycomb,       │
│                    │  │  Prometheus)     │ │  Grafana Cloud             │
│                    │  └──────────────────┘ │                             │
│                    └───────────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
```

## Signal Types

```
Traces   — distributed request flow across services
           TraceId links all spans for one request
           SpanId identifies each unit of work
           ParentSpanId builds the call tree

Metrics  — numeric measurements over time
           Counter, Histogram, Gauge
           Aggregated (no per-request cost)

Logs     — timestamped events with structured data
           Correlated to traces via TraceId/SpanId
           (OTel logs bridge connects existing loggers to OTel)
```

---

## Java Instrumentation

### Auto-Instrumentation (zero code changes)

```bash
# 1. Download the Java agent
curl -L -o opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# 2. Attach at JVM startup — instruments 100+ libraries automatically
#    (Spring, Tomcat, gRPC, Kafka, JDBC, Redis, HTTP clients, ...)
java \
  -javaagent:opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.resource.attributes=deployment.environment=production,service.version=2.1.0 \
  -Dotel.exporter.otlp.endpoint=http://collector:4317 \
  -Dotel.logs.exporter=otlp \
  -Dotel.metrics.exporter=otlp \
  -jar order-service.jar
```

```bash
# Environment variable equivalents (preferred in containers)
OTEL_SERVICE_NAME=order-service
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=2.1.0
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
OTEL_LOGS_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1     # 10% sampling
```

### Manual Instrumentation — Traces

```java
// pom.xml / build.gradle.kts
// io.opentelemetry:opentelemetry-api
// io.opentelemetry:opentelemetry-sdk
// io.opentelemetry:opentelemetry-exporter-otlp

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;

public class OrderService {

    // One tracer per class — reuse the instance (it's thread-safe)
    private static final Tracer tracer =
        GlobalOpenTelemetry.getTracer("com.example.order-service", "2.1.0");

    public Order processOrder(String orderId) {
        // Create a child span under the current context
        Span span = tracer.spanBuilder("processOrder")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();

        // CRITICAL: always put span in scope so child operations inherit it
        try (Scope scope = span.makeCurrent()) {
            // Add semantic attributes
            span.setAttribute("order.id", orderId);
            span.setAttribute("order.type", "standard");

            Order order = fetchOrder(orderId);       // auto-instrumented DB call
            validateOrder(order);
            chargePayment(order);                    // child span created here

            span.setAttribute("order.total_cents", order.getTotalCents());
            span.setStatus(StatusCode.OK);
            return order;

        } catch (OrderNotFoundException e) {
            span.setStatus(StatusCode.ERROR, "Order not found");
            span.recordException(e);   // attaches stack trace to span
            throw e;
        } finally {
            span.end();  // ALWAYS end the span; use try-finally
        }
    }
}
```

### Context Propagation Across Threads

```java
// Propagate context to async work (executor, CompletableFuture, etc.)
import io.opentelemetry.context.Context;

// WRONG: context lost across thread boundary
CompletableFuture.runAsync(() -> {
    // Current span is not visible here — new thread has no context
    doWork();
});

// RIGHT: wrap runnable with current context
Context ctx = Context.current();
CompletableFuture.runAsync(ctx.wrap(() -> {
    // span is now accessible in this thread
    doWork();
}));

// Or use context-aware executor
ExecutorService executor = Context.taskWrapping(
    Executors.newFixedThreadPool(10)
);
executor.submit(() -> doWork());  // context flows automatically
```

### Baggage — Cross-Service Data Propagation

```java
import io.opentelemetry.api.baggage.Baggage;
import io.opentelemetry.context.Context;

// Set baggage (propagates through all downstream calls via W3C Baggage header)
Baggage baggage = Baggage.current().toBuilder()
    .put("tenant.id", tenantId)
    .put("user.tier", "premium")
    .build();

try (Scope scope = baggage.makeCurrent()) {
    // All HTTP/gRPC calls from here carry X-B3-Baggage or baggage header
    orderService.process(request);
}

// Read baggage downstream (in another service)
String tenantId = Baggage.current().getEntryValue("tenant.id");
```

### Custom Attributes — Semantic Conventions

```java
import io.opentelemetry.semconv.SemanticAttributes;

// Use standard semantic convention keys (not custom strings)
span.setAttribute(SemanticAttributes.HTTP_METHOD, "POST");
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);
span.setAttribute(SemanticAttributes.DB_SYSTEM, "postgresql");
span.setAttribute(SemanticAttributes.DB_STATEMENT, "SELECT * FROM orders WHERE id = ?");
span.setAttribute(SemanticAttributes.MESSAGING_SYSTEM, "kafka");
span.setAttribute(SemanticAttributes.MESSAGING_DESTINATION_NAME, "orders.created");

// Custom business attributes — prefix with your domain
span.setAttribute("order.id", orderId);
span.setAttribute("order.status", "CONFIRMED");
span.setAttribute("payment.method", "credit_card");
```

### Spring Boot Auto-Configuration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.4.0-alpha</version>
</dependency>
```

```yaml
# application.yml — no code changes needed; Spring Boot starter auto-configures
management:
  tracing:
    sampling:
      probability: 1.0   # 100% in dev, lower in prod

spring:
  application:
    name: order-service

otel:
  exporter:
    otlp:
      endpoint: http://collector:4317
  resource:
    attributes:
      service.version: "2.1.0"
      deployment.environment: "production"
```

### Metrics SDK

```java
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.Meter;

public class OrderMetrics {

    private final LongCounter ordersCreated;
    private final LongCounter ordersFailed;
    private final DoubleHistogram processingDuration;

    public OrderMetrics(OpenTelemetry otel) {
        Meter meter = otel.getMeter("com.example.order-service");

        this.ordersCreated = meter
            .counterBuilder("orders.created.total")
            .setDescription("Total orders created successfully")
            .setUnit("{order}")
            .build();

        this.ordersFailed = meter
            .counterBuilder("orders.failed.total")
            .setDescription("Total orders that failed processing")
            .setUnit("{order}")
            .build();

        this.processingDuration = meter
            .histogramBuilder("orders.processing.duration")
            .setDescription("Order processing duration")
            .setUnit("ms")
            .build();
    }

    public void recordOrderCreated(String status, String tier) {
        ordersCreated.add(1,
            Attributes.of(
                AttributeKey.stringKey("order.status"), status,
                AttributeKey.stringKey("customer.tier"), tier
            )
        );
    }

    public void recordProcessingDuration(double ms, String status) {
        processingDuration.record(ms,
            Attributes.of(AttributeKey.stringKey("status"), status)
        );
    }
}
```

---

## Python Instrumentation

### Auto-Instrumentation

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install    # auto-detects installed frameworks and installs their plugins

# Run — instruments Flask, FastAPI, SQLAlchemy, requests, etc. automatically
OTEL_SERVICE_NAME=order-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317 \
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production \
opentelemetry-instrument python app.py
```

### Manual Instrumentation

```python
from opentelemetry import trace, metrics, baggage
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.trace import StatusCode

# SDK setup (done once at app startup)
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

# Per-module tracer
tracer = trace.get_tracer("order-service", "2.1.0")

def process_order(order_id: str):
    with tracer.start_as_current_span(
        "process_order",
        kind=trace.SpanKind.INTERNAL
    ) as span:
        span.set_attribute("order.id", order_id)

        try:
            order = fetch_order(order_id)
            span.set_attribute("order.total_cents", order.total_cents)
            span.set_status(StatusCode.OK)
            return order
        except OrderNotFoundError as e:
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)   # attaches stack trace
            raise
```

### FastAPI Integration

```python
from fastapi import FastAPI
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

app = FastAPI()

# Instrument everything
FastAPIInstrumentor.instrument_app(app)     # HTTP server spans
SQLAlchemyInstrumentor().instrument()       # DB query spans
RedisInstrumentor().instrument()            # Redis command spans
HTTPXClientInstrumentor().instrument()      # outbound HTTP spans
```

### Metrics (Python)

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

provider = MeterProvider()
metrics.set_meter_provider(provider)
meter = metrics.get_meter("order-service")

# Counter — monotonically increasing
orders_created = meter.create_counter(
    "orders.created.total",
    description="Total orders created",
    unit="{order}"
)
orders_created.add(1, {"status": "confirmed", "tier": "premium"})

# Histogram — distribution of values
processing_time = meter.create_histogram(
    "orders.processing.duration",
    unit="ms"
)
processing_time.record(145.3, {"status": "success"})

# ObservableGauge — current value (e.g., queue depth)
def get_queue_depth(options):
    yield metrics.Observation(value=queue.size(), attributes={})

meter.create_observable_gauge(
    "orders.queue.depth",
    callbacks=[get_queue_depth],
    unit="{order}"
)
```

---

## OTel Collector Configuration

```yaml
# collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317   # services send telemetry here
      http:
        endpoint: 0.0.0.0:4318
  prometheus:                     # scrape Prometheus metrics
    config:
      scrape_configs:
        - job_name: "services"
          static_configs:
            - targets: ["order-service:8080"]

processors:
  batch:                          # buffer spans before export (critical for performance)
    send_batch_size: 10000
    timeout: 5s

  memory_limiter:                 # prevent OOM; drop telemetry if memory too high
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  resource:                       # add/override resource attributes
    attributes:
      - action: insert
        key: deployment.environment
        value: production

  attributes/drop-pii:            # redact sensitive fields
    actions:
      - action: delete
        key: http.request.header.authorization
      - action: hash
        key: user.email

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

  otlp/grafana:                   # Grafana Cloud / Tempo
    endpoint: tempo:4317
    headers:
      authorization: "Bearer ${GRAFANA_API_KEY}"

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, resource, attributes/drop-pii, batch]
      exporters:  [jaeger, otlp/grafana]

    metrics:
      receivers:  [otlp, prometheus]
      processors: [memory_limiter, resource, batch]
      exporters:  [prometheusremotewrite]

    logs:
      receivers:  [otlp]
      processors: [memory_limiter, resource, batch]
      exporters:  [otlp/grafana]
```

### Collector Deployment Patterns

```
Sidecar (per pod):
  ┌─────────────────────────────────────┐
  │ Pod                                  │
  │  ┌──────────────┐ ┌───────────────┐ │
  │  │ App Container│►│ OTel Sidecar  │►│ Backend
  │  └──────────────┘ └───────────────┘ │
  └─────────────────────────────────────┘
  + Low latency; isolation per pod
  - Many collector instances; resource overhead

DaemonSet (per node):
  Node: [App1] [App2] [App3] → [OTel DaemonSet] → Backend
  + Fewer instances; shared resources
  - Node failure affects all pods on that node

Gateway (centralized):
  All services → [OTel Agent/Sidecar] → [OTel Gateway Cluster] → Backend
  + Fan-in; apply tail sampling; single egress point
  - Additional hop; single point of failure if not HA
  Recommended: agents on each node, gateway cluster for tail sampling and export
```

---

## Sampling Strategies

### Head Sampling (at trace start)

```java
// Java SDK setup with head sampling
SdkTracerProvider provider = SdkTracerProvider.builder()
    .setSampler(
        // Sample 10% of root spans; respect parent's sampling decision for child spans
        Sampler.parentBased(
            Sampler.traceIdRatioBased(0.1)  // 10%
        )
    )
    .addSpanProcessor(BatchSpanProcessor.builder(
        OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://collector:4317")
            .build()
    ).build())
    .build();
```

```python
# Python head sampling
from opentelemetry.sdk.trace.sampling import ParentBased, TraceIdRatioBased

provider = TracerProvider(
    sampler=ParentBased(root=TraceIdRatioBased(0.1))  # 10%
)
```

### Tail Sampling (at Collector — see full trace before deciding)

```yaml
# collector-config.yaml — tail sampling processor
processors:
  tail_sampling:
    decision_wait: 10s          # wait 10s for all spans in a trace
    num_traces: 100000          # number of traces to buffer
    expected_new_traces_per_sec: 10

    policies:
      # Always sample traces with errors
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }

      # Always sample slow traces (> 1s)
      - name: slow-traces
        type: latency
        latency: { threshold_ms: 1000 }

      # Sample 10% of healthy fast traces
      - name: random-10pct
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

      # Composite: errors OR slow OR 10% random
      - name: composite
        type: composite
        composite:
          max_total_spans_per_second: 5000
          policy_order: [errors, slow-traces, random-10pct]
          composite_sub_policy:
            - name: errors
              type: status_code
              status_code: { status_codes: [ERROR] }
            - name: slow-traces
              type: latency
              latency: { threshold_ms: 1000 }
            - name: random-10pct
              type: probabilistic
              probabilistic: { sampling_percentage: 10 }
          rate_allocation:
            - policy: errors
              percent: 60
            - policy: slow-traces
              percent: 30
            - policy: random-10pct
              percent: 10
```

```
Head vs Tail sampling:
  Head (at SDK):
    + Zero latency — decision at trace start
    + No collector buffering needed
    - Cannot sample based on outcome (error, latency)
    Use for: very high volume where any random 10% is good enough

  Tail (at Collector):
    + Sample ALL errors, sample based on full trace outcome
    + Best signal-to-noise
    - Requires buffering all trace spans in memory
    - Collector needs significant RAM
    Use for: production where you need 100% error visibility
```

---

## Semantic Conventions — Standard Attribute Keys

```
HTTP (server):
  http.request.method       GET, POST, ...
  http.response.status_code 200, 404, 500
  url.path                  /api/orders/123
  server.address            api.example.com
  server.port               443

HTTP (client):
  http.request.method
  http.response.status_code
  server.address
  url.full                  https://api.example.com/v1/orders

Database:
  db.system                 postgresql, mysql, redis, mongodb
  db.name                   orders_db
  db.statement              SELECT * FROM orders WHERE id = ?
  db.operation              SELECT, INSERT, UPDATE
  server.address            db-host.example.com

Messaging (Kafka, RabbitMQ):
  messaging.system          kafka, rabbitmq
  messaging.destination.name  orders.created
  messaging.operation       publish, receive, process
  messaging.kafka.consumer.group  order-processors
  messaging.message.id      msg-uuid-123

RPC (gRPC):
  rpc.system                grpc
  rpc.service               com.example.order.v1.OrderService
  rpc.method                GetOrder
  rpc.grpc.status_code      0 (OK), 5 (NOT_FOUND), ...
```

---

## Exemplars — Link Metrics to Traces

```
Exemplar: a sample trace ID embedded in a metric data point
           "This histogram bucket had p99=850ms, here is a trace ID that hit 900ms"

Configuration (Java):
  Add exemplar filter to SDK — it attaches current span's trace ID to metric recordings

Grafana usage:
  In a dashboard panel showing request latency, click a spike
  → see the exemplar trace ID → jump to Tempo to see the full trace
  = "Why did p99 spike at 14:32?" answered in 2 clicks
```

---

## Resource Attributes — Set at SDK Init

```java
// Java: set resource attributes once at startup
Resource resource = Resource.getDefault().merge(
    Resource.create(Attributes.builder()
        .put(ResourceAttributes.SERVICE_NAME,    "order-service")
        .put(ResourceAttributes.SERVICE_VERSION, "2.1.0")
        .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production")
        .put(ResourceAttributes.HOST_NAME,       System.getenv("HOSTNAME"))
        .put(ResourceAttributes.K8S_POD_NAME,    System.getenv("POD_NAME"))
        .put(ResourceAttributes.K8S_NAMESPACE_NAME, System.getenv("POD_NAMESPACE"))
        .build())
);

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setResource(resource)
    .build();
```

```python
# Python: set resource at startup
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION

resource = Resource.create({
    SERVICE_NAME: "order-service",
    SERVICE_VERSION: "2.1.0",
    "deployment.environment": "production",
    "k8s.pod.name": os.environ.get("POD_NAME", ""),
})
provider = TracerProvider(resource=resource)
```

---

## Production Checklist

```
SDK Setup:
  [ ] service.name set (not "unknown_service")
  [ ] service.version set (enables version-over-version comparison)
  [ ] deployment.environment set (dev / staging / production)
  [ ] k8s.pod.name, k8s.namespace.name set in Kubernetes
  [ ] BatchSpanProcessor used (not SimpleSpanProcessor — blocks)
  [ ] OTLP exporter pointing to Collector (not directly to backend)

Instrumentation:
  [ ] Java agent attached OR manual instrumentation on critical paths
  [ ] Context propagated across thread boundaries (Context.wrap())
  [ ] Baggage used for cross-cutting tenant/user metadata
  [ ] span.end() in finally block — never rely on GC
  [ ] Exceptions recorded with span.recordException(e)
  [ ] Status set: setStatus(OK) on success, setStatus(ERROR) on failure
  [ ] Semantic convention attribute names used (not custom strings)
  [ ] Business-relevant attributes added (order.id, payment.method)

Collector:
  [ ] memory_limiter processor configured (prevents OOM)
  [ ] batch processor configured (reduces export overhead)
  [ ] Tail sampling configured for errors and slow traces
  [ ] Collector deployed as DaemonSet or Gateway (not sidecar for scale)

Sampling:
  [ ] NOT 100% sampling in production (cost, volume)
  [ ] Tail sampling catches all errors regardless of rate
  [ ] Head sampling: ParentBased to respect upstream decision
```

---

## Anti-Patterns

```
WRONG: Not propagating context across async boundaries
  CompletableFuture.runAsync(() -> doWork());
  // span is null inside the lambda
RIGHT: Context.current().wrap(() -> doWork()) or context-aware executor.

WRONG: Not ending spans (span leak)
  Span span = tracer.spanBuilder("op").startSpan();
  doWork();
  // forgot span.end() — memory leak, corrupts trace
RIGHT: Always use try-with-resources:
  try (Scope s = span.makeCurrent()) { doWork(); } finally { span.end(); }

WRONG: 100% sampling in production
  Every trace exported = cost proportional to traffic; backend overwhelmed
RIGHT: 1-10% for healthy traffic; 100% for errors via tail sampling.

WRONG: Exporting directly from services to backends
  service → Jaeger, service → Datadog (tight coupling, different config per service)
RIGHT: service → OTel Collector → multiple backends (one config change to add/remove backends).

WRONG: Custom attribute names for standard concepts
  span.setAttribute("http_status", 200)   # non-standard
RIGHT: span.setAttribute(SemanticAttributes.HTTP_RESPONSE_STATUS_CODE, 200)

WRONG: Missing service.name resource attribute
  Traces appear as "unknown_service:java" — impossible to filter in dashboards
RIGHT: Always set service.name at SDK init.

WRONG: SimpleSpanProcessor in production
  Blocks the calling thread until span is exported — adds latency to every request
RIGHT: BatchSpanProcessor buffers and exports asynchronously.
```
