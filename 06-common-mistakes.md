# ⚠️ Common Mistakes Engineers Make

> The mistakes I've seen senior engineers make at FAANG-scale systems. Learn from them.


---

## Mistake 1: Choosing Eventual Consistency Without Understanding It

**What happens**: Team picks DynamoDB or Cassandra because "it scales." Then they're shocked when:
- A user updates their profile but sees the old data on refresh
- Two users edit the same document and one's changes disappear
- A payment is processed twice because the retry didn't see the first success

**The root cause**: "Eventual consistency" sounds harmless. In reality, it means **your system will temporarily be wrong**, and you must design for that.

**How to avoid it**:
- Map out EVERY read/write flow and ask: "What happens if this read returns stale data?"
- If the answer is "something bad," you need stronger consistency for that flow
- Use read-your-own-writes patterns (route reads to leader after writes)
- Default to strong consistency; opt into eventual consistency per use case

---

## Mistake 2: Ignoring Replication Lag

**What happens**: Team sets up PostgreSQL with read replicas. Routes all reads to replicas. User creates a post, gets redirected to their profile — post doesn't appear. User creates it again. Now there are two posts.

**The root cause**: Async replication can have seconds of lag. `SELECT * FROM posts WHERE user_id = ?` on a replica may not include the post that was just written to the primary.

**How to avoid it**:
- Route reads-after-writes to the primary for the writing user
- Use session stickiness (same user always reads from same replica)
- Monitor replication lag as a critical metric (set alerts at > 1 second)
- In critical flows (account creation, payment), ALWAYS read from primary

---

## Mistake 3: No Idempotency on Retries

**What happens**: Network timeout occurs between your service and the payment processor. You don't know if the charge went through. You retry. The user is charged twice.

**The root cause**: Not designing operations to be idempotent.

**How to avoid it**:
- Generate a unique request ID (UUID) for every operation
- Store it in your database before processing
- On retry, check if the request ID already exists
- Example: Stripe's `Idempotency-Key` header — send the same key, get the same result

```
Request 1: POST /charge {amount: $50, idempotency_key: "abc-123"}
  → Charge created, stored with key "abc-123"

Request 2 (retry): POST /charge {amount: $50, idempotency_key: "abc-123"}
  → Key "abc-123" already exists → Return same result, no double-charge
```

---

## Mistake 4: Sharding Too Late (or Too Early)

### Sharding Too Late
**What happens**: Database hits 1TB, queries slow down, single-server capacity maxed out. Now you need to shard a production database with zero downtime. This is a multi-quarter engineering project.

**Famous example**: Instagram had to migrate its PostgreSQL database mid-growth. Incredibly painful.

### Sharding Too Early
**What happens**: Team decides to shard at 10GB of data "for scale." Now every query needs a routing layer, cross-shard joins are impossible, and the team spends 80% of their time on distributed systems problems instead of product features.

**How to avoid it**:
- Start with a single database (PostgreSQL handles more than you think — often to 1-2TB)
- Design your schema so sharding is possible later (include the shard key in every table)
- Use managed databases that auto-scale (Aurora, PlanetScale, CockroachDB)
- Shard when you have evidence, not predictions

---

## Mistake 5: Treating the Cache as a Source of Truth

**What happens**: Team uses Redis as the primary read path. Redis goes down → application goes down. Or Redis data diverges from database → users see stale data → bad decisions made on stale data.

**The root cause**: Cache is not a database. It's a performance optimization layer that can be lost at any time.

**How to avoid it**:
- The database is ALWAYS the source of truth
- Application must work (slowly) if cache is empty
- Never write to cache without writing to database first (or use cache-aside pattern)
- Set TTLs on everything. No TTL = infinite stale data
- Monitor cache hit rate. Below 80%? Your cache isn't helping.

---

## Mistake 6: Not Understanding Your Database's Isolation Level

**What happens**: Application has a "buy the last item" race condition. Two users both check inventory (both see 1 remaining), both purchase. Inventory goes to -1.

**The root cause**: The default isolation level (Read Committed in PostgreSQL) doesn't prevent this. You need Serializable or an explicit `SELECT FOR UPDATE`.

**How to avoid it**:
- Know your database's default isolation level
- For any read-modify-write pattern, use explicit locking or atomic operations
- Test concurrent scenarios explicitly (not just happy-path unit tests)
- Use the strongest isolation level you can afford for financial/inventory operations

```sql
-- ❌ WRONG: Race condition with concurrent buyers
SELECT quantity FROM products WHERE id = 42;  -- Both see quantity=1
UPDATE products SET quantity = quantity - 1 WHERE id = 42;  -- Both execute

-- ✅ RIGHT: Pessimistic locking
SELECT quantity FROM products WHERE id = 42 FOR UPDATE;  -- Second caller blocks
UPDATE products SET quantity = quantity - 1 WHERE id = 42;

-- ✅ RIGHT: Optimistic / conditional update
UPDATE products SET quantity = quantity - 1 WHERE id = 42 AND quantity > 0;
-- Check affected rows. If 0 → item was already sold.
```

---

## Mistake 7: Synchronous Calls Where Async Would Work

**What happens**: User places an order. API call synchronously: validates → charges card → updates inventory → sends confirmation email → sends to warehouse → returns 200 OK. If the email service is slow, the user waits. If the warehouse service is down, the order fails.

**The root cause**: Coupling non-critical operations to the critical path.

