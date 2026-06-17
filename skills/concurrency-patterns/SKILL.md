---
name: concurrency-patterns
description: Expert concurrency patterns — invoke when designing or debugging multi-threaded code, async systems, actor models, channels, race conditions, deadlocks, or parallel data processing in Java, Kotlin, Python, Go, or Rust.
---

# Concurrency Patterns

## Concurrency Models Overview

```
MODEL              COMMUNICATION         SYNCHRONIZATION     LANGUAGES
------------------ --------------------- ------------------- -------------------------
Shared memory      Shared variables      Locks / atomics     Java, C++, Rust (Mutex)
Message passing    Mailboxes / queues    None needed         Akka, Erlang, Go channels
Async / await      Callbacks / futures   Event loop          JS, Python asyncio, C#
Data parallelism   Implicit via runtime  None visible        Rayon, NumPy, SIMD
CSP channels       Typed channels        Channel ops block   Go, Kotlin coroutines
```

### When to Use Which

| Workload              | Best Model               | Reason                                    |
|-----------------------|--------------------------|-------------------------------------------|
| CPU-bound parallelism | Data parallelism / Fork-join | Utilize multiple cores efficiently    |
| I/O-bound (many conns)| Async / virtual threads  | Avoid blocking OS threads               |
| Isolated state actors | Actor model              | Eliminate shared state problems           |
| Pipeline processing   | CSP channels             | Natural backpressure, composable stages   |
| Simple coordination   | Thread pool + futures    | Familiar, sufficient for most cases       |

---

## Common Concurrency Problems

### Race Condition

```
Thread A: read x (=0)          Thread B: read x (=0)
Thread A: x = x + 1            Thread B: x = x + 1
Thread A: write x (=1)         Thread B: write x (=1)
Result: x = 1  WRONG (expected 2)
```

Fix: synchronize the read-modify-write, or use an atomic operation.

### Deadlock

```
Thread A holds Lock1, waits for Lock2
Thread B holds Lock2, waits for Lock1
Both wait forever.
```

Fix strategies:
1. Lock ordering: always acquire locks in the same order (Lock1 before Lock2 everywhere)
2. tryLock with timeout: if timeout expires, release held locks and retry
3. Avoid holding multiple locks simultaneously (redesign with a single lock or message passing)

### Livelock

Two threads keep reacting to each other without making progress:
```
Thread A sees conflict, backs off
Thread B sees conflict, backs off at the same time
Thread A retries, sees conflict again...
```

Fix: add random jitter to backoff. `sleep(random(10ms..100ms))` before retry.

### Starvation

Low-priority thread never gets the lock because high-priority threads always win.

Fix: use fair locks (`ReentrantLock(fair = true)`). Fair locks queue waiters in FIFO order.

### False Sharing

```
Cache line (64 bytes):  [ counter_A | counter_B | ... ]
Thread A writes counter_A -> invalidates entire cache line
Thread B writes counter_B -> must reload, even though it's a different variable
```

Fix: pad variables to separate cache lines, or use `@Contended` in Java.

---

## Java / Kotlin Concurrency Primitives

### Lock Comparison

```
PRIMITIVE           REENTRANCY  FAIRNESS  CONDITION   TIMEOUT   PERFORMANCE
synchronized        yes         no        wait/notify no        baseline
ReentrantLock       yes         optional  Condition   yes       similar
ReadWriteLock       yes         optional  2 queues    yes       good for read-heavy
StampedLock         limited     no        optimistic  yes       best for read-heavy
AtomicXxx           N/A         N/A       N/A         N/A       lock-free, fastest
```

### synchronized vs ReentrantLock

```kotlin
// synchronized — simple, no timeout, not interruptible
@Synchronized
fun increment() {
    count++
}

// ReentrantLock — tryLock, interruptible, fair mode
val lock = ReentrantLock(/* fair = */ true)

fun safeUpdate() {
    if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
        try {
            count++
        } finally {
            lock.unlock()  // ALWAYS unlock in finally
        }
    } else {
        throw TimeoutException("Could not acquire lock")
    }
}
```

### ReadWriteLock

