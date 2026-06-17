---
name: redis-mastery
description: Expert Redis skill — invoke when designing caching strategies, selecting data structures, implementing distributed locks, reasoning about eviction policies, building Pub/Sub or Streams pipelines, or tuning Redis for production performance.
---

# Redis Mastery

## Data Structures and When to Use Each

Redis is not just a cache. Choosing the right data structure eliminates application-side logic and dramatically reduces round trips.

### String

The most versatile. Stores bytes: text, JSON, binary, numbers.

```python
# Simple cache
r.set("user:1001:profile", json.dumps(profile), ex=3600)  # TTL 1 hour
val = r.get("user:1001:profile")

# Atomic counter (no race condition)
r.incr("page:views:2024-01-15")
r.incrby("inventory:item:42", -1)  # decrement stock

# Distributed lock (SET NX PX = set if not exists, with ms TTL)
acquired = r.set("lock:payment:order-999", "worker-id-abc", nx=True, px=30000)
```

**SET NX PX** is the canonical single-node distributed lock. `nx=True` makes it atomic: set only if key doesn't exist. `px=30000` sets 30s TTL so lock auto-releases if holder crashes.

### Hash

Object fields without serializing the entire object. Efficient when you need individual field access or partial updates.

```python
# Store user object — update individual fields without loading full object
r.hset("user:1001", mapping={
    "name": "Alice",
    "email": "alice@example.com",
    "login_count": 0
})

r.hget("user:1001", "email")           # fetch single field
r.hincrby("user:1001", "login_count", 1)  # atomic increment of one field
r.hmget("user:1001", "name", "email")  # fetch multiple fields

# Memory efficiency: Redis uses ziplist encoding for small hashes
# (< 128 fields, values < 64 bytes) — much more compact than one key per field
```

**Use Hash over String+JSON when:** you frequently update individual fields; you need atomic field-level operations; the object has many fields but you typically only need a few.

**Use String+JSON when:** you always read/write the full object; you need the value in external systems that expect JSON.

### List

Ordered collection of strings. O(1) push/pop at both ends. O(N) access by index.

```python
# Queue: LPUSH adds to head; RPOP removes from tail (FIFO)
r.lpush("queue:emails", json.dumps(email_job))
job = r.brpop("queue:emails", timeout=5)  # blocking pop; waits up to 5s

# Stack: LPUSH + LPOP (LIFO)
r.lpush("undo:session-xyz", json.dumps(action))
last_action = r.lpop("undo:session-xyz")

# Recent items with bounded size
r.lpush("recent:user:1001:pages", "/products/42")
r.ltrim("recent:user:1001:pages", 0, 9)  # keep only 10 most recent
pages = r.lrange("recent:user:1001:pages", 0, -1)  # get all
```

**BRPOP/BLPOP** (blocking pop) is how you build a simple task queue. Workers block waiting for work; Redis wakes them when items arrive. No polling.

**Anti-pattern:** Using List as a set (checking membership with LRANGE + scan). Use Set for membership checks — O(1) vs O(N).

### Set

Unordered collection of unique strings. O(1) add, remove, membership check. Set operations (union, intersection, difference) are O(N).

```python
# Tag system
r.sadd("tags:post:123", "redis", "caching", "databases")
r.sadd("tags:post:456", "redis", "performance")

r.sismember("tags:post:123", "redis")   # True — O(1)
r.smembers("tags:post:123")            # get all tags

# Find posts with any of these tags (union)
r.sunionstore("result:query-1", "tags:post:123", "tags:post:456")

# Find posts with ALL of these tags (intersection)
r.sinterstore("result:query-2", "tags:category:tech", "tags:category:databases")

# Online users tracking
r.sadd("online:users", user_id)
r.srem("online:users", user_id)
r.scard("online:users")               # count online users — O(1)
```

### Sorted Set (ZSet)

Like Set, but each member has a floating-point score. Members ordered by score. O(log N) add/remove.

