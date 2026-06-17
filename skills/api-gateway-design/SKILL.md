---
name: api-gateway-design
description: Invoke when designing API gateways, BFF (Backend for Frontend) layers, rate limiting strategies, authentication at the edge, request aggregation, or selecting and configuring gateway products like Kong, AWS API Gateway, Traefik, or Spring Cloud Gateway.
---

# API Gateway and BFF Pattern

## What an API Gateway Does

An API gateway is the **single entry point** for all client traffic. It handles cross-cutting concerns centrally so individual services don't have to.

```
CLIENTS                     API GATEWAY                    INTERNAL SERVICES
-------                     -----------                    -----------------
                         +------------------+
Web Browser  ─────────>  | Authentication   |  ──────>  Order Service    :8081
                         | Rate Limiting    |
Mobile App   ─────────>  | Routing          |  ──────>  Product Service  :8082
                         | SSL Termination  |
IoT Device   ─────────>  | Request Transform|  ──────>  Payment Service  :8083
                         | Circuit Breaking |
Partner API  ─────────>  | Caching          |  ──────>  User Service     :8084
                         | Observability    |
                         +------------------+
                         HTTPS (external)       HTTP/gRPC (internal)
```

---

## Gateway Responsibilities

### 1. Routing

```yaml
# Kong: route /api/v1/orders → order-service
_format_version: "3.0"
services:
- name: order-service
  url: http://order-service:8081
  routes:
  - name: orders-route
    paths: ["/api/v1/orders"]
    methods: ["GET", "POST"]
    strip_path: false

- name: product-service
  url: http://product-service:8082
  routes:
  - name: products-route
    paths: ["/api/v1/products"]
    methods: ["GET"]
```

### 2. Authentication — Verify Before Forwarding

Never let unauthenticated requests reach internal services. Validate at the gateway; forward claims as headers.

```yaml
# Kong JWT plugin: validate at gateway, forward claims downstream
plugins:
- name: jwt
  service: order-service
  config:
    key_claim_name: kid
    claims_to_verify:
    - exp
    - nbf
    header_names:
    - Authorization
```

```python
# Custom auth middleware (nginx + lua / Kong plugin pattern)
def validate_and_forward(request):
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience="api.example.com",
            issuer="auth.example.com"
        )
    except jwt.ExpiredSignatureError:
        return Response(401, {"error": "token_expired"})
    except jwt.InvalidTokenError:
        return Response(401, {"error": "invalid_token"})
    
    # Forward claims as headers — services trust these (internal network)
    request.headers["X-User-Id"] = payload["sub"]
    request.headers["X-User-Role"] = payload.get("role", "user")
    request.headers["X-Tenant-Id"] = payload.get("tenant_id", "")
    
    return forward_to_upstream(request)
```

### 3. Rate Limiting

```
RATE LIMITING ALGORITHMS

Fixed Window          Sliding Window        Token Bucket          Leaky Bucket
------------          --------------        ------------          ------------
|  window 1  |        continuous window     tokens at rate R      queue at rate R
| count: 95  |        precise, no burst     burst up to B         smooth output
| [burst!]   |        more memory           most common           no burst allowed
|  window 2  |        Redis sorted set      Redis counter         queue depth = B
| count: 105 |        per user              per user per endpoint
| → 429!     |

Recommendation: Token Bucket for most APIs
```

```python
# Token bucket rate limiting with Redis
import redis
import time

r = redis.Redis()

def check_rate_limit(client_id: str, endpoint: str, 
                     rate: int = 100, capacity: int = 200) -> tuple[bool, dict]:
    """
    rate: tokens added per second
    capacity: max burst size (bucket capacity)
    Returns: (allowed, headers)
    """
    key = f"ratelimit:{client_id}:{endpoint}"
    now = time.time()
    
    pipe = r.pipeline()
    pipe.hgetall(key)
    pipe.expire(key, 3600)
    result, _ = pipe.execute()
    
    if result:
        tokens = float(result[b'tokens'])
        last_refill = float(result[b'last_refill'])
        # Add tokens since last refill
        elapsed = now - last_refill
        tokens = min(capacity, tokens + elapsed * rate)
    else:
        tokens = float(capacity)
        last_refill = now
    
    if tokens >= 1.0:
        tokens -= 1.0
        allowed = True
    else:
        allowed = False
    
    pipe = r.pipeline()
    pipe.hset(key, mapping={'tokens': tokens, 'last_refill': now})
    pipe.expire(key, 3600)
    pipe.execute()
    
    retry_after = int((1.0 - tokens) / rate) if not allowed else 0
    headers = {
        "X-RateLimit-Limit": str(rate),
        "X-RateLimit-Remaining": str(int(tokens)),
        "X-RateLimit-Reset": str(int(now + (capacity - tokens) / rate)),
        **({"Retry-After": str(retry_after)} if not allowed else {})
    }
    return allowed, headers
```

