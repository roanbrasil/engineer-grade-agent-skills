---
name: rust-systems
description: Expert Rust systems programming — ownership, lifetimes, async, traits, unsafe, and the idiomatic Rust mental model
---

# Rust Systems Programming

You are an expert Rust systems programmer. When helping with Rust code, apply the mental models, idioms, and patterns below. Prefer idiomatic Rust over literal translations from other languages.

---

## Ownership Mental Model

Rust's ownership system is a compile-time discipline, not a runtime cost. Every value has exactly one owner. When the owner goes out of scope, the value is dropped (RAII).

```
Stack frame A          Stack frame B
┌─────────────┐        ┌─────────────┐
│  s: String ─┼──────► │  heap data  │
└─────────────┘        └─────────────┘
     owner                  data

After move to fn foo(s: String):
┌─────────────┐        ┌─────────────┐
│  (invalid)  │        │  heap data  │
└─────────────┘        └─────────────┘
                  foo's s owns it now
```

**Move vs Copy**: Types implementing `Copy` (integers, bools, `&T`) are copied implicitly. Everything else is moved.

```rust
// IDIOMATIC — take ownership when you need it
fn process(data: Vec<u8>) -> usize { data.len() }

// IDIOMATIC — borrow when you only need to read
fn inspect(data: &Vec<u8>) -> usize { data.len() }

// EVEN MORE IDIOMATIC — borrow the slice, accept more types
fn inspect(data: &[u8]) -> usize { data.len() }
```

### Borrowing Rules (enforced at compile time)

```
At any point in time, you may have EITHER:
  ┌──────────────────────────────────┐
  │  Any number of shared refs &T   │  (read-only, concurrent OK)
  └──────────────────────────────────┘
             OR
  ┌──────────────────────────────────┐
  │  Exactly one exclusive ref &mut T│  (read+write, exclusive)
  └──────────────────────────────────┘
  But NEVER both simultaneously.
```

---

## Lifetimes

Lifetimes are names for scopes. They let the compiler verify that references don't outlive the data they point to. Most of the time, lifetime elision handles this automatically.

```rust
// Elision hides lifetimes here — compiler infers 'a
fn longest(x: &str, y: &str) -> &str {
    // ERROR: compiler can't tell which input the output refers to
}

// Explicit: "output lifetime is tied to the shorter of x, y"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**Lifetime anti-pattern**: fighting the borrow checker by introducing `Arc<Mutex<>>` everywhere. Often the right fix is restructuring data ownership, not shared pointers.

---

## Error Handling

```
Error handling hierarchy:
  Application code        → anyhow::Result<T>   (ergonomic, context-rich)
  Library / API boundary  → thiserror enums      (typed, matchable)
  Unrecoverable           → panic! / unwrap()    (only in tests or truly impossible cases)
```

### Idiomatic Library Error Type

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("database error: {0}")]
    Db(#[from] sqlx::Error),

    #[error("not found: {id}")]
    NotFound { id: u64 },

    #[error("invalid input: {0}")]
    InvalidInput(String),
}

// ? operator auto-converts via From impl
async fn fetch_user(id: u64, pool: &PgPool) -> Result<User, AppError> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id as i64)
        .fetch_optional(pool)
        .await?  // sqlx::Error -> AppError::Db via #[from]
        .ok_or(AppError::NotFound { id })?;
    Ok(user)
}
```

### Application Code (anyhow)

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config file: {path}"))?;
    let config: Config = toml::from_str(&content)
        .context("failed to parse config as TOML")?;
    Ok(config)
}
```

**Anti-patterns**:
- `unwrap()` in production code (use `expect("reason")` in tests, `?` in prod)
- `Box<dyn Error>` in library public APIs (loses type info for callers)
- Ignoring errors with `let _ = fallible();`

---

## Fearless Concurrency

### Shared State: `Arc<Mutex<T>>`

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// Arc = shared ownership across threads
// Mutex = exclusive access
let counter = Arc::new(Mutex::new(0u64));

let handles: Vec<_> = (0..8).map(|_| {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut n = counter.lock().unwrap();
        *n += 1;
    })
}).collect();

for h in handles { h.join().unwrap(); }
println!("{}", *counter.lock().unwrap()); // 8
```

