# 🎯 Interview Cheat Sheet — System Design with DDIA

> Everything you need to nail FAANG system design interviews, organized by DDIA concepts.

---


## The Framework: How to Answer Any System Design Question

```
Step 1: REQUIREMENTS (3-5 min)
────────────────────────────────
• Functional requirements (what does it do?)
• Non-functional requirements (scale, latency, consistency, availability)
• Back-of-envelope estimation (QPS, storage, bandwidth)

Step 2: HIGH-LEVEL DESIGN (5-10 min)
────────────────────────────────────
• APIs (REST/gRPC endpoints)
• Data model (SQL vs NoSQL, schema)
• Architecture diagram (services, databases, caches, queues)

Step 3: DEEP DIVE (15-20 min)
────────────────────────────────
• Pick 2-3 interesting components to dive deep
• Discuss tradeoffs explicitly
• Reference DDIA concepts by name

Step 4: BOTTLENECKS & IMPROVEMENTS (5 min)
──────────────────────────────────────────
• Single points of failure
• Hot spots
• Scaling strategies
• Monitoring and alerting
```

---

## The DDIA Toolkit — Concepts You'll Use in EVERY Interview

### 1. Database Selection (Chapter 2-3)

**Interviewer asks**: "What database would you use?"

| If You Need... | Choose | Why |
|----------------|--------|-----|
| Complex queries, JOINs, transactions | PostgreSQL / CockroachDB | ACID, mature ecosystem |
| Key-value at scale, single-digit ms | DynamoDB / Redis | Horizontal scaling, simple access patterns |
| Full-text search | Elasticsearch / OpenSearch | Inverted indexes, relevance scoring |
| Time-series data | InfluxDB / TimescaleDB | Optimized for time-based writes and queries |
| Graph traversals | Neo4j / Neptune | When relationships ARE the data |
| Analytics / warehousing | Snowflake / BigQuery | Column-oriented, MPP |
| Real-time analytics | ClickHouse / Druid / Pinot | Sub-second aggregations on streaming data |
| Vector similarity | Pinecone / pgvector | AI features, semantic search |

**Pro move**: State your choice AND the tradeoff. "I'd use DynamoDB for user sessions because we need single-digit millisecond latency and don't need complex queries. The tradeoff is we lose JOINs, but session data is self-contained."

### 2. Scaling Strategy (Chapters 1, 5, 6)

**Interviewer asks**: "How does this scale to 100M users?"

```
Step 1: Read replicas (Chapter 5)
  Primary DB → Read replicas (most reads don't need real-time consistency)
  
Step 2: Caching layer (Chapter 3)
  App → Redis/Memcached → DB (cache the hot 20% of data)
  
Step 3: Partitioning/Sharding (Chapter 6)
  Shard by user_id, order_id, or region
  Choose hash partitioning (even distribution) vs range partitioning (scan support)
  
Step 4: Async processing (Chapters 10-11)
  Move non-critical work to message queues (Kafka/SQS)
  Send email → Queue ✅   NOT  Send email → Synchronous API call ❌
  
Step 5: CDN + Edge
  Static content → CDN (CloudFront, Fastly)
  API responses → Edge caching where possible
```

### 3. Consistency vs. Availability (Chapter 9)

**Interviewer asks**: "What consistency model would you use?"

| System | Consistency Needed | Why |
|--------|-------------------|-----|
| Bank transfers | Strong (linearizable) | Money can't appear or disappear |
| Social media feed | Eventual | Seeing a post 2 seconds late is fine |
| E-commerce inventory | Strong for purchase, eventual for display | Show approximate count, but exactly verify at checkout |
| Ride matching | Strong-ish (serializable) | Can't assign one driver to two riders |
| Chat messages | Causal | Messages should appear in order within a conversation |
| Analytics dashboard | Eventual | Data can be minutes old |

### 4. Data Encoding (Chapter 4)

**Interviewer asks**: "How do services communicate?"

| Pattern | Use When | Example |
|---------|----------|---------|
| REST + JSON | Public APIs, low complexity | Stripe API, GitHub API |
| gRPC + Protobuf | Internal service-to-service, high throughput | Google internal, Uber |
| GraphQL | Client-controlled data fetching | GitHub v4 API, Shopify |
| Kafka + Avro | Async event streams | Data pipelines, event-driven systems |

### 5. Caching Patterns (Chapter 3)

**Interviewer asks**: "How do you handle read-heavy workloads?"

| Pattern | How | Use When |
|---------|-----|----------|
| **Cache-aside** | App checks cache first, falls back to DB, writes to cache | Most common. Simple. |
| **Write-through** | App writes to cache AND DB simultaneously | When cache must always be fresh |
| **Write-behind** | App writes to cache, cache async writes to DB | Write-heavy, can tolerate some data loss |
| **Read-through** | Cache fetches from DB on miss (cache manages itself) | Less app complexity |

**Cache invalidation strategies:**
```
TTL-based:     Set expiry time (simplest, stale data possible)
Event-based:   CDC → invalidate cache on DB change (strongest)
Write-through: Update cache on every write (consistent but slower writes)
```

**The golden rule**: "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton

### 6. Message Queues & Async (Chapter 11)

**Interviewer asks**: "What happens when this service goes down?"

```
WITHOUT queue:  User → API → Payment Service (⚡ if down, user sees error)

WITH queue:     User → API → Kafka → Payment Service
                              ↑
                         (buffered, retryable,
                          consumer can be down
                          temporarily)
```

**When to use queues:**
- Decoupling services (payment processing after order creation)
- Handling spikes (Black Friday traffic surge)
- Reliable delivery (notifications, emails)
- Fan-out (one event triggers multiple actions)

