---
name: saas-architecture
description: Design, implement, or review SaaS architecture patterns including multi-tenancy models, tenant isolation, billing, feature access control, and operational concerns.
---

# SaaS Architecture Patterns

SaaS architecture decisions made early — especially the tenancy model — are expensive to change later. Every architectural choice cascades into the data model, billing, compliance posture, and operational complexity.

```
+--------------------------------------------------------------+
|                    SaaS Platform                             |
|                                                              |
|  +------------------+    +-----------------------------+    |
|  | Self-Service     |    | Admin Portal                |    |
|  | Signup / Trial   |    | Tenant Management           |    |
|  +------------------+    +-----------------------------+    |
|           |                          |                       |
|           v                          v                       |
|  +----------------------------------------------+           |
|  |  API Gateway / Auth Middleware               |           |
|  |  - Validate JWT                              |           |
|  |  - Extract tenantId from claim               |           |
|  |  - Set request context                       |           |
|  +----------------------------------------------+           |
|           |                                                  |
|           v                                                  |
|  +----------------------------------------------+           |
|  |  Application Services                        |           |
|  |  - Entitlement check (plan.allows(feature))  |           |
|  |  - Rate limit per tenant                     |           |
|  |  - All queries scoped by tenantId            |           |
|  +----------------------------------------------+           |
|           |                    |                             |
|      +---------+        +------------+                       |
|      | Pool DB |        | Stripe     |                       |
|      | RLS on  |        | Billing    |                       |
|      | every   |        | Webhooks   |                       |
|      | table   |        +------------+                       |
|      +---------+                                             |
+--------------------------------------------------------------+
```

---

## Multi-Tenancy Models

Choose based on isolation requirements, target customer size, and operational budget.

```
SILO (one stack per tenant)
+----------+  +----------+  +----------+
| Tenant A |  | Tenant B |  | Tenant C |
| App      |  | App      |  | App      |
| DB       |  | DB       |  | DB       |
| Infra    |  | Infra    |  | Infra    |
+----------+  +----------+  +----------+
Isolation: max  Cost: high  Complexity: simple per-tenant

BRIDGE (shared infra, dedicated DB per tenant)
+-----------------------------------+
|  Shared App Tier (K8s cluster)    |
|  Shared Control Plane             |
+-----------------------------------+
  |           |           |
+----+     +----+     +----+
| DB |     | DB |     | DB |
| A  |     | B  |     | C  |
+----+     +----+     +----+
Isolation: DB  Cost: medium  Compliance: easier

POOL (shared everything)
+-----------------------------------+
|  Shared App Tier                  |
+-----------------------------------+
           |
+-----------------------------------+
|  Shared Database                  |
|  tenant_id on every table         |
|  Row-Level Security enforced      |
+-----------------------------------+
Isolation: row  Cost: low  Scale: highest
```

| Model | Use When | Avoid When |
|-------|----------|-----------|
| Silo | Enterprise compliance (HIPAA, FedRAMP), dedicated SLAs | Many small tenants; cost prohibitive |
| Bridge | Mid-market; compliance needed; some cost efficiency | Thousands of tenants; connection pool limits hit |
| Pool | SMB/PLG; startup; highest density | Enterprise with strict data isolation; RLS bugs are catastrophic |

### Hybrid: Tiered Tenancy

```
Free / Starter tier  --> Pool model
Pro tier             --> Bridge model (dedicated DB)
Enterprise tier      --> Silo model (dedicated stack)
```

---

## Tenant Isolation Strategies

### Row-Level Security (PostgreSQL RLS)

The most important defence in Pool model. RLS is enforced at the database engine level — bugs in application code cannot read across tenants.

```sql
-- Enable RLS on every multi-tenant table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;   -- applies to table owner too

-- Policy: only see rows matching current tenant
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Same pattern for INSERT (prevents inserting into wrong tenant)
CREATE POLICY tenant_insert ON orders
  FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.current_tenant')::uuid);

-- Application sets tenant context at start of each transaction
SET LOCAL app.current_tenant = '550e8400-e29b-41d4-a716-446655440000';
```

**Spring Boot RLS integration**

