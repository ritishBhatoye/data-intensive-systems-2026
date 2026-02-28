# Level-Based Checklists & Roadmap

> Comprehensive study roadmap for mastering data-intensive systems at L4, L5, and L6 levels.

---

## 1. Level Checklists

### 1.1 L4 (Senior Engineer) Readiness Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    L4 READINESS CHECKLIST                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  FUNDAMENTALS:                                                         │
│  ────────────                                                          │
│  □ Can explain the CAP theorem and its implications                    │
│  □ Understands ACID vs BASE                                           │
│  □ Knows difference between SQL and NoSQL                              │
│  □ Can design basic database schemas                                    │
│  □ Understands indexing (B-tree, when to use)                          │
│                                                                          │
│  REPLICATION & PARTITIONING:                                            │
│  ────────────────────────────                                           │
│  □ Understands single-leader, multi-leader, leaderless                │
│  □ Knows how replication lag affects consistency                       │
│  □ Can design sharding strategy (hash vs range)                       │
│  □ Understands conflict resolution strategies                          │
│                                                                          │
│  TRANSACTIONS:                                                          │
│  ────────────                                                          │
│  □ Knows isolation levels (READ COMMITTED, MVCC, SERIALIZABLE)        │
│  □ Can identify race conditions (lost updates, write skew)            │
│  □ Understands when to use distributed transactions                     │
│  □ Knows Saga pattern for distributed transactions                     │
│                                                                          │
│  STORAGE ENGINES:                                                       │
│  ────────────────                                                      │
│  □ Can explain B-tree vs LSM-tree tradeoffs                           │
│  □ Knows when to use which storage engine                             │
│  □ Understands write-ahead logs                                       │
│                                                                          │
│  MESSAGING & STREAMING:                                                │
│  ────────────────────────                                              │
│  □ Can design event-driven architecture                                │
│  □ Understands at-least-once vs exactly-once                          │
│  □ Knows Kafka basics (topics, partitions, consumers)                │
│                                                                          │
│  SYSTEM DESIGN:                                                        │
│  ──────────────                                                        │
│  □ Can complete URL shortener design                                   │
│  □ Can design Twitter timeline (push vs pull)                          │
│  □ Can design rate limiter                                            │
│  □ Can design distributed cache                                       │
│                                                                          │
│  OPERATIONS:                                                            │
│  ──────────                                                            │
│  □ Can design basic monitoring and alerting                            │
│  □ Understands SLO/SLI/SLA                                           │
│  □ Can design basic failure handling                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  INTERVIEW EXPECTATIONS:                                                │
│  ─────────────────────────                                              │
│  □ Can complete a 45-min system design interview                       │
│  □ Can discuss trade-offs                                             │
│  □ Can explain your design choices                                     │
│  □ Knows basic capacity estimation                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 L5 (Staff Engineer) Readiness Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    L5 READINESS CHECKLIST                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ALL L4 ITEMS PLUS:                                                    │
│                                                                          │
│  CONSENSUS & DISTRIBUTED SYSTEMS:                                      │
│  ─────────────────────────────────                                      │
│  □ Can explain Raft consensus algorithm                                │
│  □ Understands leader election                                        │
│  □ Knows when consensus is needed                                     │
│  □ Can explain clock synchronization issues                           │
│  □ Understands vector clocks and Lamport timestamps                   │
│                                                                          │
│  ADVANCED STORAGE:                                                      │
│  ─────────────────                                                     │
│  □ Deep understanding of B-tree and LSM internals                     │
│  □ Can optimize for specific workloads                                 │
│  □ Understands columnar storage                                       │
│  □ Knows data warehousing patterns                                    │
│                                                                          │
│  ADVANCED SCALING:                                                     │
│  ─────────────────                                                     │
│  □ Can design multi-region active-active                              │
│  □ Understands hot spotting and solutions                             │
│  □ Can design global CDNs                                              │
│  □ Knows connection pooling and optimization                           │
│                                                                          │
│  ADVANCED PATTERNS:                                                    │
│  ──────────────────                                                   │
│  □ Can implement change data capture                                  │
│  □ Understands event sourcing                                         │
│  □ Can design CQRS patterns                                          │
│  □ Knows when to use each pattern                                     │
│                                                                          │
│  CASE STUDIES:                                                         │
│  ────────────                                                         │
│  □ Can deeply design Twitter (all components)                         │
│  □ Can design Uber/Lyft ride matching                                │
│  □ Can design YouTube/Netflix streaming                               │
│  □ Can design distributed key-value store                             │
│  □ Can design payment system                                          │
│                                                                          │
│  OPERATIONS & SRE:                                                     │
│  ─────────────────                                                     │
│  □ Can design comprehensive monitoring                                  │
│  □ Can create runbooks                                                 │
│  □ Understands incident response                                      │
│  □ Can do capacity planning                                           │
│  □ Knows cost optimization strategies                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  INTERVIEW EXPECTATIONS:                                                │
│  ─────────────────────────                                              │
│  □ Anticipates follow-up questions                                     │
│  □ Can discuss production challenges                                   │
│  □ Shows depth in 2-3 areas                                            │
│  □ Mentions real-world production experience                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 L6 (Principal Engineer) Readiness Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    L6 READINESS CHECKLIST                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ALL L5 ITEMS PLUS:                                                    │
│                                                                          │
│  INDUSTRY CONTEXT:                                                     │
│  ─────────────────                                                     │
│  □ Knows how major tech companies design systems (Twitter, Netflix,    │
│    Uber, Stripe, Google)                                              │
│  □ Understands organizational patterns for data platforms              │
│  □ Can discuss data mesh principles                                   │
│  □ Knows latest trends (lakehouse, vector DBs)                       │
│                                                                          │
│  PRODUCTION EXCELLENCE:                                                │
│  ─────────────────────                                                 │
│  □ Has run multi-region systems                                       │
│  □ Has handled major incidents                                        │
│  □ Can design for regulatory compliance                               │
│  □ Understands cost optimization at scale                              │
│                                                                          │
│  ARCHITECTURE LEADERSHIP:                                              │
│  ──────────────────────                                               │
│  □ Can evaluate build vs buy decisions                                │
│  □ Can design platform strategies                                     │
│  □ Understands team structure implications                            │
│  □ Can create technical vision                                         │
│                                                                          │
│  ADVANCED TOPICS:                                                      │
│  ───────────────                                                      │
│  □ Probabilistic data structures (Bloom, HLL, Count-Min)             │
│  □ Advanced networking (TCP internals, TLS)                           │
│  □ Security patterns                                                  │
│  □ Performance tuning at scale                                        │
│                                                                          │
│  COMMUNICATION:                                                        │
│  ──────────────                                                        │
│  □ Can present to executives                                          │
│  □ Can drive architecture discussions                                 │
│  □ Can mentor senior engineers                                         │
│  □ Can negotiate technical decisions                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  INTERVIEW EXPECTATIONS:                                               │
│  ─────────────────────────                                              │
│  □ Leads the interview conversation                                    │
│  □ Shows broad and deep knowledge                                     │
│  □ Can discuss organizational/team considerations                     │
│  □ Demonstrates production judgment                                    │
│  □ Can handle rapid pivots in questions                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 12-Week Study Roadmap

