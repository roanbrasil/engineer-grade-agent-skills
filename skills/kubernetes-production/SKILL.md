---
name: kubernetes-production
description: Expert Kubernetes production operations — invoke when designing workloads, configuring autoscaling, setting up availability guarantees, hardening security posture, or diagnosing production issues beyond basic kubectl usage.
---

# Kubernetes Production Operations

## Workload Design

### Container Design Principles

**One process per container.** Not one app per container — one *process*. The container's PID 1 is the process; when it exits, the container exits. Running multiple processes requires a supervisor (s6, supervisord), which adds complexity and hides failures.

**Init containers** for setup that must complete before app starts:
- Database migration (run once, must succeed before app starts)
- Config file rendering from secrets
- Waiting for dependent services (`until nc -z db 5432; do sleep 1; done`)

Init containers run sequentially; all must succeed before app containers start. They share volumes with app containers.

**Sidecar pattern** for cross-cutting concerns:
- Log shipping (Fluentbit reading shared `/var/log` volume)
- Proxy (Envoy/Linkerd injected by service mesh)
- Secret rotation agent
- Metrics exporter for apps that don't natively expose Prometheus metrics

### Choosing the Right Workload Controller

| Controller | Use When |
|---|---|
| **Deployment** | Stateless services; rolling updates; scale horizontally |
| **StatefulSet** | Need stable network identity (`pod-0`, `pod-1`); need persistent storage that follows pod; ordered startup/shutdown |
| **DaemonSet** | One pod per node; node-level agents (log shippers, monitoring, CNI plugins, CSI drivers) |
| **Job** | Run to completion; batch processing; one-time tasks |
| **CronJob** | Scheduled Jobs; reports, cleanup, data export |

StatefulSet pods get DNS names: `pod-0.service.namespace.svc.cluster.local`. Use headless services (`clusterIP: None`) with StatefulSets.

### Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: "250m"      # 0.25 CPU cores; used for SCHEDULING
    memory: "256Mi"  # used for SCHEDULING
  limits:
    cpu: "1000m"     # enforcement: CPU throttling when exceeded
    memory: "512Mi"  # enforcement: OOMKilled when exceeded
```

**Requests = scheduling.** The scheduler places pods on nodes with enough *allocatable* capacity to satisfy requests. A node with 4 CPUs allocatable can run 16 pods requesting 250m each.

**Limits = enforcement.** The Linux kernel enforces these via cgroups:
- CPU limit: the container's CPU usage is **throttled** (not killed) when it hits the limit. The process keeps running but gets fewer CPU cycles. This can cause latency spikes even when the node has idle CPUs.
- Memory limit: the container is **OOMKilled** (SIGKILL) when it exceeds the limit.

**CPU limit caveat:** CPU throttling with limits causes latency spikes in latency-sensitive services even when the node has spare capacity. Many production teams omit CPU limits and rely only on requests. This requires strong namespace-level LimitRanges and careful capacity planning. Never omit memory limits.

**Rules:**
- Always set memory requests and limits
- Memory requests ≈ memory limits for critical services (avoids eviction under memory pressure)
- CPU requests always set; CPU limits optional for latency-sensitive workloads
- Never leave requests unset (BestEffort pods are first to be evicted)

### QoS Classes

Kubernetes assigns a QoS class based on resource configuration. Determines eviction order under node memory pressure.

| QoS Class | Condition | Eviction Priority |
|---|---|---|
| **Guaranteed** | requests == limits for all containers | Last evicted |
| **Burstable** | requests < limits for at least one container | Middle |
| **BestEffort** | No requests or limits set | First evicted |

Production critical services should be **Guaranteed**. Batch jobs can be **Burstable** or **BestEffort**.

---

## Autoscaling

### HPA (Horizontal Pod Autoscaler)

Scales pod count based on metrics. Default: CPU/memory utilization. Custom: Prometheus metrics, external metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # scale up when avg CPU > 60%
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60  # scale down at most 10% per minute
    scaleUp:
      stabilizationWindowSeconds: 0    # scale up immediately
```