```java
@Component
public class TenantAwareDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getCurrentTenant();
    }
}

@Aspect
@Component
public class TenantRlsAspect {

    @Autowired
    private EntityManager em;

    @Before("@annotation(Transactional)")
    public void setTenantContext() {
        String tenantId = TenantContext.getCurrentTenant();
        em.createNativeQuery("SET LOCAL app.current_tenant = :tenantId")
          .setParameter("tenantId", tenantId)
          .executeUpdate();
    }
}
```

### Schema Per Tenant

```sql
-- Each tenant gets their own PostgreSQL schema
CREATE SCHEMA tenant_abc123;
CREATE TABLE tenant_abc123.orders (...);

-- Set search_path to tenant schema on connection
SET search_path TO tenant_abc123, public;
```

```java
// Spring Boot: switch schema per request
@Component
public class SchemaPerTenantConnectionProvider implements MultiTenantConnectionProvider {

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        Connection connection = dataSource.getConnection();
        connection.createStatement()
            .execute("SET search_path TO " + tenantIdentifier + ", public");
        return connection;
    }
}
```

### Kubernetes Namespace Per Tenant (Enterprise Tier)

```yaml
# Namespace per tenant for enterprise silo
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acmecorp
  labels:
    tenant-id: acmecorp
    tier: enterprise

---
# NetworkPolicy: block cross-tenant traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-acmecorp
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: tenant-acmecorp
```

---

## Tenant Context Propagation

Every request must carry `tenantId` from the edge to every downstream call.

```
Request arrives at API Gateway
         |
         v
+--------------------+
| Auth Middleware    |
| - Validate JWT     |
| - Decode claims    |
| - tenantId = jwt   |
|   .claims.tenant   |
+--------------------+
         |
         v
+--------------------+
| Request Context    |   ThreadLocal / Reactor context / MDC
| TenantContext      |   Propagated automatically to:
|   .set(tenantId)   |   - DB queries (via RLS)
+--------------------+   - Outbound HTTP (header injection)
         |               - Kafka messages (header)
         v               - Log MDC (tenant_id in every log line)
+--------------------+
| Repository Layer   |
| RLS: SET LOCAL     |
| app.current_tenant |
+--------------------+
```

### JWT Claim Extraction

```java
@Component
public class TenantFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        String token = extractBearerToken(request);
        if (token != null) {
            Jwt jwt = jwtDecoder.decode(token);
            String tenantId = jwt.getClaimAsString("tenant_id");
            TenantContext.setCurrentTenant(tenantId);
            MDC.put("tenant_id", tenantId);   // appears in every log line
        }
        try {
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
            MDC.remove("tenant_id");
        }
    }
}
```

### Propagate to Downstream Services

```java
// Feign client interceptor: inject tenant header on all outbound calls
@Component
public class TenantFeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String tenantId = TenantContext.getCurrentTenant();
        if (tenantId != null) {
            template.header("X-Tenant-ID", tenantId);
        }
    }
}
```

```java
// Kafka producer: include tenantId in every message header
ProducerRecord<String, OrderCreatedEvent> record = new ProducerRecord<>(
    "orders.created", orderEvent);
record.headers().add("X-Tenant-ID",
    TenantContext.getCurrentTenant().getBytes(StandardCharsets.UTF_8));
```

```java
// Kafka consumer: extract tenant context before processing
@KafkaListener(topics = "orders.created")
public void handle(ConsumerRecord<String, OrderCreatedEvent> record) {
    String tenantId = new String(
        record.headers().lastHeader("X-Tenant-ID").value(),
        StandardCharsets.UTF_8);
    TenantContext.setCurrentTenant(tenantId);
    try {
        orderService.process(record.value());
    } finally {
        TenantContext.clear();
    }
}
```

---

## Onboarding and Provisioning

### Self-Service Signup Flow