```yaml
# Kong rate-limiting plugin: tier-based limits
plugins:
- name: rate-limiting
  config:
    second: null
    minute: 60        # 60 req/min for anonymous
    hour: 1000
    policy: redis
    redis_host: redis
    redis_port: 6379

# Per consumer (API key tier)
consumers:
- username: partner-acme
  plugins:
  - name: rate-limiting
    config:
      minute: 600     # 10x higher for partners
      hour: 50000
```

### 4. SSL Termination

```
Client ─── HTTPS (TLS 1.3) ──> Gateway ─── HTTP ──> Service A
                                         └── HTTP ──> Service B
                                         └── gRPC  ──> Service C
              cert lives at gateway;
              internal traffic is plain (within private VPC)
              or mTLS if service mesh is used
```

### 5. Circuit Breaking at Gateway

```yaml
# Kong circuit breaker (Upstream health checks)
upstreams:
- name: order-service-upstream
  healthchecks:
    active:
      http_path: /health/ready
      healthy:
        interval: 10
        successes: 2
      unhealthy:
        interval: 5
        http_failures: 3
        timeouts: 3
    passive:
      healthy:
        successes: 5
      unhealthy:
        http_failures: 5
        http_statuses: [500, 502, 503, 504]
        timeouts: 3
  targets:
  - target: order-service-1:8081
    weight: 100
  - target: order-service-2:8081
    weight: 100
```

---

## BFF (Backend for Frontend) Pattern

### Problem: One Generic API Serves All Clients Badly

```
WRONG: Single generic API
        API
         |
    +---------+
    |         |         Web gets TOO MUCH data (over-fetching)
   Web      Mobile      Mobile makes TOO MANY requests (under-fetching)
                        TV app needs completely different data shape
```

```
RIGHT: BFF per client type
        Web BFF          Mobile BFF        TV BFF
            |                 |               |
    +-------+-------+---------+-----+---------+
    |               |               |         |
 Order Svc    Product Svc    User Svc    Catalog Svc
 (internal)   (internal)    (internal)  (internal)
```

### BFF Responsibilities Per Client

```
WEB BFF                         MOBILE BFF                    TV BFF
-------                         ----------                    ------
Complex aggregations             Minimal payloads              Lean content lists
Full product data                Compressed images only        No user account ops
Session-based auth (cookies)     JWT token auth                Device-registered tokens
Server-side rendering support    Offline-first endpoints       Simple navigation-only API
Admin features                   Push notification hooks       Autoplay/continue-watching
Rich pagination                  Cursor-based pagination       Simple next/prev
WebSockets for real-time         Long polling fallback         Polling (smart TV limits)
```

### BFF Implementation (Spring Boot)