```python
# Leaderboard
r.zadd("leaderboard:weekly", {"alice": 4200, "bob": 3800, "carol": 5100})
r.zincrby("leaderboard:weekly", 100, "alice")   # alice scores 100 more points

# Top 10 (highest scores first)
top10 = r.zrange("leaderboard:weekly", 0, 9, withscores=True, rev=True)

# Rank of a player (0-indexed, lowest to highest)
rank = r.zrevrank("leaderboard:weekly", "alice")   # rank from top

# Rate limiting: sliding window
# Score = timestamp; member = request-id or timestamp
now = time.time()
pipe = r.pipeline()
pipe.zadd(f"ratelimit:user:{user_id}", {str(uuid4()): now})
pipe.zremrangebyscore(f"ratelimit:user:{user_id}", 0, now - 60)  # remove older than 60s
pipe.zcard(f"ratelimit:user:{user_id}")
_, _, count = pipe.execute()
if count > 100:
    raise RateLimitExceeded()

# Priority queue: lower score = higher priority
r.zadd("jobs:priority", {"job:critical:1": 1, "job:normal:2": 5, "job:low:3": 10})
next_job_id = r.zpopmin("jobs:priority")  # atomic pop of lowest score (highest priority)
```

### HyperLogLog

Probabilistic cardinality counting. O(1) add and count. Always ~12KB regardless of cardinality. Error rate ~0.81%.

```python
# Count unique visitors (exact count would require storing every user ID)
r.pfadd(f"visitors:2024-01-15", user_id)
count = r.pfcount(f"visitors:2024-01-15")   # ~0.81% error

# Merge multiple days
r.pfmerge("visitors:2024-01", "visitors:2024-01-01", "visitors:2024-01-02", ...)
weekly_uniques = r.pfcount("visitors:2024-01")
```

**Use when:** exact count not required; cardinality is high (millions of values); storing every value is prohibitive.

**Do not use when:** you need exact counts; cardinality < 10,000 (just use a Set).

### Streams

Append-only log with consumer groups. Like Kafka but simpler. Persisted in Redis.

```
Stream structure:
ID (auto-generated: timestamp-sequence)
│
▼
1704067200000-0  →  {event: "order.created", order_id: "42", amount: "99.99"}
1704067200001-0  →  {event: "payment.processed", order_id: "42", tx_id: "tx-99"}
1704067200500-0  →  {event: "order.shipped", order_id: "42", tracking: "1Z999"}
```

---

## Caching Patterns

### Cache-Aside (Lazy Loading)

Most common. Application manages cache explicitly.

```python
def get_user(user_id: str) -> dict:
    cache_key = f"user:{user_id}"
    
    # 1. Check cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # 2. Cache miss — load from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. Store in cache
    r.set(cache_key, json.dumps(user), ex=3600)
    return user
```

**Pros:** Only caches what's requested; cache survives Redis restart (cold cache, but no stale data).
**Cons:** Cache miss adds latency; thundering herd on popular cache misses.

### Write-Through

Write to cache and DB atomically. Cache always consistent.

```python
def update_user(user_id: str, data: dict):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id, data)
    r.set(f"user:{user_id}", json.dumps(data), ex=3600)
    # Both writes happen; if Redis fails, DB is source of truth
```

**Pros:** Cache always has latest data; no stale reads after writes.
**Cons:** Higher write latency; caches data that may never be read.

### Write-Behind (Write-Back)

Write to cache first; async flush to DB. Highest write performance; risk of data loss.

```
Client → Cache (sync, fast) → DB (async, delayed)
```

**Only use when:** write throughput is more important than durability; data can be reconstructed; losses are acceptable (analytics, metrics, view counts).

### Cache Stampede (Thundering Herd)

When a popular cache key expires, many requests simultaneously miss → all hit DB → DB overwhelmed.

**Solutions:**

