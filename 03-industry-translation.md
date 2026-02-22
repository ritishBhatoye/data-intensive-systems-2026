# 🏭 Industry Translation — DDIA Concepts → Real FAANG Systems

> Every concept from the book, mapped to real production systems at scale.

---

## Concept-to-System Mapping

### Chapter 1: Reliability, Scalability, Maintainability

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Fault tolerance via redundancy | Netflix runs in 3 AWS regions. If us-east-1 dies, traffic shifts to us-west-2 in seconds |
| Chaos engineering | Netflix Chaos Monkey randomly kills instances in production. Amazon GameDay simulates full region failures |
| Percentile-based SLOs | Google SRE defines SLOs like "p99 latency < 200ms for 99.9% of 5-minute windows" |
| Horizontal scaling | Instagram went from 1 server to thousands of sharded PostgreSQL instances |
| Operational simplicity | Uber moved from microservices chaos back to a domain-oriented architecture to reduce complexity |

### Chapter 2: Data Models

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Relational model | Stripe uses PostgreSQL heavily for financial transactions (ACID matters for money) |
| Document model | MongoDB is used by Forbes, Toyota, and many mid-stage startups for content management |
| Graph model | Facebook's social graph is a custom graph DB (TAO). LinkedIn uses graph for "People You May Know" |
| Polyglot persistence | Uber uses MySQL for trips, Redis for caching, Elasticsearch for search, Kafka for events, and a custom graph for mapping |
| Schema evolution | Airbnb migrated from a monolithic MySQL to service-oriented PostgreSQL instances with careful schema evolution using Protobuf |

### Chapter 3: Storage & Retrieval

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| B-Tree indexes | PostgreSQL's default index type. Every `CREATE INDEX` on RDS is a B-Tree |
| LSM-Tree storage | Cassandra (used by Discord for trillion+ messages), RocksDB (used as embedded engine in CockroachDB, Kafka Streams) |
| Write-Ahead Log | Every PostgreSQL write goes to WAL first. Aurora replicates the WAL across 3 AZs, 6 copies |
| Column-oriented storage | Snowflake, BigQuery, Redshift — all column stores internally. Data stored in Parquet/ORC format |
| Data warehouse (star schema) | Spotify's data warehouse: fact table = every stream event, dimension tables = users, songs, artists, countries |
| Bloom filters | Cassandra uses bloom filters to avoid unnecessary disk reads (probabilistic check: "is this key possibly here?") |

### Chapter 4: Encoding & Evolution

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Protocol Buffers | Google uses Protobuf universally. gRPC = HTTP/2 + Protobuf. Used by all Google services |
| Avro for data pipelines | LinkedIn (who created Avro) uses it with Kafka. Confluent Schema Registry enforces Avro compatibility |
| Schema registry | Confluent Schema Registry at Netflix validates every Kafka message against registered schemas |
| Rolling deployments | Meta deploys code to billions of users using gradual rollout: 0.1% → 1% → 10% → 50% → 100% |
| Backward/forward compatibility | Stripe's API versioning: every API request includes a version header. Old clients keep working for years |

### Chapter 5: Replication

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Single-leader replication | Amazon RDS: primary instance + read replicas. Writes go to primary, reads can go to replicas |
| Async replication | Most MySQL/PostgreSQL read replicas are async. GitHub experienced a replication lag incident in 2018 that caused data inconsistency |
| Multi-leader replication | Google Docs: each user's browser is a "leader" that accepts local edits, then syncs via OT/CRDT |
| Leaderless replication | Amazon DynamoDB: writes go to multiple nodes, reads do quorum reads. Shopping cart is the classic use case |
| Failover automation | AWS Aurora: automatic failover in <30 seconds. The DNS endpoint switches to the new primary |
| CRDTs | Figma uses CRDTs for real-time collaborative design. Apple Notes uses CRDTs for cross-device sync |

### Chapter 6: Partitioning

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Hash partitioning | DynamoDB: you choose a partition key, DynamoDB hashes it to determine which partition stores the item |
| Range partitioning | Google Bigtable/HBase: rows sorted by key. Useful for time-series (row key = `device_id#timestamp`) |
| Hot spots | Justin Bieber tweets → millions of fan-out writes to one partition. Twitter solved this with celebrity fan-out at read time |
| Scatter/gather queries | Elasticsearch: search query goes to ALL shards, each returns top-K results, coordinator merges them |
| Rebalancing | Kafka: adding a broker requires partition reassignment. Tools like Confluent's Cruise Control automate this |
| Consistent hashing | DynamoDB, Cassandra use consistent hashing rings. Adding a node only moves ~1/N of the data |

