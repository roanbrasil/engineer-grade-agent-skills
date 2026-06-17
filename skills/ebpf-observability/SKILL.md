---
name: ebpf-observability
description: Implement, evaluate, or troubleshoot eBPF-based observability — including Pixie, Cilium/Hubble, Pyroscope continuous profiling, bpftrace, and BCC tools.
---

# eBPF-Based Observability

eBPF (Extended Berkeley Packet Filter) lets you run sandboxed programs directly in the Linux kernel — without modifying kernel source code or loading kernel modules. For observability, this means zero-instrumentation visibility into system calls, network flows, CPU scheduling, and application internals.

```
+------------------------------------------------------------+
|  Traditional Observability vs eBPF                       |
|                                                            |
|  Traditional:                                             |
|  App Code --> SDK (OTel) --> Agent --> Backend            |
|  Requires: code change, redeploy, SDK maintenance         |
|                                                            |
|  eBPF:                                                    |
|  Linux Kernel                                             |
|  +--------------------------------------------------+     |
|  | syscalls | scheduler | network stack | filesystem |     |
|  |    ^           ^            ^              ^       |     |
|  |  [eBPF]     [eBPF]       [eBPF]         [eBPF]   |     |
|  +--------------------------------------------------+     |
|       |           |            |              |            |
|    profiles    scheduling    flows         file I/O        |
|                                                            |
|  Requires: no code change, no redeploy                   |
+------------------------------------------------------------+
```

---

## What eBPF Is

eBPF programs are loaded into the kernel at runtime and verified for safety before execution. The verifier guarantees:
- No infinite loops (bounded loop iterations only)
- No invalid memory access (bounds-checked)
- No crashing the kernel (programs are sandboxed)
- Programs terminate (proven by the verifier)

```
+------------------------------------------+
| User Space                               |
|  bpftrace / BCC / Cilium / Pixie agent   |
|  - Write eBPF program (C / BPF bytecode) |
|  - Load via bpf() syscall                |
+------------------------------------------+
              |
              v  bpf() syscall
+------------------------------------------+
| Linux Kernel                             |
|  eBPF Verifier                           |
|  - Checks safety (no loops, no OOB)      |
|  - Rejects unsafe programs               |
|             |                            |
|             v (if safe)                  |
|  JIT Compiler --> native machine code    |
|             |                            |
|             v                            |
|  Attach to hook point:                   |
|  - kprobe (any kernel function)          |
|  - tracepoint (stable kernel events)     |
|  - uprobe (user-space function)          |
|  - XDP (network packet, before kernel)   |
|  - tc (traffic control)                  |
|  - LSM (security hooks)                  |
+------------------------------------------+
              |
              v  BPF Maps (shared memory)
+------------------------------------------+
| User Space reads results                 |
| from BPF Maps (ring buffer, hash map)    |
+------------------------------------------+
```

### eBPF Hook Types

| Hook | Stability | Use Case |
|------|-----------|---------|
| Tracepoint | Stable (kernel API) | syscalls, scheduler, block I/O |
| Kprobe / kretprobe | Unstable (changes across kernel versions) | any kernel function entry/exit |
| Uprobe | Stable per binary | user-space function entry/exit |
| USDT | Stable (defined in app) | Java, Python, Node.js application events |
| XDP | Stable | high-performance packet filtering at NIC level |
| tc (traffic control) | Stable | packet shaping, filtering after NIC |

---

## bpftrace — One-Liner Tracing

`bpftrace` is an awk-like language for writing eBPF programs. Best for ad-hoc production debugging.