```python
# 1. Probabilistic early expiration (XFetch algorithm)
import math
import random

def get_with_early_expiration(key: str, beta: float = 1.0) -> Optional[str]:
    value, ttl = r.get(key), r.ttl(key)
    if value is None:
        return None  # cache miss
    
    # Probabilistically recompute before expiration
    # Higher beta = more aggressive early refresh
    if -beta * math.log(random.random()) >= ttl:
        return None  # trigger refresh
    return value

# 2. Mutex / single-flight: only one goroutine recomputes, others wait
import threading
_locks = {}

def get_with_lock(key: str, compute_fn) -> Any:
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    
    lock_key = f"lock:{key}"
    if lock_key not in _locks:
        _locks[lock_key] = threading.Lock()
    
    with _locks[lock_key]:
        cached = r.get(key)  # double-check after acquiring lock
        if cached:
            return json.loads(cached)
        value = compute_fn()
        r.set(key, json.dumps(value), ex=3600)
        return value

# 3. Stale-while-revalidate: serve stale, refresh in background
```

---

## Expiration and Eviction

### TTL Commands

```python
r.set("session:abc", data, ex=3600)      # seconds
r.set("session:abc", data, px=3600000)   # milliseconds
r.expire("key", 3600)                    # set TTL on existing key (seconds)
r.pexpire("key", 3600000)               # milliseconds
r.expireat("key", 1735689600)           # Unix timestamp
r.pexpireat("key", 1735689600000)       # Unix timestamp in ms

r.ttl("key")    # remaining seconds (-1 = no TTL, -2 = doesn't exist)
r.persist("key")  # remove TTL — make key permanent
```

### Eviction Policies

Set `maxmemory` in `redis.conf` or at runtime. When memory is full, policy determines what to evict:

| Policy | Behavior | Use When |
|---|---|---|
| `noeviction` | Return error on write; reads still work | You never want data loss; writes should fail instead |
| `allkeys-lru` | Evict least recently used key from all keys | General-purpose cache; most keys should be evicted |
| `allkeys-lfu` | Evict least frequently used from all keys | Workloads with popularity skew; hot keys stay |
| `volatile-lru` | LRU from keys with TTL set | Mix of cache and persistent data |
| `volatile-lfu` | LFU from keys with TTL set | Same; frequency-based |
| `volatile-ttl` | Evict keys with shortest TTL first | Evict "most expired soon" keys first |
| `allkeys-random` | Random eviction from all keys | Uniform access patterns |
| `volatile-random` | Random from keys with TTL | Rarely useful |

**Production recommendation:** `allkeys-lru` for pure caches. `volatile-lru` if Redis stores both cache and persistent data. Always set `maxmemory`.

```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

---

## Distributed Locks

### Basic Lock (Single Redis Instance)

```python
import uuid
import time

def acquire_lock(r, lock_name: str, ttl_ms: int = 30000) -> Optional[str]:
    """Returns lock token if acquired, None otherwise."""
    token = str(uuid.uuid4())
    acquired = r.set(f"lock:{lock_name}", token, nx=True, px=ttl_ms)
    return token if acquired else None

# CRITICAL: Release must be atomic — check token, then delete
RELEASE_SCRIPT = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""

def release_lock(r, lock_name: str, token: str) -> bool:
    """Returns True if lock was released, False if already expired or taken by another."""
    result = r.eval(RELEASE_SCRIPT, 1, f"lock:{lock_name}", token)
    return bool(result)

# Usage
token = acquire_lock(r, "payment:order-999", ttl_ms=30000)
if token is None:
    raise LockNotAcquired("Could not acquire payment lock")
try:
    process_payment()
finally:
    released = release_lock(r, "payment:order-999", token)
    if not released:
        logger.warning("Lock expired before release — check processing time vs TTL")
```

**Why Lua for release?** `GET` + `DEL` is two commands. Between them, the lock could expire and be acquired by another worker. The Lua script executes atomically on the Redis server — no race condition.

### Redlock (Multi-Instance)

For stronger guarantees, acquire lock on N/2+1 independent Redis instances:

```python
# Using redlock-py
from redlock import Redlock

dlm = Redlock([
    {"host": "redis-1", "port": 6379},
    {"host": "redis-2", "port": 6379},
    {"host": "redis-3", "port": 6379},
])

