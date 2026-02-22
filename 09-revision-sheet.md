# 📝 One-Page Revision Sheet — DDIA in 5 Minutes

> Print this. Read it before your interview. This is the entire book compressed.

---

## The Three Pillars
**Reliability** (survive faults) · **Scalability** (handle growth: use percentiles, not averages) · **Maintainability** (operable, simple, evolvable)

## Data Models
**Relational** (JOINs, ACID) → PostgreSQL | **Document** (flexible, self-contained) → MongoDB, DynamoDB | **Graph** (relationships) → Neo4j | **Vector** (AI/embeddings) → Pinecone, pgvector


## Storage Engines
**B-Tree**: read-optimized, in-place updates, O(log n) — PostgreSQL, MySQL
**LSM-Tree**: write-optimized, append-only, compaction — Cassandra, RocksDB
**Column-oriented**: OLAP, scan-heavy, great compression — Snowflake, BigQuery, ClickHouse

## Encoding
JSON (human-readable, slow) · Protobuf (gRPC, fast, schema) · Avro (Kafka, schema registry) · Parquet (analytics, columnar)
**Rule**: Add optional fields = safe. Remove/rename fields = dangerous. Use field numbers, not names.

## Replication
| Type | Writes | Read Scaling | Consistency | Example |
|------|--------|-------------|-------------|---------|
| **Single-leader** | 1 node | ✅ Replicas | Strong (leader) | RDS, Cloud SQL |
| **Multi-leader** | N nodes | ✅ | Conflict resolution | CockroachDB, Spanner |
| **Leaderless** | Any node | ✅ | Eventual (quorum) | DynamoDB, Cassandra |

**Quorum**: W + R > N. Failover risk: split brain, data loss. Replication lag → read-your-writes, monotonic reads.

## Partitioning (Sharding)
**Hash** (even distribution, no range queries) vs **Range** (range queries, hot spots)
Hot spots: salt hot keys. Rebalance: over-partition from start (1000 partitions for 10 nodes).
Secondary indexes: **local** (fast writes, scatter reads) vs **global** (fast reads, slow writes).

## Transactions
**ACID**: Atomicity (all-or-nothing), Consistency (app invariants), Isolation (concurrency), Durability (crash-safe)

| Isolation Level | Prevents | Allows | Default In |
|----------------|----------|--------|-----------|
| Read Committed | Dirty reads/writes | Read skew, lost updates | PostgreSQL |
| Snapshot (MVCC) | + Read skew | Lost updates, write skew | PG explicit |
| Serializable | Everything | Nothing (slowest) | CockroachDB |

Lost updates → atomic ops / CAS. Write skew → serializable. Distributed txns → **Saga pattern** (not 2PC).

## Distributed Systems Troubles
**Networks**: unreliable, use timeouts + retries + idempotency
**Clocks**: NTP can jump, use monotonic clocks for duration, logical clocks for ordering
**Processes**: GC pauses, VM migration — use fencing tokens for lease safety

## Consistency & Consensus
**Linearizable** (strongest, one copy) → etcd, Spanner | **Eventual** (weakest, fast) → DynamoDB, Cassandra
**CAP simplified**: During partition → pick consistency OR availability. Real tradeoff → latency vs consistency.
**Raft/Paxos**: Leader-based consensus, majority vote, log replication. Used by etcd, CockroachDB, ZooKeeper.

## Batch Processing
Unix pipes → MapReduce → **Spark** → **dbt + SQL Warehouses** (2026)
Key: idempotent jobs, replayable from source. Parquet = universal format. Joins: sort-merge, broadcast hash.

## Stream Processing
**Kafka** = durable, ordered, replayable log. Partitioned by key (ordering unit).
**CDC**: Database → Debezium → Kafka → derived stores (Elasticsearch, Redis, Snowflake)
**Event Sourcing**: Store events, derive state. Windowing: tumbling, hopping, sliding, session.
**Flink** for complex streaming. **Kafka Streams** for simpler, library-based streaming.

## The Modern Architecture Pattern (2026)
```
System of Record (PostgreSQL) → CDC → Kafka → { Search, Cache, Analytics, ML, Notifications }
```
Batch (Spark/dbt) for training + analytics. Stream (Flink) for real-time. Cache (Redis) for hot reads.

## 2026 Must-Know Tools
| Category | Tools |
|----------|-------|
| OLTP | PostgreSQL, CockroachDB, DynamoDB, PlanetScale |
| OLAP | Snowflake, BigQuery, ClickHouse, Redshift |
| Streaming | Kafka, Flink, Redpanda, Kinesis |
| Cache | Redis, Memcached, CDN edge |
| Search | Elasticsearch, Meilisearch |
| Vector | Pinecone, pgvector, Weaviate |
| Lakehouse | Iceberg, Delta Lake, Hudi |
| Orchestration | Airflow, Dagster, Prefect |
| Schema | Confluent Schema Registry, Buf, Protobuf |
| Observability | OpenTelemetry, Grafana, Datadog |

## Decision Shortcuts
- **Database?** → Complex queries = PostgreSQL. Key-value = DynamoDB. Search = Elasticsearch. Analytics = Snowflake.
- **Consistency?** → Money = strong. Social feed = eventual. Inventory = strong at checkout, eventual for display.
- **Cache?** → Repeated reads + staleness OK = cache-aside + Redis + TTL. Invalidate via CDC.
- **Queue?** → Decouple services, handle spikes, async work = Kafka/SQS. Critical path stays synchronous.
- **Shard?** → Only when single DB can't handle it. Over-partition. Hash by primary access key.

## The One Sentence
> **Choose the RIGHT tradeoffs: replicate for availability, partition for throughput, transact for correctness, stream for reactivity, batch for completeness — and always design for failure.**