**Anti-pattern:** Setting CPU target too high (>80%). Scaling is not instantaneous; if you only trigger at 80%, by the time new pods are ready you're at 100%.

### VPA (Vertical Pod Autoscaler)

Adjusts resource requests/limits based on observed usage. Three modes:
- `Off`: Only recommendation, no action
- `Initial`: Set requests only on pod creation
- `Auto`: Evict and recreate pods with updated resources (causes restarts!)

**Do not use VPA and HPA on CPU/memory simultaneously.** They fight each other: HPA scales out because CPU is high; VPA increases CPU limit; HPA scales back in; VPA decreases limit; repeat. Use VPA for non-scalable workloads (single-instance stateful apps) or combine VPA (on memory only) with HPA (on custom metrics).

### KEDA (Kubernetes Event-Driven Autoscaler)

Scale based on external event sources. Supports scale to zero.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 0      # scale to zero when idle
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: my-topic
      lagThreshold: "10"    # 1 replica per 10 messages of lag
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/...
      queueLength: "5"
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: pending_jobs
      query: sum(pending_jobs_total)
      threshold: "100"
```

Scale to zero eliminates cost for batch workloads. KEDA wakes the deployment when queue depth > 0.

### Cluster Autoscaler

Adds nodes when pods are pending due to insufficient resources. Removes underutilized nodes.

**Tuning:**
- `--scale-down-utilization-threshold=0.5`: Remove node if all pods use <50% requested
- `--scale-down-unneeded-time=10m`: Node must be unneeded for 10 minutes before removal
- Node groups: define multiple node groups for different instance types; use node affinity/taints to route workloads

**Anti-pattern:** Pods with no requests — Cluster Autoscaler cannot schedule them on new nodes accurately; may never add nodes because it thinks existing nodes have capacity.

---

## Availability

### PodDisruptionBudget (PDB)

Prevents too many pods being unavailable during voluntary disruptions (node drains, rolling upgrades, Cluster Autoscaler scale-down).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
spec:
  minAvailable: 2         # always keep at least 2 pods
  # OR:
  # maxUnavailable: 1     # allow at most 1 pod unavailable
  selector:
    matchLabels:
      app: api-server
```

**Critical:** PDBs only protect against *voluntary* disruptions. Node hardware failure is involuntary — PDBs don't apply.

**Anti-pattern:** `minAvailable: 100%` or `maxUnavailable: 0` with only 1 replica — blocks all node drains. Always run at least 2 replicas for production services.

### Pod Anti-Affinity

Spread replicas across nodes and availability zones:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:   # hard requirement
    - labelSelector:
        matchLabels:
          app: api-server
      topologyKey: kubernetes.io/hostname              # different nodes
    preferredDuringSchedulingIgnoredDuringExecution:  # soft preference
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: api-server
        topologyKey: topology.kubernetes.io/zone      # different AZs
```

`required` blocks scheduling if constraint cannot be met. Use `preferred` for AZ spreading unless you can guarantee enough AZs have nodes.

### Topology Spread Constraints

More flexible than anti-affinity for even distribution:

```yaml
topologySpreadConstraints:
- maxSkew: 1                                    # max difference between zones
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule             # or ScheduleAnyway
  labelSelector:
    matchLabels:
      app: api-server
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: api-server
```

### Probes

```yaml
startupProbe:             # gates liveness/readiness until app starts
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30    # 30 * 10s = 5 minutes to start
  periodSeconds: 10

readinessProbe:           # removes pod from Service endpoints when failing
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3     # 3 failures → not ready

livenessProbe:            # restarts container when failing
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Readiness vs Liveness:**
- **Readiness failing** → pod removed from Service load balancing; traffic stops going to it; pod is NOT restarted
- **Liveness failing** → pod is restarted (SIGTERM + SIGKILL after terminationGracePeriodSeconds)

