---
name: distributed-systems-fundamentals
description: Deep reference for distributed systems design — invoke when reasoning about consistency models, CAP tradeoffs, consensus algorithms, replication strategies, distributed transactions, or failure detection in any architecture discussion or implementation.
---

# Distributed Systems Fundamentals

## The Fundamental Problem

Distributed systems fail in ways local systems cannot. A local process either runs or crashes — you know immediately. A remote node might be slow, partitioned, or dead, and **you cannot tell the difference**.

This ambiguity is the defining characteristic of distributed systems. Every design decision flows from it.

### The 8 Fallacies of Distributed Computing

Peter Deutsch and James Gosling identified these in the 1990s. Each one bites you in production:

| Fallacy | Real Consequence |
|---|---|
| 1. The network is reliable | TCP connections drop silently; retries cause duplicate processing without idempotency |
| 2. Latency is zero | A synchronous call to a "local" service across AZs adds 1–5ms per hop; 10 hops = 10–50ms baseline |
| 3. Bandwidth is infinite | Serializing large payloads naively; sending full objects instead of diffs |
| 4. The network is secure | TLS termination at load balancer; service-to-service traffic unencrypted on internal network |
| 5. Topology doesn't change | Hardcoded IPs; services not handling DNS TTL correctly; no service discovery |
| 6. There is one administrator | Schema migrations require coordination across multiple teams; no unified change management |
| 7. Transport cost is zero | Chatty microservices; N+1 remote calls; no batching |
| 8. The network is homogeneous | Different services have different serialization formats, timeouts, retry behaviors |

### Partial Failure

