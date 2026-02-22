# 🏗️ 10 System Design Case Studies Applying DDIA Concepts

> Each case study maps directly to DDIA concepts. Build the mental bridge from theory to practice.

---

## Case Study 1: Design Twitter's Home Timeline


### Problem
- 500M users, 200M DAU
- Users follow other users, see their tweets in reverse-chronological order
- Celebrities have 50M+ followers
- Timeline must load in <200ms

### DDIA Concepts Applied

**Replication (Ch 5)**: Timeline data replicated across regions for low-latency reads.

**Partitioning (Ch 6)**: Tweets partitioned by user_id (author). Timelines are per-user, so each user's timeline is on one partition.

**Stream Processing (Ch 11)**: When a user tweets, a fan-out service pushes the tweet to all followers' timelines in real-time.

### Architecture

```
User tweets → Tweet Service (write to tweets table)
                    ↓
              Kafka (TweetCreated event)
                    ↓
              Fan-out Service
              ├── Normal users: push tweet to all followers' timeline caches (write-time fan-out)
              └── Celebrities (50M+ followers): do NOT fan-out at write time
                    ↓
              Timeline Cache (Redis)
                    ↓
              On read: merge cached timeline + celebrity tweets (read-time fan-out for celebrities)
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Write-time fan-out for normal users | Fast reads, expensive writes for high-follower users | Ch 6 (hot spots) |
| Read-time fan-out for celebrities | Slower reads, but avoids 50M writes per tweet | Ch 6, Ch 11 |
| Redis for timeline cache | Fast but limited by memory, not durable | Ch 3 |
| Eventual consistency on timeline | User sees tweet 1-2 sec late, acceptable | Ch 9 |

---

## Case Study 2: Design a Payment System (Like Stripe)

### Problem
- Millions of transactions per day
- ZERO tolerance for double-charging or lost payments
- Must handle retries safely
- Must support refunds, disputes, chargebacks

### DDIA Concepts Applied

**Transactions (Ch 7)**: Every payment MUST be atomic. Charge + record creation = one transaction or neither.

**Idempotency (Ch 12)**: Client sends Idempotency-Key. Retry = same result, not double charge.

**Event Sourcing (Ch 11)**: Every financial event is an immutable entry. Current balance is derived.

### Architecture

```
Client → API Gateway (with Idempotency-Key header)
            ↓
         Payment Service
            ↓
         ┌─────────────────────────────────────┐
         │  PostgreSQL (system of record)       │
         │  - payments table (ACID)             │
         │  - idempotency_keys table            │
         │  - ledger entries (append-only)       │
         └─────────────────────────────────────┘
            ↓ (CDC via Debezium)
         Kafka
         ├── → Analytics (Snowflake)
         ├── → Notification Service (email receipts)
         ├── → Fraud Detection (Flink, real-time scoring)
         └── → Reconciliation Service (daily batch)
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| PostgreSQL with Serializable isolation | Strongest correctness, lower throughput | Ch 7 |
| Idempotency keys | Extra storage + lookup per request, but safe retries | Ch 12 |
| Event sourcing for ledger | Can reconstruct any historical state, but storage grows | Ch 11 |
| Async notifications | User doesn't wait for email, but might not get it instantly | Ch 11 |

---

## Case Study 3: Design a Real-Time Ride Matching System (Uber)

### Problem
- Match riders with nearby drivers in <5 seconds
- Handle millions of concurrent rides
- Driver location updates every 4 seconds
- Must not assign one driver to two riders

### DDIA Concepts Applied

**Stream Processing (Ch 11)**: Driver locations are a continuous event stream.

**Partitioning (Ch 6)**: Geospatial partitioning by grid cell (geohash).

**Transactions (Ch 7)**: Driver assignment must be atomic (prevent double-booking).

### Architecture