### Chapter 7: Transactions

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| ACID transactions | Stripe: every payment must be atomic. Charge the card AND create the record, or neither |
| Read committed | PostgreSQL default. Most web apps run at this level and it's fine for 90% of use cases |
| Snapshot isolation (MVCC) | PostgreSQL MVCC: each transaction sees a consistent snapshot. Readers never block writers |
| Serializable isolation | CockroachDB uses Serializable Snapshot Isolation (SSI) as its default — no weaker option available |
| Saga pattern (distributed transactions) | Uber's trip lifecycle: create trip → dispatch driver → charge rider → pay driver. Each step is compensatable |
| Optimistic concurrency control | DynamoDB conditional writes: `PutItem` with `ConditionExpression` ensures no lost updates |

### Chapter 8: Distributed Systems Troubles

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Network partitions | AWS us-east-1 has had multiple network partition events. In 2017, an S3 outage cascaded to break half the internet |
| Timeouts & retries | AWS SDK has built-in exponential backoff with jitter. Stripe's API clients implement idempotent retries |
| Clock skew | Google Spanner uses TrueTime (GPS + atomic clocks) to bound clock uncertainty to ~7ms globally |
| GC pauses | Discord moved from Go to Rust partly to eliminate GC pauses in their message storage service |
| Split brain | Elasticsearch has had split-brain issues in production. Solution: minimum_master_nodes = (N/2)+1 |
| Circuit breakers | Netflix Hystrix (now Resilience4j): if a downstream service fails, stop calling it temporarily |

### Chapter 9: Consistency & Consensus

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| Linearizability | Google Spanner: globally linearizable reads and writes using TrueTime. Used for Google Ads billing |
| Eventual consistency | DynamoDB default. Social media feeds, product catalogs — stale data for a few seconds is OK |
| Raft consensus | etcd (Kubernetes' brain) uses Raft. CockroachDB uses Raft for every range (partition) |
| ZooKeeper/etcd | Kafka uses ZooKeeper for controller election (migrating to Raft-based KRaft). Kubernetes uses etcd for all state |
| Leader election | Redis Sentinel for Redis HA. Kubernetes controller manager leader election via etcd leases |
| Distributed locks | Redlock (Redis-based distributed locking). DynamoDB Lock Client. etcd distributed locks |

### Chapters 10-12: Data Processing

| DDIA Concept | Real-World Implementation |
|-------------|--------------------------|
| MapReduce (batch) | Google originally built Search index with MapReduce. Meta still uses Hive (MapReduce under the hood) for some ETL |
| Spark (modern batch) | Netflix uses Spark for recommendation model training on 200M+ user profiles |
| dbt (SQL transformations) | Spotify, GitLab, and thousands of companies use dbt to transform data in their warehouses |
| Kafka (event streaming) | LinkedIn processes 7+ trillion messages/day through Kafka. Uber uses Kafka for trip events |
| Flink (stream processing) | Alibaba uses Flink to process Singles' Day transactions in real-time (billions of events) |
| CDC (Change Data Capture) | Shopify uses Debezium + Kafka to replicate MySQL changes to their analytics data warehouse |
| Event sourcing | Stripe stores every financial event as an immutable log. Bank account balances are derived views |
| Lambda architecture | Netflix: batch layer (Spark on S3) for accurate recommendations + stream layer (Flink) for real-time personalization |

---


## Architecture Patterns at FAANG Scale

### Netflix: The Event-Driven Giant

```
User Action → API Gateway → Microservice
                                 ↓
                          Kafka (Event Bus)
                         ↙    ↓    ↘
                   Flink     Spark    Elasticsearch
                  (real-time) (batch)  (search)
                       ↓       ↓          ↓
                   Cassandra  S3/Iceberg  Redis
                   (viewing   (data lake) (cache)
                    history)
```

### Uber: The Polyglot Beast

```
Rider App → API Gateway → Trip Service (MySQL)
                              ↓
                         Kafka Topics
                        ↙    ↓    ↘
              Pricing     Matching    Analytics
             (Redis)     (custom)    (Spark → Hive)
                ↓           ↓            ↓
            Surge maps   Driver    Data Warehouse
                        assignment  (Presto/Trino)
```

### Stripe: The Correctness Obsessed

```
Payment API → Idempotency Layer → PostgreSQL (system of record)
                                       ↓
                                  CDC (Debezium)
                                       ↓
                                  Kafka Topics
                                 ↙    ↓    ↘
                           Search   Analytics  Notifications
                        (Elastic) (Trino/PB)   (async)
```

---

## The Pattern

Every FAANG-scale system follows this blueprint:
1. **System of Record**: One authoritative database (usually relational)
2. **Event Bus**: Kafka as the central nervous system
3. **Derived Stores**: Specialized databases fed by CDC/events
4. **Processing Layers**: Batch (Spark/dbt) + Stream (Flink/Kafka Streams)
5. **Caching Layer**: Redis/Memcached for hot data
6. **Search Layer**: Elasticsearch for full-text search
7. **Observability**: Metrics, logs, traces (OpenTelemetry, Datadog, Grafana)