**Deadlock prevention**: always acquire locks in the same order across threads. Prefer `RwLock<T>` when writes are rare.

### Message Passing: `mpsc`

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel::<String>();
let tx2 = tx.clone(); // multiple producers

thread::spawn(move || tx.send("hello".into()).unwrap());
thread::spawn(move || tx2.send("world".into()).unwrap());

for msg in rx { println!("{msg}"); } // blocks until all tx dropped
```

### Data Parallelism: Rayon

```rust
use rayon::prelude::*;

// Drop-in parallel iterator — no data races possible
let sum: u64 = (0u64..1_000_000).into_par_iter().sum();
let results: Vec<_> = data.par_iter().map(|x| expensive(x)).collect();
```

---

## Async Rust

```
Sync code:       thread blocks waiting → OS context switch
Async code:      task yields → runtime parks it, runs other tasks
                 No OS threads blocked → millions of concurrent tasks
```

### Tokio Basics

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Spawn concurrent tasks
    let (a, b) = tokio::join!(
        fetch_url("https://example.com/a"),
        fetch_url("https://example.com/b"),
    );
    println!("{} {}", a?, b?);
    Ok(())
}

async fn fetch_url(url: &str) -> anyhow::Result<String> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}
```

**Critical rules**:
- Never call blocking code (`std::fs`, `std::thread::sleep`, CPU-heavy loops) in async context — use `tokio::fs`, `tokio::time::sleep`, or `tokio::task::spawn_blocking`
- `async fn` returns a `Future` — it does nothing until awaited
- `Pin<Box<dyn Future>>` is for trait objects; prefer `async fn` or `impl Future`

```rust
// WRONG — blocks the async runtime thread
async fn bad() {
    std::thread::sleep(Duration::from_secs(1)); // starves other tasks
}

// RIGHT
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
}

// RIGHT — CPU-bound work
async fn cpu_work() -> u64 {
    tokio::task::spawn_blocking(|| expensive_computation()).await.unwrap()
}
```

---

## Zero-Cost Abstractions

### Iterators

```rust
// NOT idiomatic — C-style index loop
let mut sum = 0;
for i in 0..v.len() { sum += v[i]; }

// IDIOMATIC — iterator chain, same assembly output
let sum: i32 = v.iter().sum();

// Complex chains — still zero cost (no allocations mid-chain)
let result: Vec<String> = data
    .iter()
    .filter(|x| x.active)
    .map(|x| x.name.to_uppercase())
    .take(10)
    .collect();
```

### Monomorphization

Generics in Rust generate concrete implementations at compile time — no runtime dispatch overhead (unlike Java generics with type erasure).

```rust
// This generates two separate functions at compile time:
// max_i32(a: i32, b: i32) -> i32
// max_f64(a: f64, b: f64) -> f64
fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}
```

---

## Traits

Traits define shared behavior. They are Rust's answer to interfaces, type classes, and ad-hoc polymorphism.

```rust
// Defining a trait
pub trait Summary {
    fn summarize(&self) -> String;
    fn preview(&self) -> String {          // default implementation
        format!("{}...", &self.summarize()[..50])
    }
}

// Static dispatch (monomorphized, zero-cost)
fn print_summary(item: &impl Summary) { println!("{}", item.summarize()); }
fn print_summary<T: Summary>(item: &T) { println!("{}", item.summarize()); } // equivalent

// Dynamic dispatch (vtable, runtime cost, needed for heterogeneous collections)
fn print_any(items: &[Box<dyn Summary>]) {
    for item in items { println!("{}", item.summarize()); }
}
```

**Orphan Rule**: You can implement a trait for a type only if you own the trait OR the type. You cannot implement `Display` for `Vec<T>` in a third-party crate.

**Essential standard traits to implement**:
- `Display` / `Debug` — human-readable output
- `From<T>` / `Into<T>` — conversions (implement `From`, get `Into` free)
- `Iterator` — enables all iterator adapters
- `FromStr` — enables `.parse::<MyType>()`
- `Clone` / `Copy` — value duplication
- `PartialEq` / `Eq`, `PartialOrd` / `Ord` — comparisons

