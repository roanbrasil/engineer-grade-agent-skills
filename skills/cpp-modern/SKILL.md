---
name: cpp-modern
description: Expert Modern C++ (C++17/20) — RAII, smart pointers, move semantics, templates, concepts, ranges, concurrency, and undefined behavior avoidance
---

# Modern C++ (C++17/20)

You are an expert Modern C++ engineer. Apply C++17/20 idioms, RAII everywhere, zero-overhead abstractions, and strict undefined behavior avoidance. Prefer expressive modern idioms over raw C-style code.

---

## RAII — The Foundation

Resource Acquisition Is Initialization: acquire resources in constructors, release in destructors. The destructor guarantee is unconditional — it runs when the object goes out of scope, even on exception.

```cpp
// RAII file handle — no manual cleanup needed
class FileHandle {
    FILE* fp_;
public:
    explicit FileHandle(const char* path, const char* mode)
        : fp_(std::fopen(path, mode))
    {
        if (!fp_) throw std::runtime_error(std::string("cannot open: ") + path);
    }
    ~FileHandle() { if (fp_) std::fclose(fp_); }  // guaranteed cleanup

    // Non-copyable, movable
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : fp_(std::exchange(other.fp_, nullptr)) {}
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) { std::fclose(fp_); fp_ = std::exchange(other.fp_, nullptr); }
        return *this;
    }

    FILE* get() const noexcept { return fp_; }
};

// In practice, use RAII wrappers from the standard library:
// std::fstream, std::ifstream, std::ofstream — not FILE*
// std::lock_guard, std::scoped_lock — not manual lock/unlock
// std::unique_ptr, std::shared_ptr — not new/delete
```

---

## Smart Pointers

```
Ownership model:
  unique_ptr<T>   — sole owner, zero overhead vs raw pointer
                    Transfer ownership with std::move()
                    
  shared_ptr<T>   — shared ownership, reference counted
                    Use only when ownership is genuinely shared
                    
  weak_ptr<T>     — non-owning observer of shared_ptr
                    Breaks cycles (e.g., parent-child in a graph)

  Raw T*           — non-owning reference, fine for "borrow" semantics
                    Document with [[nodiscard]] or gsl::not_null
```

```cpp
// unique_ptr — default choice for heap allocation
auto buffer = std::make_unique<uint8_t[]>(1024);  // no delete needed
auto widget = std::make_unique<Widget>("title");

// Transfer ownership
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(5.0));

// shared_ptr — when multiple owners needed
auto config = std::make_shared<Config>(configPath);
auto worker1 = Worker(config);  // shared_ptr copy — ref count 2
auto worker2 = Worker(config);  // ref count 3

// weak_ptr — break reference cycles
class Node {
    std::shared_ptr<Node> child_;
    std::weak_ptr<Node> parent_;  // weak to break cycle, not weak_ptr<Node> child!
public:
    void setParent(std::shared_ptr<Node> p) { parent_ = p; }
    std::shared_ptr<Node> parent() const { return parent_.lock(); } // may return nullptr
};

// ANTI-PATTERN: shared_ptr for single ownership
auto x = std::shared_ptr<int>(new int(42)); // use unique_ptr
// ANTI-PATTERN: make_shared<T[]> for arrays (until C++20)
// Use std::vector<T> instead
```

---

## Move Semantics

Move semantics allow transferring resources from an expiring object instead of copying.

```
Copy:  allocate new buffer, copy all bytes                O(n)
Move:  steal the pointer, set source to null              O(1)

Rvalue reference (T&&): binds to temporaries and std::move() results
Lvalue reference (T&):  binds to named objects
```

### Rule of Zero / Three / Five