```kotlin
val rwLock = ReentrantReadWriteLock()
val cache = mutableMapOf<String, String>()

fun get(key: String): String? {
    rwLock.readLock().lock()
    try {
        return cache[key]  // multiple threads can read simultaneously
    } finally {
        rwLock.readLock().unlock()
    }
}

fun put(key: String, value: String) {
    rwLock.writeLock().lock()
    try {
        cache[key] = value  // exclusive write
    } finally {
        rwLock.writeLock().unlock()
    }
}
```

### StampedLock (Optimistic Read)

```kotlin
val stampedLock = StampedLock()
var balance = 0L

fun readBalance(): Long {
    // Try optimistic read (no lock acquired)
    val stamp = stampedLock.tryOptimisticRead()
    val value = balance
    if (stampedLock.validate(stamp)) {
        return value  // No write happened — return value without locking
    }
    // Write happened during read — fall back to read lock
    val readStamp = stampedLock.readLock()
    try {
        return balance
    } finally {
        stampedLock.unlockRead(readStamp)
    }
}

fun writeBalance(newBalance: Long) {
    val stamp = stampedLock.writeLock()
    try {
        balance = newBalance
    } finally {
        stampedLock.unlockWrite(stamp)
    }
}
```

### Atomic Classes

```kotlin
val counter = AtomicLong(0)
val ref = AtomicReference<String>("initial")

counter.incrementAndGet()                    // lock-free increment
counter.addAndGet(5)                         // lock-free add
ref.compareAndSet("initial", "updated")     // CAS — only updates if current == expected

// AtomicInteger for array indices
val index = AtomicInteger(0)
fun nextIndex(size: Int) = index.getAndIncrement() % size
```

### volatile

```kotlin
// Use ONLY for single-writer, multi-reader flags
// volatile guarantees visibility, NOT atomicity

@Volatile
var running = true  // written by one thread, read by many

fun stop() { running = false }        // single writer
fun isRunning() = running             // multiple readers OK

// WRONG: volatile for counter incremented by multiple threads
@Volatile var count = 0
fun increment() { count++ }  // NOT atomic: read -> increment -> write is 3 steps
```

### Coordination Utilities

```kotlin
// CountDownLatch: wait for N events (one-shot, cannot be reset)
val latch = CountDownLatch(3)
repeat(3) { launch { doWork(); latch.countDown() } }
latch.await()  // blocks until count reaches 0

// CyclicBarrier: rendezvous point for N threads (reusable)
val barrier = CyclicBarrier(4) { println("All threads reached barrier") }
repeat(4) { launch { doPhase1(); barrier.await(); doPhase2() } }

// Semaphore: limit concurrent access (bulkhead)
val semaphore = Semaphore(5)  // max 5 concurrent database connections
fun query() {
    semaphore.acquire()
    try { db.query() }
    finally { semaphore.release() }
}
```

### Virtual Threads (Java 21+)

```java
// Before: blocking I/O ties up a platform thread
// After: blocking I/O parks the virtual thread; platform thread is freed

// Creating virtual threads
Thread.ofVirtual().start(() -> {
    var result = httpClient.send(request, bodyHandler);  // parks, not blocks
    process(result);
});

// With ExecutorService (recommended for pools)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = requests.stream()
        .map(req -> executor.submit(() -> httpClient.send(req, bodyHandler)))
        .toList();
    var results = futures.stream().map(Future::get).toList();
}
```

Virtual threads eliminate the need for reactive/async programming for I/O-bound workloads. A server can handle 100,000 concurrent connections with blocking code that reads like single-threaded code.

---

## Actor Model (Akka / Kotlin)

### Core Concepts

```
Actor
  +-- Mailbox (message queue)
  +-- Behavior (message handler function)
  +-- State (encapsulated; not shared)
  +-- Children (spawned actors)

Actors process one message at a time — no concurrent access to their state.
Communication: send a message to an ActorRef. Fire and forget by default.
```

### Akka Typed (Java/Kotlin)

```kotlin
// Define messages
sealed class CounterCommand
data class Increment(val by: Int) : CounterCommand()
data class GetCount(val replyTo: ActorRef<Int>) : CounterCommand()
object Reset : CounterCommand()

// Define behavior
fun counterBehavior(count: Int = 0): Behavior<CounterCommand> =
    Behaviors.receive { ctx, message ->
        when (message) {
            is Increment -> counterBehavior(count + message.by)  // return new behavior
            is GetCount  -> { message.replyTo.tell(count); Behaviors.same() }
            is Reset     -> counterBehavior(0)
        }
    }

// Usage
val system = ActorSystem.create(counterBehavior(), "counter-system")
system.tell(Increment(5))
system.tell(Increment(3))

// Ask pattern (request-reply)
val count: CompletableFuture<Int> = AskPattern.ask(
    system,
    { replyTo: ActorRef<Int> -> GetCount(replyTo) },
    Duration.ofSeconds(3),
    system.scheduler()
)
```