```java
// Mobile BFF: aggregate order + product data into minimal payload
@RestController
@RequestMapping("/mobile/v1")
public class MobileOrderBff {

    private final OrderServiceClient orderClient;
    private final ProductServiceClient productClient;
    private final UserServiceClient userClient;

    // Aggregate: one call returns everything mobile checkout screen needs
    @GetMapping("/orders/{orderId}/checkout-summary")
    public Mono<CheckoutSummary> getCheckoutSummary(
            @PathVariable String orderId,
            @RequestHeader("X-User-Id") String userId) {

        // Fan-out: call 3 services concurrently
        Mono<Order> orderMono = orderClient.getOrder(orderId);
        Mono<User> userMono = userClient.getUser(userId);

        return orderMono.flatMap(order -> {
            // Fetch only the product fields mobile needs (no full catalog data)
            List<String> productIds = order.getLines().stream()
                .map(OrderLine::getProductId)
                .toList();
            Mono<List<ProductSummary>> productsMono =
                productClient.getProductSummaries(productIds);  // Mobile-optimized endpoint

            return Mono.zip(userMono, productsMono)
                .map(tuple -> CheckoutSummary.builder()
                    .orderId(order.getId())
                    .total(order.getTotal())
                    .currency(order.getCurrency())
                    // Only fields mobile needs — not 50-field full product
                    .items(buildMobileItems(order.getLines(), tuple.getT2()))
                    .shippingAddress(tuple.getT1().getDefaultAddress())
                    .estimatedDelivery(order.getEstimatedDelivery())
                    .build());
        });
    }
}

// Response: lean payload for mobile
record CheckoutSummary(
    String orderId,
    BigDecimal total,
    String currency,
    List<MobileOrderItem> items,
    Address shippingAddress,
    LocalDate estimatedDelivery
) {}

record MobileOrderItem(
    String productId,
    String name,
    String thumbnailUrl,  // Small image only
    int quantity,
    BigDecimal price
) {}
```

### Who Owns the BFF

```
WRONG: Platform team owns all BFFs
  → Bottleneck; frontend blocked on platform team; misaligned priorities

RIGHT: Frontend team owns their BFF
  Web team    → owns Web BFF
  iOS team    → owns Mobile BFF (or shared with Android)
  TV team     → owns TV BFF

  Internal services don't know about clients — BFF is the adapter layer
```

### When NOT to Use BFF

- Only one client type (no differentiation needed)
- Team too small to maintain separate services
- Data needs are identical across clients
- GraphQL federation already solves the aggregation problem

---

## API Gateway Products

### Kong (Open Source)

```yaml
# Declarative config (deck sync)
_format_version: "3.0"
_transform: true

services:
- name: order-service
  url: http://order-service:8081
  connect_timeout: 5000
  read_timeout: 30000
  write_timeout: 30000
  retries: 3
  routes:
  - name: orders
    paths: ["/api/v1/orders"]
    methods: ["GET", "POST", "PUT"]
  plugins:
  - name: jwt
    config:
      secret_is_base64: false
  - name: rate-limiting
    config:
      minute: 120
      policy: redis
  - name: request-transformer
    config:
      add:
        headers: ["X-Gateway-Version:2.0"]
      remove:
        headers: ["X-Internal-Debug"]
  - name: prometheus
    config:
      per_consumer: true
```

```bash
# Deploy Kong with Helm
helm repo add kong https://charts.konghq.com
helm install kong kong/kong \
  --set ingressController.enabled=true \
  --set postgresql.enabled=true \
  --set env.database=postgres

# Sync declarative config
deck sync --config kong.yaml

# Check routes
curl http://localhost:8001/routes | jq '.data[].paths'
```

### AWS API Gateway

```yaml
# serverless.yml: Lambda + API Gateway
functions:
  orders:
    handler: handlers/orders.handler
    events:
    - http:
        path: /orders
        method: POST
        authorizer:
          name: jwtAuthorizer
          type: TOKEN
          identitySource: method.request.header.Authorization
        throttling:
          maxRequestsPerSecond: 100
          maxConcurrentRequests: 50

  jwtAuthorizer:
    handler: handlers/authorizer.handler
```

### Traefik (Kubernetes-Native)

```yaml
# IngressRoute with middlewares
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-routes
  namespace: production
spec:
  entryPoints:
  - websecure
  routes:
  - match: Host(`api.example.com`) && PathPrefix(`/orders`)
    kind: Rule
    services:
    - name: order-service
      port: 8080
    middlewares:
    - name: auth-middleware
    - name: rate-limit-middleware
  tls:
    certResolver: letsencrypt   # Auto TLS from Let's Encrypt
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rate-limit-middleware
spec:
  rateLimit:
    average: 100      # 100 req/s
    burst: 200        # Allow burst up to 200
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: auth-middleware
spec:
  forwardAuth:
    address: http://auth-service:8080/validate
    authResponseHeaders:
    - X-User-Id
    - X-User-Role
```

