# 📋 Executive Overview — DDIA in 2 Pages

> If you only have 10 minutes, read this. Everything else is depth.

---

## Page 1: The Big Picture

### What Is a Data-Intensive Application?

Most modern apps are **data-intensive**, not compute-intensive. Your bottleneck isn't CPU — it's **data**: storing it, moving it, querying it, and keeping it consistent across machines.

Every data system is built from 5 primitives:

| Primitive | What It Does | 2026 Tools |
|-----------|-------------|------------|
| **Database** | Store & retrieve | PostgreSQL, DynamoDB, CockroachDB, MongoDB |
| **Cache** | Speed up reads | Redis, Memcached, CDN edge caches |
| **Search Index** | Find by content | Elasticsearch, Meilisearch, Pinecone (vectors) |
| **Stream Processing** | React to events | Kafka, Flink, Pulsar, AWS Kinesis |
| **Batch Processing** | Crunch large datasets | Spark, dbt, Snowflake, BigQuery |

### The Three Pillars

Every system design conversation boils down to three qualities:

```
┌─────────────────────────────────────────────┐
│                                             │
│   RELIABILITY    →  "Does it keep working   │
│                      when things go wrong?" │
│                                             │
│   SCALABILITY    →  "Can it handle growth?" │
│                                             │
│   MAINTAINABILITY → "Can my team work on    │
│                      it without crying?"    │
│                                             │
└─────────────────────────────────────────────┘
```

- **Reliability** = fault tolerance. Hardware dies, software bugs exist, humans make mistakes. Build for it.
- **Scalability** = not "will it scale" but "what happens when load changes?" Think: 10x users, 100x data.
- **Maintainability** = operability + simplicity + evolvability. This is where 90% of the cost lives.

### The Two Worlds: OLTP vs OLAP

```
OLTP (Transactions)              OLAP (Analytics)
─────────────────                ────────────────
"Give me user #42's order"       "What's the avg order value in Q4?"
Low-latency, high-frequency      Scan-heavy, column-oriented
PostgreSQL, MySQL, DynamoDB      Snowflake, BigQuery, Redshift
Row-oriented storage             Column-oriented storage
Indexes matter enormously        Compression matters enormously
```

**Key insight**: Don't run analytics on your OLTP database. Extract → Transform → Load (ETL) into a warehouse.

---

## Page 2: Distributed Systems at a Glance

### Why Distribute?

You distribute data for three reasons: **geography** (put data near users), **availability** (survive failures), **throughput** (handle more load).

### Replication — Copies of the Same Data

```
Single-Leader          Multi-Leader          Leaderless
────────────           ────────────          ──────────
One writer, N readers  Multiple writers      Any node reads/writes
Simple, proven         Multi-datacenter OK   Eventually consistent
PostgreSQL, MySQL      CockroachDB           Cassandra, DynamoDB
Failover is tricky     Conflict resolution   Quorum: W + R > N
```

**Rule of thumb**: Start with single-leader. Go multi-leader only for multi-region. Go leaderless only for availability-first workloads.

### Partitioning (Sharding) — Splitting Data Across Nodes

```
By Key Range            By Hash of Key
──────────────          ────────────────
Good for scans          Even distribution
Hot spots possible      No range queries
(e.g., all "2024" data  Key gets scattered
 on one shard)          across partitions)
```

**Production insight**: Most teams use hash-based partitioning (DynamoDB partition key, Kafka partition key, Cassandra partition key). When you need range scans, you design a compound key.

### Transactions — Grouping Operations

**ACID is not binary — it's a spectrum:**
- **Read Committed** → no dirty reads/writes (default PostgreSQL)
- **Snapshot Isolation** → consistent reads within a transaction
- **Serializable** → strongest, slowest, safest

**When you need cross-service transactions** (you usually don't): use the Saga pattern, not 2PC.

### Consistency & Consensus

**The #1 insight**: You can't have both strong consistency AND high availability during a network partition (CAP theorem). But **CAP is misleading** — the real tradeoff is **latency vs. consistency**.

```
Strong consistency  ←─────────────────→  Eventual consistency
(linearizable)                            (available)
Banks, leader election                    Social feeds, caches
Slower                                    Faster
ZooKeeper, etcd                           DynamoDB, Cassandra
```

### Data Processing Paradigms

```
Online (Request/Response)    Batch (Offline)          Stream (Near-Real-Time)
─────────────────────────    ───────────────          ──────────────────────
API serves a user            Process all data at once Process events as they arrive
ms latency                   Hours latency            Seconds latency
REST/gRPC APIs               Spark, dbt               Kafka + Flink
"What's my balance?"         "Rebuild all reports"     "Fraud detection on swipe"
```

**2026 reality**: Most companies use Kafka as the central nervous system. Batch is still king for ML training and analytics. Stream handles everything real-time.

---

## The One-Sentence Summary

> **Data-intensive systems are about choosing the RIGHT tradeoffs between reliability, scalability, and maintainability — using replication for availability, partitioning for throughput, transactions for correctness, and stream/batch processing for data flow — all while accepting that distributed systems WILL fail and designing for that failure.**
