---
name: database-design
description: Expert database design guidance — schema design, normalization, indexing strategy, zero-downtime migrations, transactions, and query optimization for production PostgreSQL systems.
---

# Database Design — Expert Reference

## Schema Design Principles

### Primary Keys

Every table gets a surrogate primary key. Natural keys become unique constraints.

```sql
-- GOOD: surrogate PK + natural key as unique constraint
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(320) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_users_email UNIQUE (email)
);

-- BAD: email as PK — it changes, joins are expensive, exposes PII in URLs
CREATE TABLE users (
    email       VARCHAR(320) PRIMARY KEY,
    ...
);
```

**Why surrogate keys:**
- Natural keys change (email changes, SSNs get reassigned)
- Stable FK references — no cascading updates
- Join performance — integer vs varchar comparison
- Hide internal identifiers from external URLs (use UUID or ULID for external-facing IDs)

### Naming Conventions

```sql
-- Tables: snake_case, plural
CREATE TABLE order_line_items ( ... );

-- Primary key: id (always)
id BIGSERIAL PRIMARY KEY

-- Foreign key: <referenced_table_singular>_id
user_id       BIGINT NOT NULL REFERENCES users(id),
product_id    BIGINT NOT NULL REFERENCES products(id),

-- Timestamps: past-tense, timezone-aware
created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
deleted_at    TIMESTAMPTZ,             -- nullable = soft delete

-- Boolean: is_ / has_ / can_ prefix
is_active     BOOLEAN NOT NULL DEFAULT TRUE,
has_verified_email BOOLEAN NOT NULL DEFAULT FALSE,

-- Indexes: idx_<table>_<columns>
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Unique constraints: uq_<table>_<columns>
CONSTRAINT uq_users_email UNIQUE (email)
```

---

## Normalization

### Normal Forms

```
1NF: Each column holds atomic values; no repeating groups
     Violation: tags VARCHAR = "python,java,sql"
     Fix: tags table with FK

2NF: 1NF + every non-key attribute depends on the WHOLE primary key
     (Only relevant when PK is composite)
     Violation: order_items(order_id, product_id, product_name)
                product_name depends only on product_id
     Fix: move product_name to products table

3NF: 2NF + no transitive dependencies (non-key → non-key)
     Violation: employees(id, dept_id, dept_name)
                dept_name depends on dept_id, not on id
     Fix: move dept_name to departments table

BCNF: 3NF + every determinant is a candidate key
      (Edge case — rarely violated in practice)
```

### When to Denormalize Deliberately

Denormalize when normalization causes unacceptable query complexity or performance cost:

```sql
-- Normalized: requires JOIN for every order total read
SELECT o.id, SUM(li.quantity * li.unit_price) as total
FROM orders o
JOIN order_line_items li ON li.order_id = o.id
GROUP BY o.id;

-- Denormalized: store computed total when line items are immutable
ALTER TABLE orders ADD COLUMN total_amount NUMERIC(12,2) NOT NULL DEFAULT 0;
-- Maintain via trigger or application logic on line item insert/update
```

**Deliberate denormalization is justified when:**
- Read:Write ratio is very high (reporting tables)
- The denormalized value is expensive to recompute
- Data is immutable after creation (event tables, audit logs)
- Document this decision in a comment or ADR

---

## Soft Delete

```sql
-- Schema
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMPTZ;

-- Soft delete
UPDATE products SET deleted_at = NOW() WHERE id = 42;

-- Query active records (add to every query — easy to forget)
SELECT * FROM products WHERE deleted_at IS NULL;

-- Restore
UPDATE products SET deleted_at = NULL WHERE id = 42;
```

### Tradeoffs

| Concern | Impact |
|---|---|
| Query complexity | Every query needs `WHERE deleted_at IS NULL` |
| FK violations | FK references to soft-deleted rows still resolve |
| Index bloat | Deleted rows stay in all indexes |
| Unique constraints | Deleted email still blocks re-registration |
| GDPR/right to erasure | Soft-delete doesn't satisfy erasure requirements |

**Fix for unique constraint with soft delete:**