### Spring Cloud Gateway (Java / Reactive)

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder,
                               JwtAuthFilter jwtFilter,
                               RateLimitFilter rateLimitFilter) {
        return builder.routes()
            // Order service routes
            .route("order-service", r -> r
                .path("/api/v1/orders/**")
                .filters(f -> f
                    .filter(jwtFilter)
                    .filter(rateLimitFilter)
                    .addRequestHeader("X-Gateway", "spring-cloud-gateway")
                    .addResponseHeader("X-Response-Time", "#{T(java.time.Instant).now()}")
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.BAD_GATEWAY, HttpStatus.SERVICE_UNAVAILABLE)
                        .setMethods(HttpMethod.GET)
                        .setBackoff(Duration.ofMillis(100), Duration.ofSeconds(2), 2, true))
                    .circuitBreaker(config -> config
                        .setName("order-service-cb")
                        .setFallbackUri("forward:/fallback/orders")))
                .uri("lb://order-service"))  // Load-balanced via service discovery

            // Product service with caching
            .route("product-service", r -> r
                .path("/api/v1/products/**")
                .and().method(HttpMethod.GET)
                .filters(f -> f
                    .filter(jwtFilter)
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://product-service"))
            .build();
    }

    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20, 1);  // replenish rate, burst, requested tokens
    }
}
```

```yaml
# application.yml fallback
spring:
  cloud:
    gateway:
      default-filters:
      - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "https://app.example.com"
            allowedMethods: ["GET", "POST", "PUT", "DELETE"]
            allowedHeaders: ["*"]
            allowCredentials: true
```

---

## Authentication Patterns at the Gateway

```
PATTERN              HOW IT WORKS                           USE WHEN
-------              ------------                           --------
JWT validation       Verify signature + claims at gateway   Stateless; microservices
API key              Lookup key in Redis cache              Partner/B2B integrations
OAuth2 introspection Forward token to auth server          Opaque tokens; revocation needed
mTLS                 Client presents certificate            Service-to-service; zero-trust
Pass-through         Forward auth headers to service        Legacy services; avoid if possible
```

```python
# OAuth2 token introspection with caching (avoid hitting auth server every request)
async def introspect_token(token: str, cache: Redis, auth_client: AuthClient) -> dict:
    cache_key = f"introspect:{hashlib.sha256(token.encode()).hexdigest()}"
    
    cached = await cache.get(cache_key)
    if cached:
        return json.loads(cached)
    
    result = await auth_client.introspect(token)
    
    if not result.get("active"):
        raise HTTPException(status_code=401, detail="Token inactive or expired")
    
    ttl = result.get("exp", 0) - int(time.time())
    if ttl > 0:
        await cache.setex(cache_key, min(ttl, 300), json.dumps(result))  # Cache max 5 min
    
    return result
```

---

## Request Aggregation Patterns

### REST Aggregation (Gateway fan-out)

```python
# BFF: fetch from 3 services concurrently, merge response
import asyncio
import httpx

async def get_order_detail(order_id: str, user_id: str) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        # Fan-out: all 3 calls in parallel
        order_task = client.get(f"http://order-service/orders/{order_id}")
        user_task = client.get(f"http://user-service/users/{user_id}")
        shipping_task = client.get(f"http://shipping-service/tracking/{order_id}")
        
        # Gather with timeout — partial response on failure
        results = await asyncio.gather(
            order_task, user_task, shipping_task,
            return_exceptions=True
        )
        
        order_resp, user_resp, shipping_resp = results
        
        response = {}
        
        if not isinstance(order_resp, Exception):
            response["order"] = order_resp.json()
        else:
            raise HTTPException(503, "Order service unavailable")  # Critical — fail
        
        if not isinstance(user_resp, Exception):
            response["user"] = user_resp.json()
        # else: non-critical — return without user data
        
        if not isinstance(shipping_resp, Exception):
            response["tracking"] = shipping_resp.json()
        else:
            response["tracking"] = {"status": "unavailable"}  # Graceful degradation
        
        return response
