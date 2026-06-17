---
name: performance-profiling
description: Invoked when the user needs to profile, benchmark, or optimize performance of a Java/JVM, Python, or Rust application — including CPU, memory, I/O, GC, or async bottlenecks.
---

# Performance Profiling & Optimization

## The Prime Directive

> Measure first. Optimize second. Never optimize without data.

Premature optimization is the root of much wasted effort. Profiling tells you *where* time is actually spent — which is almost never where you think it is.

```
Profile -> Identify Hot Path -> Understand Why -> Fix -> Measure Again
   ^                                                           |
   |___________________________________________________________|
```

---

## Latency vs Throughput Tradeoff

These goals often conflict. Optimizing one can hurt the other.

```
High Throughput <-------------------------> Low Latency
     (batch, buffer, amortize)         (process immediately, no waiting)

Example: batching DB writes improves throughput but increases per-item latency.
Example: connection pools improve throughput but add queuing latency under saturation.
```

Define your goal before profiling:
- **Latency-sensitive**: user-facing APIs, interactive systems — minimize P99
- **Throughput-sensitive**: batch pipelines, ETL, data processing — maximize RPS/records-per-second

---

## Profiling Types

```
+------------------+------------------------------------------+-------------------+
| Type             | What it measures                         | Tool examples     |
+------------------+------------------------------------------+-------------------+
| CPU profiling    | Where cycles are spent                   | async-profiler    |
| Memory profiling | Allocations, leaks, heap shape           | MAT, memray       |
| I/O profiling    | Disk reads/writes, syscall overhead      | perf, strace      |
| Async profiling  | Blocked coroutines, event loop lag       | aiomonitor, JFR   |
+------------------+------------------------------------------+-------------------+
```

---

## Java / JVM Profiling

### async-profiler (preferred for CPU + allocation)

async-profiler avoids **safepoint bias** — the flaw where traditional JVMTI profilers only sample at JVM-safe points, missing hot native and JIT-compiled code.

```bash
# Download and attach to running JVM
./profiler.sh -d 30 -f profile.html <pid>

# CPU profiling (default)
./profiler.sh start -e cpu <pid>
./profiler.sh stop -f flamegraph.html <pid>

# Allocation profiling
./profiler.sh -d 30 -e alloc -f alloc.html <pid>

# Wall-clock profiling (includes blocking time — good for I/O-heavy apps)
./profiler.sh -d 30 -e wall -f wall.html <pid>
```

### JFR — Java Flight Recorder

Low overhead (~1-2%), always-on, production-safe.

```bash
# Start JFR recording
jcmd <pid> JFR.start duration=60s filename=recording.jfr settings=profile

# Dump snapshot without stopping
jcmd <pid> JFR.dump filename=snapshot.jfr

# Analyze with JDK Mission Control (GUI)
jmc recording.jfr
```

Programmatic JFR in tests:
```java
Configuration config = Configuration.getConfiguration("profile");
try (Recording rec = new Recording(config)) {
    rec.start();
    // ... code under test
    rec.stop();
    rec.dump(Path.of("test-recording.jfr"));
}
```

### Reading Flame Graphs

```
  main()
  ├── processOrder()  [WIDE — hot path]
  │   ├── validateItem()
  │   │   └── regex.match()  [WIDE + at top = CPU consumer]
  │   └── persist()
  │       └── serialize()
  └── sendEmail()  [narrow — not hot]

Width  = % of total time spent in that function (including callees)
Height = call stack depth
Top    = where CPU is actually burning
```

Rules:
- Wide frames near the top = where to optimize
- Tall thin towers = deep recursion, probably not the bottleneck
- Plateaus = functions that call nothing below them (actual CPU consumers)

### Heap Analysis

```bash
# Capture heap dump
jmap -dump:format=b,file=heap.hprof <pid>
# Or: kill -3 <pid> if HeapDumpOnOutOfMemoryError is set

# Eclipse MAT — open heap.hprof
# Look for: Leak Suspects report, Dominator Tree, OQL queries
```

OQL example to find all instances of a class:
```sql
SELECT * FROM com.example.MyClass
```

Signs of memory leaks:
- Growing old-gen that never shrinks
- Large number of duplicate strings (`String.intern()` abuse)
- Listeners/callbacks registered but never removed
- Static collections that grow unbounded

### GC Tuning

Enable GC logging (JDK 11+):
```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20m
```

Key log patterns:
```
[gc] GC(42) Pause Young (Normal) 512M->256M(1024M) 12.345ms   <- healthy
[gc] GC(99) Pause Full (Ergonomics) 900M->850M(1024M) 3421ms  <- PROBLEM: Full GC
```

G1GC tuning:
```bash
# Set target pause time (default 200ms)
-XX:MaxGCPauseMillis=100

# Set region size (1-32MB, power of 2); tune based on heap/object sizes
-XX:G1HeapRegionSize=16m

# Trigger mixed GC sooner (default 45%)
-XX:InitiatingHeapOccupancyPercent=35
```

