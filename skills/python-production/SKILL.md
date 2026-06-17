---
name: python-production
description: Expert production Python — type hints, Pydantic v2, async/await, packaging with uv, testing with pytest, performance, and common pitfalls
---

# Production Python

You are an expert production Python engineer. Apply type hints, Pydantic v2 for validation, pytest for testing, and idiomatic Python patterns. Prefer explicit, well-typed, testable code over quick scripts.

---

## Type Hints

Type hints make code self-documenting, enable static analysis (mypy/pyright), and catch bugs before runtime.

```python
from typing import TypeVar, Generic, Protocol, runtime_checkable
from collections.abc import Sequence, Callable, Iterator

# Basic annotations
def greet(name: str, times: int = 1) -> str:
    return (f"Hello, {name}!\n" * times).strip()

# Generic functions
T = TypeVar("T")

def first(seq: Sequence[T]) -> T | None:
    return seq[0] if seq else None

# Protocol — structural typing (duck typing with type safety)
@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

def safe_close(resource: Closeable) -> None:
    resource.close()
# Works with any class that has .close() — no inheritance needed

# Generic classes
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()

# Union types (Python 3.10+)
def parse_id(raw: str | int) -> int:
    return int(raw)

# Type aliases
UserId = int
EmailAddress = str
UserMap = dict[UserId, EmailAddress]
```

### mypy Configuration

```toml
# pyproject.toml
[tool.mypy]
strict = true              # enables all optional checks
python_version = "3.12"
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true

# Per-module overrides for third-party stubs
[[tool.mypy.overrides]]
module = ["boto3.*", "botocore.*"]
ignore_missing_imports = true
```

---

## Pydantic v2

Pydantic v2 is the standard for data validation, serialization, and settings management.

```python
from pydantic import BaseModel, Field, field_validator, model_validator, EmailStr
from pydantic import ConfigDict
from datetime import datetime
from decimal import Decimal

class User(BaseModel):
    model_config = ConfigDict(
        frozen=True,              # immutable instances
        str_strip_whitespace=True,
        validate_assignment=True,
    )

    id: int
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    created_at: datetime = Field(default_factory=datetime.utcnow)
    balance: Decimal = Field(default=Decimal("0"), ge=0)

    @field_validator("name")
    @classmethod
    def name_must_be_title_case(cls, v: str) -> str:
        return v.title()

    @model_validator(mode="after")
    def check_email_domain(self) -> "User":
        if "@internal.company.com" not in self.email:
            raise ValueError("must use company email")
        return self

# Usage
user = User(id=1, name="  alice smith  ", email="alice@internal.company.com")
print(user.name)  # "Alice Smith" — stripped and title-cased

# Serialization
user.model_dump()                    # dict
user.model_dump(mode="json")         # JSON-serializable dict
user.model_dump_json()               # JSON string
User.model_validate({"id": 1, ...}) # from dict
User.model_validate_json(json_str)  # from JSON string
```

### Settings Management

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    database_url: str
    redis_url: str = "redis://localhost:6379"
    debug: bool = False
    max_connections: int = Field(default=10, ge=1, le=100)
    secret_key: str = Field(min_length=32)

# Reads from environment variables (DATABASE_URL, REDIS_URL, etc.)
settings = Settings()
```

---

## Async Python

```
Event loop model:
  Single thread runs the event loop
  I/O operations yield control back to the loop
  Other coroutines run while I/O is pending

  ┌──────────────────────────────────────────────────────────┐
  │  Event Loop                                              │
  │    [Coroutine A: await db.query()] ─── suspended        │
  │    [Coroutine B: await http.get()] ─── suspended        │
  │    [Coroutine C: computing]        ─── RUNNING          │
  │    ↑ when I/O completes, A or B becomes runnable        │
  └──────────────────────────────────────────────────────────┘
```

```python
import asyncio
import aiohttp

async def fetch(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url) as response:
        return await response.text()

async def fetch_all(urls: list[str]) -> list[str]:
    async with aiohttp.ClientSession() as session:
        # Concurrent — all requests in flight simultaneously
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# Entry point
async def main() -> None:
    urls = ["https://api.example.com/1", "https://api.example.com/2"]
    results = await fetch_all(urls)
    for r in results:
        print(r[:100])

asyncio.run(main())

# Running sync blocking code without blocking the event loop
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=4)

async def read_file_async(path: str) -> bytes:
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(executor, lambda: open(path, "rb").read())

# Or use asyncio built-ins
import aiofiles

async def read_file(path: str) -> str:
    async with aiofiles.open(path) as f:
        return await f.read()
```

**Async pitfalls**:
```python
# WRONG: blocking call in async function
async def bad() -> None:
    import time
    time.sleep(1)       # blocks the entire event loop!
    requests.get(url)   # blocking HTTP call

# RIGHT
async def good() -> None:
    await asyncio.sleep(1)
    async with aiohttp.ClientSession() as s:
        await s.get(url)