```bash
# Requires: Linux 4.9+, bpftrace installed
# Install
apt-get install bpftrace            # Ubuntu/Debian
dnf install bpftrace                # Fedora/RHEL

# Show all open() syscalls with process name and file
bpftrace -e '
  tracepoint:syscalls:sys_enter_openat {
    printf("%-16s %s\n", comm, str(args->filename));
  }'

# Count syscalls by process name (top-like view)
bpftrace -e '
  tracepoint:raw_syscalls:sys_enter {
    @[comm] = count();
  }
  interval:s:5 {
    print(@);
    clear(@);
  }'

# Trace all HTTP GET/POST at write() syscall level
bpftrace -e '
  tracepoint:syscalls:sys_enter_write {
    if (strncmp("GET ", str(args->buf), 4) == 0 ||
        strncmp("POST", str(args->buf), 4) == 0) {
      printf("%-16s fd=%-4d: %s\n", comm, args->fd, str(args->buf, 80));
    }
  }'

# Histogram of read() latency in microseconds
bpftrace -e '
  tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
  tracepoint:syscalls:sys_exit_read  {
    if (@start[tid]) {
      @latency_us = hist((nsecs - @start[tid]) / 1000);
      delete(@start[tid]);
    }
  }'

# Count TCP connections by destination port
bpftrace -e '
  kprobe:tcp_connect {
    $sk = (struct sock *)arg0;
    @[($sk->__sk_common.skc_dport >> 8) |
      (($sk->__sk_common.skc_dport & 0xff) << 8)] = count();
  }'

# Trace JVM GC pause time via USDT (Java with -XX:+ExtendedDTraceProbes)
bpftrace -e '
  usdt:/usr/lib/jvm/java-21/lib/server/libjvm.so:hotspot:gc__begin {
    @gc_start[tid] = nsecs;
  }
  usdt:/usr/lib/jvm/java-21/lib/server/libjvm.so:hotspot:gc__end {
    if (@gc_start[tid]) {
      @gc_pause_ms = hist((nsecs - @gc_start[tid]) / 1000000);
      delete(@gc_start[tid]);
    }
  }'
```

### Pre-built bpftrace Scripts

```bash
# List all available scripts
ls /usr/share/bpftrace/tools/

# Trace disk I/O latency
bpftrace /usr/share/bpftrace/tools/biolatency.bt

# Trace new process executions
bpftrace /usr/share/bpftrace/tools/execsnoop.bt

# Trace TCP retransmits
bpftrace /usr/share/bpftrace/tools/tcpretrans.bt

# CPU off-CPU time (thread blocking analysis)
bpftrace /usr/share/bpftrace/tools/offcputime.bt
```

---

## BCC — Python-Based eBPF Tools

BCC (BPF Compiler Collection) provides pre-built production-ready eBPF tools and a Python API for custom programs.

```bash
# Install BCC tools
apt-get install bpfcc-tools linux-headers-$(uname -r)

# Core debugging tools
execsnoop-bpfcc       # trace new process executions (find hidden child processes)
tcpretrans-bpfcc      # TCP retransmits (network reliability issues)
biolatency-bpfcc      # block I/O latency histogram
runqlat-bpfcc         # CPU run queue latency (scheduler contention)
profile-bpfcc         # CPU profiler with flame graph output
offcputime-bpfcc      # off-CPU time analysis (blocking, I/O wait)
memleak-bpfcc         # detect memory leaks in native code
tcpconnect-bpfcc      # trace TCP connection attempts
tcpaccept-bpfcc       # trace accepted TCP connections

# Examples
execsnoop-bpfcc                           # all new processes
tcpconnect-bpfcc -p $(pgrep java)         # TCP connects from Java process
biolatency-bpfcc -m                       # block I/O latency in milliseconds
profile-bpfcc -f 30 > /tmp/cpu.stacks    # 30s CPU profile for flame graph
```

### Custom BCC Program (Python)

```python
#!/usr/bin/env python3
"""Trace slow HTTP requests by hooking write() syscall."""

from bcc import BPF
import ctypes

program = """
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>

struct event_t {
    u32 pid;
    u64 latency_ns;
    char comm[TASK_COMM_LEN];
    char data[128];
};

BPF_HASH(start_time, u32, u64);
BPF_PERF_OUTPUT(events);

TRACEPOINT_PROBE(syscalls, sys_enter_write) {
    u32 tid = bpf_get_current_pid_tgid();
    u64 ts = bpf_ktime_get_ns();
    start_time.update(&tid, &ts);
    return 0;
}

TRACEPOINT_PROBE(syscalls, sys_exit_write) {
    u32 tid = bpf_get_current_pid_tgid();
    u64 *ts = start_time.lookup(&tid);
    if (!ts) return 0;

    u64 latency = bpf_ktime_get_ns() - *ts;
    start_time.delete(&tid);

    // Only report calls slower than 1ms
    if (latency < 1000000) return 0;

    struct event_t event = {};
    event.pid = bpf_get_current_pid_tgid() >> 32;
    event.latency_ns = latency;
    bpf_get_current_comm(&event.comm, sizeof(event.comm));
    events.perf_submit(args, &event, sizeof(event));
    return 0;
}
"""

b = BPF(text=program)

def handle_event(cpu, data, size):
    event = b["events"].event(data)
    print(f"pid={event.pid} comm={event.comm.decode()} "
          f"latency={event.latency_ns/1e6:.2f}ms")

b["events"].open_perf_buffer(handle_event)
print("Tracing slow write() calls... Ctrl-C to stop")
while True:
    b.perf_buffer_poll()
```

