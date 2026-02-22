# 📖 Chapter-by-Chapter Simplified Guide

> Every chapter of DDIA, re-explained for 2026 engineers. Zero academic fluff. All signal.

---

## Part I: Foundations of Data Systems

---


## Chapter 1: Reliability, Scalability, and Maintainability

### The Core Idea

Every data system exists to do five things: store data (databases), speed up reads (caches), search data (indexes), send messages (streams), and crunch data (batch). Your job as an engineer is to pick the right tool for each job and compose them correctly.

### Reliability — "It works even when things break"

**Things that break:**
- **Hardware**: Disks die (~2-4% annual failure rate in large datacenters). AWS literally kills instances routinely.
- **Software**: Memory leaks, cascading failures, runaway queries, config push to the wrong environment.
- **Humans**: The #1 source of outages. That `DROP TABLE` without a `WHERE` clause. That config change at 5 PM on a Friday.

**How to build reliable systems:**
1. Design for failure: redundancy at every layer
2. Chaos engineering: Netflix's Chaos Monkey isn't optional anymore — it's standard practice
3. Canary deployments: Roll out changes to 1% of traffic first
4. Circuit breakers: Prevent cascading failures (Hystrix pattern)
5. Observability: You can't fix what you can't see (OpenTelemetry, Datadog, Grafana)

**Production pitfall**: Engineers often conflate "fault" and "failure." A fault is **one thing going wrong**. A failure is **the whole system going down**. Your goal is to be fault-tolerant, not fault-free.

### Scalability — "It handles growth gracefully"

**Stop asking "Will it scale?" Start asking:**
- What are my load parameters? (QPS, data volume, read/write ratio, concurrent users)
- What happens when load parameter X doubles?

**Performance metrics that matter:**
- **Throughput**: Requests per second (batch systems)
- **Response time**: How long a user waits (online systems)
- **Always use percentiles**, not averages. P50 = median, P99 = tail latency, P999 = worst case

```
Why percentiles matter:
────────────────────────
Average response time: 200ms    ← "Looks fine!"
P99 response time: 4,500ms     ← "1% of users are FURIOUS"
P99 is often your highest-value customers (most data, most features)
```

**Scaling approaches:**
| Vertical (Scale Up) | Horizontal (Scale Out) |
|---------------------|----------------------|
| Bigger machine | More machines |
| Simple | Complex |
| Ceiling exists | Ceiling is budget |
| r6g.16xlarge → done | Sharding, load balancing, replication |

**2026 take**: Auto-scaling is default now. Kubernetes handles horizontal scaling. Serverless (Lambda, Cloud Run) abstracts it entirely. But you still need to understand *what* scales and *what doesn't*.

### Maintainability — "People can actually work on this"

Three sub-properties:
1. **Operability**: Good monitoring, automation, self-healing, runbooks. If your on-call engineer can't debug it at 3 AM from their phone, it's not operable.
2. **Simplicity**: Complexity is the #1 killer. Use abstractions. Remove dead code. Keep APIs clean.
3. **Evolvability**: Requirements change. Schema changes. APIs evolve. Design for change, not perfection.

**The real takeaway**: Maintainability is where 90% of the lifetime cost of software lives. The initial build is cheap. Living with it for 5 years is expensive.

---

## Chapter 2: Data Models and Query Languages

### The Core Idea

Your data model is the most important architectural decision you make. It shapes how you think about the problem, what queries are fast, and what changes are painful.

### Relational vs. Document vs. Graph

```
Relational (SQL)                Document (NoSQL)               Graph
────────────────                ────────────────               ─────
Tables with rows/columns        JSON/BSON blobs               Nodes + edges
Strict schema (schema-on-write) Flexible (schema-on-read)     Relationships are first-class
JOINs are powerful              JOINs are weak/manual          Traversals are cheap
PostgreSQL, MySQL, CockroachDB  MongoDB, DynamoDB, Firestore   Neo4j, Amazon Neptune, Dgraph
```

### When To Use What

| Use Case | Best Model | Why |
|----------|-----------|-----|
| E-commerce with complex relations | Relational | JOINs between orders, products, users, payments |
| User profiles, content management | Document | Self-contained documents, flexible schema |
| Social networks, fraud detection | Graph | Relationships ARE the data |
| IoT sensor data, time series | Time-series DB | InfluxDB, TimescaleDB, Prometheus |
| AI/ML embeddings, semantic search | Vector DB | Pinecone, Weaviate, pgvector |