# WRONG: fire and forget without tracking
asyncio.create_task(risky_operation())  # unhandled exceptions silently swallowed

# RIGHT: collect and handle
task = asyncio.create_task(risky_operation())
task.add_done_callback(lambda t: t.exception() and log.error(t.exception()))
```

---

## Python Packaging

```toml
# pyproject.toml — modern standard (PEP 517/518/621)
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypackage"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = [
    "pydantic>=2.0",
    "aiohttp>=3.9",
    "structlog>=24.0",
]

[project.optional-dependencies]
dev = ["pytest>=8", "mypy>=1.8", "ruff>=0.3"]
test = ["pytest>=8", "pytest-asyncio>=0.23", "pytest-cov>=5"]

[project.scripts]
myapp = "mypackage.cli:main"
```

```bash
# uv — modern Python package manager (10-100x faster than pip)
uv init myproject           # create new project
uv add pydantic aiohttp     # add dependency
uv add --dev pytest mypy    # add dev dependency
uv run pytest               # run in virtualenv
uv sync                     # sync dependencies from lockfile
uv lock                     # update lockfile

# Virtual environments
uv venv .venv               # create
source .venv/bin/activate   # activate (Linux/Mac)
```

---

## Error Handling

```python
# Custom exception hierarchy
class AppError(Exception):
    """Base exception for application errors."""

class ValidationError(AppError):
    def __init__(self, field: str, message: str) -> None:
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    def __init__(self, resource: str, id: int | str) -> None:
        super().__init__(f"{resource} with id={id!r} not found")

# contextlib for resource management
from contextlib import contextmanager, suppress

@contextmanager
def timer(label: str) -> Iterator[None]:
    import time
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.3f}s")

with timer("data processing"):
    process_data()

# suppress — ignore specific exceptions cleanly
with suppress(FileNotFoundError):
    os.remove(temp_file)  # don't crash if already gone

# Exception chaining
try:
    result = json.loads(raw_data)
except json.JSONDecodeError as e:
    raise ValidationError("body", "invalid JSON") from e
```

---

## Performance

**Profile before optimizing**.

```bash
# cProfile — function-level profiling
python -m cProfile -o output.prof myscript.py
python -m pstats output.prof

# line_profiler — line-by-line in hot functions
pip install line_profiler
# Decorate with @profile, then:
kernprof -l -v myscript.py

# memory_profiler
pip install memory_profiler
python -m memory_profiler myscript.py
```

### When to Use Alternatives

```python
# NumPy — vectorized numerical operations (C speed)
import numpy as np
arr = np.array([1.0, 2.0, 3.0])
result = np.sqrt(arr) * 2 + np.sin(arr)  # no Python loop

# Cython — annotate Python with types, compile to C
# cdef int add(int a, int b): return a + b  # in .pyx file

# ctypes — call existing C libraries
import ctypes
lib = ctypes.CDLL("./fast_lib.so")
lib.process.argtypes = [ctypes.c_int, ctypes.POINTER(ctypes.c_double)]
lib.process.restype = ctypes.c_double

# PyO3/Maturin — write Rust extensions for Python
# cargo new --lib myext && add pyo3 = {features=["extension-module"]}

# __slots__ — reduce memory per instance
class Point:
    __slots__ = ("x", "y")  # no __dict__ per instance
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
# 40-50% less memory for large numbers of instances
```

---

## Concurrency Model

```
GIL (Global Interpreter Lock):
  Only one Python thread executes bytecode at a time
  I/O releases the GIL — threading works fine for I/O-bound work
  CPU-bound: threading doesn't help — use multiprocessing or asyncio

  I/O-bound, concurrent:    asyncio (preferred), threading
  CPU-bound, parallel:      multiprocessing, concurrent.futures
  Mixed:                    asyncio + ThreadPoolExecutor for I/O,
                            ProcessPoolExecutor for CPU
```

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed

# I/O-bound: thread pool (GIL released during I/O)
with ThreadPoolExecutor(max_workers=10) as pool:
    futures = {pool.submit(fetch_url, url): url for url in urls}
    for future in as_completed(futures):
        url = futures[future]
        try:
            result = future.result()
        except Exception as e:
            print(f"Failed {url}: {e}")

# CPU-bound: process pool (no GIL contention)
with ProcessPoolExecutor() as pool:
    results = list(pool.map(heavy_computation, data_chunks))
```

---

## Testing

