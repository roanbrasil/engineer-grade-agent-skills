---
name: debugging-production
description: Debug production incidents and hard-to-reproduce bugs — use when someone asks how to diagnose a JVM memory leak, Kubernetes crash, slow query, production exception, or needs to read thread dumps, heap dumps, or flamegraphs.
---

# Debugging Production Systems

## Debugging Mindset

### Hypothesis-Driven Debugging

```
Observation: service response time increased 10x at 14:32 UTC
    |
    v
Hypothesis 1: traffic spike caused resource exhaustion
    |
    |--> Experiment: check request rate metrics at 14:32
    |--> Result: traffic normal; hypothesis rejected
    |
    v
Hypothesis 2: a deployment at 14:30 introduced a slow code path
    |
    |--> Experiment: check deploy history; compare p99 before/after
    |--> Result: deploy at 14:28 introduced a new DB query; hypothesis confirmed
    |
    v
Action: roll back or hotfix the query
```

Resist the urge to change things randomly hoping something helps. Each change:
1. Costs time to deploy and verify
2. Can introduce new bugs
3. Obscures the root cause if it has a side effect

### Reproduce First

A bug you can reproduce consistently is 80% debugged. Approaches:

- Reproduce in a staging environment with production data (anonymized)
- Replay production traffic: capture with `tcpdump`, replay with `tcpreplay` or service-specific tools
- Feature flag: route a small % of traffic to a debug-instrumented version
- Add debug logging to production temporarily (use a log level that can be changed without restart)

If you cannot reproduce, add instrumentation to gather more data. Do not guess.

### The Debugging Checklist

Before changing anything:

- [ ] Reproduce in non-production environment first
- [ ] Check logs (application + infrastructure) around the time of failure
- [ ] Check metrics for anomalies (traffic spike? memory ramp? error rate increase?)
- [ ] Check recent deployments: is there a correlation with deploy time?
- [ ] Check distributed traces for the failing request
- [ ] Form a hypothesis before changing anything
- [ ] Document your observations (others may be investigating in parallel)

---

## JVM Production Debugging

### Thread Dumps — Identify Deadlocks and Blocked Threads

```bash
# Method 1: signal (safe for production; non-destructive)
kill -3 <pid>           # output goes to stdout/stderr of the JVM process
# or
kill -QUIT <pid>

# Method 2: jstack
jstack <pid>            # print once
jstack -l <pid>         # include lock information

# Method 3: jcmd (preferred on Java 11+)
jcmd <pid> Thread.print

# Capture multiple dumps 30s apart (to identify stuck threads vs transient waits)
for i in 1 2 3; do jstack <pid> > /tmp/thread-dump-$i.txt; sleep 30; done
```

Reading a thread dump:
```
"http-nio-8080-exec-7" #45 daemon prio=5 os_prio=0 tid=0x00007f RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
    at com.mysql.cj.protocol.FullReadInputStream.readFully(...)
    ...

"http-nio-8080-exec-12" #50 daemon prio=5 os_prio=0 tid=0x00007f BLOCKED
    waiting to lock <0x00000007b> (a com.example.service.UserCache)
    at com.example.service.UserCache.get(UserCache.java:84)
```

Patterns to look for:
- `BLOCKED` on the same lock object across many threads: contention or deadlock
- `WAITING` on `java.util.concurrent.LinkedBlockingQueue.take`: thread pool idle (normal)
- All threads `WAITING` in `socketRead`: downstream service not responding
- A thread holding a lock but not releasing it: look for the thread that owns `<0x0000...>`

Deadlock detection: jstack will print "Found one Java-level deadlock" and show the cycle.

### Heap Dumps — Identify Memory Leaks

```bash
# Trigger on OOM automatically (add to JVM startup flags)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap-dumps/

# Trigger manually without restarting
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# jmap (older; works but jcmd preferred)
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
```

Analyze with Eclipse Memory Analyzer (MAT):
1. Open `heap.hprof` in MAT
2. Run "Leak Suspects Report" — MAT identifies the largest retained object trees
3. Open "Dominator Tree" — find objects retaining the most memory
4. Use OQL to query: `SELECT * FROM com.example.service.UserSession WHERE createdAt < ...`