### 2.1 Week-by-Week Plan

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    12-WEEK STUDY ROADMAP                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  WEEK 1-2: FOUNDATIONS                                                 │
│  ════════════════════════                                              │
│                                                                          │
│  □ Read Chapter 1-2: Reliability, Scalability, Data Models             │
│  □ Watch: Distributed Systems lecture series                           │
│  □ Practice: Basic system design (URL shortener)                      │
│                                                                          │
│  DELIVERABLE: Can explain three pillars and data model choices         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WEEK 3-4: STORAGE & RETRIEVAL                                        │
│  ═══════════════════════════                                           │
│                                                                          │
│  □ Read Chapter 3: Storage engines                                     │
│  □ Deep dive: B-tree vs LSM-tree                                       │
│  □ Practice: Database selection for different use cases                │
│                                                                          │
│  DELIVERABLE: Can explain storage engine tradeoffs                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WEEK 5-6: REPLICATION & PARTITIONING                                 │
│  ══════════════════════════════                                        │
│                                                                          │
│  □ Read Chapters 5-6: Replication, Partitioning                         │
│  □ Deep dive: Consensus (Raft/Paxos)                                   │
│  □ Practice: Twitter timeline design                                   │
│                                                                          │
│  DELIVERABLE: Can design sharding and replication strategies           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WEEK 7-8: TRANSACTIONS & CONSISTENCY                                 │
│  ═══════════════════════════════                                       │
│                                                                          │
│  □ Read Chapters 7, 9: Transactions, Consensus                        │
│  □ Deep dive: Isolation levels, distributed transactions               │
│  □ Practice: Payment system design                                      │
│                                                                          │
│  DELIVERABLE: Can explain ACID and consistency tradeoffs               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WEEK 9-10: BATCH & STREAM PROCESSING                                 │
│  ═══════════════════════════════                                       │
│                                                                          │
│  □ Read Chapters 10-11: Batch, Stream                                  │
│  □ Deep dive: Kafka, Flink, Spark                                     │
│  □ Practice: Analytics system design                                   │
│                                                                          │
│  DELIVERABLE: Can design event-driven architectures                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WEEK 11: OPERATIONS & PRODUCTION                                      │
│  ════════════════════════════                                          │
│                                                                          │
│  □ Study monitoring and alerting                                       │
│  □ Understand SLO/SLI/SLA                                              │
│  □ Study incident response                                             │
│  □ Review case studies: how real systems handle failures              │
│                                                                          │
│  DELIVERABLE: Can discuss production concerns                          │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WEEK 12: INTEGRATION & MOCK INTERVIEWS                               │
│  ═══════════════════════════════════                                   │
│                                                                          │
│  □ Complete 3 full mock interviews                                    │
│  □ Focus on communication and trade-offs                               │
│  □ Review common pitfalls                                              │
│  □ Practice back-of-envelope calculations                               │
│                                                                          │
│  DELIVERABLE: Ready for interview!                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Daily Study Schedule (2 hours/day)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DAILY STUDY SCHEDULE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  MORNING (30 min):                                                     │
│  ─────────────────                                                     │
│  • Read or watch material (new topic)                                 │
│  • Take notes on key concepts                                         │
│                                                                          │
│  EVENING (90 min):                                                    │
│  ─────────────────                                                     │
│  • Work through chapter in detail                                     │
│  • Practice system design questions                                   │
│  • Code examples if applicable                                        │
│                                                                          │
│  WEEKLY:                                                               │
│  ───────                                                               │
│  • 1 mock interview (45 min) + review (15 min)                       │
│  • 1 review session of previous week's topics                         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  TIME BREAKDOWN:                                                       │
│  ──────────────                                                        │
│  • Theory: 40%                                                         │
│  • System design practice: 40%                                        │
│  • Mock interviews: 20%                                               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PRIORITY ORDER:                                                       │
│  ──────────────                                                        │
│  1. System design practice (most important)                            │
│  2. Trade-off knowledge                                               │
│  3. Production considerations                                         │
│  4. Deep technical details                                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Quick Reference Index