---

## Pixie — Auto-Instrumented Kubernetes Observability

Pixie uses eBPF to automatically instrument every service in your Kubernetes cluster — no code changes, no sidecars.

```
+-----------------------------------------------+
|  Kubernetes Cluster                           |
|                                               |
|  +------------------+  +------------------+  |
|  | Service A        |  | Service B        |  |
|  | (no OTel SDK)    |  | (no OTel SDK)    |  |
|  +------------------+  +------------------+  |
|        |  HTTP/gRPC            |              |
|        v                       v              |
|  +-------------------------------------+      |
|  | Pixie Vizier (DaemonSet per node)  |      |
|  | eBPF: captures all traffic         |      |
|  | Stores in-cluster (no egress)      |      |
|  +-------------------------------------+      |
|                                               |
|  Pixie UI / PxL queries                      |
|  px/http_data     -- HTTP request traces      |
|  px/mysql_data    -- MySQL query traces       |
|  px/jvm           -- JVM heap + GC metrics   |
+-----------------------------------------------+
```

### Install Pixie

```bash
# Install px CLI
bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"

# Deploy Pixie into your cluster
px deploy

# Run a PxL script
px run px/http_data               # all HTTP traffic for 30s
px run px/mysql_data              # all MySQL queries
px run px/node_memory_usage       # memory per node
px run px/jvm_stats               # JVM heap, GC, threads

# Live UI
px live px/service_stats          # per-service latency/error rate/RPS
```

### PxL Query (SQL-like)

```python
# PxL: find slowest HTTP endpoints in the last 5 minutes
import px

df = px.DataFrame(table='http_events', start_time='-5m')
df = df[df.resp_status >= 400]           # only errors
df.latency_ms = df.latency / 1e6
df = df.groupby(['service', 'req_path']).agg(
    p99_latency_ms=('latency_ms', px.quantiles(0.99)),
    error_count=('resp_status', px.count),
)
df = df.sort(by='p99_latency_ms', ascending=False)
px.display(df, 'Slow/Erroring Endpoints')
```

### What Pixie Auto-Detects (No Code Changes)

- HTTP/1.x, HTTP/2, gRPC (encrypted HTTPS via uprobes on OpenSSL)
- MySQL, PostgreSQL, Redis, Cassandra, MongoDB
- DNS queries and responses
- JVM metrics (heap, GC, class loading) via JVM USDT probes
- CPU profiles per service (flame graph)

---

## Cilium and Hubble — eBPF Networking + Observability

Cilium replaces iptables with eBPF for Kubernetes networking. Hubble is the built-in observability layer.

```
+--------------------------------------------------+
|  Cilium Architecture                            |
|                                                  |
|  Pod A ---[eBPF datapath]---> Pod B             |
|              |                                   |
|              | Flow recorded                     |
|              v                                   |
|  Hubble Agent (per node)                        |
|    - HTTP method, path, status                  |
|    - DNS query/response                         |
|    - TCP connection open/close                  |
|    - Policy verdict (allowed/denied)            |
|              |                                   |
|              v                                   |
|  Hubble Relay (cluster-wide aggregation)        |
|              |                                   |
|              v                                   |
|  Hubble UI  /  Grafana / Prometheus             |
+--------------------------------------------------+
```

### Install Cilium + Hubble

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail-with-body \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz" \
  | sudo tar xz -C /usr/local/bin

# Install Cilium (replace kube-proxy)
cilium install --version 1.15.0

# Enable Hubble
cilium hubble enable --ui

# Verify
cilium status --wait

# Port-forward Hubble UI
cilium hubble ui   # opens browser at localhost:12000
```

### Hubble CLI — Flow Inspection

```bash
# Install hubble CLI
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail-with-body \
  "https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz" \
  | sudo tar xz -C /usr/local/bin

# Port-forward Hubble relay
cilium hubble port-forward &

# Watch all flows
hubble observe

# Filter to a specific namespace
hubble observe --namespace production

# Filter HTTP flows only
hubble observe --protocol http

# Filter by source pod
hubble observe --from-pod production/frontend

# Watch for dropped packets (policy violations)
hubble observe --verdict DROPPED