Common leak patterns:
- Static collections that grow without eviction (`static Map<UserId, Session>`)
- Event listeners not deregistered
- ThreadLocal variables not cleaned up in thread pools
- Cache without size bounds or TTL

### Java Flight Recorder — Low-Overhead Continuous Profiling

```bash
# Start recording (< 1% overhead; safe for production)
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr settings=profile

# Or start without time limit and dump on demand
jcmd <pid> JFR.start name=continuous settings=default
jcmd <pid> JFR.dump name=continuous filename=/tmp/recording.jfr
jcmd <pid> JFR.stop name=continuous

# Analyze with JDK Mission Control (JMC)
jmc  # open GUI and load /tmp/recording.jfr
```

JFR captures: CPU profile, memory allocation profile, GC events, lock contention, I/O waits, network activity — all with minimal overhead.

### async-profiler — CPU and Allocation Flamegraph

```bash
# Download
curl -L https://github.com/async-profiler/async-profiler/releases/latest/download/async-profiler-linux-x64.tar.gz | tar xz

# CPU flamegraph (30 second sample)
./asprof -d 30 -f /tmp/cpu-flamegraph.html <pid>

# Allocation flamegraph (find what's creating garbage)
./asprof -d 30 -e alloc -f /tmp/alloc-flamegraph.html <pid>

# Wall-clock profiling (includes I/O wait; good for latency investigations)
./asprof -d 30 -e wall -t -f /tmp/wall-flamegraph.html <pid>
```

Reading flamegraphs:
- X-axis: proportion of time spent (wider = more time)
- Y-axis: call stack (bottom = main; top = leaf function)
- Look for wide bars near the top: functions where most time is spent
- Click to zoom; use search to find a specific class or method

### GC Analysis

JVM startup flags for GC logging:
```bash
-Xlog:gc*:file=/tmp/gc.log:time,uptime:filecount=5,filesize=10m
```

Key GC events to look for:
```
[info][gc] GC(42) Pause Young (Allocation Failure) ... 250ms  <-- high pause; memory pressure
[info][gc] GC(43) Pause Full (Heap Dump Initiated) ...       <-- OOM imminent
[info][gc] GC(44) To-space exhausted                         <-- G1: region allocation failure
```