**Anti-patterns:**
- Liveness probe that checks external dependencies (DB, downstream services) — cascade failure; one flaky DB causes all pods to restart in a loop
- Liveness = readiness probe — use readiness for dependency checks, liveness for app deadlock detection only
- No startup probe with long init time — liveness probe fires before app finishes starting, causing restart loop
- Liveness `initialDelaySeconds` too short — same as above

**Deployment stability fields:**
```yaml
spec:
  minReadySeconds: 10          # pod must be ready for 10s before considered "available"
  progressDeadlineSeconds: 600 # rollout fails if not progressed in 10 minutes
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0        # never reduce available pods during rollout
```

---

## Networking

### Service Types

```
ClusterIP (default): Internal-only VIP; DNS: <name>.<ns>.svc.cluster.local
NodePort: Exposes on each node's IP:port (30000-32767); for dev/testing
LoadBalancer: Provisions cloud LB; production ingress for L4 traffic
ExternalName: DNS CNAME alias; for migrating to/from external services
```

**Headless service** (`clusterIP: None`): No VIP; DNS returns individual pod IPs. Use with StatefulSets so clients can connect to specific pods.

### Ingress

Routes external HTTP/HTTPS to Services. Controller-specific (nginx, traefik, ALB):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
```

### NetworkPolicy

Default: all pods can communicate with all pods. NetworkPolicy restricts this.

**Always start with default-deny in every namespace, then explicitly allow:**

```yaml
# 1. Default deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 2. Allow api-server to receive traffic from ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - port: 8080

---
# 3. Allow api-server egress to database and DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
  - to:                 # DNS
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

**Anti-pattern:** No NetworkPolicy → any compromised pod can reach any other pod or sensitive internal service.

---

## Storage

```yaml
# StorageClass for dynamic provisioning
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Retain       # Retain | Delete; use Retain for production data
volumeBindingMode: WaitForFirstConsumer  # delays PV creation until pod scheduled (AZ aware)
allowVolumeExpansion: true

---
# StatefulSet with PVC templates
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]    # single node read/write
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

**Access modes:**
- `ReadWriteOnce` (RWO): One node can mount read/write. EBS, GCE PD.
- `ReadWriteMany` (RWX): Multiple nodes can mount read/write. NFS, EFS, Ceph.
- `ReadOnlyMany` (ROX): Multiple nodes read-only. Config files, static assets.

**`reclaimPolicy: Retain`:** When PVC is deleted, the PV and underlying storage are kept. Required for production databases. `Delete` removes the storage — dangerous.

---

## Security

### RBAC

```yaml
# Role: namespace-scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]    # never use ["*"]

---
# ServiceAccount: one per app, not "default"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server
  namespace: production
automountServiceAccountToken: false   # opt-in; mount only if needed

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-server-pod-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: api-server
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**RBAC anti-patterns:**
- `cluster-admin` for application service accounts
- Wildcard verbs (`["*"]`) or resources (`["*"]`)
- Using `default` ServiceAccount for apps
- ClusterRoleBinding when RoleBinding (namespace-scoped) is sufficient

### Pod Security Context

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true     # forces explicit volume mounts for writable paths
      capabilities:
        drop:
        - ALL                           # drop all Linux capabilities
        add:                            # add only what's needed
        - NET_BIND_SERVICE              # only if binding port < 1024
    volumeMounts:
    - name: tmp
      mountPath: /tmp                  # writable tmp since root fs is read-only
  volumes:
  - name: tmp
    emptyDir: {}
```

**Pod Security Standards** (enforced at namespace level):
- `Privileged`: No restrictions; for system namespaces only
- `Baseline`: Prevents most privilege escalations; reasonable default
- `Restricted`: Enforces all hardening; apply to all production workloads

```bash
kubectl label namespace production pod-security.kubernetes.io/enforce=restricted
kubectl label namespace production pod-security.kubernetes.io/warn=restricted
kubectl label namespace production pod-security.kubernetes.io/audit=restricted
```

### Secrets Management

**Never store secrets in:**
- Environment variables (visible in `kubectl describe pod`, logs, crash dumps)
- ConfigMaps
- Git repositories
- Container images

**Production approach:** External Secrets Operator + Vault/AWS Secrets Manager/GCP Secret Manager:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials          # creates this Kubernetes Secret
  data:
  - secretKey: password
    remoteRef:
      key: production/db
      property: password
```