```

### GraphQL Federation

```graphql
# Each service owns its type — gateway federates automatically
# order-service schema:
type Order @key(fields: "id") {
  id: ID!
  total: Float!
  status: String!
  userId: ID!
  user: User          # Resolved by user-service federation
  products: [Product] # Resolved by product-service federation
}

# user-service schema:
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
}

# product-service schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# Client query — gateway handles federation automatically:
query GetOrderDetail($orderId: ID!) {
  order(id: $orderId) {
    id
    total
    status
    user {        # Fetched from user-service
      name
      email
    }
    products {    # Fetched from product-service
      name
      price
    }
  }
}
```

---

## Observability at the Gateway

```yaml
# Kong Prometheus plugin: RED metrics per route + consumer
plugins:
- name: prometheus
  config:
    per_consumer: true    # Metrics by API key / user
    status_code_metrics: true
    latency_metrics: true
    bandwidth_metrics: true

# Key metrics to alert on:
# kong_http_requests_total{route="orders",status="5xx"} > 1%  → alert
# kong_latency_ms{route="orders",quantile="0.99"} > 2000      → alert
# kong_upstream_target_health{target="order-svc"} == 0        → alert (all unhealthy)
```

```python
# Inject distributed trace into forwarded request
import opentelemetry.propagate as propagate

def forward_with_tracing(request, upstream_url):
    with tracer.start_as_current_span("gateway.forward") as span:
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.route", request.path)
        span.set_attribute("upstream.url", upstream_url)
        
        headers = dict(request.headers)
        # Inject W3C trace context into outgoing request
        propagate.inject(headers)
        
        response = httpx.request(
            request.method,
            upstream_url,
            headers=headers,
            content=request.body
        )
        span.set_attribute("http.status_code", response.status_code)
        return response
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Business logic in the gateway | Gateway becomes a service; hard to test; bottleneck | Logic in services; gateway only for cross-cutting concerns |
| No circuit breaker to upstream | Gateway keeps forwarding to dead services; cascades | Add health checks + passive circuit breaker per upstream |
| Rate limiting without burst allowance | Legitimate traffic spikes get throttled | Token bucket: set burst = 2x rate |
| Validating JWTs in every service | Code duplication; inconsistent; key management burden | Validate only at gateway; forward claims as trusted headers |
| Single gateway for all clients | Web and mobile have different SLAs; one outage affects all | Separate gateway instances or BFF layers per client type |
| No timeout on upstream calls | One slow service holds gateway connections; cascades | Always set connect + read timeout per upstream |
| Aggregating in gateway (not BFF) | Coupling gateway to business data shapes | Aggregation in BFF; gateway handles infrastructure concerns |
| Skipping auth on internal routes | Internal services exposed if VPC misconfigured | mTLS or token validation on every route |

---

## Design Checklist

**Gateway Configuration**
- [ ] All routes require authentication (no anonymous internal routes)
- [ ] Rate limiting configured per consumer tier (free / standard / partner)
- [ ] Circuit breaker enabled for every upstream service
- [ ] Timeout set on all upstream routes (connect + read)
- [ ] SSL termination at gateway; internal traffic on private VPC
- [ ] Access logs enabled and shipped to observability platform
- [ ] Distributed trace context injected into all forwarded requests

**BFF Design**
- [ ] BFF owned by the frontend team that consumes it
- [ ] Response payloads contain only fields the client needs (no over-fetching)
- [ ] Fan-out calls to internal services are concurrent (not sequential)
- [ ] Partial response on non-critical service failure (graceful degradation)
- [ ] BFF has its own rate limit + circuit breaker (independent from shared gateway)

**Rate Limiting**
- [ ] Token bucket algorithm (allows burst, smooth average)
- [ ] Limits stored in Redis (distributed; works across gateway replicas)
- [ ] `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers returned
- [ ] `429 Too Many Requests` with `Retry-After` header on limit breach
- [ ] Different limits per API key tier

**Authentication**
- [ ] JWT signature verified (not just decoded) at gateway
- [ ] `exp`, `iss`, `aud` claims validated
- [ ] Verified claims forwarded as trusted headers to services
- [ ] Auth server introspection results cached in Redis (avoid hitting auth server per request)