### Supervision

```kotlin
// Parent supervises children; decides what to do on failure
Behaviors.supervise(childBehavior())
    .onFailure<DatabaseException>(SupervisorStrategy.restart())
    .onFailure<UnrecoverableException>(SupervisorStrategy.stop())

// Supervision strategies
// resume()  — ignore the failure, continue with current state
// restart() — reset state to initial, restart behavior
// stop()    — stop the actor permanently
// escalate  — (typed: throw, parent handles it)
```

---

## CSP Channels (Kotlin Coroutines / Go)

### Kotlin Channels

```kotlin
// Basic channel
val channel = Channel<Int>(capacity = 10)  // buffered

// Producer coroutine
launch {
    for (i in 1..5) {
        channel.send(i)  // suspends if buffer full
    }
    channel.close()
}

// Consumer coroutine
launch {
    for (value in channel) {  // receives until closed
        println("Received: $value")
    }
}

// produce builder (cleaner)
val numbers: ReceiveChannel<Int> = produce {
    for (i in 1..5) send(i)
}
```

### Pipeline Pattern

```kotlin
fun CoroutineScope.naturals(start: Int): ReceiveChannel<Int> = produce {
    var n = start
    while (true) send(n++)
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (n in numbers) send(n * n)
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int): ReceiveChannel<Int> = produce {
    for (n in numbers) if (n % prime != 0) send(n)
}

// Usage: naturals -> square -> filter
val nums = naturals(2)
val squares = square(nums)
val filtered = filter(squares, 4)
```

### Fan-Out and Fan-In

```kotlin
// Fan-out: distribute work to multiple workers
fun CoroutineScope.fanOut(
    source: ReceiveChannel<Task>,
    workerCount: Int
): List<ReceiveChannel<Result>> {
    return (1..workerCount).map { workerId ->
        produce {
            for (task in source) {
                send(processTask(workerId, task))
            }
        }
    }
}

// Fan-in: merge multiple channels into one
fun CoroutineScope.fanIn(vararg channels: ReceiveChannel<Result>): ReceiveChannel<Result> = produce {
    for (channel in channels) {
        launch {
            for (result in channel) send(result)
        }
    }
}
```

### select (Non-Blocking Choice)

```kotlin
select<Unit> {
    channelA.onReceive { value -> handleA(value) }
    channelB.onReceive { value -> handleB(value) }
    onTimeout(1000) { handleTimeout() }
}
```

### Go Channels (Reference)

```go
// Goroutine + channel
ch := make(chan int, 5)  // buffered
go func() {
    for i := 0; i < 5; i++ { ch <- i }
    close(ch)
}()
for v := range ch { fmt.Println(v) }

// select
select {
case v := <-chA: handleA(v)
case v := <-chB: handleB(v)
case <-time.After(time.Second): handleTimeout()
}
```

---

## Python Concurrency

### The GIL Explained

```
threading module:
  - OS threads exist
  - GIL (Global Interpreter Lock) allows only one thread to execute Python bytecode at a time
  - I/O releases the GIL — threads CAN overlap during I/O
  - CPU-bound: threads are useless; use multiprocessing

asyncio:
  - Single thread, single GIL holder
  - Coroutines cooperate by yielding at await points
  - No parallelism; great for I/O concurrency with minimal overhead
```

### threading (I/O-Bound)

```python
import threading
from queue import Queue

def worker(q: Queue):
    while True:
        url = q.get()
        if url is None:
            break
        response = requests.get(url)  # GIL released during I/O
        process(response)
        q.task_done()

queue = Queue()
threads = [threading.Thread(target=worker, args=(queue,)) for _ in range(10)]
for t in threads:
    t.start()

for url in urls:
    queue.put(url)

queue.join()
for _ in threads:
    queue.put(None)  # stop signal
```

### multiprocessing (CPU-Bound)