```
User submits email
       |
       v
+------------------+
| Email Verified?  |--No--> Send verification email --> wait
+------------------+
       |Yes
       v
+---------------------------+
| Create Tenant Record      |
| - Generate tenant_id (UUID)|
| - plan = TRIAL             |
| - trial_expires_at = now   |
|   + 14 days                |
+---------------------------+
       |
       v
+---------------------------+
| Provision Resources       |
| - Create DB schema         |
| - Create S3 prefix         |
| - Create Stripe customer   |
| - Seed default config      |
+---------------------------+
       |
       v
+---------------------------+
| Create First User         |
| - role = ADMIN             |
| - Send welcome email       |
| - Redirect to onboarding  |
+---------------------------+
```

```java
@Service
@Transactional
public class TenantProvisioningService {

    public Tenant provision(SignupRequest request) {
        // Idempotent: check if already provisioned
        return tenantRepository.findByEmail(request.email())
            .orElseGet(() -> doProvision(request));
    }

    private Tenant doProvision(SignupRequest request) {
        Tenant tenant = tenantRepository.save(Tenant.builder()
            .id(UUID.randomUUID())
            .email(request.email())
            .plan(Plan.TRIAL)
            .trialExpiresAt(Instant.now().plus(14, DAYS))
            .build());

        // Each step is idempotent
        schemaProvisioner.provision(tenant.getId());     // CREATE SCHEMA IF NOT EXISTS
        s3Provisioner.provision(tenant.getId());         // idempotent S3 prefix setup
        stripeProvisioner.provision(tenant);             // create customer if not exists

        eventPublisher.publishEvent(new TenantProvisionedEvent(tenant));
        return tenant;
    }
}
```

### Idempotency Pattern

Every provisioning step must tolerate being called twice (network retries, job restarts).

```java
public void provisionSchema(UUID tenantId) {
    String schema = "tenant_" + tenantId.toString().replace("-", "");
    jdbcTemplate.execute("CREATE SCHEMA IF NOT EXISTS " + schema);  // IF NOT EXISTS = idempotent
    flyway.configure()
        .schemas(schema)
        .locations("classpath:db/tenant-migration")
        .load()
        .migrate();   // Flyway tracks applied migrations; safe to call repeatedly
}
```

---

## Billing and Subscription

### Stripe Integration

```
Tenant signs up --> Stripe Customer created
       |
       v
Selects plan --> Stripe Subscription created
       |              (Product + Price)
       v
Feature use --> Entitlement check via plan field
       |
       v
Monthly usage --> Stripe Meter records
       |
       v
Invoice generated --> Stripe charges card
       |
       v
Stripe Webhook --> /webhooks/stripe
  - payment_intent.succeeded --> activate subscription
  - invoice.payment_failed  --> notify + grace period
  - customer.subscription.deleted --> downgrade/cancel
```

```java
@Service
public class BillingService {

    private final Stripe stripe;

    public String createCustomer(Tenant tenant) {
        CustomerCreateParams params = CustomerCreateParams.builder()
            .setEmail(tenant.getEmail())
            .setName(tenant.getName())
            .putMetadata("tenant_id", tenant.getId().toString())
            .build();
        Customer customer = Customer.create(params);
        return customer.getId();
    }

    public Subscription subscribe(String customerId, String priceId) {
        SubscriptionCreateParams params = SubscriptionCreateParams.builder()
            .setCustomer(customerId)
            .addItem(SubscriptionCreateParams.Item.builder()
                .setPrice(priceId)
                .build())
            .setPaymentBehavior(
                SubscriptionCreateParams.PaymentBehavior.DEFAULT_INCOMPLETE)
            .addExpand("latest_invoice.payment_intent")
            .build();
        return Subscription.create(params);
    }

    public void recordUsage(String subscriptionItemId, long quantity) {
        UsageRecordCreateOnSubscriptionItemParams params =
            UsageRecordCreateOnSubscriptionItemParams.builder()
                .setQuantity(quantity)
                .setTimestamp(Instant.now().getEpochSecond())
                .setAction(UsageRecordCreateOnSubscriptionItemParams.Action.INCREMENT)
                .build();
        UsageRecord.createOnSubscriptionItem(subscriptionItemId, params);
    }
}
```

### Stripe Webhook Handler