```sql
-- Allow same email if previously deleted
CREATE UNIQUE INDEX uq_users_active_email
  ON users(email)
  WHERE deleted_at IS NULL;
```

**Alternative: archive table pattern**

```sql
-- Move deleted rows to archive table, remove from main table
INSERT INTO users_archive SELECT *, NOW() as archived_at FROM users WHERE id = 42;
DELETE FROM users WHERE id = 42;
```

---

## Indexing Strategy

### Index Types (PostgreSQL)

```
B-Tree (default)
  ├── Equality: WHERE email = 'x@y.com'
  ├── Range: WHERE created_at > '2024-01-01'
  ├── ORDER BY, LIMIT
  └── LIKE 'prefix%' (but NOT LIKE '%suffix')

Hash
  ├── Equality only: WHERE id = 42
  └── Slightly faster than B-Tree for equality, no range support

GIN (Generalized Inverted Index)
  ├── JSONB columns: WHERE metadata @> '{"status": "active"}'
  ├── Full-text search: WHERE to_tsvector('english', body) @@ 'query'
  └── Array columns: WHERE tags @> ARRAY['python']

GiST
  └── Geometric types, range types, full-text (alternative to GIN)

BRIN (Block Range Index)
  └── Very large tables with natural physical ordering (time-series)
      Tiny index, trades precision for size
```

### Partial Index

```sql
-- Index only active users — much smaller, queries on active users are faster
CREATE INDEX idx_users_active_email
  ON users(email)
  WHERE deleted_at IS NULL;

-- Index only recent orders (last 90 days common query path)
CREATE INDEX idx_orders_recent
  ON orders(created_at DESC, user_id)
  WHERE created_at > NOW() - INTERVAL '90 days';
-- NOTE: static condition — use with partitioning for dynamic recency
```

### Composite Index

Column order is critical. Rule: **most selective first for equality predicates, then range/sort columns**.

```sql
-- Query pattern: WHERE status = 'active' AND created_at > '2024-01-01' ORDER BY created_at DESC
-- Good: equality column first, then range column
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- Bad: range column first — can't efficiently filter on status after range
CREATE INDEX idx_orders_created_status ON orders(created_at DESC, status);
```

```
Index column order guide:
1. Equality predicates (= , IN) — in order of selectivity, highest first
2. Range predicates (>, <, BETWEEN) — one range column max
3. ORDER BY columns — must match query sort direction
```

### Covering Index (Index-Only Scan)

Include all columns the query needs — avoids heap access entirely.

```sql
-- Query: SELECT user_id, status FROM orders WHERE order_date = '2024-01-15'
-- Covering index — includes user_id and status in the index
CREATE INDEX idx_orders_date_covering
  ON orders(order_date)
  INCLUDE (user_id, status);
-- PostgreSQL 11+ INCLUDE syntax — INCLUDE columns not used in WHERE, just returned
```

---

## N+1 Query Problem

**The problem:** Fetching a list, then querying for each item individually.

```python
# ANTI-PATTERN: N+1 in Python ORM (SQLAlchemy)
orders = session.query(Order).limit(100).all()  # 1 query
for order in orders:
    print(order.customer.name)  # N queries — one per order
```

**Fix 1: JOIN / eager load**

```python
# SQLAlchemy: joinedload
from sqlalchemy.orm import joinedload

orders = (
    session.query(Order)
    .options(joinedload(Order.customer))
    .limit(100)
    .all()
)
# 1 query with JOIN — customer loaded
```

**Fix 2: IN query (batch load)**

```sql
-- Fetch order IDs, then batch-load customers
SELECT * FROM customers WHERE id IN (1, 2, 3, ..., 100);
```

**Fix 3: DataLoader (GraphQL / async contexts)**

```python
# DataLoader batches individual loads within a single tick
from strawberry.dataloader import DataLoader

async def load_customers(customer_ids: list[int]) -> list[Customer]:
    return await Customer.filter(id__in=customer_ids)

customer_loader = DataLoader(load_fn=load_customers)

# Each field resolver calls customer_loader.load(order.customer_id)
# DataLoader batches all calls in the same async tick into one query
```

---

## EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name, o.total_amount
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 50;
```

### Reading Query Plans

```
Seq Scan on orders  (cost=0.00..45000.00 rows=1200 width=40)
  Filter: (status = 'pending' AND created_at > ...)
  Rows Removed by Filter: 498800
```

| Plan node | Meaning | What to do |
|---|---|---|
| `Seq Scan` | Full table scan | Add index if table is large and filter is selective |
| `Index Scan` | Follows index, then fetches heap | Good; consider covering index if width is large |
| `Index Only Scan` | Reads index only, no heap | Best for read-heavy queries |
| `Bitmap Heap Scan` | Batch heap access after bitmap index scan | Good for moderate selectivity |
| `Hash Join` | Build hash table on smaller relation | Fine; check if inner table fits in `work_mem` |
| `Nested Loop` | For each outer row, scan inner | Good with small sets; bad with large |
| `Sort` | In-memory or on-disk sort | Add index matching ORDER BY; or increase `work_mem` |

**Key numbers to watch:**
```
cost=0.00..45000.00   → estimated startup..total cost (arbitrary units)
rows=1200             → estimated rows; actual rows shown with ANALYZE
actual time=0.043..892.000  → actual milliseconds startup..total
Buffers: shared hit=4500 read=12000  → 12000 disk reads is expensive
```

---

## Zero-Downtime Migrations

### Guiding Principle

```
Deploy ≠ Migrate.
Run migrations separately from deploys.
New code must handle BOTH old schema (rollback) and new schema (forward).
```

### Add Column — Safe

```sql
-- SAFE: nullable column or column with DEFAULT
ALTER TABLE users ADD COLUMN middle_name VARCHAR(100);
ALTER TABLE users ADD COLUMN is_verified BOOLEAN NOT NULL DEFAULT FALSE;

-- DANGEROUS on large tables without concurrent: full table rewrite
-- PostgreSQL 11+: DEFAULT is stored in catalog, not rewritten — safe
-- PostgreSQL < 11: triggers a full table rewrite for NOT NULL + DEFAULT
```

### Rename Column — Expand-Contract

```sql
-- Step 1 — Expand: add new column
ALTER TABLE orders ADD COLUMN order_total NUMERIC(12,2);

-- Step 2 — Dual-write: update both columns in application
-- (during transition period, app writes both old and new column)
UPDATE orders SET order_total = amount;  -- backfill

-- Step 3 — Switch reads: application reads from order_total only

-- Step 4 — Contract: drop old column (in a later deploy)
ALTER TABLE orders DROP COLUMN amount;
```

### Remove Column — Application First

```sql
-- Step 1: Remove column references from application code (deploy)
-- Step 2: After old code is no longer running:
ALTER TABLE orders DROP COLUMN deprecated_field;
-- PostgreSQL: DROP COLUMN is a catalog-only operation — instant
```

### Add Index — Always CONCURRENTLY

```sql
-- DANGEROUS: acquires exclusive lock, blocks writes for minutes on large tables
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- SAFE: builds index without blocking writes (takes longer, but non-blocking)
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- If CONCURRENTLY fails, it leaves an INVALID index — clean up:
DROP INDEX CONCURRENTLY idx_orders_user_id;
-- Then retry
```

### Change Column Type — Never Directly

```sql
-- DANGEROUS: rewrite + lock
ALTER TABLE events ALTER COLUMN event_id TYPE BIGINT;

-- SAFE: Expand-Contract
-- Step 1: add new column
ALTER TABLE events ADD COLUMN event_id_new BIGINT;

-- Step 2: backfill
UPDATE events SET event_id_new = event_id::BIGINT;

-- Step 3: add constraint, switch application
ALTER TABLE events ALTER COLUMN event_id_new SET NOT NULL;
-- Application now reads/writes event_id_new

