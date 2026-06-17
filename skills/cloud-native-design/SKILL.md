---
name: cloud-native-design
description: Invoke when designing, reviewing, or refactoring services to be cloud-native — covering 12-Factor principles, stateless design, graceful shutdown, container best practices, GitOps, and cloud-native patterns like sidecar and bulkhead.
---

# Cloud-Native Design Principles

## What "Cloud-Native" Means

Cloud-native applications are designed to exploit elastic, distributed cloud infrastructure: they scale horizontally, recover automatically, deploy frequently, and treat infrastructure as disposable. The foundation is the **12-Factor App** methodology, extended for modern container/Kubernetes environments.

```
CLOUD-NATIVE PROPERTIES
+-----------------------------------------------------------+
|  Stateless     Horizontally    Immutable    Observability  |
|  processes  +  scalable     +  artifacts +  built-in      |
+-----------------------------------------------------------+
|  Declarative   Automated       Fast           GitOps       |
|  config     +  recovery    +  startup  +  driven          |
+-----------------------------------------------------------+
```

---

## The 12-Factor App (2025 Edition)

### Factor 1 — Codebase
One codebase per service in version control. Multiple environments (dev/staging/prod) are **deploys** of the same codebase — never separate repos per environment.

```
WRONG                           RIGHT
------                          -----
/repos/my-service-dev           /repos/my-service
/repos/my-service-staging         └── deploys to: dev / staging / prod
/repos/my-service-prod              via env vars + CI/CD pipeline
```

### Factor 2 — Dependencies
Explicitly declare all dependencies. Never rely on system-installed packages.

```toml
# Rust: Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

```xml
<!-- Java: pom.xml — no "apt install libssl" in Dockerfile -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```python
# Python: requirements.txt or pyproject.toml — pin versions
fastapi==0.111.0
uvicorn[standard]==0.30.0
httpx==0.27.0
```

### Factor 3 — Config
Store config in environment variables. Never hardcode. Never commit secrets to repo.

```python
# WRONG
DATABASE_URL = "postgresql://prod-host:5432/mydb"
SECRET_KEY = "supersecret123"

# RIGHT
import os
DATABASE_URL = os.environ["DATABASE_URL"]           # Fails fast if missing
SECRET_KEY = os.environ.get("SECRET_KEY")           # Optional
LOG_LEVEL = os.environ.get("LOG_LEVEL", "INFO")     # With default
```

```yaml
# Kubernetes: inject config via env vars (not hardcoded in image)
env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: url
- name: KAFKA_BROKERS
  valueFrom:
    configMapKeyRef:
      name: kafka-config
      key: brokers
```

**Config hierarchy (strictest to most flexible):**
1. Secrets (Vault, AWS Secrets Manager, K8s Secrets) — credentials
2. Environment-specific ConfigMaps — per-env tuning
3. Application defaults — safe fallbacks

### Factor 4 — Backing Services
Treat databases, caches, and message brokers as **attached resources** — swap without code changes.

```
App code references: DATABASE_URL, REDIS_URL, KAFKA_BROKERS
                        |               |           |
                   RDS Postgres    ElastiCache   MSK Kafka    (prod)
                   LocalPostgres   Local Redis   Local Kafka  (dev)
                   TestContainers  TestContainers TestContainers (test)
```

### Factor 5 — Build / Release / Run
Strict separation between stages. Releases are immutable.

```
BUILD stage:          source + deps → executable artifact (Docker image)
RELEASE stage:        artifact + config → versioned release (image:sha256-abc123)
RUN stage:            execute release in environment (kubectl rollout)

NEVER: ssh into prod and git pull + restart
NEVER: edit files in a running container
```

```bash
# Immutable release: tag by git SHA, never overwrite :latest in prod
docker build -t my-service:${GIT_SHA} .
docker push my-service:${GIT_SHA}
# Deploy specific SHA — rollback = deploy previous SHA
kubectl set image deployment/my-service app=my-service:${GIT_SHA}
```

### Factor 6 — Processes
Services are **stateless** and **share nothing**. All persistent state lives in backing services.

```
WRONG (stateful process)          RIGHT (stateless process)
------------------------          ------------------------
User session in memory            Session stored in Redis
Local file upload cache           Files in S3/GCS
In-memory job queue               Jobs in SQS/RabbitMQ
Sticky sessions (session affinity) Any pod can serve any request
```

### Factor 7 — Port Binding
Service is self-contained. It exports a service via a port — no runtime web server injection.

```python
# RIGHT: app binds its own port
if __name__ == "__main__":
    uvicorn.run("app:app", host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

```dockerfile
# Self-contained — no external web server needed
EXPOSE 8080
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Factor 8 — Concurrency
Scale out via the process model (horizontal). Don't thread-hack to scale.

```
Scale UP (wrong for cloud-native): bigger VM, more threads, more heap
Scale OUT (right):                  more pods, each small, stateless
                                    Kubernetes HPA handles this automatically
```