```java
@RestController
@RequestMapping("/webhooks/stripe")
public class StripeWebhookController {

    @PostMapping
    public ResponseEntity<Void> handle(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String sigHeader) {

        Event event = Webhook.constructEvent(payload, sigHeader, webhookSecret);

        switch (event.getType()) {
            case "customer.subscription.updated" -> handleSubscriptionUpdated(event);
            case "invoice.payment_failed"        -> handlePaymentFailed(event);
            case "customer.subscription.deleted" -> handleSubscriptionCancelled(event);
        }
        return ResponseEntity.ok().build();
    }
}
```

---

## Feature Access Control

### Plan Hierarchy

```java
public enum Plan {
    FREE, STARTER, PRO, ENTERPRISE;

    public boolean allows(Feature feature) {
        return feature.minimumPlan().ordinal() <= this.ordinal();
    }
}

public enum Feature {
    BASIC_API(Plan.FREE),
    ADVANCED_ANALYTICS(Plan.PRO),
    CUSTOM_ROLES(Plan.PRO),
    SSO(Plan.ENTERPRISE),
    AUDIT_LOG(Plan.ENTERPRISE),
    SLA_99_9(Plan.ENTERPRISE);

    private final Plan minimumPlan;
    Feature(Plan minimumPlan) { this.minimumPlan = minimumPlan; }
    public Plan minimumPlan() { return minimumPlan; }
}
```

### Entitlement Enforcement

```java
// Annotation-based enforcement (AOP)
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresPlan {
    Feature value();
}

@Aspect
@Component
public class EntitlementAspect {

    @Before("@annotation(requiresPlan)")
    public void checkEntitlement(RequiresPlan requiresPlan) {
        Tenant tenant = tenantRepository.findById(TenantContext.getCurrentTenant())
            .orElseThrow();
        if (!tenant.getPlan().allows(requiresPlan.value())) {
            throw new PlanUpgradeRequiredException(
                requiresPlan.value(), tenant.getPlan());
        }
    }
}

// Usage
@RequiresPlan(Feature.ADVANCED_ANALYTICS)
public AnalyticsReport generateReport(ReportRequest request) { ... }
```

### Usage Quota Enforcement

```java
@Service
public class QuotaService {

    private final RedisTemplate<String, Long> redis;

    public void checkAndIncrement(UUID tenantId, QuotaType type) {
        String key = "quota:%s:%s:%s".formatted(
            tenantId, type, YearMonth.now());

        Long current = redis.opsForValue().increment(key);

        // Set expiry on first write (auto-reset monthly)
        if (current == 1) {
            redis.expire(key, Duration.ofDays(35));
        }

        Plan plan = tenantRepository.getPlan(tenantId);
        long limit = plan.limitFor(type);

        if (current > limit) {
            throw new QuotaExceededException(type, limit, current);
        }
    }
}
```

---

## Operational Concerns

### Noisy Neighbor Protection

```java
// Rate limiting per tenant with Bucket4j
@Component
public class TenantRateLimiter {

    private final Map<UUID, Bucket> buckets = new ConcurrentHashMap<>();

    public void checkLimit(UUID tenantId, Plan plan) {
        Bucket bucket = buckets.computeIfAbsent(tenantId,
            id -> buildBucket(plan));
        if (!bucket.tryConsume(1)) {
            throw new RateLimitExceededException(tenantId);
        }
    }

    private Bucket buildBucket(Plan plan) {
        Bandwidth limit = switch (plan) {
            case FREE       -> Bandwidth.simple(100,  Duration.ofMinute(1));
            case STARTER    -> Bandwidth.simple(500,  Duration.ofMinute(1));
            case PRO        -> Bandwidth.simple(2000, Duration.ofMinute(1));
            case ENTERPRISE -> Bandwidth.simple(10000, Duration.ofMinute(1));
        };
        return Bucket.builder().addLimit(limit).build();
    }
}
```

### Per-Tenant Observability

```java
// Micrometer: tag all metrics with tenant_id
@Component
public class TenantMetrics {

    private final MeterRegistry registry;

    public void recordApiCall(UUID tenantId, String endpoint, long durationMs) {
        Timer.builder("api.request.duration")
            .tag("tenant_id", tenantId.toString())
            .tag("endpoint", endpoint)
            .register(registry)
            .record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

```yaml
# Grafana query: per-tenant API latency
# PromQL
histogram_quantile(0.99, 
  sum by (tenant_id, le) (
    rate(api_request_duration_bucket[5m])
  )
)
```

### Tenant Offboarding (GDPR Right to Erasure)

```java
@Service
public class TenantOffboardingService {