### The Schema Spectrum

```
Schema-on-Write (Relational)     Schema-on-Read (Document)
───────────────────────────      ─────────────────────────
Database enforces structure      Application enforces structure
ALTER TABLE to change            Just write new fields
Safer, but migrations are painful Flexible, but garbage-in garbage-out
Think: static typing              Think: dynamic typing
```

**Production insight**: In 2026, the line has blurred enormously. PostgreSQL has excellent JSON support. MongoDB has schema validation. DynamoDB has strongly-typed attributes. Pick based on your access patterns, not ideology.

### The SQL vs. NoSQL Decision

**Use SQL when:**
- You have complex relationships between entities
- You need ACID transactions across multiple tables
- Your access patterns are diverse and unpredictable
- You want the ecosystem (ORMs, migrations, tooling)

**Use NoSQL when:**
- You have clear, simple access patterns (key-value lookup)
- Your data is naturally hierarchical/document-shaped
- You need horizontal scaling from day one
- Schema flexibility is important (rapid iteration)

**The 2026 truth**: Most companies use both. PostgreSQL for the core transactional system. DynamoDB or Redis for high-throughput, simple lookups. Elasticsearch for search. This is called **polyglot persistence**.

---

## Chapter 3: Storage and Retrieval

### The Core Idea

How databases physically store and retrieve data determines everything about performance. Understanding this lets you pick the right database and the right indexes.

### Two Families of Storage Engines

```
B-Tree (Page-Oriented)                LSM-Tree (Log-Structured)
──────────────────────                ─────────────────────────
The "classic" approach                 The "newer" approach
Fixed-size pages (~4KB)                Sorted string tables (SSTables)
In-place updates                       Append-only, then compact
Reads are fast                         Writes are fast
PostgreSQL, MySQL, Oracle              RocksDB, Cassandra, LevelDB
Good for read-heavy workloads          Good for write-heavy workloads
```

### B-Trees — The Mental Model

Think of a B-Tree as a **sorted filing cabinet**. Every page is a drawer. The root page tells you which drawer to check. Each drawer either contains more drawers (internal nodes) or the actual data (leaf nodes).

```
        [10 | 20 | 30]          ← Root: "Go left for <10, middle for 10-20..."
       /      |      \
   [3|5|7]  [12|15]  [22|25|28] ← Leaf pages with actual data
```

- **Reads**: O(log n) — follow the tree from root to leaf
- **Writes**: Find the page, update in place. If the page is full, split it.
- **Crash safety**: Write-Ahead Log (WAL) — write the change to a log FIRST, then apply it.

### LSM-Trees — The Mental Model

Think of an LSM-Tree as a **stack of sorted notebooks**. You write everything to the current notebook (memtable, in-memory). When it's full, you flush it to disk as a sorted file. Periodically, you merge old files together (compaction).

```
Write → [Memtable (RAM)]
              ↓ (flush when full)
         [SSTable Level 0]  ← Most recent
         [SSTable Level 1]  ← Older
         [SSTable Level 2]  ← Oldest
              ↑ (compaction merges levels)
```

- **Reads**: Check memtable → Level 0 → Level 1 → ... (use bloom filters to skip levels)
- **Writes**: Append to memtable (blazing fast)
- **Tradeoff**: Slower reads (multiple levels to check), but way faster writes

### When To Use Which

| Factor | B-Tree | LSM-Tree |
|--------|--------|----------|
| Read-heavy workload | ✅ Winner | Slower (multiple levels) |
| Write-heavy workload | Slower (random I/O) | ✅ Winner |
| Predictable performance | ✅ Consistent | Compaction can cause spikes |
| Space efficiency | Less efficient | ✅ Better compression |
| Production example | PostgreSQL default | Cassandra, RocksDB |

### OLTP vs. OLAP Storage

| | OLTP | OLAP |
|-|------|------|
| Access pattern | Point lookups, small updates | Full-column scans, aggregations |
| Storage layout | Row-oriented | **Column-oriented** |
| Why | You read all fields of a row | You read one field across millions of rows |
| 2026 tools | PostgreSQL, MySQL, DynamoDB | Snowflake, BigQuery, Redshift, ClickHouse |