```yaml
# Horizontal Pod Autoscaler: scale based on CPU/RPS
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Factor 9 — Disposability
Fast startup (< 10 seconds). Graceful shutdown (drain in-flight requests on SIGTERM).

```python
# Python FastAPI: graceful shutdown
import signal
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await db.connect()
    yield
    # Shutdown — runs when SIGTERM received
    await db.disconnect()
    await cache.close()

app = FastAPI(lifespan=lifespan)
```

```java
// Spring Boot: graceful shutdown built-in
// application.yaml
server:
  shutdown: graceful          # Wait for in-flight requests
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```go
// Go: SIGTERM handler
func main() {
    srv := &http.Server{Addr: ":8080", Handler: router}
    
    go srv.ListenAndServe()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)  // Drain in-flight requests
}
```

### Factor 10 — Dev/Prod Parity
Minimize gap between dev and prod: same Docker image, same backing service versions.

```yaml
# docker-compose.yml: local dev uses same images as prod
services:
  postgres:
    image: postgres:16.3        # Same major version as RDS prod
  redis:
    image: redis:7.2-alpine     # Same as ElastiCache prod
  kafka:
    image: confluentinc/cp-kafka:7.6.0
```

```java
// TestContainers: integration tests use real databases, not H2 in-memory
@Testcontainers
class OrderRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16.3");
    
    @DynamicPropertySource
    static void configureProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }
}
```

### Factor 11 — Logs
Treat logs as event streams. Write to stdout. Let the platform aggregate.

```python
# WRONG: write to files
logging.basicConfig(filename='/var/log/app.log')

# RIGHT: write to stdout in structured JSON
import structlog
log = structlog.get_logger()
log.info("order_created", order_id=order.id, amount=order.total, user_id=user.id)
# Output: {"event": "order_created", "order_id": "abc", "amount": 99.99, ...}
```

```
App stdout → Fluentbit/Fluentd (DaemonSet) → Elasticsearch / Loki / CloudWatch
                                              (Platform aggregates — app doesn't care)
```

### Factor 12 — Admin Processes
One-off admin tasks run as separate commands in the same codebase — not SSH sessions into prod.

```bash
# WRONG: ssh into pod and run migration
kubectl exec -it <pod> -- bash
python manage.py migrate

# RIGHT: Kubernetes Job for migrations (runs before deployment)
kubectl apply -f migration-job.yaml

# Or init container in Deployment
initContainers:
- name: run-migrations
  image: my-service:${GIT_SHA}
  command: ["python", "manage.py", "migrate"]
```

---

## Cloud-Native Patterns

### Health Endpoints

```python
# FastAPI: liveness vs readiness
@app.get("/health/live")
async def liveness():
    # Am I running? (simple — never check external deps here)
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness():
    # Can I serve traffic? (check DB, cache connectivity)
    checks = {}
    try:
        await db.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"
        raise HTTPException(status_code=503, detail=checks)
    
    try:
        await redis.ping()
        checks["cache"] = "ok"
    except Exception:
        checks["cache"] = "error"
        raise HTTPException(status_code=503, detail=checks)
    
    return checks
```

```yaml
# Kubernetes probes
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3    # Restart after 3 failures

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2    # Remove from load balancer after 2 failures

startupProbe:
  httpGet:
    path: /health/live
    port: 8080
  failureThreshold: 30   # Give slow-starting apps 30 * 10s = 5 min
  periodSeconds: 10
```

### Sidecar Pattern

```
POD
+---------------------------------------+
|  Main Container (business logic)      |
|  my-service :8080                     |
|    - knows nothing about logging      |
|    - knows nothing about secrets      |
|    - knows nothing about networking   |
+---------------------------------------+
|  Sidecar: Fluentbit                   |  <- Ships logs to Elasticsearch
|  Sidecar: Vault Agent                 |  <- Injects secrets into files
|  Sidecar: Envoy (Istio)              |  <- mTLS + traffic management
+---------------------------------------+
         Shared: volumes, network namespace
```

```yaml
# Vault Agent sidecar: inject secrets without app SDK
spec:
  serviceAccountName: my-service
  containers:
  - name: my-service
    image: my-service:v1.2.3
    volumeMounts:
    - name: secrets
      mountPath: /vault/secrets
      readOnly: true
  - name: vault-agent
    image: vault:1.16
    args: ["agent", "-config=/vault/config/config.hcl"]
    volumeMounts:
    - name: vault-config
      mountPath: /vault/config
    - name: secrets
      mountPath: /vault/secrets
  volumes:
  - name: secrets
    emptyDir:
      medium: Memory   # tmpfs — secrets never touch disk
```

### Bulkhead Pattern in Kubernetes