-- Step 4: drop old column, rename new
ALTER TABLE events DROP COLUMN event_id;
ALTER TABLE events RENAME COLUMN event_id_new TO event_id;
```

### Migration Decision Matrix

```
Operation                    Safe?   Notes
────────────────────────────────────────────────────────────
ADD COLUMN nullable           YES    Instant (catalog only)
ADD COLUMN NOT NULL DEFAULT   YES*   PG11+: catalog only; PG<11: table rewrite
ADD COLUMN NOT NULL no default NO    Requires backfill first
DROP COLUMN                   YES    Instant (catalog only in PG)
RENAME COLUMN                 NO     Use Expand-Contract
ADD INDEX                     NO     Use CREATE INDEX CONCURRENTLY
ADD UNIQUE CONSTRAINT         NO     Use CREATE UNIQUE INDEX CONCURRENTLY first
CHANGE TYPE                   NO     Use Expand-Contract
DROP TABLE                    NO     Remove from app first, then drop
```

---

## Transactions and Isolation

### ACID

```
Atomicity    — All operations in a transaction succeed, or none do.
Consistency  — Transaction brings database from one valid state to another.
Isolation    — Concurrent transactions behave as if serialized.
Durability   — Committed transactions survive crashes.
```

### Isolation Levels

```sql
-- Read Committed (PostgreSQL default)
-- Can see: dirty reads NO, non-repeatable reads YES, phantom reads YES
-- Use for: most OLTP operations; reads are fast

-- Repeatable Read
-- Can see: dirty reads NO, non-repeatable reads NO, phantom reads NO*
-- Use for: financial reports, any query that reads same row twice and expects consistency
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Serializable
-- Full serializability; slowest; use for: complex business invariants (preventing double-booking)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Practical example — bank transfer:**

```sql
BEGIN;
-- Repeatable Read ensures balance read is stable within transaction
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- lock row
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

---

## Locking Strategies

### Optimistic Locking

```sql
-- Schema: version column
ALTER TABLE products ADD COLUMN version INTEGER NOT NULL DEFAULT 1;

-- Application read
SELECT id, price, version FROM products WHERE id = 42;
-- Returns: id=42, price=99.99, version=7