lock = dlm.lock("payment:order-999", ttl=30000)
if lock:
    try:
        process_payment()
    finally:
        dlm.unlock(lock)
```

**Redlock controversy:** Martin Kleppmann showed Redlock can fail to provide mutual exclusion under certain failure scenarios (clock jumps, GC pauses). For truly critical locks (distributed commit coordination), use ZooKeeper or etcd which provide linearizable guarantees.

**Use single-instance lock when:** brief lock duplication on Redis failure is acceptable.
**Use Redlock when:** better availability than single instance; eventual consistency of the lock is acceptable.
**Use ZooKeeper/etcd when:** strict mutual exclusion is required regardless of Redis failures.

---

## Pub/Sub vs Streams

```
              Pub/Sub                    Streams
              ───────                    ───────
Persistence   None (fire-and-forget)    Yes (persisted to memory/AOF/RDB)
Consumer      All subscribers get all   Consumer groups: each message
delivery      messages                  delivered to one consumer in group
Replay        No                        Yes (read from any ID)
Ordering      Per-channel               Per-stream (within shard)
Scale         Limited (all msgs to all  Consumer groups scale horizontally
              subscribers)
Backpressure  None                      MAXLEN cap on stream size
Acknowledgment No                       Yes (XACK)
```

**Use Pub/Sub for:** real-time notifications where loss is acceptable; broadcasting (chat, live updates, cache invalidation signals).

**Use Streams for:** reliable message processing; event sourcing; audit logs; replacing simple Kafka use cases.

---

## Redis Streams as Lightweight Kafka

### Producer

```python
# XADD: append to stream; * = auto-generate ID (timestamp-sequence)
entry_id = r.xadd("orders", {
    "event": "order.created",
    "order_id": "42",
    "amount": "99.99",
    "customer_id": "1001"
})
# entry_id: "1704067200000-0"

# Cap stream size (trim to last 10000 entries)
r.xadd("orders", {"event": "..."}, maxlen=10000, approximate=True)
```

### Consumer Group

```python
# Create consumer group (read from beginning with $=only new messages)
try:
    r.xgroup_create("orders", "payment-service", id="0", mkstream=True)
except ResponseError:
    pass  # group already exists

# Consumer reads undelivered messages (> = pending for this group)
def process_messages(consumer_name: str):
    while True:
        # Read up to 10 new messages, block 1 second if none
        messages = r.xreadgroup(
            "payment-service", consumer_name,
            {"orders": ">"},   # ">" = new messages not yet delivered to this group
            count=10,
            block=1000
        )
        
        for stream_name, entries in (messages or []):
            for entry_id, fields in entries:
                try:
                    handle_order_event(fields)
                    r.xack("orders", "payment-service", entry_id)  # ACK = processed
                except Exception as e:
                    logger.error(f"Failed to process {entry_id}: {e}")
                    # DO NOT ACK — will appear in XPENDING for retry/DLQ
```

### Pending Entries and Recovery

```python
# See unacknowledged messages
pending = r.xpending("orders", "payment-service")
# {'pending': 3, 'min': '1704067200000-0', 'max': '1704067200002-0', 'consumers': [...]}

# Detailed pending entries
pending_detail = r.xpending_range("orders", "payment-service", min="-", max="+", count=100)

# Reclaim messages idle for > 60 seconds (e.g., consumer crashed)
def reclaim_idle_messages(consumer_name: str):
    claimed = r.xautoclaim(
        "orders", "payment-service", consumer_name,
        min_idle_time=60000,   # 60 seconds in ms
        start_id="0-0",
        count=10
    )
    # Process claimed messages normally
```

---

## Pipelining and Transactions

### Pipelining

Batch multiple commands to reduce round-trip latency. Commands are NOT atomic — other clients can interleave.

```python
# Without pipeline: N round trips
for user_id in user_ids:
    r.get(f"user:{user_id}:score")