```python
# conftest.py — shared fixtures
import pytest
from collections.abc import Generator

@pytest.fixture(scope="session")
def db_engine() -> Generator[Engine, None, None]:
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture
def db_session(db_engine: Engine) -> Generator[Session, None, None]:
    with Session(db_engine) as session:
        yield session
        session.rollback()  # clean up after each test

# Parameterized tests
@pytest.mark.parametrize("input,expected", [
    ("hello world", "Hello World"),
    ("  alice  ", "Alice"),
    ("", ""),
])
def test_title_case(input: str, expected: str) -> None:
    assert title_case(input) == expected

# Async tests
@pytest.mark.asyncio
async def test_fetch_user() -> None:
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/1")
    assert response.status_code == 200
    assert response.json()["name"] == "Alice"

# Hypothesis — property-based testing
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_idempotent(lst: list[int]) -> None:
    sorted_once = sorted(lst)
    sorted_twice = sorted(sorted_once)
    assert sorted_once == sorted_twice

@given(st.text(min_size=1))
def test_title_case_never_empty(s: str) -> None:
    result = title_case(s)
    assert len(result) > 0
```

---

## Design Patterns in Python

```python
# Dataclasses — structured data with less boilerplate
from dataclasses import dataclass, field

@dataclass
class Config:
    host: str
    port: int = 8080
    tags: list[str] = field(default_factory=list)
    debug: bool = False

    def __post_init__(self) -> None:
        if not 0 < self.port < 65536:
            raise ValueError(f"Invalid port: {self.port}")

# ABC — abstract base classes for interfaces
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> None: ...

    @abstractmethod
    async def send_async(self, message: str, recipient: str) -> None: ...

class EmailNotifier(Notifier):
    def send(self, message: str, recipient: str) -> None:
        send_email(recipient, message)

    async def send_async(self, message: str, recipient: str) -> None:
        await async_send_email(recipient, message)
```

---

## Common Pitfalls

```python
# 1. Mutable default arguments — classic Python gotcha
# WRONG
def append_to(item, lst=[]):  # lst is created ONCE at function definition
    lst.append(item)
    return lst

append_to(1)  # [1]
append_to(2)  # [1, 2] — unexpected!

# RIGHT
def append_to(item: int, lst: list[int] | None = None) -> list[int]:
    if lst is None:
        lst = []
    lst.append(item)
    return lst

# 2. Late binding closures
# WRONG
funcs = [lambda: i for i in range(3)]
funcs[0]()  # 2, not 0 — all capture the same 'i'

# RIGHT
funcs = [lambda i=i: i for i in range(3)]
funcs[0]()  # 0

# 3. Circular imports — restructure to break cycles
# module_a.py imports from module_b.py, module_b.py imports from module_a.py
# Fix: extract shared types to a third module, or use TYPE_CHECKING guard
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from module_b import TypeFromB  # only imported for type checking, not at runtime

# 4. Global mutable state
# WRONG
_cache: dict[str, Any] = {}  # module-level mutable state

# RIGHT: encapsulate in a class, inject it
class Cache:
    def __init__(self) -> None:
        self._store: dict[str, Any] = {}
    def get(self, key: str) -> Any | None: ...
    def set(self, key: str, value: Any) -> None: ...

# 5. Not closing resources
# WRONG
f = open("file.txt")
data = f.read()  # if exception here, file never closed

# RIGHT
with open("file.txt") as f:
    data = f.read()
```

---

## Essential Tooling

```bash
# Linting and formatting
ruff check .          # fast linter (replaces flake8, pylint, isort)
ruff format .         # fast formatter (replaces black)

# Type checking
mypy src/             # static type checking
pyright src/          # Microsoft's type checker (faster, stricter)

# Testing
pytest -x -v          # stop on first failure, verbose
pytest --cov=src      # with coverage
pytest -k "test_user" # run matching tests only

# Security
pip-audit             # check for known vulnerabilities in dependencies
bandit -r src/        # security linter

# Pre-commit hooks
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    hooks: [{id: ruff}, {id: ruff-format}]
  - repo: https://github.com/pre-commit/mirrors-mypy
    hooks: [{id: mypy}]
```

---

## Checklist

- [ ] All function signatures have type annotations (inputs and return)
- [ ] mypy (or pyright) passes with strict mode
- [ ] Pydantic v2 used for all external data validation
- [ ] No mutable default arguments (`= []`, `= {}`, `= set()`)
- [ ] All resources opened with `with` statement (context manager)
- [ ] Async functions use `await`-compatible I/O (aiohttp, aiofiles, asyncpg)
- [ ] No blocking calls (`time.sleep`, `requests.get`) inside async functions
- [ ] CPU-bound work uses `ProcessPoolExecutor` or C extensions
- [ ] Custom exceptions inherit from a project base exception class
- [ ] pytest fixtures used for setup/teardown, not `setUp`/`tearDown`
- [ ] `hypothesis` used for property-based testing of data processing functions
- [ ] `pyproject.toml` used (not `setup.py`)
- [ ] `uv` used for dependency management and virtual environments
- [ ] `ruff` for linting and formatting (single tool, replaces flake8 + black + isort)
- [ ] No bare `except:` — always catch specific exception types
- [ ] `__slots__` considered for classes instantiated millions of times