```python
from multiprocessing import Pool
import os

def cpu_bound_task(data: list[int]) -> int:
    return sum(x * x for x in data)  # CPU-intensive

if __name__ == "__main__":
    chunks = [list(range(i, i + 1000)) for i in range(0, 100000, 1000)]

    with Pool(processes=os.cpu_count()) as pool:
        results = pool.map(cpu_bound_task, chunks)  # true parallelism

    total = sum(results)
```

### asyncio (I/O Concurrency)

```python
import asyncio
import aiohttp

async def fetch(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url) as response:
        return await response.text()

async def fetch_all(urls: list[str]) -> list[str]:
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)  # concurrent, not parallel

async def main():
    results = await fetch_all(["https://example.com"] * 100)
    print(f"Fetched {len(results)} pages")

asyncio.run(main())
```

### concurrent.futures (High-Level Abstraction)

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# I/O-bound: ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(requests.get, url) for url in urls]
    results = [f.result() for f in futures]

# CPU-bound: ProcessPoolExecutor
with ProcessPoolExecutor() as executor:
    results = list(executor.map(cpu_heavy, data_chunks))
```

---

## Rust Concurrency

### Ownership Prevents Data Races

```rust
// This does NOT compile — Rust prevents sharing mutable state
let mut v = vec![1, 2, 3];
let t1 = thread::spawn(|| v.push(4));  // ERROR: v moved into closure
let t2 = thread::spawn(|| v.push(5));  // ERROR: v already moved
```

### Arc<Mutex<T>> (Shared Mutable State)

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0i64));

let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut count = counter.lock().unwrap();
        *count += 1;
        // lock released when `count` drops
    })
}).collect();

for h in handles { h.join().unwrap(); }
println!("Final: {}", *counter.lock().unwrap());
```

### mpsc Channel

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel::<String>();

// Multiple producers
for i in 0..5 {
    let tx = tx.clone();
    thread::spawn(move || {
        tx.send(format!("message from thread {}", i)).unwrap();
    });
}
drop(tx);  // drop original sender so rx knows when all are done

// Single consumer
for msg in rx {
    println!("Received: {}", msg);
}
```

### Rayon (Data Parallelism)

```rust
use rayon::prelude::*;

let data: Vec<i64> = (0..1_000_000).collect();

// Sequential
let sum_seq: i64 = data.iter().map(|x| x * x).sum();

// Parallel — change iter() to par_iter()
let sum_par: i64 = data.par_iter().map(|x| x * x).sum();
// Rayon divides data across CPU cores automatically
```

### tokio (Async Runtime)

```rust
use tokio::{spawn, select, time::{sleep, Duration}};

#[tokio::main]
async fn main() {
    // Concurrent tasks
    let t1 = spawn(async { fetch_data("https://api-a.example.com").await });
    let t2 = spawn(async { fetch_data("https://api-b.example.com").await });

    let (r1, r2) = tokio::join!(t1, t2);  // wait for both

    // Race: first one wins
    select! {
        result = fast_op()   => println!("fast won: {:?}", result),
        result = slow_op()   => println!("slow won: {:?}", result),
        _ = sleep(Duration::from_secs(5)) => println!("timeout"),
    }
}
```

---

## Producer-Consumer Pattern

### Java (BlockingQueue)

```java
public class ProducerConsumer {
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