Upload `gc.log` to GCEasy (https://gceasy.io) for visualization of GC activity, throughput, and pause times.

Common GC problems and causes:

```
Long GC pauses (>500ms)     --> heap too small; increase -Xmx; or memory leak
High GC frequency           --> too many short-lived objects; check allocation flamegraph
To-space exhausted (G1)     --> concurrent marking not keeping up; increase heap or GC threads
Humongous allocation        --> single object > region size; allocating large arrays in a loop
```

### Arthas — Hot Diagnostics Without Restart

```bash
# Download and attach to running JVM
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar <pid>

# Inside Arthas REPL:

# Watch method arguments and return value in real time
watch com.example.service.OrderService createOrder '{params, returnObj}' -x 2

# Trace method call tree with timing (find which sub-call is slow)
trace com.example.service.OrderService createOrder '#cost > 100'  # only > 100ms

# Print current class bytecode (verify hot-swap worked)
jad com.example.service.OrderService

# Hot-swap a class without restart (load compiled .class file)
redefine /tmp/OrderService.class

# Show class loading info
classloader -a | grep OrderService

# Thread analysis
thread -n 5  # top 5 CPU-consuming threads
thread -b    # detect deadlocks
```

---

## Python Production Debugging

### Thread and CPU Profiling

```bash
# py-spy: sampling profiler; attaches to running process; minimal overhead
pip install py-spy

# Print current stack traces of all threads (like jstack)
py-spy dump --pid <pid>

# CPU flamegraph (30 second sample)
py-spy record --pid <pid> --output /tmp/profile.svg --duration 30

# Live top-like view
py-spy top --pid <pid>
```

### faulthandler — Crash Diagnostics

```python
# Add to application startup (before anything else)
import faulthandler
import signal

faulthandler.enable()  # print Python traceback on SIGSEGV / SIGFPE / SIGABRT

# Register signal to dump traceback on demand (without killing the process)
faulthandler.register(signal.SIGUSR2, all_threads=True, chain=False)
```

```bash
# Trigger traceback dump from outside the process
kill -USR2 <pid>
# Output appears in stderr of the Python process
```

### manhole — Interactive REPL in Running Process

```python
# Add to application startup
import manhole
manhole.install()  # creates /tmp/manhole-<pid> Unix socket
```

```bash
# Connect to running process
manhole-cli <pid>
# Now you have a Python REPL inside the live process:
>>> import gc; gc.get_referrers(some_object)
>>> import tracemalloc; tracemalloc.start(); # start memory tracing
```

### pdb Post-Mortem in Development

```python
try:
    process_order(order)
except Exception:
    import pdb
    pdb.post_mortem()  # opens interactive debugger at the point of exception
```

### Memory Profiling

```bash
pip install memray

# Profile a running process
memray attach <pid>
memray flamegraph output.bin

# Profile a script from start
memray run --output output.bin python myscript.py
memray flamegraph output.bin
```

---

## Kubernetes Debugging

### Pod Diagnostics

```bash
# Pod status, resource usage, and event log
kubectl describe pod <pod-name> -n <namespace>

# Key fields to check in describe output:
# - Conditions: Ready/NotReady, Initialized, PodScheduled
# - Reason: OOMKilled, CrashLoopBackOff, Evicted
# - Events: image pull failures, liveness probe failures, resource pressure

# Logs from the current container
kubectl logs <pod-name> -n <namespace>

# Logs from the PREVIOUS (crashed) container
kubectl logs <pod-name> -n <namespace> --previous

# Stream logs in real time
kubectl logs -f <pod-name> -n <namespace>

# Logs from all containers in a pod
kubectl logs <pod-name> -n <namespace> --all-containers=true
```

### Interactive Debugging

```bash
# Shell into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run a command in a container
kubectl exec <pod-name> -n <namespace> -- curl -s localhost:8080/actuator/health

# Copy files out of a container (e.g., heap dumps, thread dumps)
kubectl cp <namespace>/<pod-name>:/tmp/heap.hprof ./heap.hprof

# Port-forward to access an internal service locally
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
# Then: curl localhost:8080/api/health
```

### Ephemeral Debug Containers

```bash
# Inject a debug container into a running pod
# (useful when the main container doesn't have debugging tools)
kubectl debug -it <pod-name> \
  --image=nicolaka/netshoot \  # container with network tools (curl, dig, tcpdump, netstat)
  --target=<container-name> \
  -n <namespace>

# Inside the debug container:
curl localhost:8080/health       # test the app
netstat -tlnp                    # check open ports
tcpdump -i eth0 port 8080       # capture traffic
dig my-service.my-namespace.svc.cluster.local  # DNS resolution check
```

### Cluster-Wide Event Timeline

```bash
# See all events sorted by time (find what happened when)
kubectl get events --sort-by='.lastTimestamp' -A

# Filter by namespace and type
kubectl get events -n <namespace> --field-selector type=Warning

# Watch events in real time
kubectl get events -n <namespace> -w
```

### Resource Pressure Debugging

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -n <namespace> --sort-by=memory

# Check resource limits and requests
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'

# If a pod is OOMKilled:
kubectl describe pod <pod-name> | grep -A5 "OOMKilled\|Last State"
# Increase memory limit or fix the memory leak
```

### CrashLoopBackOff Diagnosis

```
CrashLoopBackOff means: the container started, crashed, and Kubernetes is backing off before restarting it.

Diagnosis sequence:
1. kubectl logs <pod> --previous          --> what was the last error?
2. kubectl describe pod <pod>             --> what exit code? (OOMKilled=137, segfault=139, app error=1)
3. Check liveness probe config            --> probe too aggressive? not enough startup time?
4. Check environment variables / secrets  --> missing config causes immediate crash
5. Check init containers                  --> did init container fail?
```

---

## Distributed Tracing for Debugging

### Finding the Slow Span

```
Request: POST /api/orders
Total duration: 3.2s  <-- 10x normal

Trace breakdown:
  [OrderController.createOrder]          50ms
  [OrderService.validate]                20ms
  [InventoryService HTTP call]           2800ms  <-- SLOW SPAN
    [HTTP GET /inventory/check]          2800ms
      [InventoryController.check]        50ms
      [InventoryRepository.findByIds]    2700ms  <-- root cause: N+1 query
```

Action: examine the `InventoryRepository.findByIds` span — it has 47 child spans, each a separate DB query. Fix: use a batch query.

### Finding the Error Span

```
Request: POST /api/checkout
Result: 500 Internal Server Error

Trace:
  [CheckoutController] OK
  [CheckoutService]    OK
  [PaymentService HTTP call]  ERROR
    status: 422
    error: "Card declined: insufficient funds"
    <-- This is the root cause; all upstream services returned 500 as a result
```

Without tracing, you would see a 500 from CheckoutController and have to search logs across 3 services.

### Tail-Based Sampling

Problem: sampling at the start of a request means you may not capture the slow or failing requests you care about most.

Tail-based sampling: collect all spans, decide to keep or drop the trace AFTER it completes based on:
- Final status (keep all errors)
- Duration (keep all p99+ latency)
- Random sampling for the rest

OpenTelemetry Collector configuration:
```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 1000}
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: {sampling_percentage: 5}
```

---

## Network Debugging

### Capture Traffic

```bash
# Capture HTTP traffic to/from port 8080
tcpdump -i any -w /tmp/capture.pcap port 8080

