---
name: fastapi-production
description: Expert FastAPI production guidance — async patterns, dependency injection, SQLAlchemy 2.0, auth, testing, middleware, and deployment
---

# FastAPI Production Expert

You are an expert in building production-grade FastAPI applications. Apply async-first patterns, Pydantic v2 validation, and production-hardened deployment practices.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                     FastAPI Application                        │
│                                                                │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌────────────┐  │
│  │Middleware│  │  Router   │  │  Deps    │  │ Background │  │
│  │          │  │           │  │          │  │  Tasks     │  │
│  │ CORS     │  │ /v1/users │  │ get_db() │  │ Celery     │  │
│  │ ReqID    │  │ /v1/items │  │ get_user │  │ APScheduler│  │
│  │ Logging  │  │ /v1/auth  │  │ auth_dep │  │            │  │
│  └──────────┘  └───────────┘  └──────────┘  └────────────┘  │
│        │              │              │                         │
│        ▼              ▼              ▼                         │
│  ┌─────────────────────────────────────────┐                  │
│  │           Route Handlers                │                  │
│  │   async def endpoint(                   │                  │
│  │       data: RequestModel,               │                  │
│  │       db: AsyncSession = Depends(...),  │                  │
│  │       user: User = Depends(auth),       │                  │
│  │   ) -> ResponseModel                    │                  │
│  └─────────────────────────────────────────┘                  │
│        │                                                       │
│        ▼                                                       │
│  ┌─────────────────┐   ┌─────────────────────────────────┐   │
│  │  Service Layer  │   │   Repository / SQLAlchemy 2.0   │   │
│  │  (business      │──>│   AsyncSession + Alembic        │   │
│  │   logic)        │   │   migrations                    │   │
│  └─────────────────┘   └─────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

---

## 1. FastAPI Fundamentals

### Path Operations with Pydantic v2

```python
from fastapi import FastAPI, HTTPException, Query, Path
from pydantic import BaseModel, Field, EmailStr, field_validator, model_validator
from typing import Annotated
from datetime import datetime
import uuid

app = FastAPI(
    title="My API",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
)

# --- Request Models ---
class CreateUserRequest(BaseModel):
    email: EmailStr
    first_name: str = Field(..., min_length=1, max_length=100)
    last_name: str = Field(..., min_length=1, max_length=100)
    age: int = Field(..., ge=0, le=150)
    role: Literal["admin", "user", "viewer"] = "user"

    @field_validator("email")
    @classmethod
    def email_must_be_corporate(cls, v: str) -> str:
        if v.endswith("@example.com"):
            raise ValueError("Corporate emails not allowed in this context")
        return v.lower()

    @model_validator(mode="after")
    def check_admin_age(self) -> "CreateUserRequest":
        if self.role == "admin" and self.age < 18:
            raise ValueError("Admins must be 18 or older")
        return self

# --- Response Models (separate from request) ---
class UserResponse(BaseModel):
    id: uuid.UUID
    email: str
    first_name: str
    last_name: str
    role: str
    created_at: datetime

    model_config = {"from_attributes": True}  # pydantic v2: replaces orm_mode

# --- Route with explicit response model ---
@app.post(
    "/v1/users",
    response_model=UserResponse,
    status_code=201,
    summary="Create a new user",
    tags=["users"],
)
async def create_user(
    body: CreateUserRequest,
    db: Annotated[AsyncSession, Depends(get_db)],
    current_user: Annotated[User, Depends(require_role("admin"))],
) -> UserResponse:
    user = await user_service.create(db, body, created_by=current_user.id)
    return UserResponse.model_validate(user)
```

### response_model vs response_model_exclude

```python
# response_model: validates output, filters extra fields, generates schema
@app.get("/users/{id}", response_model=UserResponse)

# response_model_exclude: when you need to return most fields but hide a few
@app.get("/users/{id}", response_model=UserResponse,
         response_model_exclude={"password_hash", "internal_notes"})

# response_model_include: only return specific fields
@app.get("/users/{id}/summary", response_model=UserResponse,
         response_model_include={"id", "email", "role"})

# return None to skip response validation (e.g., streaming responses)
@app.get("/users/{id}", response_model=None)
async def get_user_stream(id: uuid.UUID) -> StreamingResponse:
    ...
```

---

## 2. Async Endpoints

### When to use async def vs def