**How to avoid it**:
```
CRITICAL PATH (synchronous):          NON-CRITICAL (async via queue):
─────────────────────────────         ────────────────────────────────
✅ Validate order                     📧 Send confirmation email
✅ Charge payment                     📦 Notify warehouse
✅ Create order record                📊 Update analytics
✅ Return 200 to user                 🔍 Update search index
                                      📱 Send push notification
```

- Anything that isn't needed for the user's response goes to a queue
- Use Kafka or SQS for reliable async processing
- This also makes the system resilient to downstream failures

---

## Mistake 8: One Big Database for Everything

**What happens**: Single PostgreSQL instance serves the web app, analytics, search, and background jobs. Analytics queries lock tables. Background jobs starve the web app of connections. Search is slow because PostgreSQL's `LIKE '%term%'` is terrible.

**The root cause**: Using one tool for fundamentally different access patterns.

**How to avoid it**:
- OLTP workload → PostgreSQL (or DynamoDB)
- Analytics → Data warehouse (Snowflake, BigQuery)
- Search → Elasticsearch
- Cache/sessions → Redis
- Use CDC (Debezium → Kafka) to keep them in sync
- Each system is optimized for its specific access pattern

---

## Mistake 9: Ignoring Backpressure

**What happens**: Producer sends messages at 10K/sec. Consumer processes at 2K/sec. Queue grows unboundedly. Eventually: OOM, disk full, cascade failure.

**The root cause**: No feedback mechanism from consumer to producer.

**How to avoid it**:
- Set bounded queue sizes (Kafka retention, SQS DLQ)
- Implement rate limiting at the API gateway
- Use circuit breakers to stop calling overloaded services
- Monitor queue depth / consumer lag as critical metrics
- Design your system to gracefully degrade under overload (drop low-priority work, not everything)

---

## Mistake 10: Distributed Transactions with 2PC

**What happens**: Team uses 2PC to coordinate writes across two databases. Coordinator crashes mid-commit. Both databases are stuck holding locks. Application is frozen until someone manually intervenes.

**The root cause**: 2PC is a blocking protocol. Coordinator failure = everyone blocked.

**How to avoid it**:
- Use the Saga pattern instead of 2PC
  - **Choreography**: Each service publishes events, next service reacts
  - **Orchestration**: Central orchestrator tells each service what to do
- Each step in the saga has a compensating action (undo)
- Accept that you get eventual consistency, not immediate

```
Saga for order processing:
  1. Create Order (pending) → if fails, done
  2. Reserve Inventory → if fails, cancel order
  3. Charge Payment → if fails, release inventory, cancel order
  4. Confirm Order (success) → emit OrderConfirmed event
```

---

## Mistake 11: Not Planning for Data Migration

**What happens**: Schema changes require downtime. Adding a column locks the table for hours. Changing a data format breaks all consumers.

**How to avoid it**:
- Always use backward-compatible schema changes (add optional fields, never remove or rename)
- Use the expand-contract pattern: add new field → migrate data → update consumers → remove old field
- In Kafka: use Schema Registry with compatibility checks
- In databases: use online DDL tools (pt-online-schema-change, gh-ost for MySQL, pgroll for PostgreSQL)

---

## Mistake 12: Underestimating Hot Spots

**What happens**: Team hash-partitions by user_id. One user has 10M followers. Every action by that user fans out to 10M writes on one partition. That partition melts.

**Real example**: Twitter's celebrity problem. Justin Bieber tweeting shouldn't melt your infrastructure.

**How to avoid it**:
- Identify hot keys in advance (celebrities, viral content, popular products)
- Add jitter/salt to hot keys (split celebrity_user_id into 100 sub-keys)
- Use read-time fan-out for high-follower users, write-time fan-out for normal users
- DynamoDB adaptive capacity handles some of this automatically, but not all

---

## Mistake 13: Logging Everything in Production

**What happens**: Team logs every request at DEBUG level. 100K QPS × 1KB per log = 100MB/sec of logs. ELK cluster costs more than the application infrastructure. Disk fills up. Nobody reads the logs anyway.

**How to avoid it**:
- Log at appropriate levels (ERROR in prod, DEBUG only when investigating)
- Sample high-volume logs (log 1% of successful requests, 100% of errors)
- Use structured logging (JSON) for machine-parseability
- Set retention policies (7 days for debug, 30 days for info, 1 year for errors)
- Calculate: QPS × avg_log_size × 86400 = daily log volume. Know this number.

---

## Mistake 14: No Dead Letter Queue

**What happens**: A malformed message enters Kafka. Consumer crashes trying to process it. Consumer restarts, picks up same message, crashes again. Infinite restart loop. Mean while, all messages behind it are stuck.

**The root cause**: The "poison pill" problem. One bad message blocks the entire queue.

**How to avoid it**:
- After N retries (typically 3-5), move the message to a Dead Letter Queue (DLQ)
- Monitor the DLQ size (alert on growth)
- Build tooling to inspect, replay, or discard DLQ messages
- Never let a single bad message block the processing of subsequent messages

---

## The Meta-Mistake: Not Measuring Before Optimizing

> "Premature optimization is the root of all evil" — Knuth

**What happens**: Team spends 3 months building a distributed cache, sharding layer, and event-driven architecture. The app has 500 users and a single PostgreSQL instance at 5% CPU utilization.

**How to avoid it**:
1. Build the simplest thing that works
2. Measure actual production performance
3. Find the actual bottleneck
4. Optimize THAT specific bottleneck
5. Repeat

**The 2026 addendum**: With managed cloud services, you can often scale by just changing a configuration slider, not by re-architecting your system.
