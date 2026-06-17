---
name: service-mesh
description: Invoke when designing, configuring, or troubleshooting service-to-service communication with Istio or Linkerd — including mTLS, traffic management, canary deployments, circuit breaking, and mesh observability.
---

# Service Mesh Expert (Istio + Linkerd)

## What Is a Service Mesh

A service mesh is an **infrastructure layer** that handles service-to-service communication transparently — without changing application code. It intercepts all network traffic through sidecar proxies injected alongside each service pod (or via eBPF kernel-level interception in newer implementations).

```
WITHOUT MESH                          WITH MESH (Sidecar)
-----------                           -------------------
Service A ──HTTP──> Service B         Service A ──> [Envoy] ──mTLS──> [Envoy] ──> Service B
                                         sidecar                        sidecar
  App handles: retry, timeout,         Mesh handles: retry, timeout,
  auth, tracing                        mTLS, tracing, metrics — app knows nothing
```

**Core capabilities the mesh provides — zero app code changes:**
- **mTLS**: automatic certificate rotation, encrypted + authenticated service traffic
- **Traffic management**: weighted routing, header-based routing, retries, timeouts
- **Observability**: RED metrics (Rate, Errors, Duration) per service pair; distributed tracing
- **Policy enforcement**: authorization policies (who can call whom)
- **Fault injection**: inject latency/errors for chaos testing in production-like conditions

---

## Istio

### Architecture

```
CONTROL PLANE (istiod)
+------------------------------------------+
|  Pilot          Citadel         Galley    |
|  (routing)      (certs/mTLS)   (config)  |
+------------------------------------------+
           |               |
    xDS API (gRPC)   cert distribution
           |               |
DATA PLANE (per pod)
+--------------------+   +--------------------+
| App Container      |   | App Container      |
| [Service A :8080]  |   | [Service B :8080]  |
|                    |   |                    |
| [Envoy :15001]     |   | [Envoy :15001]     |
+--------------------+   +--------------------+
     sidecar                  sidecar
```

**istiod components:**
- **Pilot**: pushes routing config (xDS) to Envoy sidecars; service discovery
- **Citadel**: CA; issues and rotates X.509 certificates for workload identity
- **Galley**: config validation and distribution

### Sidecar Injection

```bash
# Enable auto-injection for a namespace
kubectl label namespace default istio-injection=enabled

# Verify injection
kubectl get namespace default --show-labels

# Manual injection (useful for debugging)
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# Verify sidecar is running alongside app
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'
# Output: my-app istio-proxy
```

### mTLS: STRICT Mode

```yaml
# PeerAuthentication — enforce mTLS for all workloads in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # STRICT = only mTLS accepted; PERMISSIVE = both plain + mTLS
---
# Namespace-wide STRICT, but one workload in PERMISSIVE (legacy migration)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service-permissive
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE
```

```bash
# Verify mTLS is active between two services
istioctl x authz check <pod-name>
istioctl proxy-config secret <pod-name> -n production

# Check if traffic is encrypted
kubectl exec <pod> -c istio-proxy -- curl -s http://localhost:15000/config_dump | grep tls
```

### Authorization Policy (who can call whom)

```yaml
# Only allow order-service to call payment-service on /pay endpoint
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/order-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/pay"]
```

### Traffic Management

#### VirtualService — routing rules

```yaml
# Canary: 90% stable, 10% canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
  namespace: production
spec:
  hosts:
  - product-service
  http:
  - match:
    - headers:
        x-canary-user:
          exact: "true"
    route:
    - destination:
        host: product-service
        subset: canary
      weight: 100
  - route:
    - destination:
        host: product-service
        subset: stable
      weight: 90
    - destination:
        host: product-service
        subset: canary
      weight: 10
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure,retriable-4xx
---
# Fault injection: delay 2s for 10% of requests (chaos testing)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-chaos
  namespace: staging
spec:
  hosts:
  - product-service
  http:
  - fault:
      delay:
        percentage:
          value: 10
        fixedDelay: 2s
      abort:
        percentage:
          value: 5
        httpStatus: 503
    route:
    - destination:
        host: product-service
        subset: stable
```

#### DestinationRule — load balancing + circuit breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service
  namespace: production
spec:
  host: product-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN          # ROUND_ROBIN | RANDOM | LEAST_CONN | PASSTHROUGH
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 200
        maxRequestsPerConnection: 10
    outlierDetection:             # Circuit breaker: eject unhealthy hosts
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s               # Scan interval
      baseEjectionTime: 30s       # How long to eject
      maxEjectionPercent: 50      # Never eject more than 50% of hosts
      minHealthPercent: 30
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 10   # Limit canary load
```

#### Gateway — TLS termination at mesh edge

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE            # TLS termination; MUTUAL for client cert required
      credentialName: api-tls-cert   # Kubernetes TLS secret
    hosts:
    - "api.example.com"
  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true     # Force HTTPS
    hosts:
    - "api.example.com"
---
# VirtualService binding to Gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routes
  namespace: production
spec:
  hosts:
  - "api.example.com"
  gateways:
  - api-gateway
  http:
  - match:
    - uri:
        prefix: /orders
    route:
    - destination:
        host: order-service
        port:
          number: 8080
  - match:
    - uri:
        prefix: /products
    route:
    - destination:
        host: product-service
        port:
          number: 8080
```

### Observability

```bash
# Access Kiali (topology graph)
istioctl dashboard kiali

# Access Jaeger (distributed tracing)
istioctl dashboard jaeger

# Access Grafana (RED metrics dashboards)
istioctl dashboard grafana

# Check mesh status
istioctl analyze --namespace production

# View proxy config for a pod
istioctl proxy-config routes <pod-name> -n production
istioctl proxy-config clusters <pod-name> -n production
istioctl proxy-config listeners <pod-name> -n production

# Top traffic by service
kubectl -n production exec -it <pod> -c istio-proxy -- pilot-agent request GET /stats/prometheus | grep istio_requests_total
```