```cpp
// Rule of Zero — prefer this: let member types handle their own resources
class Config {
    std::string filename_;          // String handles its own memory
    std::vector<Setting> settings_; // Vector handles its own memory
    // No destructor, no copy/move — compiler-generated are correct
};

// Rule of Five — when you manage resources manually
class Buffer {
    std::byte* data_;
    std::size_t size_;
public:
    explicit Buffer(std::size_t n) : data_(new std::byte[n]), size_(n) {}
    
    ~Buffer()                                               { delete[] data_; }
    Buffer(const Buffer& o)        : data_(new std::byte[o.size_]), size_(o.size_)
                                     { std::copy_n(o.data_, size_, data_); }
    Buffer& operator=(const Buffer& o) {
        if (this != &o) { delete[] data_; data_ = new std::byte[o.size_]; size_ = o.size_;
                          std::copy_n(o.data_, size_, data_); }
        return *this;
    }
    Buffer(Buffer&& o) noexcept    : data_(std::exchange(o.data_, nullptr)), size_(o.size_) {}
    Buffer& operator=(Buffer&& o) noexcept {
        if (this != &o) { delete[] data_; data_ = std::exchange(o.data_, nullptr); size_ = o.size_; }
        return *this;
    }
};
```

### Perfect Forwarding

```cpp
// Forward arguments preserving their value category (lvalue/rvalue)
template<typename T, typename... Args>
std::unique_ptr<T> make(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// std::move: cast to rvalue reference (doesn't actually move)
std::string a = "hello";
std::string b = std::move(a);  // a is now in "valid but unspecified state"
// Don't use 'a' after move without reassigning it
```

---

## Templates

### Function and Class Templates

```cpp
template<typename T>
T clamp(T val, T lo, T hi) {
    return val < lo ? lo : val > hi ? hi : val;
}

// Variadic templates
template<typename... Args>
void log(std::string_view fmt, Args&&... args) {
    std::cout << std::vformat(fmt, std::make_format_args(args...)) << '\n';
}

// Fold expressions (C++17)
template<typename... Ts>
auto sum(Ts... vals) { return (vals + ...); }   // unary right fold
auto s = sum(1, 2, 3, 4);  // 10
```

### Concepts (C++20) — Replaces SFINAE

```cpp
// C++20 concepts — expressive constraints
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<typename T>
concept Printable = requires(T t, std::ostream& os) {
    { os << t } -> std::convertible_to<std::ostream&>;
};

// Using concepts
template<Numeric T>
T square(T x) { return x * x; }

template<typename Container>
concept Sortable = requires(Container c) {
    std::begin(c); std::end(c);
    requires std::sortable<decltype(std::begin(c))>;
};

template<Sortable C>
void sort_container(C& c) { std::sort(std::begin(c), std::end(c)); }
```

---

## Modern Idioms (C++17)

### Structured Bindings

```cpp
auto [min, max] = std::minmax_element(v.begin(), v.end());

for (auto& [key, value] : config_map) {
    std::cout << key << " = " << value << '\n';
}

// Return multiple values cleanly
std::pair<bool, std::string> validate(std::string_view input) {
    if (input.empty()) return {false, "empty input"};
    return {true, ""};
}
auto [ok, error] = validate(input);
```

### `std::optional`, `std::variant`, `std::string_view`

```cpp
// optional — nullable value without heap allocation
std::optional<User> findUser(int id) {
    if (auto it = db.find(id); it != db.end())
        return it->second;
    return std::nullopt;
}

auto user = findUser(42);
if (user) process(*user);              // dereference if present
auto name = user.value_or("unknown"); // with default

// variant — type-safe union
using JsonValue = std::variant<std::nullptr_t, bool, int64_t, double, std::string>;

JsonValue val = 42;
std::visit(overloaded{
    [](std::nullptr_t)  { std::cout << "null\n"; },
    [](bool b)          { std::cout << std::boolalpha << b << '\n'; },
    [](int64_t i)       { std::cout << i << '\n'; },
    [](double d)        { std::cout << d << '\n'; },
    [](const std::string& s) { std::cout << '"' << s << '"' << '\n'; },
}, val);

// string_view — non-owning string reference (zero copy)
void process(std::string_view sv) {
    auto trimmed = sv.substr(sv.find_first_not_of(' ')); // no allocation
}
process("literal");          // no std::string construction
process(std::string("heap")); // views into string, no copy
```

### `if constexpr`

```cpp
template<typename T>
std::string describe(T val) {
    if constexpr (std::is_integral_v<T>)
        return "integer: " + std::to_string(val);
    else if constexpr (std::is_floating_point_v<T>)
        return "float: " + std::to_string(val);
    else
        return "other";
    // Dead branches are not compiled — no linker errors
}
```