# With pipeline: 1 round trip
pipe = r.pipeline(transaction=False)  # transaction=False = no MULTI/EXEC wrapping
for user_id in user_ids:
    pipe.get(f"user:{user_id}:score")
results = pipe.execute()
```

### MULTI/EXEC (Transactions)

All commands in MULTI block are queued and executed atomically as a batch. No other client can interleave.

```python
# Transfer points between users atomically
pipe = r.pipeline()  # transaction=True by default
pipe.multi()
pipe.decrby("user:1001:points", 100)
pipe.incrby("user:1002:points", 100)
pipe.execute()
# Either both execute or neither does (if connection drops before EXEC)
```

**Caveat:** Redis transactions are NOT like SQL transactions. If a command fails inside MULTI/EXEC (e.g., wrong type), the other commands still execute. EXEC does not roll back on command error.

### WATCH (Optimistic Locking)

Watch keys before MULTI; if any watched key changes before EXEC, transaction aborts.

```python
def transfer_points(from_user: str, to_user: str, amount: int, max_retries: int = 3):
    for attempt in range(max_retries):
        with r.pipeline() as pipe:
            try:
                pipe.watch(f"user:{from_user}:points")
                balance = int(pipe.get(f"user:{from_user}:points") or 0)
                
                if balance < amount:
                    raise InsufficientPoints()
                
                pipe.multi()
                pipe.decrby(f"user:{from_user}:points", amount)
                pipe.incrby(f"user:{to_user}:points", amount)
                pipe.execute()
                return  # success
                
            except WatchError:
                # Another client modified the watched key — retry
                continue
    raise MaxRetriesExceeded()
```

### Lua Scripts (Atomic, Complex Logic)

For compare-and-swap or any logic that must be atomic:

```python
# Atomic decrement only if >= amount (no negative balance)
DEBIT_SCRIPT = """
local balance = tonumber(redis.call("get", KEYS[1]))
if balance == nil then
    return {err = "account_not_found"}
end
if balance < tonumber(ARGV[1]) then
    return {err = "insufficient_funds"}
end
return redis.call("decrby", KEYS[1], ARGV[1])
"""

result = r.eval(DEBIT_SCRIPT, 1, f"account:{account_id}:balance", amount)
```

Lua scripts execute atomically on the Redis server. No other commands can run while a script executes. Keep scripts short to avoid blocking.

---

## Java and Kotlin Examples

### Lettuce (Reactive, Kotlin)

```kotlin
// Lettuce with Kotlin coroutines
val client = RedisClient.create("redis://localhost:6379")
val connection = client.connect().coroutines()  // coroutine API

// Distributed lock
suspend fun withLock(lockName: String, ttlMs: Long, block: suspend () -> Unit) {
    val token = UUID.randomUUID().toString()
    val acquired = connection.set(
        "lock:$lockName", token,
        SetArgs.Builder.nx().px(ttlMs)
    ) == "OK"
    
    if (!acquired) throw LockNotAcquiredException(lockName)
    
    try {
        block()
    } finally {
        val script = """
            if redis.call("get",KEYS[1]) == ARGV[1] then
                return redis.call("del",KEYS[1])
            else
                return 0
            end
        """.trimIndent()
        connection.eval<Long>(script, ScriptOutputType.INTEGER, arrayOf("lock:$lockName"), token)
    }
}

// Pipeline with Lettuce
suspend fun batchGetScores(userIds: List<String>): List<Long?> {
    return connection.pipelined {
        userIds.map { get("user:$it:score") }
    }.map { it.toLongOrNull() }
}
```

### Jedis (Java)

```java
// Jedis with connection pool
JedisPool pool = new JedisPool(new JedisPoolConfig(), "localhost", 6379);

// Cache-aside pattern
public User getUser(String userId) {
    String cacheKey = "user:" + userId;
    try (Jedis jedis = pool.getResource()) {
        String cached = jedis.get(cacheKey);
        if (cached != null) {
            return objectMapper.readValue(cached, User.class);
        }
        User user = userRepository.findById(userId);
        jedis.setex(cacheKey, 3600, objectMapper.writeValueAsString(user));
        return user;
    }
}