# JSON output for further processing
hubble observe --output json | jq '.flow | {src: .source.pod_name, dst: .destination.pod_name, type: .l7.type}'
```

### Hubble Metrics in Grafana

```yaml
# Cilium values for Prometheus metrics
# cilium-values.yaml
hubble:
  enabled: true
  metrics:
    enabled:
      - dns:query;ignoreAAAA
      - drop
      - tcp
      - flow
      - port-distribution
      - icmp
      - http
    serviceMonitor:
      enabled: true   # requires Prometheus Operator
```

### mTLS Without Sidecars (Cilium 1.14+)

```yaml
# CiliumNetworkPolicy with mutual auth (no Istio/Envoy needed)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: require-mtls
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: order-service
      authentication:
        mode: required   # enforce mTLS between these services
```

---

## Continuous Profiling with Pyroscope

Pyroscope collects CPU and memory profiles continuously from all processes using eBPF — no code change needed.

```
+--------------------------------------------------+
|  Pyroscope Architecture                         |
|                                                  |
|  Pyroscope Agent (eBPF, DaemonSet)              |
|  - Sample stack traces every 10ms (perf_events) |
|  - Attribute to PID / container / K8s pod       |
|  - No app instrumentation needed                |
|             |                                    |
|             v                                    |
|  Pyroscope Server                               |
|  - Aggregate flame graphs per service           |
|  - Store time-series profiles                   |
|  - Label: service, pod, namespace               |
|             |                                    |
|             v                                    |
|  Grafana (Pyroscope datasource)                 |
|  - Correlate profiles with traces (Tempo)       |
|  - "Why is p99 high?" --> drill to flame graph  |
+--------------------------------------------------+
```

### Deploy Pyroscope in Kubernetes

```bash
# Add Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Pyroscope with eBPF profiling
helm install pyroscope grafana/pyroscope \
  --namespace monitoring \
  --set ebpf.enabled=true \
  --set ebpf.spy=true
```

```yaml
# pyroscope-values.yaml
pyroscope:
  extraArgs:
    - -scrape-configs.reload-period=30s

ebpf:
  enabled: true
  # Profile all processes including JVM, Python, Rust, Go
  # No per-language agent needed for CPU profiles

# Grafana integration
grafana:
  enabled: true
  sidecar:
    datasources:
      enabled: true
```

### Pyroscope SDK — Push Profiles (When You Need App Labels)

```java
// Java: push profiles with custom labels (service name, version)
// build.gradle: implementation 'io.pyroscope:agent:0.13.0'

PyroscopeAgent.start(
    new Config.Builder()
        .setApplicationName("order-service")
        .setProfilingEvent(EventType.ITIMER)    // CPU
        .setProfilingAlloc("512k")              // allocation profiling
        .setServerAddress("http://pyroscope:4040")
        .setLabels(Map.of(
            "region", System.getenv("AWS_REGION"),
            "version", System.getenv("APP_VERSION")
        ))
        .build()
);
```

```go
// Go: SDK for labeled profiles
import "github.com/grafana/pyroscope-go"

profiler, _ := pyroscope.Start(pyroscope.Config{
    ApplicationName: "order-service",
    ServerAddress:   "http://pyroscope:4040",
    ProfileTypes: []pyroscope.ProfileType{
        pyroscope.ProfileCPU,
        pyroscope.ProfileAllocObjects,
        pyroscope.ProfileAllocSpace,
        pyroscope.ProfileInuseObjects,
        pyroscope.ProfileInuseSpace,
    },
    Tags: map[string]string{
        "region":  os.Getenv("AWS_REGION"),
        "version": os.Getenv("APP_VERSION"),
    },
})
defer profiler.Stop()
```

### Correlate Profiles With Traces (Grafana)

```yaml
# grafana datasource: link Tempo traces to Pyroscope profiles
apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    url: http://tempo:3200
    jsonData:
      tracesToProfiles:
        datasourceUid: pyroscope
        tags: [{ key: "service.name", value: "service_name" }]
        profileTypeId: "process_cpu:cpu:nanoseconds:cpu:nanoseconds"

  - name: Pyroscope
    type: grafana-pyroscope-datasource
    url: http://pyroscope:4040