### 3.1 Topic-to-Chapter Mapping

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    QUICK REFERENCE INDEX                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  FUNDAMENTALS                                                          │
│  ────────────                                                          │
│  Reliability, Scalability, Maintainability → 01-executive-overview      │
│  Data Models (SQL vs NoSQL) → 02-chapters-simplified                  │
│  Schema Design → api-design/01-api-schema-design-guide                 │
│                                                                          │
│  STORAGE                                                                │
│  ───────                                                               │
│  Storage Engines → fundamentals/10-storage-engines-deep-dive           │
│  B-tree → fundamentals/10-storage-engines-deep-dive                   │
│  LSM-tree → fundamentals/10-storage-engines-deep-dive                  │
│  Columnar → 02-chapters-simplified                                     │
│                                                                          │
│  DISTRIBUTED SYSTEMS                                                   │
│  ─────────────────                                                     │
│  Replication → 02-chapters-simplified                                  │
│  Partitioning → 02-chapters-simplified                                  │
│  Transactions → 02-chapters-simplified                                 │
│  Consensus → 02-chapters-simplified                                    │
│  Consistency Models → 02-chapters-simplified                          │
│                                                                          │
│  NETWORKING                                                            │
│  ───────────                                                           │
│  TCP/UDP → fundamentals/11-networking-deep-dive                        │
│  DNS → fundamentals/11-networking-deep-dive                           │
│  TLS → fundamentals/11-networking-deep-dive                            │
│  Load Balancing → fundamentals/11-networking-deep-dive                  │
│                                                                          │
│  ADVANCED TOPICS                                                       │
│  ───────────────                                                       │
│  Probabilistic Structures → fundamentals/12-probabilistic-ds           │
│  Bloom Filters → fundamentals/12-probabilistic-ds                      │
│  HyperLogLog → fundamentals/12-probabilistic-ds                         │
│  Count-Min Sketch → fundamentals/12-probabilistic-ds                    │
│                                                                          │
│  OPERATIONS                                                            │
│  ───────────                                                           │
│  Monitoring → operations/01-operations-production-guide                 │
│  SRE → operations/01-operations-production-guide                       │
│  Incident Response → operations/01-operations-production-guide           │
│  Cost Optimization → operations/01-operations-production-guide          │
│                                                                          │
│  CASE STUDIES                                                          │
│  ───────────                                                           │
│  Twitter → 08-case-studies                                            │
│  Payment System → 08-case-studies                                      │
│  Uber → 08-case-studies                                                │
│  YouTube/Netflix → 08-case-studies                                     │
│                                                                          │
│  INTERVIEW                                                             │
│  ─────────                                                             │
│  Framework → interview/01-interview-preparation                        │
│  Scoring Rubric → interview/01-interview-preparation                   │
│  Common Questions → 05-interview-cheatsheet                            │
│  Mistakes → 06-common-mistakes                                         │
│                                                                          │
│  TOOLS                                                                 │
│  ─────                                                                 │
│  PostgreSQL → 03-industry-translation                                  │
│  Redis → 03-industry-translation                                      │
│  Kafka → 04-2026-updates                                              │
│  Snowflake → 04-2026-updates                                          │
│  Databricks → 04-2026-updates                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Interview Quick Reference

