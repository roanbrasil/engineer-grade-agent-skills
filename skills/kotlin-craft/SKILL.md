---
name: kotlin-craft
description: Expert Kotlin — null safety, coroutines, Flow, sealed classes, scope functions, DSLs, and idiomatic Kotlin over Java idioms
---

# Kotlin Craft

You are an expert Kotlin engineer. Apply idiomatic Kotlin patterns, coroutine-based concurrency, and null-safe design. Prefer concise, expressive Kotlin over verbose Java-style code.

---

## Null Safety

Kotlin's type system distinguishes nullable (`String?`) from non-nullable (`String`) at compile time.

```
Type System:
  String     — guaranteed non-null (compiler enforces)
  String?    — may be null (must handle before use)
  String!!   — "trust me it's non-null" (NullPointerException waiting to happen)
```

```kotlin
// Safe call operator — returns null if receiver is null
val length: Int? = name?.length

// Elvis operator — provide default if null
val displayName: String = name ?: "Anonymous"

// Safe call chain
val city: String? = user?.address?.city?.uppercase()

// let — run block only if non-null
user?.let { u ->
    println("User: ${u.name}, email: ${u.email}")
}

// requireNotNull / checkNotNull — validate at boundaries
fun process(data: ByteArray?) {
    val bytes = requireNotNull(data) { "data must not be null" }
    // bytes is ByteArray (non-null) here
}
```

**`!!` is a smell**: every `!!` is a potential NPE. If you find yourself writing `!!`, ask whether the type should be non-nullable, or handle nullability explicitly.

```kotlin
// WRONG
val name = user!!.name  // crashes if user is null

// RIGHT: restructure so user is non-null when needed
fun displayUser(user: User) {  // caller guarantees non-null
    println(user.name)
}
// or handle null explicitly
val name = user?.name ?: return
```

---

## Data Classes, Sealed Classes, Enums, Objects

### Data Classes

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String,
)
// Auto-generates: equals, hashCode, toString, copy, componentN functions

// copy() for immutable updates
val updated = user.copy(email = "new@example.com")

// Destructuring via componentN
val (id, name, email) = user
```

### Sealed Classes — Exhaustive Hierarchies

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// when is exhaustive — compiler verifies all cases
fun <T> Result<T>.getOrDefault(default: T): T = when (this) {
    is Result.Success  -> data
    is Result.Failure  -> default
    is Result.Loading  -> default
}
```

### Companion Objects and Object Declarations

```kotlin
class ApiClient private constructor(val baseUrl: String) {
    companion object {
        // Factory method — idiomatic alternative to static factory
        fun create(baseUrl: String): ApiClient = ApiClient(baseUrl)
        
        const val DEFAULT_TIMEOUT = 30_000L
    }
}

// Singleton — object declaration
object EventBus {
    private val listeners = mutableListOf<(Event) -> Unit>()
    fun subscribe(handler: (Event) -> Unit) { listeners += handler }
    fun publish(event: Event) { listeners.forEach { it(event) } }
}
```

---

## Extension Functions

Extensions add functions to existing types without inheritance.

```kotlin
// Idiomatic — adds methods to String without subclassing
fun String.toSlug(): String = lowercase().replace(Regex("[^a-z0-9]+"), "-")

fun <T> List<T>.second(): T {
    if (size < 2) throw NoSuchElementException("List has fewer than 2 elements")
    return this[1]
}

// Usage reads like a method call
"Hello World!".toSlug()  // "hello-world"
listOf("a", "b", "c").second()  // "b"
```