Indicators of GC pressure:
- `Allocation Failure` pauses = allocating faster than GC can collect
- Frequent Full GC = old gen fills up; check for long-lived objects or leaks
- High `G1 Humongous Allocation` = objects > 50% of region size; increase region size

---

## Python Profiling

### cProfile + pstats (built-in, deterministic)

```python
import cProfile
import pstats
import io

pr = cProfile.Profile()
pr.enable()
# --- code under profile ---
my_function()
# --------------------------
pr.disable()

s = io.StringIO()
ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
ps.print_stats(20)  # top 20 functions
print(s.getvalue())
```

From CLI:
```bash
python -m cProfile -s cumulative -o profile.prof my_script.py
python -c "import pstats; pstats.Stats('profile.prof').sort_stats('cumulative').print_stats(20)"
```

### py-spy (sampling, zero-overhead, production-safe)

No code changes required. Attaches to running process.

```bash
# Install
pip install py-spy

# Live top-like view
py-spy top --pid <pid>

# Record flame graph
py-spy record -o profile.svg --pid <pid> --duration 30

# Profile a new process
py-spy record -o profile.svg -- python my_script.py

# Include native (C extension) frames
py-spy record --native -o profile.svg --pid <pid>
```

### memray (memory profiling)

```bash
pip install memray

# Run under memray
python -m memray run -o output.bin my_script.py

# Generate flame graph
python -m memray flamegraph output.bin

# Live TUI view
python -m memray run --live my_script.py
```

Programmatic:
```python
import memray

with memray.Tracker("output.bin"):
    my_function_to_profile()
```

### scalene (CPU + memory + GPU combined)

```bash
pip install scalene
python -m scalene my_script.py

# Output: annotated source showing CPU% and memory allocation per line
```

### asyncio profiling

```bash
# Enable asyncio debug mode (shows slow callbacks)
PYTHONASYNCIODEBUG=1 python my_app.py

# aiomonitor — interactive debugger for running asyncio app
pip install aiomonitor
```

```python
import asyncio
import aiomonitor

async def main():
    # ... your async app
    pass

loop = asyncio.get_event_loop()
with aiomonitor.start_monitor(loop):
    loop.run_until_complete(main())
```

Signs of blocked event loop:
- `asyncio: <Task> took 0.XXXs to complete` warnings in debug mode
- Missing `await` on blocking calls (file I/O, `time.sleep`, CPU-heavy code)
- Use `run_in_executor` for CPU or blocking I/O tasks

---

## Rust Profiling

### perf + flamegraph crate

```bash
# Install tools
cargo install flamegraph

# Build with debug symbols (keep optimizations)
# In Cargo.toml:
[profile.release]
debug = true

# Generate flamegraph (requires perf or dtrace)
cargo flamegraph --bin my_binary -- --arg1 --arg2

# Or with perf directly
perf record -g ./target/release/my_binary
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
```

### criterion (micro-benchmarking)

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

```bash
cargo bench
# Output: HTML report in target/criterion/
```

Key criterion concepts:
- `black_box()`: prevents compiler from optimizing away the computation
- Automatic warm-up and statistical analysis (mean, std dev, outliers)
- Detects regressions vs previous runs

### valgrind / heaptrack

```bash
# valgrind (slow but thorough — use on small inputs)
valgrind --tool=massif --pages-as-heap=yes ./target/debug/my_binary
ms_print massif.out.<pid> | head -50

# heaptrack (faster)
heaptrack ./target/release/my_binary
heaptrack_gui heaptrack.my_binary.<pid>.gz
```

### DHAT (heap allocation patterns)

```bash
valgrind --tool=dhat ./target/debug/my_binary
# Open dhat-viewer (web UI) with dhat.out.<pid>
```

DHAT shows: peak allocation sites, total bytes allocated, live at peak.

---

## General Optimization Patterns

### 1. Algorithmic Complexity First

No micro-optimization beats a better algorithm:
```
O(n²) at n=10,000 = 100,000,000 ops
O(n log n) at n=10,000 = 132,877 ops   <- 750x fewer operations
```

Profile first to confirm the algorithm IS the bottleneck.

### 2. Cache Efficiency

```
L1 cache: ~4 cycles  (32KB)
L2 cache: ~12 cycles (256KB)
L3 cache: ~40 cycles (8MB)
RAM:      ~200 cycles

Cache miss costs 50x more than a cache hit.
```

**Array-of-Structs vs Struct-of-Arrays:**
```rust
// Array-of-Structs (AoS) — bad for processing single field over all elements
struct Particle { x: f32, y: f32, vx: f32, vy: f32 }
let particles: Vec<Particle> = ...;

// Struct-of-Arrays (SoA) — cache-friendly when iterating over x,y only
struct Particles { x: Vec<f32>, y: Vec<f32>, vx: Vec<f32>, vy: Vec<f32> }
```

### 3. Database Queries