# Capture and display in terminal (no file)
tcpdump -i any -A port 8080 | grep -A5 'POST\|GET\|HTTP'

# Inside Kubernetes pod (if tcpdump is available)
kubectl exec -it <pod> -- tcpdump -i eth0 -w /tmp/cap.pcap port 8080
kubectl cp <namespace>/<pod>:/tmp/cap.pcap ./cap.pcap
# Open cap.pcap in Wireshark
```

### HTTP Diagnostics

```bash
# Verbose HTTP including headers and timing
curl -v https://api.example.com/health

# Detailed timing breakdown
curl -w "\n\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nWait: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://api.example.com/health

# Follow redirects, show each hop
curl -L -v https://api.example.com/orders/123

# Test with specific TLS version (debug TLS handshake failures)
curl --tlsv1.2 -v https://api.example.com/health
```

### Network Connectivity Checks

```bash
# Check open ports and what's listening
ss -tlnp                          # Linux (preferred over netstat)
netstat -tlnp                     # older systems

# Test TCP connectivity
nc -zv database.internal 5432    # can I reach this host/port?

# DNS resolution
dig api.example.com               # full DNS resolution chain
dig @8.8.8.8 api.example.com      # query specific DNS server
nslookup api.example.com          # simpler alternative

# DNS in Kubernetes
# Check cluster DNS resolution from inside a pod:
kubectl exec -it <pod> -- nslookup my-service.my-namespace.svc.cluster.local
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

---

## Database Debugging

### PostgreSQL

```sql
-- Find long-running queries (running > 1 minute)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND (now() - pg_stat_activity.query_start) > interval '1 minute'
ORDER BY duration DESC;

-- Find blocking queries (lock chains)
SELECT
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocked.wait_event_type,
    blocked.wait_event
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Kill a specific blocking query (use carefully in production)
SELECT pg_terminate_backend(<blocking_pid>);

-- Explain a slow query with actual execution stats
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'PENDING';
-- Look for: Seq Scan on large tables, high actual rows vs estimated rows, buffer misses
```

Key fields in EXPLAIN ANALYZE output:
```
Seq Scan on orders  (cost=0.00..89432.00 rows=1 width=200)
                    (actual time=0.042..2341.2 rows=1 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 2341198        <-- scanned 2M rows to find 1: missing index
Buffers: shared hit=0 read=14422        <-- 0 cache hits: cold data or huge table
```

Auto-explain for slow queries in production:
```sql
-- Enable in postgresql.conf:
-- shared_preload_libraries = 'auto_explain'
-- auto_explain.log_min_duration = '1s'   -- log plans for queries > 1s
-- auto_explain.log_analyze = true
-- auto_explain.log_buffers = true
```

### MySQL / MariaDB

```sql
-- Show currently running queries
SHOW FULL PROCESSLIST;

-- Kill a blocking query
KILL QUERY <thread_id>;

-- Find table locks
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- Slow query log (enable in my.cnf)
-- slow_query_log = 1
-- slow_query_log_file = /var/log/mysql/slow.log
-- long_query_time = 1

-- Explain a slow query
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 42;
```