-- Application update (check version hasn't changed)
UPDATE products
SET price = 109.99, version = version + 1
WHERE id = 42 AND version = 7;
-- If 0 rows affected → someone else updated it → retry or conflict error
```

```python
# Python (SQLAlchemy) — optimistic locking
class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    price = Column(Numeric(10, 2))
    version = Column(Integer, nullable=False, default=1)

    __mapper_args__ = {"version_id_col": version}  # SQLAlchemy handles it automatically
```

### Pessimistic Locking

```sql
-- SELECT FOR UPDATE: lock rows for the duration of the transaction
BEGIN;
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
-- Other transactions block on this row until COMMIT/ROLLBACK

-- SELECT FOR UPDATE SKIP LOCKED: skip locked rows (queue processing)
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

**Optimistic vs Pessimistic:**

```
Optimistic locking:
  + No lock held during think time
  + Better throughput under low contention
  - Requires retry logic in application
  - Bad under high contention (many retries = wasted work)

Pessimistic locking:
  + Guarantees no conflict
  - Deadlock risk (always acquire locks in same order)
  - Serializes access — lower throughput under high load
  Use for: inventory reservation, seat booking, anything where retries are expensive
```

---

## Connection Pooling

### PgBouncer Modes

```
Session mode    — Connection assigned for client session lifetime.
                  Compatible with all PostgreSQL features.
                  Pool size = max concurrent sessions.

Transaction mode — Connection assigned for single transaction.
                   Cannot use: LISTEN/NOTIFY, server-side cursors,
                   prepared statements (by default), session-level settings.
                   Pool size = max concurrent transactions (much smaller).
                   RECOMMENDED for most web apps.

Statement mode   — Connection returned after each statement.
                   Cannot use: transactions, multi-statement queries.
                   Use only for simple, single-statement workloads.
```

### Pool Size Formula

```
pool_size = (CPU cores × 2) + effective_disk_spindles

For cloud/SSD: effective_disk_spindles = 1
Example: 8 core machine with SSD → pool_size = 17

Application pool size per instance:
  total_db_pool_size / number_of_app_instances

Rule: Start with 10-20 connections per app instance. Monitor
      pg_stat_activity and pool wait time. Increase only if wait
      time is high and CPU is not saturated.
```

```yaml
# PgBouncer config
[pgbouncer]
pool_mode = transaction
max_client_conn = 1000      # max app connections to PgBouncer
default_pool_size = 20      # connections PgBouncer opens to PostgreSQL
min_pool_size = 5
reserve_pool_size = 5       # extra connections for bursts
server_idle_timeout = 600   # close idle server connections after 10 min
```

---

## JSON Columns

### When to Use JSONB

```sql
-- GOOD use: flexible attributes that vary per product category
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(500) NOT NULL,
    category    VARCHAR(100) NOT NULL,
    -- Electronics have voltage/wattage; clothing has size/material; etc.
    attributes  JSONB NOT NULL DEFAULT '{}'
);

-- Query JSON attributes
SELECT name FROM products
WHERE attributes @> '{"color": "red"}'   -- containment operator
  AND attributes->>'brand' = 'Nike';     -- key extraction

-- Index for JSONB containment queries
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);
```

### When NOT to Use JSON

```
Use normalized columns when:
- The field is used in WHERE clauses, JOINs, or GROUP BY
- You need foreign key constraints on the field
- The schema is stable and well-understood
- You need CHECK constraints on the field value

Using JSON for these creates: slow queries, no constraint enforcement,
difficult schema evolution.
```

---

## Partitioning

Use when tables exceed ~100M rows, or when archival/retention is needed.

```sql
-- Range partitioning by date (common for time-series / events)
CREATE TABLE events (
    id          BIGSERIAL,
    occurred_at TIMESTAMPTZ NOT NULL,
    type        VARCHAR(100) NOT NULL,
    payload     JSONB
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- List partitioning by region
CREATE TABLE orders (
    id      BIGSERIAL,
    region  VARCHAR(20) NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('us-east', 'us-west');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('eu-central', 'eu-west');

-- Hash partitioning (even distribution by PK)
CREATE TABLE users (id BIGSERIAL, ...) PARTITION BY HASH (id);
CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- etc.
```

**Partition benefits:**
- Partition pruning — queries touching one partition skip all others
- Archival by dropping a partition (instant) instead of DELETE (slow)
- Parallel query across partitions

---

## Migration Tooling

### Alembic (Python)

```python
# alembic/env.py — autogenerate migrations from SQLAlchemy models
# alembic revision --autogenerate -m "add user phone number"

# Generated migration
def upgrade() -> None:
    op.add_column(
        "users",
        sa.Column("phone_number", sa.String(20), nullable=True)
    )
    # Safe: nullable column

def downgrade() -> None:
    op.drop_column("users", "phone_number")
```

```bash
# Run migrations
alembic upgrade head

# Check current version
alembic current

# Generate SQL without running (for DBA review)
alembic upgrade head --sql > migration.sql

# Rollback one step
alembic downgrade -1
```

### Flyway (JVM)

```sql
-- V1__initial_schema.sql
-- V2__add_user_phone.sql
-- Naming: V{version}__{description}.sql

-- V3__add_index_orders_user_id.sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
-- NOTE: CONCURRENTLY cannot run inside a transaction.
-- Flyway wraps migrations in transactions by default.
-- Use: flyway.mixed=true and mark this migration as non-transactional
```

```yaml
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=migration_user
flyway.mixed=true                    # allow CONCURRENTLY in same run
flyway.outOfOrder=false              # enforce sequential versioning
flyway.validateOnMigrate=true        # check checksum on startup
```

---

## Query Optimization Checklist

```
[ ] EXPLAIN ANALYZE on any query taking > 100ms in prod
[ ] Seq Scan on large table (>10k rows) with selective filter → add index
[ ] Check rows estimate vs actual rows — large discrepancy → run ANALYZE
[ ] Sort node with large rows → add index matching ORDER BY
[ ] N+1 detected in ORM query log → eager load or batch query
[ ] Connection wait time high → increase pool size or reduce connection hold time
[ ] Index on low-cardinality column (status: active/inactive) → partial index
[ ] Unused indexes → DROP them (write overhead, no read benefit)
[ ] Text search → use GIN + tsvector, not LIKE '%pattern%'
[ ] Large IN() list → JOIN to a temp table or VALUES clause instead
```