    class Producer implements Runnable {
        @Override public void run() {
            for (Task task : source) {
                try {
                    queue.put(task);  // blocks if queue full (backpressure)
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
    }

    class Consumer implements Runnable {
        @Override public void run() {
            while (true) {
                try {
                    Task task = queue.take();  // blocks if empty
                    process(task);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
    }
}
```

### Kotlin (Channel + Coroutines)

```kotlin
fun CoroutineScope.producer(channel: SendChannel<Task>) = launch {
    source.forEach { task ->
        channel.send(task)  // suspends if channel full — cooperative backpressure
    }
    channel.close()
}

fun CoroutineScope.consumer(channel: ReceiveChannel<Task>) = launch {
    for (task in channel) {
        process(task)
    }
}

runBlocking {
    val channel = Channel<Task>(capacity = 100)
    val prod = producer(channel)
    val consumers = (1..4).map { consumer(channel) }
    prod.join()
    consumers.joinAll()
}
```

### Python (asyncio.Queue)

```python
import asyncio

async def producer(queue: asyncio.Queue, items: list):
    for item in items:
        await queue.put(item)  # suspends if full
    await queue.put(None)  # poison pill

async def consumer(queue: asyncio.Queue, worker_id: int):
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        await process(item)
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=50)
    tasks = [
        asyncio.create_task(producer(queue, data)),
        *[asyncio.create_task(consumer(queue, i)) for i in range(4)]
    ]
    await asyncio.gather(*tasks)
```

### Rust (tokio mpsc)

```rust
use tokio::sync::mpsc;

async fn producer(tx: mpsc::Sender<Task>, items: Vec<Task>) {
    for item in items {
        tx.send(item).await.expect("receiver dropped");
    }
    // tx dropped here; receiver sees channel closed
}

async fn consumer(mut rx: mpsc::Receiver<Task>) {
    while let Some(task) = rx.recv().await {
        process(task).await;
    }
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel::<Task>(100);  // bounded channel
    tokio::join!(
        producer(tx, generate_tasks()),
        consumer(rx)
    );
}
```

---

## Patterns Quick Reference

### Backpressure

Producers must not overwhelm consumers. Mechanisms:

```
Java: BlockingQueue.put() blocks when full
Kotlin: Channel.send() suspends when full (buffered) or waits for receiver (rendezvous)
Python: asyncio.Queue.put() awaits when full
Rust: mpsc::Sender.send().await suspends when channel full
Reactive Streams: request(n) protocol — consumer controls flow rate
```

### Bulkhead (Resource Isolation)

```kotlin
// Separate thread pools for different concerns
val dbPool    = Executors.newFixedThreadPool(10)
val httpPool  = Executors.newFixedThreadPool(20)
val workerPool = Executors.newVirtualThreadPerTaskExecutor()

// Failure in dbPool does not starve httpPool
// Kotlin: separate Dispatchers
val dbDispatcher = Executors.newFixedThreadPool(10).asCoroutineDispatcher()
withContext(dbDispatcher) { db.query() }
```

### Thread Confinement

```kotlin
// UI thread: all UI updates on a single thread (no synchronization needed)
withContext(Dispatchers.Main) { textView.text = result }

// Single-threaded dispatcher: state owned by one coroutine
val stateDispatcher = newSingleThreadContext("state-thread")
withContext(stateDispatcher) { state.update() }  // safe without locks
```

---

## Anti-Patterns

| Anti-Pattern                                | Problem                         | Fix                                    |
|---------------------------------------------|---------------------------------|----------------------------------------|
| `synchronized` on `this` in public class    | External code can deadlock you  | Lock on private object                 |
| Lock acquired in wrong order                | Deadlock                        | Define a global lock order             |
| `Thread.sleep()` as synchronization         | Race condition window           | Use `CountDownLatch` or `CompletableFuture` |
| `volatile` for counter shared by N writers  | Not atomic: 3-step read-modify-write | Use `AtomicInteger`               |
| `catch (InterruptedException e) {}`         | Swallows interrupt signal       | Re-interrupt: `Thread.currentThread().interrupt()` |
| Unbounded thread pool                       | OOM under load                  | Use bounded pool or virtual threads    |
| Shared mutable state across actors          | Concurrency bugs return         | Actors own their state; message-only   |
| `asyncio` for CPU-bound work                | Blocks event loop               | Use `loop.run_in_executor(ProcessPoolExecutor())` |
| Channel with no capacity limit              | Unbounded memory growth         | Set explicit capacity for backpressure |
| `Mutex::lock().unwrap()` in production Rust | Panics on poisoned mutex        | Handle poisoned mutex explicitly       |

## Concurrency Checklist

- [ ] Identified workload type: CPU-bound vs I/O-bound
- [ ] Chosen the right concurrency model for the workload
- [ ] Shared mutable state protected with appropriate synchronization
- [ ] Lock acquisition order documented and enforced (no deadlocks)
- [ ] `InterruptedException` handled correctly (re-interrupt, not swallowed)
- [ ] Thread pool sizes are bounded and tuned
- [ ] Backpressure mechanism in place for all producer-consumer flows
- [ ] Volatile used only for single-writer flags (not counters)
- [ ] Atomic classes used for lock-free counters and references
- [ ] Actor/channel-based code: no shared mutable state between actors
- [ ] Timeouts on all blocking operations
- [ ] Virtual threads considered for I/O-heavy Java 21+ workloads
- [ ] Tests include concurrency stress test (multiple threads, high iteration count)