```sql
-- Always EXPLAIN ANALYZE, never just EXPLAIN
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

Warning signs:
- `Seq Scan` on large tables = missing index
- `Nested Loop` with large outer sets = N+1 problem
- `Hash Join` on huge datasets = consider materialized views
- High `Buffers: shared hit=X` with `read=Y` = cache miss ratio

N+1 detection:
```python
# Bad: 1 query for orders + N queries for each user
orders = Order.objects.all()
for order in orders:
    print(order.user.name)  # N queries!

# Good: JOIN in one query
orders = Order.objects.select_related('user').all()
```

### 4. Network I/O

```
Round trip reduction strategies:
- HTTP/2 multiplexing: multiple requests on one connection
- Request batching: send 100 items in one call vs 100 calls
- GraphQL or multi-fetch: fetch related data in one request
- Connection pooling: reuse TCP connections (HikariCP, pgbouncer)
- Compression: gzip/br for JSON payloads > 1KB
```

### 5. Serialization Benchmarks

```
Format       | Size (relative) | Speed (relative) | Schema required?
-------------|-----------------|------------------|------------------
JSON         | 1.0x (baseline) | 1.0x             | No
MessagePack  | 0.6x            | 2.0x             | No
Protobuf     | 0.4x            | 3.5x             | Yes
Avro         | 0.4x            | 3.0x             | Yes (registry)
FlatBuffers  | 0.8x            | 8.0x (zero-copy) | Yes
```

Profile serialization only if it shows up in the flame graph. For most services, it does not.

---

## Continuous Profiling in Production

```
+------------+     push profiles     +------------------+
| App (JVM)  | ------------------->  | Pyroscope Server |
| App (Go)   |   (via agent/SDK)     | (profile store)  |
| App (Py)   |                       +------------------+
+------------+                              |
                                            v
                                    +--------------+
                                    | Grafana UI   |
                                    | (explore by  |
                                    |  time/tag)   |
                                    +--------------+
```

Pyroscope setup (Java):
```java
// Add to startup
Pyroscope.start(new Config.Builder()
    .setApplicationName("my-service")
    .setServerAddress("http://pyroscope:4040")
    .setProfilingEvent(EventType.ITIMER)
    .build());
```

pprof HTTP endpoint (Go-style, also available in JVM via pyroscope):
```
GET /debug/pprof/profile?seconds=30   <- CPU profile
GET /debug/pprof/heap                 <- heap snapshot
GET /debug/pprof/goroutine            <- goroutine stacks
```

---

## Benchmark Hygiene Checklist

```
[ ] Warm-up: run 5-10 iterations before measuring (JIT compilation, disk cache)
[ ] Multiple iterations: 30+ samples minimum for statistical significance
[ ] Control for variance: pin CPU frequency, disable turbo boost, use isolated cores
[ ] Don't benchmark JIT startup: separate startup cost from steady-state cost
[ ] Use realistic data: production-like sizes, not toy inputs
[ ] Measure the right thing: wall clock for latency; CPU time to exclude scheduling noise
[ ] Black-box results: prevent dead code elimination (criterion's black_box, JMH's Blackhole)
[ ] Report percentiles: P50, P95, P99 — not averages (averages hide tail latency)
[ ] Separate concerns: one benchmark per hypothesis
[ ] Re-run after changes: never trust a single run comparison
```

---

## Anti-Patterns

```
WRONG: Optimize code that is not in the hot path.
RIGHT: Profile first, focus on top 3 hotspots in the flame graph.

WRONG: Use averages to compare benchmarks ("average latency is 20ms").
RIGHT: Use percentiles. P99 = 500ms with P50 = 10ms means you have a serious tail problem.

WRONG: Optimize in development on a Mac and assume it holds in production Linux.
RIGHT: Profile on production hardware or as close as possible.

WRONG: Add indexes to every column "just in case."
RIGHT: Add indexes where EXPLAIN ANALYZE shows Seq Scans on hot queries.

WRONG: Micro-optimize a O(n²) algorithm with SIMD.
RIGHT: Replace O(n²) with O(n log n) first.

WRONG: Profile with heap dump on a live production system mid-traffic.
RIGHT: Use JFR (low overhead) or take heap dumps during off-peak, or in canary instance.
```

---

## Quick Reference: Tool Selection

```
Symptom                          | Tool to use
---------------------------------|-------------------------------------------
High CPU in Java                 | async-profiler (CPU mode), JFR
Memory leak in Java              | async-profiler (alloc mode), Eclipse MAT
Slow Python function             | cProfile, scalene
Memory leak in Python            | memray
High CPU Python (no code change) | py-spy
Asyncio hanging / slow           | aiomonitor, PYTHONASYNCIODEBUG=1
Rust performance regression      | cargo flamegraph, criterion
Rust memory usage high           | heaptrack, DHAT
Slow SQL queries                 | EXPLAIN ANALYZE, pg_stat_statements
Production CPU profile           | Pyroscope continuous profiling
```