### Column-Oriented Storage — The Insight

```
Row storage (OLTP):
Row 1: [user_id=1, name="Alice", age=30, city="NYC"]
Row 2: [user_id=2, name="Bob",   age=25, city="SF"]

Column storage (OLAP):
user_id: [1, 2, 3, 4, 5, ...]
name:    ["Alice", "Bob", "Charlie", ...]
age:     [30, 25, 35, ...]
city:    ["NYC", "SF", "NYC", ...]
```

**Why this matters**: Query `SELECT AVG(age) FROM users WHERE city = 'NYC'` only reads TWO columns instead of ALL columns. Plus columns of the same type compress incredibly well (bitmap encoding, run-length encoding).

**2026 update**: Parquet is the universal columnar file format. Every data warehouse and lakehouse uses it. If you're doing analytics, you're probably using Parquet under the hood.

---

## Chapter 4: Encoding and Evolution

### The Core Idea

Your system will evolve. Old code and new code will run simultaneously during deployments. Your serialization format must handle **backward compatibility** (new code reads old data) and **forward compatibility** (old code reads new data).

### Encoding Formats — Decision Matrix

| Format | Schema | Human-Readable | Performance | When To Use |
|--------|--------|----------------|-------------|-------------|
| JSON | Flexible | ✅ Yes | Slow | APIs, configs, small payloads |
| Protocol Buffers | Required (.proto) | ❌ No | ✅ Fast | gRPC services, internal comms |
| Avro | Required | ❌ No | ✅ Fast | Kafka messages, data pipelines |
| Parquet | Embedded | ❌ No | ✅ Fast | Analytics, data lakes |
| Thrift | Required (.thrift) | ❌ No | Fast | Legacy Facebook systems |

### Compatibility Rules

```
Adding a new OPTIONAL field    → ✅ Backward + Forward compatible
Adding a new REQUIRED field    → ❌ Breaks backward compatibility
Removing a field               → ⚠️ Okay if it was optional
Changing a field type           → ⚠️ Careful — widening OK, narrowing dangerous
Renaming a field                → ❌ Never (use field IDs/numbers instead)
```

**Production insight**: Protobuf and Avro use field NUMBERS, not names. This is why they handle evolution so well. `string name = 1;` — the `1` is what's stored in the wire format.

### Modes of Dataflow

| Dataflow | How Data Moves | Compatibility Burden | Example |
|----------|---------------|---------------------|---------|
| Via databases | Write then read later | Reader must handle all versions ever written | Schema migrations |
| Via service calls (REST/gRPC) | Request → Response | Client + Server must agree | API versioning |
| Via message queues | Producer → Broker → Consumer | Producers and consumers evolve independently | Kafka topics |

**2026 take**: Schema registries (Confluent Schema Registry, AWS Glue Schema Registry) are essential. They enforce compatibility at the pipeline level, not just in code.

---

## Part II: Distributed Data

---

## Chapter 5: Replication

### The Core Idea

Replication means keeping copies of the same data on multiple machines. You do it for **fault tolerance**, **low latency** (data near users), and **read throughput**.

### Three Replication Architectures

#### 1. Single-Leader (Master-Slave)

```
           ┌──────────┐
Writes ──→ │  LEADER   │ ──→ Replication Log
           └──────────┘
              ↓    ↓    ↓
          ┌──────┐ ┌──────┐ ┌──────┐
Reads ──→ │Follow│ │Follow│ │Follow│
          └──────┘ └──────┘ └──────┘
```

**How it works**: All writes go to one node (leader). Leader sends changes to followers via replication log. Reads can go to any node.

**Sync vs. Async replication:**
- **Synchronous**: Write isn't confirmed until follower has it. Durable but slow. One slow follower blocks everything.
- **Asynchronous**: Write confirmed immediately. Fast but data can be lost if leader crashes. This is what most systems use.
- **Semi-synchronous**: One follower is sync, rest are async. Practical middle ground.

**The Failover Problem:**
When the leader dies, you need a new leader (failover). This is HARD:
- New leader may be missing recent writes → data loss
- Old leader comes back thinking it's still leader → **split brain** (two leaders accepting writes)
- Timeout too short → false failovers under load
- Timeout too long → long outage