```

---

## eBPF Kernel Requirements and Portability

```
+------------------------------------------+
| Kernel Version Requirements              |
|                                          |
| 4.9  - kprobes, tracepoints, BPF maps   |
| 4.18 - BTF (type info for CO-RE)         |
| 5.2  - BPF_PROG_TYPE_TRACING            |
| 5.5  - BPF ring buffer                   |
| 5.8  - BPF iterator, fentry/fexit        |
| 5.13 - BPF CO-RE stable                 |
|                                          |
| Check your kernel:                       |
|   uname -r                              |
| Check BTF support:                       |
|   ls /sys/kernel/btf/vmlinux            |
+------------------------------------------+
```

### BPF CO-RE (Compile Once, Run Everywhere)

CO-RE allows a single eBPF binary to run across different kernel versions without recompilation.

```bash
# Check if BTF is available (needed for CO-RE)
ls /sys/kernel/btf/vmlinux   # exists on kernels compiled with CONFIG_DEBUG_INFO_BTF=y

# Amazon Linux 2023, Ubuntu 22.04+, RHEL 9+: BTF enabled by default
# GKE, EKS (Amazon Linux 2023 node groups): BTF supported
# EKS Bottlerocket: BTF supported

# bpftool: inspect running eBPF programs
bpftool prog list                          # list loaded eBPF programs
bpftool map list                           # list BPF maps
bpftool map dump id <map_id>              # dump map contents
bpftool net list                           # show eBPF attached to network interfaces
bpftool perf list                          # show eBPF attached to perf events
```

---

## When to Use eBPF Observability

```
Decision tree:

Can you modify the application?
  YES --> Use OTel SDK (richer business context, distributed tracing)
  NO  --> eBPF is your only option for visibility

Need kernel-level visibility?
  (syscalls, network bytes, scheduler, file I/O)
  YES --> eBPF only; OTel SDK can't see this layer
  NO  --> OTel SDK sufficient

Performance-critical path? (< 1% overhead budget)
  YES --> eBPF (< 0.5% overhead); OTel can be 1-5%
  NO  --> OTel SDK easier to maintain

Kubernetes service mesh (zero-config mTLS + L7 visibility)?
  YES --> Cilium + Hubble replaces Istio/Envoy; uses eBPF
  NO  --> N/A

Continuous CPU/memory profiling?
  YES --> Pyroscope with eBPF spy (no SDK needed)
       OR Pyroscope SDK for labeled profiles

Ad-hoc production debugging (legacy app, no source)?
  YES --> bpftrace one-liners or BCC tools
```

---

## Limitations

| Limitation | Detail | Workaround |
|-----------|--------|-----------|
| Kernel version | Many features need 5.8+ | Check `uname -r`; upgrade if needed |
| BTF required for CO-RE | Without BTF, must compile eBPF per kernel version | Use `libbpf-bootstrap` + CO-RE; modern distros have BTF |
| Container security restrictions | eBPF needs `CAP_BPF` or `CAP_SYS_ADMIN` | DaemonSet with privileged=true; restrict to observability namespace |
| Not for high-cardinality business metrics | BPF maps have size limits; no string interpolation | Use OTel SDK for business events; eBPF for infrastructure |
| Encrypted user-space traffic | HTTPS encrypted before kernel TCP layer | Uprobe on SSL library (`libssl.so`) or use Pixie's TLS interception |
| JVM JIT-compiled code | JIT-compiled methods have no stable symbols | Use async-profiler or JVM USDT probes (`-XX:+ExtendedDTraceProbes`) |

---

## Checklist

### Before Deploying eBPF Tools
- [ ] `uname -r` — kernel 5.8+ for full feature set
- [ ] `ls /sys/kernel/btf/vmlinux` — BTF available for CO-RE
- [ ] Node security group / PSA allows `CAP_BPF` for DaemonSet
- [ ] Confirm eBPF tool overhead acceptable (< 1% CPU on idle node)

### Pixie Deployment
- [ ] Pixie Vizier deployed as DaemonSet
- [ ] Test: `px run px/http_data` returns data within 30s
- [ ] Verify TLS interception works: `px run px/http_data` shows HTTPS traffic

### Cilium + Hubble
- [ ] `cilium status` shows all components healthy
- [ ] `hubble observe` returns flows
- [ ] Hubble metrics exported to Prometheus
- [ ] Grafana dashboard showing per-service HTTP error rate

### Pyroscope Profiling
- [ ] eBPF spy enabled in Helm values
- [ ] Flame graph visible in Grafana for at least one service
- [ ] Profile-to-trace correlation configured (Tempo + Pyroscope datasources linked)

### bpftrace / BCC (Ad-Hoc)
- [ ] `bpftrace --version` installed; verify version 0.16+
- [ ] Run `bpftrace -l 'tracepoint:syscalls:*'` to verify tracepoints available
- [ ] Test: `bpftrace -e 'BEGIN { printf("eBPF works\n"); exit(); }'`