---

## Concurrency (C++11/17/20)

```cpp
// Basic thread
std::thread t([]{ doWork(); });
t.join(); // or t.detach() — but prefer join (safer)

// std::jthread (C++20) — automatically joins, supports cancellation
{
    std::jthread worker([](std::stop_token st) {
        while (!st.stop_requested()) {
            doWork();
        }
    });
} // jthread destructor requests stop and joins

// Mutex + RAII lock
std::mutex mtx;
std::shared_mutex rwMtx; // readers/writer lock

{
    std::lock_guard<std::mutex> lock(mtx);    // exclusive
    // critical section
}

{
    std::shared_lock<std::shared_mutex> rlock(rwMtx); // multiple readers OK
    read_data();
}

{
    std::unique_lock<std::shared_mutex> wlock(rwMtx); // exclusive writer
    write_data();
}

// Scoped lock — multiple mutexes deadlock-free
std::scoped_lock lock(mtx1, mtx2); // acquires both atomically

// Atomic — for simple shared scalars
std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed); // no sync needed for count
counter.store(0, std::memory_order_release);
int val = counter.load(std::memory_order_acquire);

// Condition variable — for wait/notify
std::condition_variable cv;
bool ready = false;

// Producer:
{
    std::lock_guard<std::mutex> lk(mtx);
    ready = true;
}
cv.notify_one();

// Consumer:
std::unique_lock<std::mutex> lk(mtx);
cv.wait(lk, [] { return ready; }); // spurious wakeup safe
```

---

## Ranges (C++20)

```cpp
#include <ranges>

std::vector<int> v = {5, 3, 1, 4, 2};

// Lazy pipeline — no intermediate allocations
auto result = v
    | std::views::filter([](int x) { return x > 2; })
    | std::views::transform([](int x) { return x * x; })
    | std::views::take(3);

// Materialize only when needed
std::vector<int> out(result.begin(), result.end()); // {25, 9, 16}

// Range algorithms
std::ranges::sort(v);
auto it = std::ranges::find(v, 3);
bool found = std::ranges::contains(v, 3); // C++23, or use ranges::find
```

---

## Error Handling

```
Exceptions:      stack-unwinding, RAII cleanup, standard C++ mechanism
                 Cost: zero when not thrown (with modern compilers + -O2)
Error codes:     low overhead, explicit, composable — better for hot paths
std::expected:   C++23 — functional error handling without exceptions
```

```cpp
// Exception approach — for truly exceptional conditions
struct ParseError : std::runtime_error {
    explicit ParseError(std::string_view msg) : std::runtime_error(std::string(msg)) {}
};

Config parseConfig(std::string_view json) {
    if (json.empty()) throw ParseError("empty input");
    // ...
}

// Error code approach — for expected failures, hot paths
enum class ParseResult { Ok, EmptyInput, InvalidJson, UnknownKey };

ParseResult parseConfig(std::string_view json, Config& out) noexcept {
    if (json.empty()) return ParseResult::EmptyInput;
    // ...
    return ParseResult::Ok;
}

// std::expected (C++23) — functional, composable
std::expected<Config, ParseError> parseConfig(std::string_view json) {
    if (json.empty()) return std::unexpected(ParseError{"empty input"});
    // ...
    return Config{};
}

auto result = parseConfig(input)
    .transform([](Config c) { return c.validate(); })
    .value_or(Config::defaults());
```

---

## Undefined Behavior — Common Causes

```cpp
// 1. Signed integer overflow (use unsigned or check first)
int x = INT_MAX;
int y = x + 1;  // UB — signed overflow

// 2. Null pointer dereference
int* p = nullptr;
*p = 5;  // UB

// 3. Use-after-free
auto* p = new int(42);
delete p;
*p = 1;  // UB

// 4. Out-of-bounds access
int arr[5];
arr[5] = 0;   // UB — index 5 is past the end
arr[-1] = 0;  // UB

// 5. Data race (two threads, at least one write, no sync)
int shared = 0;
std::thread t1([&]{ shared++; });
std::thread t2([&]{ shared++; }); // UB — data race
// Fix: std::atomic<int> shared{0};

// 6. Strict aliasing violation
float f = 1.0f;
int* ip = reinterpret_cast<int*>(&f); // UB to read through ip
// Fix: use memcpy to type-pun

// Detection tools
// -fsanitize=address,undefined (clang/gcc)
// -fsanitize=thread (for data races)
// Valgrind: valgrind --tool=memcheck ./program
```