    @Transactional
    public void offboard(UUID tenantId, OffboardingReason reason) {
        // 1. Export data before deletion
        if (reason.requiresDataExport()) {
            dataExportService.export(tenantId);
        }

        // 2. Cancel Stripe subscription
        billingService.cancelSubscription(tenantId);

        // 3. Delete application data (RLS will scope to tenant)
        TenantContext.setCurrentTenant(tenantId.toString());
        dataErasureService.eraseAllTenantData(tenantId);

        // 4. Drop DB schema (Bridge/Silo model)
        if (tenancyModel == TenancyModel.BRIDGE) {
            schemaService.dropSchema(tenantId);
        }

        // 5. Remove S3 data
        s3Service.deletePrefix("tenants/" + tenantId + "/");

        // 6. Mark tenant as deleted (soft delete for audit trail)
        tenantRepository.softDelete(tenantId, reason);

        // 7. Audit log
        auditLog.record(AuditEvent.TENANT_OFFBOARDED, tenantId, reason);
    }
}
```

---

## Data Residency and Compliance

```
+--------------------------------------------------+
| Global SaaS Multi-Region                        |
|                                                  |
|  Request --> Cloudflare Workers (Edge routing)  |
|                    |                             |
|         +----------+-----------+                 |
|         |                      |                 |
|    EU-WEST-1              US-EAST-1              |
|  +------------+         +------------+           |
|  | App (EU)   |         | App (US)   |           |
|  | DB (EU)    |         | DB (US)    |           |
|  | EU tenants |         | US tenants |           |
|  +------------+         +------------+           |
|                                                  |
|  Tenant record stores: region = EU | US | APAC  |
+--------------------------------------------------+
```

```java
// Cloudflare Worker: route to correct region
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const tenantId = extractTenantId(request)
  const region = await getRegionForTenant(tenantId)  // KV lookup
  const origin = REGION_ORIGINS[region]
  return fetch(new Request(origin + request.url.pathname, request))
}
```

### SOC2 Checklist

- [ ] Audit log: every data access and mutation recorded with `tenant_id`, `user_id`, `timestamp`, `action`
- [ ] Encryption at rest: DB encryption enabled (AWS RDS: automatic; self-hosted: pgcrypto)
- [ ] Encryption in transit: TLS 1.2+ everywhere; HSTS header set
- [ ] Access control: RBAC enforced; no shared credentials; service accounts per service
- [ ] Vulnerability scanning: container images scanned in CI; dependency CVE alerts enabled

---

## Anti-Patterns

- **Forgetting `tenant_id` on a table** — cross-tenant data leak; add `CHECK (tenant_id IS NOT NULL)` and RLS as defence-in-depth
- **Trusting `X-Tenant-ID` from external clients** — only trust this header from your own gateway; validate JWT at edge
- **Single Stripe customer for all tenants** — lose per-tenant billing visibility; one customer per tenant
- **No offboarding plan** — GDPR fine risk; design deletion before you need it
- **Metrics without tenant tag** — can't debug noisy neighbor; always tag with `tenant_id`
- **Running provisioning as a synchronous API call** — timeouts on slow provisioning; use async job with status polling

## Checklist

- [ ] Tenancy model chosen and documented (Pool/Bridge/Silo) with rationale
- [ ] `tenant_id` on every multi-tenant table; RLS policy applied
- [ ] JWT contains `tenant_id` claim; middleware sets `TenantContext` on every request
- [ ] `tenant_id` included in Kafka message headers
- [ ] All metrics and logs tagged with `tenant_id`
- [ ] Entitlement check before every gated feature
- [ ] Per-tenant rate limits configured
- [ ] Trial period enforced; auto-downgrade on expiry
- [ ] GDPR data erasure procedure documented and tested
- [ ] Stripe webhook handler idempotent (Stripe can retry webhooks)
- [ ] Data residency: tenant's region stored; routing enforced