### 4.1 Key Numbers to Memorize

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KEY NUMBERS TO MEMORIZE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  LATENCY                                                               │
│  ───────                                                               │
│  L1 cache: 0.5 ns                                                     │
│  L2 cache: 7 ns                                                        │
│  RAM: 100 ns                                                           │
│  SSD: 1 ms                                                             │
│  HDD: 10 ms                                                            │
│  Same DC round-trip: 0.5 ms                                            │
│  Cross-continental: 150 ms                                              │
│                                                                          │
│  QPS                                                                   │
│  ───                                                                   │
│  Simple DB server: 1K QPS                                              │
│  Optimized server: 10K QPS                                             │
│  Redis: 100K QPS                                                       │
│  Kafka partition: 10K msgs/sec                                         │
│                                                                          │
│  STORAGE                                                               │
│  ───────                                                               │
│  B-tree: 200 keys/page                                                 │
│  Bloom filter: 1.2 bytes/item                                          │
│  HyperLogLog: 12KB for billions                                        │
│                                                                          │
│  SCALING                                                               │
│  ───────                                                               │
│  Replication lag: 0-10 seconds                                         │
│  Failover time: 10-30 seconds                                          │
│  Rebalancing: minutes to hours                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Decision Tree Quick Reference

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DECISION TREE QUICK REFERENCE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  DATABASE SELECTION:                                                   │
│  ───────────────────                                                   │
│  Complex queries + ACID → PostgreSQL                                   │
│  Simple access patterns + scale → DynamoDB                             │
│  Full-text search → Elasticsearch                                      │
│  Analytics → ClickHouse/Snowflake                                     │
│  Time-series → TimescaleDB/InfluxDB                                    │
│  Graph relationships → Neo4j                                           │
│  AI embeddings → Pinecone/pgvector                                      │
│                                                                          │
│  REPLICATION:                                                          │
│  ───────────                                                           │
│  Single region, read scaling → Single-leader                          │
│  Multi-region writes → Multi-leader (NewSQL)                          │
│  Highest availability → Leaderless                                     │
│                                                                          │
│  CONSISTENCY:                                                          │
│  ───────────                                                          │
│  Financial transactions → Linearizable                                 │
│  User data → Eventual or causal                                        │
│  Social feeds → Eventual                                               │
│                                                                          │
│  CACHING:                                                             │
│  ────────                                                             │
│  Read-heavy → Cache-aside                                             │
│  Write-heavy → Write-through                                           │
│  Need freshness → Write-through + TTL                                   │
│                                                                          │
│  PARTITIONING:                                                        │
│  ─────────────                                                         │
│  Need range queries → Range partitioning                               │
│  Need even distribution → Hash partitioning                            │
│  Hot keys → Salt + consistent hashing                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Summary

### Key Reminders

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FINAL REMINDERS                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. CLARIFY BEFORE DESIGNING                                           │
│     Ask about scale, constraints, priorities                            │
│                                                                          │
│  2. COMMUNICATE THROUGHOUT                                             │
│     Think out loud, ask questions, check in                            │
│                                                                          │
│  3. DISCUSS TRADE-OFFS                                                 │
│     There's no perfect solution, show you understand this               │
│                                                                          │
│  4. GO DEEP ON 2-3 AREAS                                               │
│     Don't stay surface-level on everything                             │
│                                                                          │
│  5. INCLUDE PRODUCTION CONCERNS                                        │
│     Monitoring, failure handling, SLOs matter                          │
│                                                                          │
│  6. PRACTICE, PRACTICE, PRACTICE                                       │
│     Mock interviews are the key to improvement                          │
│                                                                          │
│  7. STAY CALM                                                         │
│     It's okay to pause and think before answering                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  GOOD LUCK! 🎯                                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