### Connection Pool Exhaustion

Symptom: requests time out; `Unable to acquire connection within timeout`.

```bash
# Java / HikariCP: check pool metrics
# Add to application.properties:
management.endpoint.metrics.enabled=true
# Then: GET /actuator/metrics/hikaricp.connections.active
# GET /actuator/metrics/hikaricp.connections.pending

# PostgreSQL: see all open connections by application
SELECT application_name, count(*) FROM pg_stat_activity GROUP BY 1 ORDER BY 2 DESC;

# Check max_connections setting
SHOW max_connections;

# If connection pool is exhausted: thread dump may show threads waiting for DB connection
# Look for: "waiting for connection" in thread dump
```

---

## Common Production Incident Patterns

### Pattern: Sudden Latency Spike

```
Investigation sequence:
1. Check error rate (errors? or just slow?)
2. Check traffic volume (spike? load shedding needed?)
3. Check downstream services latency (trace shows which span is slow)
4. Check DB: slow queries? lock contention?
5. Check GC: long GC pauses eating into request time?
6. Check deploy history: any deployment 5-30 minutes before the spike?
7. Check CPU/memory: resource saturation?
```

### Pattern: Memory Leak (Heap Growing Until OOM)

```
Symptoms: memory usage ramps over hours/days; eventually OOM crash

Investigation:
1. Check GC logs: is GC running frequently but unable to free memory?
2. Thread dump: any threads holding references to large objects?
3. Heap dump (when heap is near peak): open in MAT, run Leak Suspects
4. Check dominator tree: which object type is retaining the most memory?
5. Common culprits: unbounded caches, static collections, event listeners,
   session objects not expiring, ThreadLocal in thread pools
```

### Pattern: Cascading Failure

```
Symptoms: multiple services failing; alert storm

Investigation:
1. Identify the "first" failure: check trace/log timestamps across services
2. The first service to fail is the root cause; others are victims
3. Usually: a core dependency (DB, cache, auth service) fails
4. Downstream services retry → amplify load → worsen the failure

Mitigation:
- Circuit breaker opens → stops cascading retries
- Timeout on all downstream calls → fail fast instead of holding threads
- Bulkhead pattern → one failing dependency doesn't exhaust all threads
```

### Pattern: CrashLoopBackOff in Kubernetes

```
Exit code meanings:
  0     --> application exited normally (should not restart unless job)
  1     --> application error (check logs)
  137   --> OOM killed (SIGKILL; memory limit exceeded)
  139   --> segfault (SIGSEGV; JVM crash or native code bug)
  143   --> graceful shutdown (SIGTERM)

Action for exit code 137:
  1. Increase memory limit temporarily to restore service
  2. Take heap dump on next restart (before OOM):
     kubectl exec -it <pod> -- jcmd <pid> GC.heap_dump /tmp/heap.hprof
  3. kubectl cp to pull the dump
  4. Analyze with MAT

Action for exit code 1:
  kubectl logs <pod> --previous | tail -100
  (find the exception that caused the crash)
```

---

## Observability Infrastructure for Effective Debugging

Without the right instrumentation, production debugging is guesswork. Ensure:

**Structured Logging**
```json
{
  "timestamp": "2024-01-15T14:32:01.234Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123def456",
  "span_id": "789ghi",
  "user_id": "user-42",
  "order_id": "order-789",
  "message": "Payment processing failed",
  "error": "Card declined: insufficient funds",
  "duration_ms": 234
}
```

**Metrics (RED Method)**
- Request Rate: how many requests per second?
- Error Rate: what % are failing?
- Duration: what is the latency distribution (p50, p95, p99)?

**Distributed Tracing**
- Every external request should have a trace ID
- Propagate trace ID via HTTP headers (`traceparent` W3C standard)
- Sample at least 5% of traffic; 100% of errors

**Health Endpoints**
```
GET /actuator/health          --> liveness: is the process alive?
GET /actuator/health/readiness --> readiness: can it serve traffic?
GET /actuator/metrics         --> current metric values
GET /actuator/info            --> build version, git commit
```