```
async def   → I/O bound work: database, HTTP calls, file I/O, Redis, message queue
              FastAPI runs this in the event loop — do NOT call blocking code here

def         → CPU bound or when using blocking libraries (sync ORM, blocking HTTP)
              FastAPI runs this in a thread pool executor automatically

RULE: If you use httpx.AsyncClient, asyncpg, aioredis, aioboto3 → async def
      If you use requests, psycopg2, SQLAlchemy sync session    → def (thread pool)
      Never call time.sleep() in async def — use await asyncio.sleep()
```

```python
import asyncio
import httpx
from fastapi import FastAPI

app = FastAPI()

# CORRECT: async I/O
@app.get("/weather")
async def get_weather(city: str) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.weather.com/{city}")
        resp.raise_for_status()
        return resp.json()

# CORRECT: CPU-bound stays as def (runs in thread pool)
@app.post("/compute")
def heavy_computation(data: list[float]) -> dict:
    import numpy as np
    result = np.fft.fft(data)
    return {"result": result.tolist()}

# WRONG: blocking call in async context → starves the event loop
@app.get("/bad")
async def bad_endpoint():
    import time
    time.sleep(2)   # NEVER do this — use await asyncio.sleep(2)
    return {"status": "done"}
```

### SQLAlchemy Async Sessions

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql+asyncpg://user:pass@host/db"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,         # validate connection before use
    pool_recycle=3600,          # recycle connections every hour
    echo=False,                 # set to True for SQL debugging
)

AsyncSessionFactory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,     # avoid lazy-loading issues after commit
    autoflush=False,
    autocommit=False,
)

class Base(DeclarativeBase):
    pass

# --- Async query patterns ---
from sqlalchemy import select, update, delete, func

async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    result = await db.execute(stmt)
    return result.scalar_one_or_none()

async def get_users_paginated(
    db: AsyncSession, page: int, size: int
) -> tuple[list[User], int]:
    # count and data in parallel
    count_stmt = select(func.count()).select_from(User).where(User.active == True)
    data_stmt = (
        select(User)
        .where(User.active == True)
        .order_by(User.created_at.desc())
        .offset(page * size)
        .limit(size)
    )
    count_result, data_result = await asyncio.gather(
        db.execute(count_stmt),
        db.execute(data_stmt),
    )
    return data_result.scalars().all(), count_result.scalar()

async def update_user_status(db: AsyncSession, user_id: uuid.UUID, active: bool) -> int:
    stmt = (
        update(User)
        .where(User.id == user_id)
        .values(active=active, updated_at=datetime.utcnow())
        .returning(User.id)
    )
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount
```

---

## 3. Dependency Injection

### Database Session Dependency

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionFactory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# Type alias for cleaner signatures
DBSession = Annotated[AsyncSession, Depends(get_db)]
```

### Authentication Dependencies

```python
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt

security = HTTPBearer()

async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(security)],
    db: DBSession,
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

    user = await user_repo.get_by_id(db, uuid.UUID(user_id))
    if user is None or not user.active:
        raise HTTPException(status_code=401, detail="User not found or inactive")
    return user

CurrentUser = Annotated[User, Depends(get_current_user)]

def require_role(*roles: str):
    """Factory for role-based dependencies."""
    async def check_role(user: CurrentUser) -> User:
        if user.role not in roles:
            raise HTTPException(
                status_code=403,
                detail=f"Required role: {roles}, user role: {user.role}"
            )
        return user
    return check_role

AdminUser = Annotated[User, Depends(require_role("admin"))]

# Usage
@app.delete("/v1/users/{id}")
async def delete_user(
    id: uuid.UUID,
    db: DBSession,
    _: AdminUser,       # just enforce role, don't need user data
) -> None:
    await user_service.delete(db, id)
```

### Layered Dependencies

```python
# Layered: Settings → DB → Repository → Service
# Each layer testable in isolation

@lru_cache
def get_settings() -> Settings:
    return Settings()

async def get_db(settings: Annotated[Settings, Depends(get_settings)]):
    ...

def get_user_repo(db: DBSession) -> UserRepository:
    return UserRepository(db)

def get_user_service(
    repo: Annotated[UserRepository, Depends(get_user_repo)],
    cache: Annotated[RedisCache, Depends(get_cache)],
) -> UserService:
    return UserService(repo, cache)
```

---