**2026 tooling**: Managed databases (RDS, Cloud SQL, Aurora) handle failover automatically — but you still need to understand it for system design interviews.

#### 2. Multi-Leader

```
   Datacenter A              Datacenter B
   ┌──────────┐              ┌──────────┐
   │ LEADER A │ ←──────────→ │ LEADER B │
   └──────────┘   conflict   └──────────┘
      ↓   ↓       resolution    ↓   ↓
   followers                  followers
```

**When to use**: Multi-region deployments where you need local writes in each region.

**The big problem**: Conflict resolution. User edits a document in US while another edits it in EU.

**Conflict resolution strategies:**
| Strategy | How | Tradeoff |
|----------|-----|---------|
| Last Write Wins (LWW) | Timestamp decides | Data loss — later timestamp wins |
| Merge values | Application logic | Complex but preserves data |
| User resolution | "Which version do you want?" | Good UX but bad for automated systems |
| CRDTs | Mathematical merge | No conflicts possible but limited data types |

**2026 tools**: CockroachDB, Google Spanner, YugabyteDB are "NewSQL" — they give you multi-region writes with strong consistency (using synchronized clocks + Raft consensus).

#### 3. Leaderless (Dynamo-style)

```
Client writes to ALL replicas simultaneously
Client reads from MULTIPLE replicas simultaneously
No single point of failure
```

**Quorum formula**: `W + R > N` where:
- N = total replicas
- W = number of writes that must succeed
- R = number of reads you do

**Common config**: N=3, W=2, R=2. You read from 2 of 3 nodes, ensuring at least one has the latest write.

**When to use**: When availability matters more than consistency. Shopping carts, session stores, metrics.

**When NOT to use**: Banking, inventory management, anything where stale reads are unacceptable.

### Replication Lag Problems

| Problem | What Happens | Solution |
|---------|-------------|---------|
| Read-after-write | User writes something, refreshes, doesn't see it | Read your own writes from leader |
| Monotonic reads | User sees newer data, then older data (time travel) | Sticky sessions (always read same replica) |
| Consistent prefix | Causal events appear out of order | Write causally related data to same partition |

---

## Chapter 6: Partitioning (Sharding)

### The Core Idea

When one machine can't hold or serve all your data, you split it across multiple machines. Each piece is a **partition** (or shard). The goal: spread data AND load evenly.

### Two Partitioning Strategies