---

## Performance

**Always profile first** — never guess where the bottleneck is.

```bash
# Profiling
g++ -O2 -pg -o program main.cpp && ./program && gprof program gmon.out
perf record -g ./program && perf report

# Compiler flags for production
-O2 -march=native -DNDEBUG    # standard release
-O3 -march=native             # aggressive optimization (check for UB first)
```

**Cache-friendly patterns**:
```cpp
// BAD: array of pointers — pointer chasing, cache misses
std::vector<std::unique_ptr<Entity>> entities;  // each Entity on separate heap page

// GOOD: contiguous storage
std::vector<Entity> entities;  // all Entities contiguous in memory

// GOOD for heterogeneous: SoA layout for hot loop data
struct Particles {
    std::vector<float> x, y, z;    // hot: position
    std::vector<float> vx, vy, vz; // hot: velocity
    std::vector<std::string> names; // cold: metadata
};
```

**Branch prediction**:
```cpp
// Help the compiler with [[likely]] / [[unlikely]] (C++20)
if (ptr) [[likely]] {
    process(ptr);
} else [[unlikely]] {
    handleNull();
}
```

---

## CMake Modern Patterns

```cmake
cmake_minimum_required(VERSION 3.20)
project(myapp CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Targets — not global variables
add_library(core STATIC src/core.cpp)
target_include_directories(core PUBLIC include)
target_compile_options(core PRIVATE -Wall -Wextra -Wpedantic)

add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE core)

# Find packages
find_package(fmt REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)

# Sanitizers for debug builds
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(myapp PRIVATE -fsanitize=address,undefined)
    target_link_options(myapp PRIVATE -fsanitize=address,undefined)
endif()
```

```bash
# vcpkg — C++ package manager
vcpkg install fmt spdlog catch2

# conan — alternative package manager
conan install . --build=missing -s build_type=Release
```

---

## Anti-Patterns

| Anti-pattern                              | Modern alternative                              |
|-------------------------------------------|-------------------------------------------------|
| `new` / `delete` manually                 | `std::make_unique<T>()`, `std::vector<T>`       |
| `char*` C-strings                         | `std::string`, `std::string_view`               |
| `void*` type erasure                      | `std::any`, `std::variant`, templates           |
| `printf` family                           | `std::format` (C++20) or `{fmt}` library        |
| `reinterpret_cast` for type punning       | `std::memcpy`, `std::bit_cast` (C++20)          |
| Raw arrays `int arr[N]`                   | `std::array<int, N>` or `std::vector<int>`      |
| `using namespace std;` in headers         | Fully qualified names in headers                |
| `#define` constants                       | `constexpr` or `enum class`                     |
| C-style casts `(int)x`                   | `static_cast<int>(x)` — explicit and searchable |
| Returning multiple values via output params| Structured bindings + `std::pair`/`std::tuple`  |
| `std::endl` in loops                      | `'\n'` — endl flushes, '\n' doesn't             |

---

## Checklist

- [ ] RAII used for all resources — no manual `delete`, `fclose`, `unlock`
- [ ] `std::unique_ptr` is the default heap allocation choice
- [ ] `std::shared_ptr` used only when ownership is genuinely shared
- [ ] Rule of Zero applied — let member types manage themselves
- [ ] No raw `new`/`delete` in application code
- [ ] `std::string_view` for read-only string parameters
- [ ] `noexcept` applied to move constructors and move assignment operators
- [ ] No signed integer overflow (use `unsigned` or check bounds)
- [ ] No use-after-move (set moved-from to known state or don't use)
- [ ] Sanitizers enabled in debug/CI builds (`-fsanitize=address,undefined,thread`)
- [ ] Concepts (C++20) used instead of SFINAE for template constraints
- [ ] `std::scoped_lock` for multi-mutex acquisition
- [ ] CMake uses target-based commands (`target_link_libraries`, not `link_libraries`)
- [ ] Profiling done before any optimization claims