## 4. Background Tasks

### FastAPI BackgroundTasks (simple, same process)

```python
from fastapi import BackgroundTasks

async def send_welcome_email(email: str, name: str) -> None:
    """Runs after response is sent — do not raise exceptions here."""
    await email_client.send(
        to=email,
        subject="Welcome!",
        template="welcome",
        context={"name": name},
    )

@app.post("/v1/users", response_model=UserResponse, status_code=201)
async def create_user(
    body: CreateUserRequest,
    background_tasks: BackgroundTasks,
    db: DBSession,
) -> UserResponse:
    user = await user_service.create(db, body)
    background_tasks.add_task(send_welcome_email, user.email, user.first_name)
    return UserResponse.model_validate(user)
```

### Celery Integration (distributed, durable)

```python
# celery_app.py
from celery import Celery

celery_app = Celery(
    "worker",
    broker="redis://redis:6379/0",
    backend="redis://redis:6379/1",
    include=["app.tasks.email", "app.tasks.reports"],
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,            # ack after task completes, not before
    worker_prefetch_multiplier=1,   # prevent one worker hoarding all tasks
)

# tasks/email.py
@celery_app.task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
    retry_jitter=True,
)
def send_email_task(self, to: str, subject: str, template: str, context: dict) -> None:
    try:
        email_client.send(to=to, subject=subject, template=template, context=context)
    except EmailProviderError as e:
        raise self.retry(exc=e)

# In FastAPI endpoint
@app.post("/v1/users", status_code=201)
async def create_user(body: CreateUserRequest, db: DBSession) -> UserResponse:
    user = await user_service.create(db, body)
    send_email_task.delay(user.email, "Welcome!", "welcome", {"name": user.first_name})
    return UserResponse.model_validate(user)
```

---

## 5. Authentication

### OAuth2 with JWT

```python
from datetime import timedelta
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from passlib.context import CryptContext
from jose import jwt

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/v1/auth/token")

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int

def create_access_token(subject: str, additional_claims: dict = {}) -> str:
    expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = {
        "sub": subject,
        "exp": expire,
        "iat": datetime.utcnow(),
        "jti": str(uuid.uuid4()),   # unique token ID for revocation
        **additional_claims,
    }
    return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")

@app.post("/v1/auth/token", response_model=TokenResponse)
async def login(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
    db: DBSession,
) -> TokenResponse:
    user = await user_repo.get_by_email(db, form_data.username)
    if not user or not pwd_context.verify(form_data.password, user.password_hash):
        raise HTTPException(
            status_code=401,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    if not user.active:
        raise HTTPException(status_code=403, detail="Account disabled")

    access_token = create_access_token(str(user.id), {"role": user.role})
    refresh_token = create_refresh_token(str(user.id))
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        expires_in=settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60,
    )
```

### API Key Authentication

```python
from fastapi import Security
from fastapi.security.api_key import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def verify_api_key(
    api_key: Annotated[str | None, Security(api_key_header)],
    db: DBSession,
) -> ApiKey:
    if not api_key:
        raise HTTPException(status_code=401, detail="API key required")

    key_hash = hashlib.sha256(api_key.encode()).hexdigest()
    key_record = await api_key_repo.get_by_hash(db, key_hash)

    if not key_record or not key_record.active:
        raise HTTPException(status_code=401, detail="Invalid or revoked API key")
    if key_record.expires_at and key_record.expires_at < datetime.utcnow():
        raise HTTPException(status_code=401, detail="API key expired")

    await api_key_repo.update_last_used(db, key_record.id)
    return key_record
```

---

## 6. Database Patterns

### Repository Pattern

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update

class UserRepository:
    def __init__(self, db: AsyncSession) -> None:
        self.db = db

    async def get_by_id(self, user_id: uuid.UUID) -> User | None:
        return await self.db.get(User, user_id)

    async def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def list_paginated(
        self, *, page: int = 0, size: int = 20, active_only: bool = True
    ) -> tuple[list[User], int]:
        base = select(User)
        if active_only:
            base = base.where(User.active == True)

        total_stmt = select(func.count()).select_from(base.subquery())
        data_stmt = base.order_by(User.created_at.desc()).offset(page * size).limit(size)

        total, rows = await asyncio.gather(
            self.db.scalar(total_stmt),
            self.db.scalars(data_stmt),
        )
        return list(rows.all()), total or 0

    async def create(self, data: dict) -> User:
        user = User(**data)
        self.db.add(user)
        await self.db.flush()  # get generated ID without committing
        await self.db.refresh(user)
        return user

    async def update(self, user_id: uuid.UUID, data: dict) -> User | None:
        stmt = (
            update(User)
            .where(User.id == user_id)
            .values(**data, updated_at=datetime.utcnow())
            .returning(User)
        )
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()
```

### Alembic Migrations

```bash
# Initialize
alembic init alembic