```
Driver App → Location Service → Kafka (driver_locations topic)
                                        ↓
                                  Location Index (Redis GeoHash)
                                  
Rider App → Matching Service → Query nearby drivers from Redis
                              → Lock driver (Redis SETNX or PostgreSQL SELECT FOR UPDATE)
                              → Create trip record
                              → Notify driver + rider
                                        ↓
                                  Trip Service (PostgreSQL)
                                        ↓ (CDC)
                                  Kafka → Analytics / Pricing / ETA
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Redis for driver locations | Blazing fast geo-queries, but data can be lost on crash | Ch 3, Ch 5 |
| Geohash partitioning | Efficient spatial queries, but rebalancing on boundary changes | Ch 6 |
| Distributed lock for driver assignment | Prevents double-booking, adds latency | Ch 7, Ch 9 |
| Eventual consistency for location | 4-sec-old location is fine for matching | Ch 9 |

---

## Case Study 4: Design a Distributed Key-Value Store (Like DynamoDB)

### Problem
- Store billions of key-value pairs
- Single-digit millisecond reads and writes
- 99.99% availability
- Automatic horizontal scaling

### DDIA Concepts Applied

**Leaderless Replication (Ch 5)**: Any node can accept reads/writes.

**Consistent Hashing (Ch 6)**: Keys distributed across nodes via hash ring.

**Quorum (Ch 5)**: W + R > N for consistency guarantees.

### Architecture

```
Client SDK → Hash(key) → Determine responsible nodes on hash ring
              ↓
         Write to W nodes in parallel
         Read from R nodes in parallel
              ↓
         Response to client (latest version wins)
         
Background:
  - Anti-entropy: Merkle trees detect divergence between replicas
  - Hinted handoff: If target node is down, neighbor stores temporarily
  - Gossip protocol: Nodes discover each other and detect failures
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Leaderless with quorum | High availability, eventual consistency | Ch 5 |
| Consistent hashing | Minimal data movement when nodes join/leave | Ch 6 |
| Last-Write-Wins for conflict | Simple but can lose concurrent updates | Ch 5 |
| N=3, W=2, R=2 | Balance of consistency and availability | Ch 5 |

---

## Case Study 5: Design a Search Engine (Like Elasticsearch)

### Problem
- Index billions of documents
- Full-text search with relevance ranking
- Sub-second query latency
- Near-real-time indexing (document appears in search within 1 sec)

### DDIA Concepts Applied

**Storage Engine (Ch 3)**: Inverted index (variant of LSM-tree for term lookups).

**Partitioning (Ch 6)**: Index sharded across nodes. Queries scatter/gather.

**Batch Processing (Ch 10)**: Initial index build is a batch job.

### Architecture

```
Documents → Kafka → Indexing Service
                         ↓
                    ┌──────────────────────────────┐
                    │  Elasticsearch Cluster         │
                    │  ┌─────┐ ┌─────┐ ┌─────┐     │
                    │  │Shard│ │Shard│ │Shard│      │
                    │  │  0  │ │  1  │ │  2  │      │
                    │  └─────┘ └─────┘ └─────┘     │
                    │  Each shard = inverted index   │
                    │  Each shard has replicas        │
                    └──────────────────────────────┘
                              ↑
Search Query → Coordinator Node → Fan-out to all shards
                                → Each shard returns top-K
                                → Coordinator merges and re-ranks
                                → Return final results
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Scatter/gather query pattern | All shards queried = tail latency amplification | Ch 6 |
| Near-real-time indexing (1 sec refresh) | Faster search freshness, but more I/O | Ch 11 |
| Inverted index (LSM-based) | Fast writes (append), periodic merge/compaction | Ch 3 |
| Replication per shard | Read throughput + fault tolerance, more storage | Ch 5 |

---

## Case Study 6: Design a Chat System (WhatsApp/Slack)

### Problem
- 1-on-1 and group messaging
- Messages must be delivered in order
- Offline users should receive messages when they come back online
- End-to-end encryption (E2EE)

### DDIA Concepts Applied

**Partitioning (Ch 6)**: Messages partitioned by conversation_id.

**Replication (Ch 5)**: Messages replicated for durability.

**Ordering (Ch 9)**: Causal consistency within a conversation.

### Architecture

```
Sender → WebSocket Gateway → Message Service
                                   ↓
                              Kafka (partitioned by conversation_id)
                              → guarantees ordering within conversation
                                   ↓
                              Message Storage (Cassandra)
                              → partition key = conversation_id
                              → clustering key = message_timestamp
                                   ↓
                              Delivery Service
                              ├── Recipient ONLINE → Push via WebSocket
                              └── Recipient OFFLINE → Store in "pending" queue
                                                    → Push notification (APNs/FCM)
                                                    → Deliver when recipient reconnects
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Kafka partitioned by conversation_id | Ordering guaranteed within conversation, not across | Ch 6, Ch 9 |
| Cassandra for storage | Writes are fast (LSM), wide-row model fits chat well | Ch 3 |
| WebSocket for real-time delivery | Persistent connection, but connection management is complex | Ch 11 |
| Eventual consistency for group chats | Members might see messages in slightly different order | Ch 9 |