---

## Smart Pointers: When Each

```
Box<T>       — heap allocation, single owner, no overhead
               Use: recursive types, large stack values, trait objects

Rc<T>        — shared ownership, single-threaded, reference counted
               Use: graph/tree nodes in single-threaded code

Arc<T>       — shared ownership, thread-safe, atomic ref count
               Use: shared data across threads

Cow<'a, T>  — Clone-On-Write, borrows until mutation needed
               Use: functions that sometimes need to modify, sometimes not

&T / &mut T — zero-cost references (prefer these whenever lifetimes allow)
```

```rust
// Cow example: avoid allocation when no modification needed
use std::borrow::Cow;

fn normalize(s: &str) -> Cow<str> {
    if s.contains('_') {
        Cow::Owned(s.replace('_', "-"))  // allocates only when needed
    } else {
        Cow::Borrowed(s)                  // zero allocation
    }
}
```

---

## Unsafe Rust

`unsafe` blocks opt out of specific compiler guarantees. You, not the compiler, must uphold the invariants.

**Justified uses**:
- FFI with C libraries
- Low-level memory operations (allocators, SIMD)
- Performance-critical hot paths after profiling proves it necessary
- Implementing core abstractions (e.g., `Arc`, `Mutex` themselves use unsafe)

**Invariants you must uphold inside `unsafe`**:
1. No null pointer dereferences
2. No dangling pointers (data must outlive the reference)
3. No data races (Rust's type system normally prevents these)
4. No aliased mutable references
5. No uninitialized memory reads
6. Uphold `Send` and `Sync` semantics if implementing them manually

```rust
// Wrap unsafe in a safe API — the public interface must be sound
pub fn split_at_midpoint(s: &str) -> (&str, &str) {
    let mid = s.len() / 2;
    // Safety: mid is always <= s.len(), valid UTF-8 boundary assumed
    // (real code would use s.char_indices() to find a valid boundary)
    unsafe {
        (
            std::str::from_utf8_unchecked(&s.as_bytes()[..mid]),
            std::str::from_utf8_unchecked(&s.as_bytes()[mid..]),
        )
    }
}
```

---

## Idiomatic Patterns

### Builder Pattern

```rust
#[derive(Debug)]
pub struct Request {
    url: String,
    timeout: Duration,
    retries: u32,
}

pub struct RequestBuilder {
    url: String,
    timeout: Duration,
    retries: u32,
}

impl RequestBuilder {
    pub fn new(url: impl Into<String>) -> Self {
        Self { url: url.into(), timeout: Duration::from_secs(30), retries: 3 }
    }
    pub fn timeout(mut self, t: Duration) -> Self { self.timeout = t; self }
    pub fn retries(mut self, n: u32) -> Self { self.retries = n; self }
    pub fn build(self) -> Request {
        Request { url: self.url, timeout: self.timeout, retries: self.retries }
    }
}

let req = RequestBuilder::new("https://api.example.com")
    .timeout(Duration::from_secs(10))
    .retries(5)
    .build();
```

### Newtype Pattern

```rust
// Prevent mixing up semantically different values of the same type
struct Meters(f64);
struct Kilograms(f64);

fn calculate_force(mass: Kilograms, accel: f64) -> f64 {
    mass.0 * accel
}
// compile error: calculate_force(Meters(5.0), 9.8) — can't mix up
```

### Typestate Pattern

```rust
// Encode valid state transitions in the type system
struct Connection<State> { inner: TcpStream, _state: std::marker::PhantomData<State> }
struct Disconnected;
struct Connected;
struct Authenticated;

impl Connection<Disconnected> {
    pub fn connect(addr: &str) -> Result<Connection<Connected>, io::Error> { ... }
}
impl Connection<Connected> {
    pub fn authenticate(self, creds: &Credentials) -> Result<Connection<Authenticated>, AuthError> { ... }
}
impl Connection<Authenticated> {
    pub fn send_command(&mut self, cmd: &str) -> Result<Response, io::Error> { ... }
}
// Can't call send_command on a Disconnected or Connected state — compile error
```

---

## Cargo Workspace

```toml
# Workspace Cargo.toml
[workspace]
members = ["crates/core", "crates/api", "crates/cli"]
resolver = "2"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
anyhow = "1"

# crates/api/Cargo.toml
[dependencies]
tokio.workspace = true        # version from workspace
serde.workspace = true
anyhow.workspace = true
core = { path = "../core" }   # internal crate
```

---

## Essential Crate Ecosystem

| Crate       | Purpose                                | Key feature                          |
|-------------|----------------------------------------|--------------------------------------|
| `serde`     | Serialization/deserialization          | `#[derive(Serialize, Deserialize)]`  |
| `tokio`     | Async runtime                          | Tasks, timers, I/O, channels         |
| `axum`      | Web framework (tokio-native)           | Type-safe extractors, tower-based    |
| `sqlx`      | Async SQL (compile-time query checks)  | `query_as!` macro verifies at compile|
| `clap`      | CLI argument parsing                   | `#[derive(Parser)]`                  |
| `tracing`   | Structured logging/instrumentation     | Spans, async-aware                   |
| `thiserror` | Library error types                    | `#[derive(Error)]`                   |
| `anyhow`    | Application error handling             | `.context()`, `bail!`                |
| `rayon`     | Data parallelism                       | `.par_iter()`                        |
| `criterion` | Benchmarking                           | Statistical microbenchmarks          |

---

## Performance

**Profile first. Never guess.**

```bash
# CPU profiling
cargo build --release
perf record --call-graph dwarf ./target/release/myapp
perf report

# Flamegraph (install cargo-flamegraph)
cargo flamegraph --bin myapp

# Memory profiling
valgrind --tool=massif ./target/release/myapp
```

**Cache-friendly data structures**:
```rust
// BAD: Vec<Box<dyn Trait>> — pointer chasing, cache misses
// GOOD: Vec<ConcreteType> — contiguous memory

// Prefer SoA (Structure of Arrays) over AoS for hot loops
struct Particles {
    x: Vec<f32>,   // all x values contiguous
    y: Vec<f32>,   // all y values contiguous
    mass: Vec<f32>,
}
// vs AoS: Vec<Particle { x, y, mass }> — worse for SIMD
```

**SIMD** (portable via `std::simd` or `packed_simd`):
```rust
#![feature(portable_simd)]
use std::simd::f32x8;

fn dot_product_simd(a: &[f32], b: &[f32]) -> f32 {
    a.chunks_exact(8).zip(b.chunks_exact(8))
        .map(|(a, b)| (f32x8::from_slice(a) * f32x8::from_slice(b)).reduce_sum())
        .sum()
}
```

---

## Anti-Patterns

| Anti-pattern                        | Idiomatic alternative                          |
|-------------------------------------|------------------------------------------------|
| `.clone()` to satisfy borrow checker| Restructure ownership or use references        |
| `Arc<Mutex<>>` everywhere           | Often unnecessary — restructure data ownership |
| `unwrap()` in production            | `?` operator or explicit error handling        |
| Returning `impl Iterator` boxing    | Use `-> impl Iterator<Item=T>` without boxing  |
| `for i in 0..v.len() { v[i] }`     | `for item in &v` or `.iter()`                  |
| `String` param when `&str` works    | Accept `&str`, callers can deref String        |
| Mutating through shared reference   | Use `Cell<T>` or `RefCell<T>` if needed        |
| `panic!` for recoverable errors     | `Result<T, E>`                                 |

---

## Checklist

- [ ] No `unwrap()` or `expect()` in non-test production paths
- [ ] Public library functions return typed `Result<T, MyError>` (thiserror)
- [ ] No blocking calls inside `async fn` — use tokio equivalents
- [ ] Lifetimes explicit only when elision is insufficient
- [ ] `Arc<Mutex<T>>` justified (not default shared-state choice)
- [ ] `unsafe` blocks have a `// Safety:` comment explaining invariants
- [ ] Iterators preferred over manual index loops
- [ ] Builder/newtype/typestate used to enforce invariants at compile time
- [ ] `cargo clippy -- -D warnings` passes clean
- [ ] `cargo test` covers happy path and error cases
- [ ] Release profile used for benchmarking (`cargo build --release`)
- [ ] Workspace used for multi-crate projects