**When extension functions become confusing**:
- When they access private state (they can't — they only see public API)
- When the receiver type is too broad (e.g., `Any.myExtension()`)
- When they shadow member functions unexpectedly
- In deeply nested scope function chains — extract to named functions

---

## Scope Functions — Decision Tree

```
┌─────────────────────────────────────────────────────────────────┐
│ Scope function decision tree                                    │
│                                                                 │
│  Need the result of the block?                                  │
│    YES → let (it), run (this)                                   │
│    NO  → also (it), apply (this), with (this)                  │
│                                                                 │
│  Need to refer to context object as 'this' (implicit receiver)? │
│    YES → run, apply, with                                       │
│    NO (use 'it' explicitly) → let, also                        │
│                                                                 │
│  Common uses:                                                   │
│    let   → null checks, transform and use result               │
│    run   → run multiple operations, return result              │
│    with  → group operations on an object (non-extension)       │
│    apply → object configuration (returns receiver)             │
│    also  → side effects (logging, validation)                  │
└─────────────────────────────────────────────────────────────────┘
```

```kotlin
// let — null safety + transformation
val length = name?.let { it.trim().length } ?: 0

// apply — object configuration, returns the object
val request = HttpRequest().apply {
    url = "https://api.example.com"
    timeout = 30_000
    headers["Authorization"] = "Bearer $token"
}

// also — side effects without changing the chain
val users = repository.findAll()
    .also { log.debug("Loaded ${it.size} users") }

// run — multiple operations, return result
val summary = user.run {
    "$name (${email}) — ${orders.size} orders"
}

// with — operations on an existing object
val stats = with(calculator) {
    add(revenue)
    subtract(costs)
    divide(months)
    result()
}
```

---

## Coroutines

Kotlin coroutines are lightweight — millions can run on a handful of threads.

```
Thread model comparison:
  Threads:      Each blocked thread holds OS resource (~1MB stack)
  Coroutines:   Suspended coroutine holds only a few hundred bytes
                Blocking suspends the coroutine, not the thread

  ┌─────────────────────────────────────────────────┐
  │  Thread 1: [Coroutine A runs] [Coroutine C runs]│
  │  Thread 2: [Coroutine B runs] [Coroutine A runs]│
  │           ↑ A suspended, thread picks up C/B   │
  └─────────────────────────────────────────────────┘
```

### Structured Concurrency

```kotlin
// Structured concurrency: child coroutines cannot outlive parent scope
suspend fun loadDashboard(): Dashboard = coroutineScope {
    // Both launched concurrently within this scope
    val user   = async { fetchUser() }
    val orders = async { fetchOrders() }
    // If either fails, the other is cancelled, and the scope fails
    Dashboard(user.await(), orders.await())
}

// CoroutineScope for ongoing work (e.g., ViewModel)
class DashboardViewModel : ViewModel() {
    fun refresh() {
        viewModelScope.launch {          // cancelled when ViewModel is cleared
            try {
                val data = loadDashboard()
                _uiState.value = UiState.Success(data)
            } catch (e: CancellationException) {
                throw e                  // ALWAYS rethrow CancellationException
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}
```

### Dispatchers

```kotlin
Dispatchers.Main      // Android/UI thread — UI updates only
Dispatchers.IO        // I/O thread pool — network, disk (up to 64 threads)
Dispatchers.Default   // CPU thread pool — computation (= CPU cores)
Dispatchers.Unconfined // Inherits caller thread — avoid in production

// Switch dispatcher for blocking I/O in a suspend function
suspend fun readFile(path: String): String = withContext(Dispatchers.IO) {
    File(path).readText()
}

// CPU-bound work
suspend fun processData(data: List<Int>): List<Int> = withContext(Dispatchers.Default) {
    data.map { heavyComputation(it) }
}
```

---

## Flow

Flow is Kotlin's async stream — cold (nothing runs until collected).

```
Channel: hot stream — produces even without a collector (like a queue)
Flow:    cold stream — produces only when collected (like a function call)
StateFlow: hot, always has a value, replays last value to new collectors
SharedFlow: hot, configurable replay, multiple subscribers
```

### Basic Flow

```kotlin
// Cold flow — nothing happens until collect
fun fetchPrices(): Flow<Price> = flow {
    while (true) {
        emit(api.getPrice())          // suspend, emit value
        delay(1_000)                   // wait 1 second
    }
}

// Collecting
viewModelScope.launch {
    fetchPrices()
        .filter { it.change > 0.01 }
        .map { PriceUi(it.symbol, it.formatted()) }
        .flowOn(Dispatchers.IO)        // upstream runs on IO thread
        .collect { price ->            // collect on current dispatcher
            _prices.value = price
        }
}
```

### StateFlow and SharedFlow

```kotlin
// StateFlow — current state holder (replaces LiveData in Kotlin)
class SearchViewModel : ViewModel() {
    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    private val _results = MutableStateFlow<List<Result>>(emptyList())
    val results: StateFlow<List<Result>> = _results.asStateFlow()

    init {
        viewModelScope.launch {
            _query
                .debounce(300)                  // wait for typing to stop
                .filter { it.length >= 3 }
                .distinctUntilChanged()
                .flatMapLatest { q ->           // cancel previous search on new query
                    repository.search(q)
                }
                .collect { _results.value = it }
        }
    }
}

// SharedFlow — for events (one-time, multiple subscribers)
class EventBus {
    private val _events = MutableSharedFlow<AppEvent>()
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()
    suspend fun emit(event: AppEvent) { _events.emit(event) }
}
```

### Flow Operators

```kotlin
flow
    .map { transform(it) }
    .filter { predicate(it) }
    .flatMapMerge(concurrency = 4) { item -> // concurrent inner flows
        loadDetails(item)
    }
    .flatMapLatest { query ->          // cancel previous, start new
        search(query)
    }
    .combine(otherFlow) { a, b ->      // zip latest from both
        combine(a, b)
    }
    .catch { e -> emit(fallback()) }   // upstream error handling
    .onEach { log(it) }               // side effects
    .buffer(Channel.BUFFERED)         // decouple producer/consumer speed
```

---

## Kotlin DSL

Type-safe builders create clean DSL syntax using lambdas with receivers.

```kotlin
// HTML builder DSL
@DslMarker annotation class HtmlDsl

@HtmlDsl
class HtmlBuilder {
    private val content = StringBuilder()
    fun div(block: HtmlBuilder.() -> Unit) {
        content.append("<div>")
        this.block()
        content.append("</div>")
    }
    fun p(text: String) { content.append("<p>$text</p>") }
    fun build() = content.toString()
}

fun html(block: HtmlBuilder.() -> Unit): String =
    HtmlBuilder().apply(block).build()

// Usage
val page = html {
    div {
        p("Hello, World!")
        p("Kotlin DSLs are elegant")
    }
}
```

---

## Java Interop

```kotlin
// @JvmStatic — expose companion object method as Java static
class Config {
    companion object {
        @JvmStatic
        fun load(path: String): Config = Config()  // callable as Config.load() from Java
    }
}

// @JvmOverloads — generate Java overloads for default parameter functions
class HttpClient @JvmOverloads constructor(
    val baseUrl: String,
    val timeout: Long = 30_000L,
    val retries: Int = 3,
)
// Generates in Java: HttpClient(url), HttpClient(url, timeout), HttpClient(url, timeout, retries)

// @JvmField — expose property as Java field (no getter/setter)
class Constants {
    companion object {
        @JvmField val MAX_SIZE = 1024
    }
}

// Handling SAM (Single Abstract Method) interfaces
// Kotlin lambdas auto-convert to Java SAM interfaces
button.setOnClickListener { view -> handleClick(view) }  // Runnable, Callable, etc.
```

---

## Idiomatic vs Non-Idiomatic

```kotlin
// --- Null checks ---
// Non-idiomatic (Java style)
if (name != null) {
    println(name.length)
}
// Idiomatic
name?.let { println(it.length) }
println(name?.length ?: 0)

// --- Mutable vs immutable ---
// Non-idiomatic
var list = mutableListOf<String>()
var count = 0
// Idiomatic
val list = mutableListOf<String>()   // val binding, mutable content
val count by lazy { computeCount() } // lazy initialization

// --- String templates ---
// Non-idiomatic
val msg = "Hello, " + name + "! You have " + count + " messages."
// Idiomatic
val msg = "Hello, $name! You have $count messages."
val detail = "User: ${user.name.trim().uppercase()}"

// --- When expressions ---
// Non-idiomatic
if (x == 1) "one"
else if (x == 2) "two"
else "other"
// Idiomatic
when (x) {
    1 -> "one"
    2 -> "two"
    else -> "other"
}

// --- Collections ---
// Non-idiomatic
val result = ArrayList<String>()
for (item in items) {
    if (item.active) result.add(item.name.uppercase())
}
// Idiomatic
val result = items.filter { it.active }.map { it.name.uppercase() }
```

---

## Testing

```kotlin
// Kotest — expressive Kotlin-native test framework
class UserServiceTest : FunSpec({
    val repo = mockk<UserRepository>()
    val service = UserService(repo)

    test("findById returns user when found") {
        val user = User(1L, "Alice", "alice@example.com")
        every { repo.findById(1L) } returns user

        val result = service.findById(1L)

        result shouldBe user
        verify(exactly = 1) { repo.findById(1L) }
    }

    test("findById throws when not found") {
        every { repo.findById(99L) } returns null
        shouldThrow<NotFoundException> { service.findById(99L) }
    }
})

// MockK — Kotlin-native mocking
val gateway = mockk<PaymentGateway>()
coEvery { gateway.charge(any()) } returns PaymentResult.SUCCESS  // suspend function mock
coVerify { gateway.charge(match { it.amount > 0 }) }

// Testing coroutines with TestCoroutineScope
@Test
fun `search debounces rapid input`() = runTest {
    val viewModel = SearchViewModel(repo)
    viewModel.query.value = "a"
    viewModel.query.value = "ab"
    viewModel.query.value = "abc"
    advanceTimeBy(400)  // advance virtual time past debounce
    coVerify(exactly = 1) { repo.search("abc") }
}
```

---

## Anti-Patterns

| Anti-pattern                          | Idiomatic alternative                              |
|---------------------------------------|----------------------------------------------------|
| `!!` everywhere                       | Restructure types to be non-null, or use `?:`      |
| `@JvmStatic` top-level functions      | Top-level Kotlin functions (no class needed)       |
| Mutable data classes (var fields)     | `val` fields + `.copy()` for updates               |
| `object : Runnable { override fun run() {...} }` | Lambda: `{ ... }` (SAM conversion)   |
| `lateinit var` for nullable fields    | `var field: Type? = null` and null-safe access     |
| Checking `if (optional.isPresent)`    | Use Kotlin nullable types instead of Java Optional |
| `runBlocking` in production coroutines| Only in tests and `main()`                         |
| Ignoring `CancellationException`      | Always rethrow it                                  |
| `.value` on StateFlow from wrong dispatcher | Collect in the right dispatcher/scope         |

---

## Checklist

- [ ] `val` preferred over `var` everywhere possible
- [ ] No `!!` in production code (treat each as a bug to fix)
- [ ] Sealed classes used for finite state hierarchies (not enums with behavior)
- [ ] `when` expressions are exhaustive (sealed classes enforce this)
- [ ] Coroutines use structured concurrency (`coroutineScope`, `viewModelScope`)
- [ ] `CancellationException` is always rethrown
- [ ] `withContext(Dispatchers.IO)` for blocking I/O in suspend functions
- [ ] Flow collected with `.flowOn()` for upstream dispatcher changes
- [ ] `StateFlow` for state, `SharedFlow` for events
- [ ] Scope functions (`let`, `apply`, etc.) used with intent, not overused
- [ ] Extension functions are in appropriate scope (not polluting global namespace)
- [ ] MockK for mocking, not Mockito (handles suspend functions natively)
- [ ] `runTest` from `kotlinx-coroutines-test` for coroutine tests