In a local system, a function either returns or throws. In a distributed system, a call to a remote service can:
- Return successfully
- Return an error
- Time out (the remote might have succeeded or failed — you don't know)
- Never return (network black hole)

**This is partial failure.** The caller must decide what to do without knowing the actual state of the callee. Idempotent operations let you retry safely. Non-idempotent operations require deduplication keys, two-phase protocols, or sagas.

---

## Consistency Models

Ordered from strongest (most correct, most expensive) to weakest (most available, most performant).

```
STRONGEST ──────────────────────────────────────── WEAKEST
     │                                                  │
Linearizability → Sequential → Causal → Eventual  →  Chaos
     │                                                  │
   Spanner      ZooKeeper    Cassandra    DynamoDB
```

### Linearizability (External Consistency)

Every operation appears to take effect instantaneously at some point between its invocation and its response. All nodes observe a single, globally consistent order consistent with real time.

- If write W completes before read R begins, R must see W.
- Most expensive: requires coordination across all replicas before returning.
- Enables: distributed locks, leader election, compare-and-swap.
- Systems: Google Spanner, ZooKeeper (for znodes), etcd, CockroachDB.

**When you need it:** financial transactions, distributed locks, configuration that must be consistent across all nodes.

### Sequential Consistency

All nodes observe operations in the same order, but that order need not correspond to real time. A read after a write (by wall clock) might not see the write if they're on different nodes.

- Cheaper than linearizability (no real-time synchronization).
- Difficult to implement correctly; often confused with linearizability.

### Causal Consistency

If operation A causally precedes operation B (A happened before B, or B depends on A's result), all nodes see A before B. Causally unrelated operations can be seen in any order.

- "You can see my reply only after seeing the message I'm replying to."
- Used in: some distributed databases, collaborative editing systems.
- Tracked with vector clocks or version vectors.

### Eventual Consistency

Given no new writes, all replicas will eventually converge to the same value. No guarantees on how long "eventually" takes. No guarantees on what you read in the meantime.

- Highest availability, lowest latency.
- Requires conflict resolution strategy.
- Systems: Cassandra, DynamoDB, CouchDB, Redis Cluster.

**When you accept it:** shopping cart totals, social media likes, DNS propagation, recommendation systems.

### Session Guarantees (Practical Middle Ground)

These are weaker than causal consistency but stronger than pure eventual consistency. Applied per-client session:

- **Read-your-writes:** After you write, your subsequent reads see that write (even from different replicas).
- **Monotonic reads:** Once you read a value, you never see an older value.
- **Monotonic writes:** Your writes are applied in the order you issued them.
- **Writes-follow-reads:** A write after a read is applied on a replica that has the value you read (or newer).

Most distributed databases implement these for practical usability.

---

## CAP Theorem

Eric Brewer's theorem: a distributed system can guarantee at most two of:

- **C**onsistency (every read sees the most recent write or an error)
- **A**vailability (every request receives a response, not an error)
- **P**artition Tolerance (the system continues operating despite network partitions)

```
          Consistency
              /\
             /  \
            /    \
           /  CA  \   ← doesn't exist in practice
          /        \
         /----  ----|
        / CP  |  AP  \
       /      |       \
      /       |        \
Availability ──────── Partition Tolerance
```

**You must tolerate partitions.** Networks partition. The real choice is what happens during a partition:

- **CP (Consistency + Partition Tolerance):** Refuse requests during partition to preserve consistency.
  - ZooKeeper, HBase, Consul, etcd
  - Use when: distributed locks, leader election, config that must be consistent

- **AP (Availability + Partition Tolerance):** Serve potentially stale data during partition.
  - Cassandra, DynamoDB, CouchDB, Riak
  - Use when: user-facing reads where staleness is acceptable, high write throughput

- **CA (Consistency + Availability):** Only possible if you assume no partitions — not realistic in distributed systems. Single-node relational databases are CA *within a single node*.

### PACELC: Extending CAP

CAP only describes behavior during partitions. **PACELC** (Daniel Abadi, 2012) says: *Even without a Partition, there's a tradeoff between latency and consistency.*

```
If Partition:
  → Choose between Availability and Consistency (CAP)
Else (normal operation):
  → Choose between Latency and Consistency

System        | Partition | Else
--------------|-----------|---------
DynamoDB      | A         | L
Cassandra     | A         | L
CockroachDB   | C         | C
Spanner       | C         | C
ZooKeeper     | C         | C
```

---

## Time in Distributed Systems

**Wall clocks lie.** NTP drift between nodes can be milliseconds to seconds. Leap seconds cause clocks to jump or repeat. Clock skew invalidates any reasoning based on timestamps alone.

### Problems With Wall Clocks

- Two events with timestamps T1 < T2 may not have happened in that order
- "Last write wins" with wall clocks causes data loss on concurrent writes with clock skew
- NTP correction can cause time to go backwards

### Lamport Timestamps (Logical Clocks)

Each node maintains a counter. Rules:
1. Increment counter before each event
2. When sending a message, attach counter value
3. On receiving a message, `counter = max(local, received) + 1`

Establishes **happens-before** (→) relationship:
- If A → B, then L(A) < L(B)
- If L(A) < L(B), A may or may not have happened before B (can't distinguish concurrent from ordered)

```
Node 1:  1 ──── 2 ────────── 4 ──── 5
              send(2)      recv(3)
                │               │
Node 2:  1 ──── 2 ──── 3 ──── 4
                     recv(2) send(3)
```

### Vector Clocks

Each node maintains a vector of counters, one per node. Rules:
1. Increment own counter for each event
2. On send, attach full vector
3. On receive, take element-wise max, then increment own counter

Enables **detecting concurrent writes**:
- V1 < V2 if every element of V1 ≤ corresponding element of V2 (and at least one is strictly less)
- If neither V1 ≤ V2 nor V2 ≤ V1, the events are concurrent

```
Initial:  [A:0, B:0, C:0]

A writes: [A:1, B:0, C:0]
A→B:      B receives, updates to [A:1, B:1, C:0]
C writes: [A:0, B:0, C:1]

A:1 and C:1 are concurrent — conflict!
B has seen A's write, C has not — causal ordering preserved
```

Used in: Amazon Dynamo, Riak, CRDTs.

### Hybrid Logical Clocks (HLC)

Combines physical time and logical counters:
- `HLC = (physical_time, logical_counter)`
- When physical clock advances, reset logical counter
- When receiving message, take max of physical times; break ties with logical counter

Properties:
- Close to wall clock (within NTP uncertainty bounds)
- Captures causality like logical clocks
- Used in: CockroachDB, YugabyteDB

### Google TrueTime

Atomic clocks + GPS receivers in every datacenter. TrueTime API returns `[earliest, latest]` for current time — **bounded uncertainty**, not a point value.

Spanner waits out the uncertainty window before committing transactions, guaranteeing external consistency. This "commit wait" is typically 7–10ms.

---

## Consensus

**The problem:** Get N nodes to agree on a single value despite node crashes and network partitions, with the following guarantees:
- **Safety:** All nodes that decide, decide the same value
- **Liveness:** The system eventually makes progress

FLP Impossibility (1985): In an asynchronous system with even one possible crash failure, no deterministic consensus algorithm can guarantee both safety and liveness. Practical systems sidestep this by using timeouts (adding weak synchrony assumptions).

### Paxos

The original consensus algorithm. Roles: Proposers, Acceptors, Learners.

**Phase 1 (Prepare):**
1. Proposer picks proposal number N, sends `Prepare(N)` to majority of acceptors
2. Acceptors respond with promise not to accept proposals < N, plus any previously accepted value

**Phase 2 (Accept):**
1. If proposer receives majority promises, sends `Accept(N, value)` to acceptors
2. Acceptors accept if no higher-numbered prepare received since their promise
3. Once majority accepts, value is chosen

Hard to implement correctly: Multi-Paxos (for log replication), leader leases, reconfiguration — each adds complexity. Chubby (Google's distributed lock) is built on Paxos; Lamport says even Google got it wrong in the paper.

### Raft

Designed for understandability. Decomposed into three problems: leader election, log replication, safety.

```
         ┌─────────────────────────────────────┐
         │            RAFT CLUSTER             │
         │                                     │
         │   ┌──────────┐    ┌──────────┐     │
         │   │ Follower │    │ Follower │     │
         │   └──────────┘    └──────────┘     │
         │          │               │          │
         │          └───────┬───────┘          │
         │                  │                  │
         │           ┌──────────┐              │
         │           │  Leader  │              │
         │           └──────────┘              │
         │                                     │
         │   Term: a monotonically increasing  │
         │   number; each election starts new  │
         │   term. Prevents stale leaders.     │
         └─────────────────────────────────────┘
```

**Leader Election:**
1. Node starts as follower; starts election timer on timeout
2. Increments term, transitions to candidate, votes for itself
3. Sends `RequestVote` to all nodes
4. Node grants vote if: term is current, hasn't voted this term, candidate's log is at least as up-to-date
5. Candidate winning majority → becomes leader; sends heartbeats to suppress new elections

**Log Replication:**
1. Client sends command to leader
2. Leader appends to local log (uncommitted)
3. Leader sends `AppendEntries` to all followers
4. Once majority acknowledges, leader commits, returns to client
5. Leader notifies followers of commit on next heartbeat; followers apply to state machine

**Safety:** A leader cannot commit entries from previous terms directly; it must commit at least one entry from its own term first (Log Matching Property).

```
Log replication flow:
Client → Leader: "SET x=5"
Leader: [idx=42, term=3, SET x=5] (uncommitted)
Leader → Follower1: AppendEntries(42, SET x=5)
Leader → Follower2: AppendEntries(42, SET x=5)
Follower1 → Leader: ACK
Leader: majority reached → commit idx=42
Leader → Client: OK
(followers learn of commit on next heartbeat)
```

**Used in:** etcd (Kubernetes), CockroachDB, TiKV (TiDB), Consul, InfluxDB.

---

## Replication Patterns

### Single-Leader (Master-Slave)

```
         ┌──────────┐
Writes → │  Leader  │
         └────┬─────┘
         ┌────┴────┐
    ┌────┴──┐  ┌───┴────┐
    │Replica│  │Replica │ ← Reads
    └───────┘  └────────┘
```

- One primary accepts all writes; replicates to replicas
- Reads can go to replicas (stale) or leader (consistent)
- **Failover:** promote a replica; risk of split-brain if old leader not properly fenced
- **SPOF risk:** if leader dies before replication, data loss
- Used in: PostgreSQL streaming replication, MySQL binlog replication, Redis primary-replica

**Anti-pattern:** Reading from replicas for data that was just written (violates read-your-writes without sticky sessions or replication lag awareness).

### Multi-Leader (Multi-Master)

Multiple nodes accept writes. Useful for multi-datacenter writes (one leader per DC).

**Conflict resolution strategies:**
- **Last Write Wins (LWW):** Timestamp-based; causes data loss on concurrent writes
- **Application-level merge:** Application decides (e.g., shopping cart union)
- **CRDT:** Data structure designed to merge without conflict (see below)
- **Custom:** Operational transformation (used in Google Docs)

### Leaderless (Dynamo-style)

```
         Client
        /   |   \
       /    |    \
   Node1  Node2  Node3
     W      W      -    ← Write to W nodes
     R      -      R    ← Read from R nodes
     
  W + R > N → quorum; can detect stale reads
  W=2, R=2, N=3 → tolerates 1 failure
```

- Client (or coordinator) sends writes to multiple nodes; reads from multiple nodes
- **Quorum:** W (write) + R (read) > N (total) ensures at least one node has latest write
- **Sloppy quorum + hinted handoff:** During partition, accept writes on non-home nodes; replay later
- **Read repair:** On read, detect stale values, write latest back to stale nodes
- **Anti-entropy:** Background process syncs replicas using Merkle trees

Used in: Amazon DynamoDB, Cassandra, Riak.

### Synchronous vs Asynchronous Replication

| | Synchronous | Asynchronous |
|---|---|---|
| Durability | Write confirmed only after replica ACK | Write confirmed immediately; replica may lag |
| Latency | Higher (network RTT on critical path) | Lower |
| Availability | Lower (replica slowness blocks writes) | Higher |
| Data loss on leader failure | Zero | Up to replication lag |

**Semi-synchronous (MySQL):** At least one replica must ACK before leader returns. Balance between full sync and full async.

---

## Distributed Transactions

### Two-Phase Commit (2PC)

Atomic commit across multiple participants.

```
         ┌─────────────┐
         │ Coordinator │
         └──────┬──────┘
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌──────┐    ┌──────┐    ┌──────┐
│  P1  │    │  P2  │    │  P3  │
└──────┘    └──────┘    └──────┘

Phase 1 (Prepare):
  Coordinator → Participants: "Can you commit?"
  Participants: lock resources, write to redo log, reply YES/NO

Phase 2 (Commit/Abort):
  All YES → Coordinator logs COMMIT, sends COMMIT to all
  Any NO  → Coordinator logs ABORT, sends ABORT to all
  Participants: commit or rollback, release locks
```

**Problems:**
- Coordinator is SPOF: if coordinator crashes after sending PREPARE but before COMMIT, participants are blocked with locks held
- Blocking protocol: participants cannot make progress without coordinator
- Vulnerable to network partitions between phases

### Three-Phase Commit (3PC)

Adds a "pre-commit" phase to reduce blocking window. Not widely used because:
- Still not safe under network partition
- Added complexity for marginal benefit
- Most systems prefer Sagas or Raft-based transactions

### Sagas

Sequence of local transactions, each publishing events that trigger the next. Compensating transactions undo completed steps on failure.

```
Order Saga:
  1. Create Order (local tx)
        ↓ success
  2. Reserve Inventory (local tx)
        ↓ success
  3. Charge Payment (local tx)
        ↓ failure
  COMPENSATE:
  3. Void payment charge (compensating tx)
  2. Release inventory reservation (compensating tx)
  1. Cancel order (compensating tx)
```

**Choreography:** Each service listens for events, decides what to do. Decentralized; hard to track saga state.

**Orchestration:** Central saga orchestrator commands each service. Explicit flow; easier to monitor; orchestrator can become SPOF.

**Key constraint:** Compensating transactions must be idempotent and must always succeed (or you need compensating-compensating transactions...).

### CRDTs (Conflict-free Replicated Data Types)

Data structures with a merge function that is commutative, associative, and idempotent. Replicas can diverge and merge without coordination.

- **G-Counter (Grow-only):** Per-node counters; total = sum. Merges by taking max per node.
- **PN-Counter:** Two G-Counters (increments + decrements).
- **G-Set:** Union-only set. Merge = union.
- **2P-Set:** Add + remove sets; once removed, never re-added.
- **LWW-Element-Set:** Timestamps on add/remove; last timestamp wins.
- **OR-Set (Observed-Remove):** Each add gets unique tag; remove removes specific tag. Allows re-add.
- **RGA (Replicated Growable Array):** For collaborative text editing.

Used in: Riak, Redis CRDT module, Automerge (collaborative editing), distributed shopping carts.

---

## Failure Detection

### Heartbeats + Timeout

Each node sends heartbeats periodically. If heartbeat not received within timeout, node suspected failed.

**Problem:** Timeout tuning. Too short → false positives (slow network triggers failover). Too long → slow failure detection.

```
timeout = avg_heartbeat_interval + max_expected_network_delay + safety_margin
```

### Phi Accrual Failure Detector

Used in Akka, Cassandra. Instead of binary (alive/dead), outputs a suspicion level φ.

- φ increases as time since last heartbeat grows relative to historical heartbeat intervals
- Application chooses threshold: φ > 8 → suspect; φ > 16 → act
- Adapts to network conditions; reduces false positives during transient slowdowns

### SWIM Protocol (Scalable Weakly-consistent Infection-style Membership)

Used in Consul, Serf, memberlist. Gossip-based; O(log N) messages instead of O(N²) for N nodes.

```
Failure detection:
  A pings B directly
  B doesn't respond within timeout
  A asks C, D, E to ping B indirectly (k random nodes)
  If none get response → B marked suspect
  After suspicion timeout with no refutation → B declared dead

Dissemination:
  Membership changes (join/leave/fail) piggyback on ping/ack messages
  Gossip spreads to whole cluster in O(log N) rounds
```

Properties:
- **Completeness:** Every failed node is eventually detected
- **Accuracy:** Low false-positive rate (indirect pings distinguish network partition from node failure)
- **Scalability:** Message complexity independent of cluster size

---

## Anti-Patterns

**Distributed monolith:** Microservices that must all be deployed together and call each other synchronously in a chain. Combines the worst of monolith (tight coupling) and distributed (network failures, latency).

**Shared database across services:** Destroys service autonomy; one service's schema change breaks others.

**Synchronous calls for everything:** Long synchronous chains amplify failure probability. If each call is 99.9% reliable and you have 10 calls in a chain: 0.999^10 = 99.0% reliability. 50 calls: 95.1%.

**No idempotency:** Retrying non-idempotent operations (e.g., charge payment) causes duplicates. Use idempotency keys.

**Ignoring the happy path:** Designing only for success; no timeout, no retry, no circuit breaker.

**Clock-based ordering without bounds:** Using `System.currentTimeMillis()` to order events across nodes will cause data loss.

**Missing back-pressure:** Producer generates events faster than consumer can process; unbounded queue growth → OOM.

---

## Quick Reference Checklist

When designing a distributed system, answer these:

- [ ] What consistency model is required? (linearizability for locks, eventual for counters)
- [ ] What happens during a network partition? (CP: reject requests; AP: serve stale data)
- [ ] How are concurrent writes detected and resolved? (vector clocks, CRDTs, LWW)
- [ ] Is the operation idempotent? Can it be retried safely?
- [ ] What is the failure detection strategy and timeout tuning?
- [ ] Is distributed transaction needed, or can Sagas work?
- [ ] What is the replication lag acceptable for reads?
- [ ] Are there clock-dependent operations? Use logical or hybrid clocks.
- [ ] How does the system handle slow consumers (back-pressure)?
- [ ] Is there a single point of failure? (coordinator in 2PC, single leader without automatic failover)