---

## Case Study 7: Design a Video Streaming Platform (Netflix/YouTube)

### Problem
- 200M+ users streaming simultaneously
- Video must be available in multiple resolutions/codecs
- Sub-second startup time
- Global availability

### DDIA Concepts Applied

**Batch Processing (Ch 10)**: Video encoding is a massive batch job.

**Partitioning (Ch 6)**: Video content distributed across CDN edge nodes.

**Replication (Ch 5)**: Popular content replicated to more CDN locations.

### Architecture

```
Upload:
  Creator → Upload Service → S3 (raw video)
                                 ↓
                           Encoding Pipeline (batch job on Kubernetes)
                           → Encode to 5+ resolutions (4K, 1080p, 720p, 480p, 360p)
                           → Encode to multiple codecs (H.264, H.265, AV1, VP9)
                           → Generate adaptive bitrate manifests (HLS/DASH)
                                 ↓
                           CDN Origin (S3/Cloud Storage)
                           → Distribute to CDN edge nodes globally

Playback:
  User → CDN Edge (nearest server) → Stream video chunks
         ↑ Adaptive bitrate: client adjusts quality based on bandwidth
         
  User → API → Recommendation Service (ML model, batch-trained on Spark)
              → Viewing History (Cassandra, partitioned by user_id)
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Batch transcoding | Can take minutes/hours, but done once per video | Ch 10 |
| CDN edge caching | Low latency globally, but cache invalidation for takedowns | Ch 3, Ch 5 |
| Multiple resolutions | Storage cost multiplied, but every user gets good experience | Ch 10 |
| Eventual consistency for view counts | Counts might be seconds behind, but no one notices | Ch 9 |

---

## Case Study 8: Design a Fraud Detection System

### Problem
- Detect fraudulent transactions in <100ms
- Process millions of transactions per second
- Minimize false positives (blocking legitimate users)
- Learn from new fraud patterns quickly

### DDIA Concepts Applied

**Stream Processing (Ch 11)**: Real-time scoring of every transaction.

**CEP (Ch 11)**: Complex event patterns (3 failed logins + purchase from new country).

**Batch Processing (Ch 10)**: Model training on historical data.

### Architecture

```
Transaction → API Gateway → Kafka (transactions topic)
                                    ↓
              ┌──────────────── Flink Stream Processor ──────────────────┐
              │  1. Enrich: lookup user profile, device fingerprint      │
              │  2. Feature extraction: velocity, geolocation, amount    │
              │  3. Rule engine: hard rules (blocked countries, limits)  │
              │  4. ML scoring: run model inference (fraud probability)  │
              │  5. Decision: approve / flag / block                     │
              └─────────────────────────────────────────────────────────┘
                    ↓                           ↓
              APPROVE → continue            FLAG/BLOCK → 
              transaction                   alert team + 
                                           hold transaction
                                           
Batch layer:
  Historical transactions (S3/Parquet) → Spark → Train ML model → Deploy to Flink
  Retrain weekly with new labeled fraud data
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| Flink for real-time scoring | Sub-100ms latency, but complex to operate | Ch 11 |
| Kafka for event bus | Durable, replayable, ordered | Ch 11 |
| Batch-trained model | Learns from full dataset, but stale by days | Ch 10 |
| Hard rules + ML hybrid | Rules catch known fraud, ML catches novel patterns | Ch 11 |

---

## Case Study 9: Design a Distributed Task Queue (Like Celery/SQS)