# Create migration
alembic revision --autogenerate -m "add_users_table"

# Apply migrations
alembic upgrade head

# Rollback one step
alembic downgrade -1

# Show history
alembic history --verbose
```

```python
# alembic/env.py — async support
from sqlalchemy.ext.asyncio import create_async_engine

def run_migrations_online():
    connectable = create_async_engine(settings.DATABASE_URL)

    async def run_async_migrations():
        async with connectable.connect() as connection:
            await connection.run_sync(do_run_migrations)

    asyncio.run(run_async_migrations())
```

---

## 7. Testing

### TestClient for sync tests

```python
from fastapi.testclient import TestClient

def test_create_user_returns_201(client: TestClient, admin_token: str):
    resp = client.post(
        "/v1/users",
        json={"email": "test@example.com", "first_name": "Test", "last_name": "User", "age": 25},
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert resp.status_code == 201
    data = resp.json()
    assert data["email"] == "test@example.com"
    assert "id" in data
```

### pytest-asyncio + httpx AsyncClient

```python
# conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

@pytest_asyncio.fixture
async def db_session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async_session = async_sessionmaker(engine, expire_on_commit=False)
    async with async_session() as session:
        yield session
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest_asyncio.fixture
async def client(db_session):
    app.dependency_overrides[get_db] = lambda: db_session
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
    app.dependency_overrides.clear()

# test_users.py
@pytest.mark.asyncio
async def test_get_user_returns_404_when_not_found(client: AsyncClient):
    resp = await client.get("/v1/users/00000000-0000-0000-0000-000000000000")
    assert resp.status_code == 404
    assert resp.json()["detail"] == "User not found"

@pytest.mark.asyncio
async def test_list_users_pagination(client: AsyncClient, admin_token: str):
    resp = await client.get(
        "/v1/users?page=0&size=5",
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert resp.status_code == 200
    data = resp.json()
    assert "items" in data
    assert "total" in data
    assert len(data["items"]) <= 5
```

---

## 8. Error Handling

### HTTPException and Custom Exception Handlers

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError

# Domain exceptions
class ResourceNotFoundError(Exception):
    def __init__(self, resource: str, id: Any) -> None:
        self.resource = resource
        self.id = id

class ConflictError(Exception):
    def __init__(self, message: str) -> None:
        self.message = message

# Register handlers
@app.exception_handler(ResourceNotFoundError)
async def not_found_handler(request: Request, exc: ResourceNotFoundError) -> JSONResponse:
    return JSONResponse(
        status_code=404,
        content={"detail": f"{exc.resource} with id '{exc.id}' not found"},
    )

@app.exception_handler(ConflictError)
async def conflict_handler(request: Request, exc: ConflictError) -> JSONResponse:
    return JSONResponse(status_code=409, content={"detail": exc.message})

# Pydantic v2 validation errors → consistent error format
@app.exception_handler(RequestValidationError)
async def validation_error_handler(
    request: Request, exc: RequestValidationError
) -> JSONResponse:
    errors = [
        {
            "field": ".".join(str(loc) for loc in error["loc"][1:]),  # skip "body"
            "message": error["msg"],
            "type": error["type"],
        }
        for error in exc.errors()
    ]
    return JSONResponse(status_code=422, content={"detail": "Validation error", "errors": errors})

# Catch-all for unexpected errors (never leak stack traces)
@app.exception_handler(Exception)
async def generic_error_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.error("Unhandled exception", exc_info=exc, extra={"path": str(request.url)})
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"},
    )
```

---

## 9. Middleware

### Production Middleware Stack

```python
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
import time
import uuid

# Order matters: outermost middleware runs first on request, last on response

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,   # never use ["*"] in production
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
)

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=settings.ALLOWED_HOSTS,
)

@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    request.state.request_id = request_id
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    start_time = time.monotonic()
    response = await call_next(request)
    duration_ms = (time.monotonic() - start_time) * 1000
    logger.info(
        "http_request",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status_code": response.status_code,
            "duration_ms": round(duration_ms, 2),
            "request_id": getattr(request.state, "request_id", None),
        },
    )
    return response
```

### Rate Limiting (slowapi)

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address, default_limits=["100/minute"])
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/v1/auth/token")
@limiter.limit("10/minute")   # stricter limit on auth endpoints
async def login(request: Request, form_data: OAuth2PasswordRequestForm = Depends()):
    ...
```

---

## 10. Production Deployment

### Gunicorn + Uvicorn Workers

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["gunicorn", "app.main:app",
     "--worker-class", "uvicorn.workers.UvicornWorker",
     "--workers", "4",
     "--bind", "0.0.0.0:8000",
     "--timeout", "120",
     "--keep-alive", "5",
     "--access-logfile", "-",
     "--error-logfile", "-",
     "--log-level", "info"]
```

```
Worker count formula: (2 * CPU_count) + 1
4 CPU → 9 workers

For async FastAPI with async endpoints: 1-4 workers is often enough
(each worker handles many concurrent requests via async event loop)
```

### Health Endpoints

```python
from fastapi import status

@app.get("/health/live", status_code=200, include_in_schema=False)
async def liveness() -> dict:
    """Kubernetes liveness probe — is the process running?"""
    return {"status": "ok"}

@app.get("/health/ready", status_code=200, include_in_schema=False)
async def readiness(db: DBSession) -> dict:
    """Kubernetes readiness probe — can the app serve traffic?"""
    try:
        await db.execute(text("SELECT 1"))
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail=f"Database not ready: {e}",
        )
    return {"status": "ready", "db": "ok"}
```

### Graceful Shutdown

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("Starting application")
    await engine.connect()  # warm up connection pool
    await redis_client.ping()
    yield
    # Shutdown (runs when SIGTERM received)
    logger.info("Shutting down — draining connections")
    await engine.dispose()
    await redis_client.aclose()
    logger.info("Shutdown complete")

app = FastAPI(lifespan=lifespan)
```

---

## 11. API Design

### Versioning

```python
from fastapi import APIRouter

# Version prefix on all routers
v1_router = APIRouter(prefix="/v1")
v2_router = APIRouter(prefix="/v2")

v1_router.include_router(users_v1_router, prefix="/users", tags=["users"])
v2_router.include_router(users_v2_router, prefix="/users", tags=["users-v2"])

app.include_router(v1_router)
app.include_router(v2_router)
```

### OpenAPI Customization

```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    schema = get_openapi(
        title="My Production API",
        version="1.0.0",
        description="## Authentication\nUse Bearer token. See /v1/auth/token.",
        routes=app.routes,
    )
    # Add security scheme globally
    schema["components"]["securitySchemes"] = {
        "BearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}
    }
    schema["security"] = [{"BearerAuth": []}]
    app.openapi_schema = schema
    return schema

app.openapi = custom_openapi
```

---

## Checklist

### New FastAPI Service

- [ ] Pydantic v2 models with field validators for all inputs
- [ ] Separate request and response models (never expose DB models directly)
- [ ] `async def` for all I/O bound endpoints, `def` for CPU-bound
- [ ] `expire_on_commit=False` on async session factory
- [ ] Database session auto-commits on success, rolls back on exception
- [ ] Custom exception handlers registered (never expose stack traces)
- [ ] Request ID middleware for distributed tracing correlation
- [ ] Health liveness + readiness endpoints
- [ ] Rate limiting on auth endpoints
- [ ] CORS configured with explicit allowed origins (not `*`)

### Testing

- [ ] Dependency override for `get_db` using test session
- [ ] Async tests use `pytest-asyncio` + `httpx.AsyncClient`
- [ ] Each test cleans up its own data (or uses transactions rolled back)
- [ ] Auth tokens generated with test secrets, not production secrets

### Deployment

- [ ] Gunicorn + UvicornWorker, not bare uvicorn
- [ ] Non-root user in Docker image
- [ ] `lifespan` context manager for startup/shutdown
- [ ] Alembic migrations run before app starts (init container or entrypoint)
- [ ] Kubernetes liveness and readiness probes configured
- [ ] Secrets injected as environment variables, not baked into image