**Automatic metrics exposed (no instrumentation needed):**
- `istio_requests_total` — request count by source, destination, response code
- `istio_request_duration_milliseconds` — latency histogram
- `istio_tcp_connections_opened_total` / `closed_total` — TCP connection tracking

---

## Linkerd

### Architecture (lighter than Istio)

```
CONTROL PLANE
+------------------------------------------+
|  linkerd-controller   linkerd-identity    |
|  linkerd-destination  linkerd-proxy-injector|
+------------------------------------------+
           |
DATA PLANE (per pod)
+-----------------------------+
| App Container :8080        |
| linkerd2-proxy :4143/4191  |  <- Rust, purpose-built (vs Envoy)
+-----------------------------+
```

**Key differences from Istio:**
| Feature | Linkerd | Istio |
|---------|---------|-------|
| Proxy | linkerd2-proxy (Rust) | Envoy (C++) |
| Config complexity | Low | High |
| Resource footprint | ~50MB/pod | ~150MB/pod |
| Traffic API | SMI (simple) | VirtualService (powerful) |
| mTLS | Auto, zero-config | Requires PeerAuthentication |
| Ecosystem | Smaller | Large (plugins, addons) |

### Install and Inject

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Validate cluster
linkerd check --pre

# Install control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check

# Inject sidecar into a deployment
kubectl get deploy order-service -o yaml | linkerd inject - | kubectl apply -f -

# Or annotate namespace for auto-injection
kubectl annotate namespace production linkerd.io/inject=enabled

# Verify injection
linkerd check --proxy -n production
```

### mTLS — Automatic, Zero-Config

```bash
# Verify mTLS between services (no YAML needed — it's on by default)
linkerd viz edges deployment -n production

# Tap live traffic (inspect requests in real time)
linkerd viz tap deployment/order-service -n production

# Check TLS status
linkerd viz edges pod -n production
```

### Traffic Split (Canary via SMI)

```yaml
# SMI TrafficSplit: 90/10 canary
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: order-service-split
  namespace: production
spec:
  service: order-service
  backends:
  - service: order-service-stable
    weight: 900m        # 90% (milliweight)
  - service: order-service-canary
    weight: 100m        # 10%
```

### Observability — Golden Metrics

```bash
# Service-level golden metrics: success rate, RPS, p99 latency
linkerd viz stat deployments -n production

# Per-route metrics
linkerd viz stat routes -n production deploy/order-service

# Live request tap
linkerd viz tap deploy/order-service -n production --to deploy/payment-service

# Open dashboard
linkerd viz dashboard
```

**Golden metrics per service (automatic, no SDK needed):**
- Success rate (non-5xx %)
- Request rate (RPS)
- Latency (p50, p95, p99)

---

## When to Use a Service Mesh

### Use a Mesh When:
- More than 5 microservices communicating with each other
- Compliance requires encrypted service-to-service traffic (mTLS)
- Canary deployments or A/B testing needed without app changes
- Distributed tracing without instrumenting every service
- Fine-grained traffic control (retries, timeouts, fault injection) centrally
- Zero-trust network security model required

### Skip the Mesh When:
- Monolith or fewer than 3 services — overhead is not justified
- Team lacks bandwidth to operate and maintain the control plane
- Latency budget is extremely tight — sidecar adds ~1-5ms per hop
- Kubernetes expertise is limited — Istio complexity is real
- Simple use case: a library solves it (Resilience4j, Polly)

### Alternatives to a Full Mesh:

```
Need                          Alternative
----                          -----------
Circuit breaker               Resilience4j (Java), Polly (.NET), resilience (Go)
Distributed tracing           OpenTelemetry SDK in each service
mTLS                          cert-manager + Kubernetes NetworkPolicy
Traffic splitting             Kubernetes native: two Deployments + manual weight
Rate limiting                 API Gateway (Kong, nginx)
```

---

## Anti-Patterns

- **Enabling STRICT mTLS before all services are injected** — breaks traffic to uninjected services; use PERMISSIVE during migration
- **Using VirtualService without matching DestinationRule subsets** — subset not found → 503s
- **Istio on a 1-node dev cluster** — control plane alone consumes ~500MB RAM
- **Ignoring `istioctl analyze`** — it catches misconfigured resources before they cause issues
- **Too many retries without idempotency** — retrying non-idempotent POST requests causes duplicate operations
- **Giant outlierDetection ejection windows** — ejecting 100% of hosts causes full outage; always set `maxEjectionPercent < 100`
- **Not setting resource limits on Envoy sidecars** — noisy neighbor problem; sidecars consume unbounded CPU during traffic spikes

---

## Checklist: Mesh Rollout

- [ ] Inject sidecars into non-critical namespaces first; verify mTLS with `istioctl x authz check`
- [ ] Set `mode: PERMISSIVE` initially; migrate to `STRICT` after all services injected
- [ ] Define DestinationRule before VirtualService — subsets must exist before routing references them
- [ ] Set `outlierDetection.maxEjectionPercent` to 50% or less
- [ ] Add retry budgets only to idempotent endpoints (GET, PUT with idempotency key)
- [ ] Configure resource requests/limits for `istio-proxy` container
- [ ] Test fault injection in staging before relying on retries in production
- [ ] Set up Kiali + Grafana dashboards for topology visibility from day one
- [ ] Run `istioctl analyze --namespace <ns>` as part of CI/CD pipeline
- [ ] Document egress policy — block all external traffic by default via `outboundTrafficPolicy: REGISTRY_ONLY`