The Kubernetes Secret (created by ESO) is mounted as a volume or env var. Rotation happens automatically based on `refreshInterval`.

---

## Observability

### Metrics

```yaml
# ServiceMonitor (Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server
spec:
  selector:
    matchLabels:
      app: api-server
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

Essential dashboards:
- `kube_pod_container_resource_requests` vs `container_cpu_usage_seconds_total` — resource utilization
- `kube_pod_status_phase` — pod lifecycle issues
- `container_oom_events_total` — memory problems
- `kube_deployment_status_replicas_unavailable` — availability

### Logging Best Practices

- Write structured JSON to stdout/stderr only — never to files on disk
- Include: `timestamp`, `level`, `service`, `trace_id`, `span_id`, `request_id`
- Fluentbit DaemonSet reads from node log files, enriches with pod metadata, ships to destination
- Never log PII or secrets; scrub before logging

### Essential kubectl Commands for Incident Response

```bash
# What's happening right now
kubectl get events --sort-by='.lastTimestamp' -n production
kubectl get pods -n production --field-selector=status.phase!=Running

# Why did that pod die
kubectl describe pod <pod> -n production   # Events section is gold
kubectl logs <pod> -n production --previous  # previous container's logs

# Resource pressure
kubectl top nodes
kubectl top pods -n production --sort-by=memory

# What would happen if I delete this node
kubectl drain <node> --ignore-daemonsets --dry-run=client

# NetworkPolicy debugging
kubectl exec -it <debug-pod> -- nc -zv <target-svc> <port>

# RBAC debugging
kubectl auth can-i get pods --as=system:serviceaccount:production:api-server -n production
```

---

## Production Checklist

Every production workload must have:

### Reliability
- [ ] `resources.requests` and `resources.limits` set on every container
- [ ] `readinessProbe` configured (appropriate path, thresholds)
- [ ] `livenessProbe` configured (does NOT check external dependencies)
- [ ] `startupProbe` configured if app takes > 30s to start
- [ ] `minReadySeconds >= 10` on Deployments
- [ ] `PodDisruptionBudget` created for `minAvailable >= 1` (or maxUnavailable)
- [ ] Replicas >= 2 for any service that needs to survive node drain
- [ ] Pod anti-affinity: `requiredDuringScheduling` on hostname, `preferred` on zone
- [ ] HPA configured with appropriate target utilization (50-70%)

### Security
- [ ] Dedicated ServiceAccount (not `default`); `automountServiceAccountToken: false` if token not needed
- [ ] RBAC: Role/RoleBinding with minimal permissions; no wildcards
- [ ] `securityContext.runAsNonRoot: true`
- [ ] `securityContext.allowPrivilegeEscalation: false`
- [ ] `securityContext.readOnlyRootFilesystem: true`
- [ ] `capabilities.drop: [ALL]`
- [ ] Secrets from External Secrets Operator, not hardcoded
- [ ] NetworkPolicy: default-deny in namespace + explicit allow rules

### Observability
- [ ] Structured JSON logging to stdout
- [ ] `/metrics` endpoint with ServiceMonitor
- [ ] Alerts on: error rate, latency p99, pod crash loops, memory near limit
- [ ] Tracing instrumentation with trace/span IDs in logs

### Operations
- [ ] `terminationGracePeriodSeconds` tuned to match app shutdown time
- [ ] SIGTERM handler: app drains in-flight requests before exiting
- [ ] `preStop` hook with sleep if using a service mesh (gives time for endpoint removal)
- [ ] Image tag: never `:latest`; use digest or immutable tag

```yaml
# Graceful shutdown pattern
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]   # allow endpoint removal before traffic stops
```