---

## Back-of-Envelope Estimation Cheat Sheet

### Numbers Every Engineer Should Know

```
Read 1 MB from RAM:         ~250 μs        (0.25 ms)
Read 1 MB from SSD:         ~1 ms
Read 1 MB from HDD:         ~20 ms
Send 1 MB over 1 Gbps:      ~10 ms
Round trip within datacenter: ~0.5 ms
Round trip CA → Netherlands: ~150 ms

QPS estimates:
1 server (single-threaded):  ~1K QPS
1 server (multi-threaded):   ~10-50K QPS
Redis:                       ~100K QPS per instance
Kafka partition:             ~10K msg/sec per partition
DynamoDB:                    ~25K WCU per table (soft limit)
```

### Quick Estimation Framework

```
Users:    100M monthly active users
DAU:      ~30M (30% of MAU)
QPS:      30M / 86400 ≈ 350 QPS average
Peak QPS: 350 × 3 ≈ 1,000 QPS (3x peak-to-average)
Storage:  100M users × 1KB per user = 100 GB
          + 30M daily events × 500B = 15 GB/day ≈ 5.5 TB/year
```

---

## Top 20 System Design Questions → DDIA Concepts

| # | Question | Key DDIA Concepts |
|---|----------|-------------------|
| 1 | Design a URL shortener | Hash partitioning, consistent hashing, caching |
| 2 | Design Twitter / X | Fan-out on read vs write, caching, partitioning, eventual consistency |
| 3 | Design a chat system (WhatsApp) | Stream processing, partitioning by conversation, WebSockets, message ordering |
| 4 | Design a news feed | Fan-out, caching, eventual consistency, ranking pipeline (batch + stream) |
| 5 | Design YouTube / Netflix | CDN, encoding formats, blob storage, recommendation system (batch ML) |
| 6 | Design Uber / Lyft | Geospatial partitioning, real-time stream processing, strong consistency for matching |
| 7 | Design a rate limiter | Redis counters, sliding window, distributed counting |
| 8 | Design a search engine | Inverted indexes (LSM-tree based), partitioning, MapReduce for index building |
| 9 | Design a key-value store | LSM-trees, partitioning, replication, consistency levels |
| 10 | Design a notification system | Message queues, fan-out, priority queues, rate limiting |
| 11 | Design Google Docs | CRDTs/OT, event sourcing, conflict resolution, multi-leader replication |
| 12 | Design a payment system | ACID transactions, idempotency keys, saga pattern, event sourcing |
| 13 | Design an ad click aggregator | Stream processing (Flink), exactly-once semantics, windowing |
| 14 | Design a metric monitoring system | Time-series DB, push vs pull, aggregation, alerting pipeline |
| 15 | Design Ticketmaster | Strong consistency (inventory), distributed locks, queue-based fairness |
| 16 | Design a web crawler | Batch processing, URL frontier queue, politeness constraints, deduplication |
| 17 | Design a file storage (Dropbox) | Chunking, CDC for sync, conflict resolution, metadata vs content storage |
| 18 | Design an e-commerce system | ACID for orders, eventual consistency for catalog, CDC to search index |
| 19 | Design a logging system | Stream processing, partitioning by service, compression, retention policies |
| 20 | Design a recommendation engine | Batch (model training on Spark), stream (real-time scoring), feature store |

---

## Killer Phrases for Interviews

Use these to demonstrate deep understanding:

| When Discussing... | Say This |
|-------------------|----------|
| Database choice | "We need to consider our access patterns. If we're mostly doing point lookups by ID, a key-value store like DynamoDB is ideal. But if we need complex joins, PostgreSQL gives us ACID and query flexibility." |
| Consistency | "I'd use eventual consistency for the feed since users can tolerate a few seconds of staleness. But for the payment flow, we need serializable transactions to prevent double-charging." |
| Partitioning | "We'll hash-partition by user_id to evenly distribute load. The tradeoff is we lose efficient range queries across users, but our dominant access pattern is per-user lookups." |
| Replication | "We'll use single-leader replication with async followers for read scaling. The risk is replication lag causing stale reads, so for read-after-write consistency, the user's own requests route to the primary." |
| Caching | "I'd use cache-aside with Redis and a TTL of 5 minutes. For cache invalidation on writes, we can use CDC from the primary database to Kafka, which triggers cache invalidation." |
| Batch vs Stream | "The recommendation model trains in batch on Spark because it needs the full dataset. But the real-time re-ranking uses a Flink stream processor reading user activity from Kafka." |
| Fault tolerance | "We need to handle this idempotently. The client generates a UUID for each request, and the server deduplicates using that ID. This way, retries are safe." |
| Tradeoffs | "This gives us strong consistency at the cost of latency. If we relaxed to eventual consistency, we'd get lower latency but risk showing stale data." |

---

## Red Flags — What NOT to Say

| ❌ Don't Say | ✅ Say Instead |
|-------------|---------------|
| "We'll just use MongoDB for everything" | "We'll choose the right database for each access pattern" |
| "We'll add more servers" | "We'll horizontally scale by partitioning on [specific key]" |
| "It'll be consistent" | "We'll use [specific isolation level] because [specific reason]" |
| "We'll cache everything" | "We'll cache the hot data with TTL and invalidate via CDC" |
| "Two-phase commit for distributed transactions" | "We'll use the Saga pattern with compensation logic" |
| "NoSQL is faster" | "NoSQL trades query flexibility for horizontal scalability" |
| "We'll figure out consistency later" | "We need to decide consistency requirements upfront because it affects our architecture" |