// Transaction with WATCH
public void transferPoints(String fromUser, String toUser, int amount) {
    try (Jedis jedis = pool.getResource()) {
        for (int attempt = 0; attempt < 3; attempt++) {
            String fromKey = "user:" + fromUser + ":points";
            jedis.watch(fromKey);
            
            int balance = Integer.parseInt(jedis.get(fromKey));
            if (balance < amount) throw new InsufficientPointsException();
            
            Transaction tx = jedis.multi();
            tx.decrBy(fromKey, amount);
            tx.incrBy("user:" + toUser + ":points", amount);
            List<Object> result = tx.exec();
            
            if (result != null) return;  // success
            // result == null means WATCH fired — retry
        }
        throw new MaxRetriesExceededException();
    }
}
```

### Spring Data Redis (Java/Kotlin)

```kotlin
@Service
class UserCacheService(
    private val redisTemplate: RedisTemplate<String, String>,
    private val objectMapper: ObjectMapper
) {
    private val ops = redisTemplate.opsForValue()
    
    fun getOrLoad(userId: String, loader: () -> User): User {
        val key = "user:$userId"
        val cached = ops.get(key)
        if (cached != null) return objectMapper.readValue(cached)
        
        val user = loader()
        ops.set(key, objectMapper.writeValueAsString(user), Duration.ofHours(1))
        return user
    }
    
    fun invalidate(userId: String) {
        redisTemplate.delete("user:$userId")
    }
}
```

### redis-py (Python)

```python
import redis
from redis.retry import Retry
from redis.backoff import ExponentialBackoff

# Production connection with retry
r = redis.Redis(
    host="redis-master",
    port=6379,
    db=0,
    decode_responses=True,
    socket_timeout=1.0,
    socket_connect_timeout=1.0,
    retry=Retry(ExponentialBackoff(), retries=3),
    retry_on_error=[redis.ConnectionError, redis.TimeoutError],
    health_check_interval=30,
)

# Sorted set rate limiter
def check_rate_limit(user_id: str, max_requests: int = 100, window_seconds: int = 60) -> bool:
    key = f"ratelimit:{user_id}"
    now = time.time()
    window_start = now - window_seconds
    
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)
    pipe.zadd(key, {str(uuid4()): now})
    pipe.zcard(key)
    pipe.expire(key, window_seconds + 1)
    _, _, count, _ = pipe.execute()
    
    return count <= max_requests
```

---

## Performance Tuning

### Finding Problems

```bash
# Slow log — commands taking longer than slowlog-log-slower-than microseconds
redis-cli SLOWLOG GET 10
redis-cli SLOWLOG LEN
redis-cli SLOWLOG RESET
# In redis.conf: slowlog-log-slower-than 10000  (10ms)

# Memory analysis
redis-cli MEMORY USAGE user:1001:profile    # bytes used by this key
redis-cli MEMORY DOCTOR                     # diagnostic report
redis-cli INFO memory                       # full memory stats

# Key analysis (use SCAN — NEVER KEYS * in production)
redis-cli --scan --pattern "session:*" | head -20

# Monitor commands in real time (use sparingly — impacts performance)
redis-cli MONITOR

# Latency analysis
redis-cli --latency -h redis-host
redis-cli --latency-history -h redis-host -i 5  # sample every 5s
```

### Dangerous Commands — Never in Production

```
KEYS *          → O(N) over all keys; blocks Redis event loop; causes latency spike
SMEMBERS        → O(N) for large sets; fetch all members at once
LRANGE 0 -1     → O(N) for large lists; same problem
FLUSHALL        → Deletes everything; irreversible
DEBUG SLEEP     → Blocks for N seconds
```

**Replace KEYS with SCAN:**

```python
# KEYS * is dangerous — blocks Redis
# NEVER: r.keys("session:*")

# SCAN iterates in batches — safe for production
def scan_keys(pattern: str, count: int = 100):
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=count)
        yield from keys
        if cursor == 0:
            break