#### Key-Range Partitioning
```
Partition 1: A-F     Partition 2: G-M     Partition 3: N-Z
 [Adams]              [Garcia]              [Nguyen]
 [Brown]              [Johnson]             [Parker]
 [Chen]               [Lee]                 [Williams]
```
- ✅ Great for range queries ("all users A-C")
- ❌ Can create hot spots (all today's data hits one partition)

#### Hash Partitioning
```
hash("Adams")  % 3 = 1  → Partition 1
hash("Garcia") % 3 = 0  → Partition 0
hash("Lee")    % 3 = 2  → Partition 2
```
- ✅ Even distribution
- ❌ No range queries possible
- This is what Kafka, DynamoDB, and Cassandra use by default

### The Hot Spot Problem

Even with hashing, hot spots happen. Example: A celebrity tweets → millions of writes to one partition key.

**Solutions:**
1. Add random suffix to hot keys (e.g., `celebrity_123_rand(0-99)`) — splits writes across 100 partitions. Reads must query all 100 and merge.
2. Application-level sharding for known hot keys
3. DynamoDB adaptive capacity (automatic in 2026)

### Secondary Indexes on Partitioned Data

| Approach | How | Tradeoff |
|----------|-----|---------|
| Local index (per-partition) | Each partition indexes its own data | Writes fast, reads must scatter/gather ALL partitions |
| Global index (term-partitioned) | One global index, also partitioned | Reads fast, writes are slower (must update multiple partitions) |

**2026 reality**: Most managed databases handle this transparently. But if you're designing from scratch: local indexes = simpler writes, global indexes = simpler reads. Pick based on your dominant access pattern.

### Rebalancing

When you add/remove nodes, data must move. Strategies:
- **Fixed number of partitions**: Create way more partitions than nodes. When you add nodes, just move partitions. (Elasticsearch, Riak, Couchbase)
- **Dynamic partitioning**: Split partitions when they get too big. (HBase, DynamoDB)
- **Proportional to nodes**: Fixed partitions per node. (Cassandra)

**Rule of thumb**: Over-partition from the start. 10 nodes? Create 1000 partitions. Adding node 11 just rebalances ~100 partitions.

---

## Chapter 7: Transactions

### The Core Idea

Transactions are the database's way of letting your application group reads and writes into a single logical unit that either **fully succeeds** or **fully fails**. They protect you from concurrency bugs and partial failures.

### ACID — What It Actually Means

| Property | Real Meaning | NOT What You Might Think |
|----------|-------------|--------------------------|
| **Atomicity** | All-or-nothing. Crash mid-write → rollback everything | NOT about concurrency (that's isolation) |
| **Consistency** | App invariants are maintained (e.g., account balance ≥ 0) | NOT a database guarantee — it's YOUR responsibility |
| **Isolation** | Concurrent transactions don't interfere with each other | Comes in multiple levels with different tradeoffs |
| **Durability** | Committed data survives crashes | Doesn't mean immune to disk destruction |

### Isolation Levels — The Spectrum

```
WEAKEST                                                    STRONGEST
   │                                                          │
   ▼                                                          ▼
Read        Snapshot          Repeatable      Serializable
Committed   Isolation         Read            
─────────   ─────────         ──────────      ────────────
No dirty    Consistent        + prevents      True serial
reads/      point-in-time     lost updates    execution
writes      view                              
Default PG  PostgreSQL MVCC   MySQL default   Strictest
Fast        Fast              Moderate        Slowest
```

### Common Concurrency Bugs

| Bug | What Happens | Example | Prevention |
|-----|-------------|---------|------------|
| **Dirty read** | Read uncommitted data | See a half-finished bank transfer | Read Committed |
| **Dirty write** | Overwrite uncommitted data | Two users update same row simultaneously | Row-level locks |
| **Read skew** | Inconsistent reads across queries | Backup sees account A debited but B not yet credited | Snapshot Isolation |
| **Lost update** | Read-modify-write race | Two users increment counter, one increment lost | Atomic ops, compare-and-set |
| **Write skew** | Two transactions read then write based on read | Two doctors go off-call simultaneously (both see "2 on-call") | Serializable isolation |
| **Phantom** | Query results change mid-transaction | Insert matches a query's WHERE clause during execution | Serializable isolation |

### Achieving Serializability

| Method | How | Tradeoff |
|--------|-----|---------|
| **Actual Serial Execution** | One thread, one transaction at a time | Simple but limited throughput. Works for Redis, VoltDB |
| **Two-Phase Locking (2PL)** | Readers and writers block each other | Strong but terrible latency under contention |
| **Serializable Snapshot Isolation (SSI)** | Optimistic: run in parallel, detect conflicts, abort | Best of both worlds. CockroachDB, PostgreSQL use this |

### Transaction Decision Framework

```
Do you need transactions across multiple tables/rows?
├── No → Use atomic single-row operations (most NoSQL supports this)
└── Yes → Do you need them across services/databases?
    ├── No → Use database transactions (PostgreSQL, MySQL)
    └── Yes → Use the Saga pattern (choreography or orchestration)
              ⚠️ Avoid distributed 2PC unless you absolutely must
```

**Production pitfall**: "We don't need transactions" is the most dangerous sentence in distributed systems. You probably do. Eventual consistency bugs are subtle, appear months later, and are incredibly hard to debug.

---

## Chapter 8: The Trouble with Distributed Systems

### The Core Idea

In a distributed system, **things WILL go wrong in ways you can't predict**. Networks are unreliable, clocks are unsynchronized, and processes can pause at any time. Your job is not to prevent failures but to **tolerate** them.

### The Three Fundamental Problems

#### 1. Unreliable Networks

```
You send a request. What happened?
─────────────────────────────────
✅ Request arrived, response on the way
❌ Request was lost
❌ Request is queued somewhere
❌ Remote node crashed
❌ Remote node is slow (GC pause)
❌ Response was lost
❌ Response is queued
YOU CAN'T TELL WHICH ONE IT IS.
```

**The only tool you have: timeouts.** But:
- Too short → false positives (healthy node declared dead)
- Too long → slow failure detection
- There is no "correct" timeout — use adaptive timeouts based on observed latency distributions

**2026 practice**: Service meshes (Istio, Linkerd) handle timeouts, retries, and circuit breakers automatically. But you still need to understand failure modes.

#### 2. Unreliable Clocks

```
Two types of clocks:
──────────────────────
Time-of-day clock     → "What time is it?"
                         Synced via NTP, can jump backward
                         DON'T use for measuring duration
                         
Monotonic clock       → "How long has it been?"
                         Always moves forward
                         Meaningless absolute value
                         USE THIS for timeouts and latency measurement
```

**The distributed ordering problem**: You cannot use timestamps to determine which event happened first across machines. NTP accuracy is 10-100ms at best.

**Solution**: Use **logical clocks** (Lamport timestamps, vector clocks) for ordering events. Or use **Google Spanner's TrueTime** — GPS + atomic clocks to bound uncertainty to ~7ms globally.

**Production horror story**: Your system uses `LastWriteWins` with timestamps. Machine A's clock is 5ms ahead. Machine A's "later" write overwrites Machine B's actually-later write. You silently lose data. Forever.

#### 3. Process Pauses

A node can pause at any time for:
- **GC pauses**: Java GC can freeze for seconds
- **VM live migration**: Cloud provider moves your VM
- **Disk I/O**: Waiting for disk (especially if swap is used)
- **Context switches**: Overloaded CPU

**Implication**: A node might think it holds a lease/lock, do a GC pause, come back, and the lease has expired. But it doesn't know that. It continues operating as if it holds the lock.

**Solution**: Fencing tokens. Every lock/lease comes with a monotonically increasing number. The storage system rejects writes with old fencing tokens.

### The Core Insight

> **You must build reliable systems from unreliable components.** This is the fundamental challenge of distributed computing. You do this through redundancy, consensus algorithms, timeouts, retries with idempotency, and accepting that "exactly-once" is really "effectively-once."

---

## Chapter 9: Consistency and Consensus

### The Core Idea

When multiple nodes need to agree on something — who is the leader, whether to commit a transaction, what order events happened in — you need **consensus**. It's the hardest problem in distributed systems.

### Consistency Models — Ranked

```
STRONGEST ──────────────────────────────────────── WEAKEST

Linearizable      Sequential       Causal          Eventual
                  Consistency      Consistency      Consistency
─────────────     ──────────       ──────────       ──────────
"Appears as       "All nodes see   "If A caused B,  "Eventually,
 one copy"         same order"      see A before B"  all agree"
 
Spanner, etcd     ZooKeeper        Kafka partitions  Cassandra,
                                                     DynamoDB

Slowest           Moderate         Practical          Fastest
```

### Linearizability — When You Need It

- **Leader election**: All nodes must agree who the leader is
- **Uniqueness constraints**: Only one user gets username "alice"
- **Distributed locking**: Only one process holds the lock
- **Bank account balances**: Can't go below zero

### The CAP Theorem — Demystified

Forget "pick 2 out of 3." The real story:

> **During a network partition, you must choose: consistency OR availability. You can't have both.**

But network partitions are rare. The practical tradeoff is:

> **Consistency vs. Latency**: Even without partitions, linearizability is SLOW because every read must check the latest write across all replicas.

**This is why most systems choose eventual consistency** — it's not about partitions, it's about performance.

### Consensus Algorithms

| Algorithm | Used By | Key Idea |
|-----------|---------|----------|
| **Raft** | etcd, CockroachDB, Consul | Leader-based, easy to understand, log replication |
| **Paxos** | Google Chubby, Megastore | Academic gold standard, hard to implement |
| **Zab** | ZooKeeper | Like Paxos, optimized for primary-backup |
| **Multi-Paxos/Raft** | Most production systems | Used for total order broadcast |

**What consensus gives you:**
- **Uniform agreement**: No two nodes decide differently
- **Integrity**: No node decides twice
- **Validity**: Only proposed values are decided
- **Termination**: Eventually a decision is reached (liveness)

### 2PC vs. Consensus

| | Two-Phase Commit (2PC) | Consensus (Raft/Paxos) |
|-|----------------------|----------------------|
| **Coordinator fails** | Blocked forever (or until recovery) | New leader elected |
| **Fault tolerance** | None — coordinator is SPOF | Tolerates minority failures |
| **Use case** | Cross-database transactions | Leader election, distributed locks |
| **2026 verdict** | Avoid if possible | Use etcd/ZooKeeper |

**Production insight**: 2PC is a blocking protocol. If the coordinator crashes after sending PREPARE but before sending COMMIT, all participants are stuck holding locks. In production, this means your database is frozen until someone manually intervenes. This is why modern systems prefer Raft-based consensus.

---

## Part III: Derived Data

---

## Chapter 10: Batch Processing

### The Core Idea

Batch processing takes a large, bounded dataset, processes all of it, and produces output. It's the workhorse of data engineering. Think: "run this job nightly on all of yesterday's data."

### The Unix Philosophy → MapReduce → Modern Batch

```
1970s-2000s: Unix pipes       cat access.log | grep "ERROR" | sort | uniq -c
2004-2015:   MapReduce        Hadoop, massive scale, but slow and painful
2015-2020:   Spark            10-100x faster than MapReduce (in-memory)
2020-2026:   dbt + Warehouses SQL transformations in Snowflake/BigQuery
```

### MapReduce — Understand It, Don't Use It

**The model:**
1. **Map**: Read input, extract key-value pairs
2. **Shuffle**: Group all values by key
3. **Reduce**: Aggregate values for each key

**Why it mattered**: It proved you could process petabytes by distributing across thousands of commodity machines.

**Why you don't use it in 2026**: It writes intermediate results to disk between every stage. Spark (in-memory) is 10-100x faster. And for most analytics, SQL on Snowflake/BigQuery is even simpler.

### Modern Batch Processing Stack (2026)

```
Raw Data (S3/GCS)
    ↓
Ingestion (Fivetran, Airbyte, custom scripts)
    ↓
Data Lake (S3 + Parquet/Delta Lake/Iceberg)
    ↓
Transformation (dbt + SQL, or Spark for heavy lifting)
    ↓
Data Warehouse (Snowflake, BigQuery, Redshift)
    ↓
BI/Analytics (Looker, Tableau, Metabase)
```

**Key principle**: Batch processing should be **idempotent**. Run it twice, get the same result. This makes retry, debugging, and reprocessing trivial.

### Join Patterns in Distributed Processing

| Join Type | How | When |
|-----------|-----|------|
| Sort-merge join | Both datasets sorted by join key, merge like a zipper | Large-large joins |
| Broadcast hash join | Small dataset loaded into memory on every node | Small-large joins |
| Partitioned hash join | Both datasets partitioned the same way | Pre-partitioned data |

**2026 reality**: You almost never implement these yourself. Spark's optimizer picks the best join strategy automatically. In Snowflake/BigQuery, you just write SQL.

---

## Chapter 11: Stream Processing

### The Core Idea

Stream processing handles unbounded data — events that keep arriving continuously. Instead of waiting for "all the data" like batch, you process each event as it arrives (or close to it).

### The Event Streaming Platform

```
Producers → [  Topic  ] → Consumers
             (Partitioned, replicated, durable log)

2026 tools: Kafka, Amazon Kinesis, Pulsar, Redpanda
```

**Kafka is the de facto standard** because:
- Durable (writes to disk, replicated)
- Ordered (within a partition)
- Replayable (consumers can rewind)
- Scalable (partition = unit of parallelism)

### Two Types of Message Systems

| Traditional Broker (RabbitMQ) | Log-Based Broker (Kafka) |
|-------------------------------|--------------------------|
| Message deleted after delivery | Messages retained (configurable) |
| No ordering guarantee | Ordered within partition |
| Push to consumers | Consumers pull at their own pace |
| Good for task queues | Good for event streams |
| Fan-out to one consumer per message | Fan-out to all consumer groups |

### Change Data Capture (CDC)

**The problem**: You have data in PostgreSQL but need it in Elasticsearch for search and Snowflake for analytics. How do you keep them in sync?

**The solution**: Capture every change to the database and stream it to downstream systems.

```
PostgreSQL → Debezium → Kafka → Elasticsearch
                              → Snowflake
                              → Redis cache
```

**2026 tools**: Debezium (open source), AWS DMS, Fivetran, Airbyte, pg_logical decoding.

**Key insight**: CDC makes your primary database the **single source of truth** (system of record). Everything else is a derived view. This is the most robust way to keep multiple systems in sync.

### Event Sourcing

Instead of storing current state, store the **sequence of events** that led to the current state.

```
Traditional:                    Event Sourced:
┌───────────────────┐           ┌─────────────────────────────┐
│ balance: $150     │           │ AccountCreated: $0          │
│ (current state)   │           │ Deposited: $200             │
└───────────────────┘           │ Withdrew: $50               │
                                │ → current balance: $150     │
                                └─────────────────────────────┘
```

**When to use**: Audit logs, financial systems, collaborative editing, anything where "how did we get here?" matters.

**When NOT to use**: Simple CRUD apps where you just need current state. The complexity isn't worth it.

### Stream Processing Patterns

| Pattern | What It Does | Example |
|---------|-------------|---------|
| **Filtering** | Drop events that don't match | Filter out non-error logs |
| **Enriching** | Add context to events | Add user profile data to click events |
| **Aggregating** | Compute metrics over windows | Clicks per minute, rolling averages |
| **Joining** | Combine two streams | Match ad impressions with clicks |
| **CEP** | Detect complex patterns | Fraud detection: 3 failed logins + purchase from new country |

### Windowing

```
Tumbling:    |  0-5  |  5-10  | 10-15 |     Fixed, non-overlapping
Hopping:     | 0-5 | 1-6 | 2-7 |           Fixed, overlapping
Sliding:     [any events within 5 sec of each other]
Session:     [user activity until 30 min inactivity]
```

### Stream Processing Frameworks (2026)

| Tool | Strength | Use When |
|------|----------|----------|
| **Kafka Streams** | Library, no cluster needed | Java/Kotlin apps, simpler use cases |
| **Apache Flink** | True streaming, low latency | Sub-second processing, complex event processing |
| **Spark Structured Streaming** | Micro-batch, strong batch integration | You already use Spark for batch |
| **AWS Kinesis Data Analytics** | Serverless, SQL-based | AWS-native, simple transformations |

---

## Chapter 12: The Future of Data Systems

### The Core Idea

No single database can do everything well. The future is about **composing specialized tools** into a cohesive data system, using event logs as the glue.

### The Unbundled Database

A traditional database bundles many features: storage, indexing, query processing, replication, transactions. The modern approach **unbundles** these:

```
System of Record (PostgreSQL)
    ↓ CDC / Event Log (Kafka)
    ├── Search (Elasticsearch)
    ├── Cache (Redis)
    ├── Analytics (Snowflake)
    ├── ML Features (Feature Store)
    └── Recommendations (Vector DB)
```

**Each component is specialized** and can be developed, scaled, and maintained independently. The event log is the **single source of truth** that ties everything together.

### Lambda vs. Kappa Architecture

| | Lambda | Kappa |
|-|--------|-------|
| **Idea** | Run batch + stream in parallel | Use stream processing for everything |
| **Batch layer** | Reprocesses all historical data for accuracy | Replay the event log when needed |
| **Complexity** | High (maintain two systems) | Lower (one system) |
| **2026 verdict** | Still used for ML pipelines | Preferred for most real-time systems |

### End-to-End Correctness

**The key insight**: Exactly-once processing doesn't exist at the database level alone. You need it **end-to-end**: from the client, through the message broker, through the processor, to the output.

**How to achieve it:**
1. **Idempotent operations**: Same operation applied twice = same result
2. **Unique operation IDs**: Client generates UUID, passes it through entire pipeline
3. **Exactly-once delivery**: Kafka transactions + idempotent producers

**Production insight**: The hardest bugs are the ones where you process a payment twice, or send a notification twice. Always design with idempotency keys. Stripe does this — every API call has an `Idempotency-Key` header.

### Data Ethics and Privacy (2026 Addition)

This wasn't a major chapter in 2017 but is critical now:
- **GDPR, CCPA, DMA**: You must be able to delete a user's data from ALL derived systems
- **Right to explanation**: If AI makes a decision, you may need an audit trail (event sourcing helps)
- **Data lineage**: Where did this data come from? Who transformed it? Tools: DataHub, OpenLineage
- **PII handling**: Encryption at rest, in transit, and in use. Tokenization for sensitive fields.