```
WRONG: All services share one namespace, one resource pool
  → background jobs starve checkout service during peak

RIGHT: Namespace per criticality tier
  production/
    namespace: critical        (checkout, payment, auth)   — quota: 40 CPU, 80GB RAM
    namespace: standard        (catalog, search, reviews)  — quota: 20 CPU, 40GB RAM
    namespace: background      (jobs, reports, ML)         — quota: 10 CPU, 20GB RAM
```

```yaml
# ResourceQuota for critical namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: critical-quota
  namespace: critical
spec:
  hard:
    requests.cpu: "40"
    requests.memory: 80Gi
    limits.cpu: "60"
    limits.memory: 100Gi
    pods: "200"
---
# LimitRange: default limits per container
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: critical
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: 512Mi
    defaultRequest:
      cpu: "100m"
      memory: 128Mi
```

### GitOps (ArgoCD)

```
Developer                 Git                    ArgoCD              Kubernetes
    |                      |                       |                      |
    |-- git push --------> |                       |                      |
    |                      |-- webhook / poll ----> |                      |
    |                      |                       |-- diff desired vs    |
    |                      |                       |   actual state       |
    |                      |                       |-- kubectl apply ---> |
    |                      |                       |                      |
    |                      |                       |<-- sync status ----  |
```

```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/my-org/k8s-manifests
    targetRevision: main
    path: services/order-service/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual kubectl changes
    syncOptions:
    - CreateNamespace=true
```

### Event-Driven Cloud Native

```yaml
# Knative Service: scale to zero when no traffic
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-processor
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"    # Scale to zero
        autoscaling.knative.dev/max-scale: "50"
    spec:
      containers:
      - image: event-processor:v1.2.3
---
# CloudEvents: standard event envelope
{
  "specversion": "1.0",
  "type": "com.example.order.created",
  "source": "/order-service",
  "id": "A234-1234-1234",
  "time": "2025-01-15T17:31:00Z",
  "datacontenttype": "application/json",
  "data": {
    "orderId": "abc-123",
    "userId": "user-456",
    "total": 99.99
  }
}
```

---

## Container Best Practices

### Distroless + Multi-Stage Build

```dockerfile
# Stage 1: Build (heavy — has compiler, build tools)
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# Stage 2: Runtime (minimal — no shell, no package manager)
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

```dockerfile
# Java: distroless JRE
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

FROM gcr.io/distroless/java21-debian12:nonroot
COPY --from=builder /app/target/app.jar /app.jar
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Security in Kubernetes Pod Spec

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true     # Can't write to container filesystem
      capabilities:
        drop:
        - ALL                           # Drop all Linux capabilities
    volumeMounts:
    - name: tmp
      mountPath: /tmp                   # Allow writes only to specific dirs
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Config in Docker image | Rebuilding for each environment | Inject via env vars / ConfigMaps |
| Session state in pod memory | Breaks horizontal scaling; sticky sessions needed | Redis for session store |
| Writing logs to files in container | Lost when pod dies; hard to aggregate | Write to stdout |
| SSH into prod to fix things | Not reproducible; config drift; audit nightmare | GitOps + immutable deploys |
| Huge container images | Slow pull times; large attack surface | Multi-stage + distroless |
| Running as root in container | Privilege escalation risk | `USER nonroot` + drop ALL caps |
| No liveness/readiness probes | K8s sends traffic to dead pods | Always define both probes |
| No resource limits | Noisy neighbor starves other pods | Always set requests + limits |
| Single replica in production | No HA; rolling update causes downtime | `minReplicas: 2` minimum |
| Mutable image tags (:latest) | Non-reproducible deploys; hard rollback | Tag by git SHA |

---

## Cloud-Native Readiness Checklist

**Application Design**
- [ ] Stateless: no in-memory session, no local disk state
- [ ] Config via environment variables only; no hardcoded values
- [ ] SIGTERM handler: drain in-flight requests, close connections cleanly
- [ ] Startup time < 10 seconds
- [ ] Structured JSON logs to stdout

**Container**
- [ ] Multi-stage Dockerfile: build stage separate from runtime
- [ ] Distroless or minimal base image (Alpine if distroless not feasible)
- [ ] Non-root user (`USER 1001`)
- [ ] No secrets in image layers (`docker history` check)
- [ ] Image tagged by git SHA, not `:latest`

**Kubernetes**
- [ ] CPU and memory requests + limits on every container
- [ ] Liveness probe: checks if process is alive
- [ ] Readiness probe: checks if ready to serve traffic (DB + cache connected)
- [ ] `minReplicas: 2` for all production services
- [ ] PodDisruptionBudget: `minAvailable: 1`
- [ ] `readOnlyRootFilesystem: true`
- [ ] Resource quotas on namespaces

**Operations**
- [ ] GitOps: all manifests in Git; no manual kubectl apply in prod
- [ ] HPA configured for CPU or custom metric autoscaling
- [ ] Alerts on error rate, p99 latency, pod crash loops
- [ ] Rollback procedure: `kubectl rollout undo` or ArgoCD history