for key in scan_keys("session:expired:*"):
    r.delete(key)
```

### Configuration Tuning

```
# redis.conf — production settings

# Memory
maxmemory 8gb
maxmemory-policy allkeys-lru
maxmemory-samples 10             # LRU sample size; higher = more accurate, more CPU

# Persistence (choose based on durability needs)
save ""                          # disable RDB for pure cache
appendonly yes                   # AOF for durability
appendfsync everysec             # fsync every second; balance durability/performance
no-appendfsync-on-rewrite yes    # don't fsync during BGREWRITE

# Network
tcp-backlog 511
timeout 300                      # close idle client connections after 5 min
tcp-keepalive 60

# Slow log
slowlog-log-slower-than 10000    # log commands > 10ms
slowlog-max-len 128

# Latency monitoring
latency-monitor-threshold 100    # track events > 100ms

# Active defragmentation (Redis 4+)
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
```

### Key Naming Conventions

```
# Use colons as separators — human-readable, compatible with RedisInsight
user:1001:profile
user:1001:sessions
session:abc123:data
cache:product:42:details
lock:payment:order-999
queue:emails:high-priority
ratelimit:api:user:1001

# Add expiry group prefix for bulk invalidation
cache:v2:product:42     # v2 prefix allows invalidating all v1 cache on deploy
```

---

## Cluster and High Availability

### Sentinel (HA for Single Primary)

```
        ┌──────────────┐
        │   Sentinel1  │
        └──────┬───────┘
               │ monitor
        ┌──────┴───────┐
        │   Sentinel2  │──── quorum: 2 of 3 sentinels must agree
        └──────┬───────┘        on failover
               │
        ┌──────┴───────┐
        │   Sentinel3  │
        └──────┬───────┘
               │
         ┌─────┴──────┐
    ┌────┴────┐   ┌───┴──────┐
    │ Primary │   │ Replica  │
    └─────────┘   └──────────┘
```

Minimum: 3 Sentinel nodes (quorum = 2). Sentinel auto-promotes replica to primary on failure. Clients discover primary by querying Sentinel.

### Cluster (Horizontal Sharding)

```
16384 hash slots distributed across nodes:
Node1: slots 0-5460
Node2: slots 5461-10922
Node3: slots 10923-16383
(each node has a replica)

Key → CRC16(key) % 16384 → slot → node
```

```python
# Cluster client handles redirection automatically
from redis.cluster import RedisCluster

r = RedisCluster(
    host="redis-cluster-1",
    port=6379,
    decode_responses=True,
)

# Hash tags: force multiple keys to same slot (for multi-key operations)
r.set("{user:1001}.profile", json.dumps(profile))
r.set("{user:1001}.sessions", sessions_data)
# Both land on same slot → MULTI/EXEC works across these keys
```

**Cluster limitations:**
- MULTI/EXEC only works for keys on the same slot
- Lua scripts must operate on keys in the same slot
- Database selection (`SELECT 1`) not supported

---

## Anti-Patterns Summary

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `KEYS *` in production | O(N) blocks Redis event loop | Use `SCAN` with cursor |
| No `maxmemory` set | Redis OOMs and crashes | Always set `maxmemory` |
| Storing large values (>1MB) | Slow network transfer, memory fragmentation | Split or compress values |
| No TTL on cache keys | Memory grows unbounded | Always set TTL; use `EXPIRE` |
| String+JSON for partial updates | Must deserialize/serialize full object to change one field | Use Hash for field-level access |
| `GET`+`DEL` for lock release | Race: lock can expire between GET and DEL | Lua script for atomic release |
| No connection pooling | New TCP connection per command | Use JedisPool, Lettuce pool |
| Storing secrets in plain text | Redis is often accessible within network | Encrypt values; restrict network access |
| Large SMEMBERS/LRANGE | O(N) response; blocks Redis | Use SSCAN/LRANGE with bounded range |
| SELECT across databases in Cluster | Not supported in Redis Cluster | Use key prefixes instead |