### Problem
- Process millions of background tasks (emails, image processing, report generation)
- At-least-once delivery
- No task should be processed twice (or design for idempotency)
- Priority queues
- Dead letter queue for failed tasks

### DDIA Concepts Applied

**Message Brokers (Ch 11)**: Core building block — producer/consumer pattern.

**Exactly-once Semantics (Ch 12)**: Idempotency keys for safe retries.

**Partitioning (Ch 6)**: Tasks distributed across workers.

### Architecture

```
Producer Service → Kafka / SQS (task queue, partitioned by task type)
                        ↓
                  Worker Pool (Kubernetes pods, auto-scaled)
                  ├── Worker 1: processes email tasks
                  ├── Worker 2: processes image processing tasks
                  └── Worker 3: processes report generation tasks
                        ↓
                  On success: ACK, mark task complete
                  On failure: retry (up to 3x) → move to DLQ
                        ↓
                  Dead Letter Queue (DLQ)
                  → Monitored, alerting, manual review
                  
Observability:
  → Task throughput, latency, failure rate
  → Queue depth (backlog alert if growing)
  → Worker utilization
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| At-least-once with idempotency | Simpler than exactly-once, safe with idempotency keys | Ch 12 |
| Partition by task type | Isolates failures, but uneven load distribution possible | Ch 6 |
| Visibility timeout (SQS-style) | Prevents task starvation, but task might be processed twice if worker is slow | Ch 11 |
| DLQ for poison pills | Failed tasks don't block the queue | Ch 11 |

---

## Case Study 10: Design a Real-Time Analytics Dashboard (Like Datadog)

### Problem
- Ingest millions of metrics per second from thousands of services
- Query: "What's the P99 latency of service X in the last 5 minutes?"
- Sub-second query response
- 30-day retention at per-second granularity, 1-year at per-minute granularity

### DDIA Concepts Applied

**Stream Processing (Ch 11)**: Pre-aggregate metrics as they arrive.

**Column-Oriented Storage (Ch 3)**: Analytics queries scan specific columns over time ranges.

**Partitioning (Ch 6)**: Metrics partitioned by service + time range.

### Architecture

```
Services → OpenTelemetry Agent → Kafka (metrics topic)
                                        ↓
                                  Flink (stream processor)
                                  ├── Pre-aggregate: compute 1-min rollups in real-time
                                  ├── Store raw data → ClickHouse (hot storage, 30 days)
                                  └── Store rollups → ClickHouse (warm storage, 1 year)
                                  
Dashboard → Query Service → ClickHouse
            ↓
            "P99(latency) WHERE service='api' AND time > now()-5m"
            → ClickHouse scans only the 'latency' column for 'api' partition
            → Sub-second response

Downsampling job (batch):
  Raw data (30 days → expire) → 1-min rollups (1 year → expire) → 1-hour rollups (forever)
```

### Key Tradeoffs

| Decision | Tradeoff | DDIA Chapter |
|----------|---------|-------------|
| ClickHouse for storage | Insanely fast column scans, but not great for point lookups | Ch 3 |
| Pre-aggregation in Flink | Reduces query-time computation, but loses some granularity | Ch 11 |
| Time-based partitioning | Efficient time-range queries, easy to expire old data | Ch 6 |
| Downsampling for retention | Saves storage, but you lose second-level precision after 30 days | Ch 10 |

---

## Cross-Cutting Patterns Across All Case Studies

| Pattern | Used In | DDIA Chapter |
|---------|---------|-------------|
| Kafka as event bus | ALL of them | Ch 11 |
| CDC (database → Kafka) | Payment, search, e-commerce | Ch 11 |
| Partitioning by primary key | ALL of them | Ch 6 |
| Cache-aside with Redis | Twitter, Uber, chat | Ch 3 |
| Batch for ML + Stream for serving | Fraud, Netflix, ads | Ch 10, 11 |
| Idempotency keys | Payment, task queue | Ch 12 |
| Saga pattern | Payment, e-commerce | Ch 7 |
| Scatter/gather queries | Search, analytics | Ch 6 |
| Column-oriented for analytics | Dashboard, analytics | Ch 3 |
| Dead letter queues | Task queue, streaming | Ch 11 |
