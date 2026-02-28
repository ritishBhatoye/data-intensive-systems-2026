# 💎 Designing Data-Intensive Applications: 2026 Staff Edition

> **The high-signal, zero-fluff playbook for modern distributed systems.**
> *A comprehensive curriculum for FAANG L5-L6+ system design mastery.*

---

## 🎯 The "Staff Engineer" Filter

Most engineers read the 600-page classic but fail to translate academic theory into production reality. This project is the high-bandwidth rewrite that bridges that gap.

*   **Stop** wading through academic depth.
*   **Start** mastering the 20% of architectural patterns that drive 80% of FAANG-scale decisions.

---

## 📚 Curriculum Structure

### Core Fundamentals

| File | What It Covers |
|------|----------------|
| [Executive Overview](01-executive-overview.md) | 2-page ultra-simplified overview of the entire book |
| [Chapter-by-Chapter Guide](02-chapters-simplified.md) | All 12 chapters, re-explained simply with 2026 context |
| [Industry Translation](03-industry-translation.md) | Every concept → real FAANG system mapping |
| [2026 Updates](04-2026-updates.md) | What changed since 2017, new tools/paradigms |

### Deep-Dive Fundamentals

| Directory | File | What It Covers |
|-----------|------|----------------|
| fundamentals/ | [Storage Engines Deep Dive](fundamentals/10-storage-engines-deep-dive.md) | B-tree vs LSM-tree, production math, decision framework |
| fundamentals/ | [Networking Deep Dive](fundamentals/11-networking-deep-dive.md) | TCP/UDP, DNS, TLS, load balancing, latency math |
| fundamentals/ | [Probabilistic Data Structures](fundamentals/12-probabilistic-data-structures.md) | Bloom filters, HyperLogLog, Count-Min Sketch |

### System Design Case Studies

| Directory | File | What It Covers |
|-----------|------|----------------|
| case-studies/ | [Twitter Timeline Expanded](case-studies/01-twitter-timeline-expanded.md) | Full requirements, API design, capacity math, failure analysis |

### Operations & Production

| Directory | File | What It Covers |
|-----------|------|----------------|
| operations/ | [Operations & Production Guide](operations/01-operations-production-guide.md) | Monitoring, SRE, incident response, runbooks, cost optimization |

### API & Schema Design

| Directory | File | What It Covers |
|-----------|------|----------------|
| api-design/ | [API & Schema Design Guide](api-design/01-api-schema-design-guide.md) | REST/gRPC/GraphQL, versioning, pagination, schema patterns |

### Interview Preparation

| Directory | File | What It Covers |
|-----------|------|----------------|
| interview/ | [Interview Preparation](interview/01-interview-preparation.md) | Framework, scoring rubric, mock templates |
| interview/ | [Level-Based Checklists](interview/02-level-based-checklists-roadmap.md) | L4/L5/L6 readiness, 12-week roadmap, quick reference |

### Reference Sheets

| File | What It Covers |
|------|----------------|
| [Interview Cheat Sheet](05-interview-cheatsheet.md) | System design interview preparation guide |
| [Common Mistakes](06-common-mistakes.md) | Mistakes engineers make in production |
| [Mental Models](07-mental-models.md) | Visual thinking + memory frameworks |
| [Case Studies](08-case-studies.md) | 10 system design case studies applying DDIA |
| [Revision Sheet](09-revision-sheet.md) | One-page ultra-compressed revision |

---

## 🎯 Who This Is For

- **L4 Engineers** preparing for senior/staff promotions
- **L5 Engineers** targeting staff engineer roles
- **System design interview candidates** (L5–L7)
- **Backend engineers** who want to understand *why* things work, not just *how*

---

## ⚡ How To Use This

### For Interview Prep (12-Week Plan)

1. **Week 1-2**: Read Executive Overview + Chapters 1-2
2. **Week 3-4**: Study Storage & Networking deep dives
3. **Week 5-6**: Master Replication & Partitioning
4. **Week 7-8**: Deep dive Transactions & Consensus
5. **Week 9-10**: Learn Batch & Stream Processing
6. **Week 11**: Operations & Production content
7. **Week 12**: Mock interviews + review

### Quick Reference (Day Before Interview)

1. Skim [Revision Sheet](09-revision-sheet.md) (5 min)
2. Review [Interview Cheat Sheet](05-interview-cheatsheet.md) (15 min)
3. Review [Mental Models](07-mental-models.md) (10 min)
4. Check [Level-Based Checklists](interview/02-level-based-checklists-roadmap.md) (10 min)

### Deep Learning (Weekend Deep Dive)

1. Work through any fundamental deep-dive chapter
2. Study expanded case studies
3. Practice with mock interviews

---

## 📊 New in 2026 Edition

### Added Content

- ✅ **Storage Engines Deep Dive** - B-tree vs LSM-tree with production math
- ✅ **Networking Deep Dive** - TCP/UDP, DNS, TLS, load balancing
- ✅ **Probabilistic Data Structures** - Bloom filters, HyperLogLog, Count-Min Sketch
- ✅ **Expanded Case Studies** - Full system design with requirements, API, capacity planning
- ✅ **Operations & Production** - Monitoring, SRE, incident response, cost optimization
- ✅ **API & Schema Design** - REST vs gRPC vs GraphQL, versioning, pagination
- ✅ **Interview Scoring Rubrics** - FAANG-level scoring with level-specific expectations
- ✅ **Level-Based Checklists** - L4/L5/L6 readiness with 12-week roadmap

### Key Updates

- All case studies expanded with Mermaid diagrams
- Added capacity estimation math
- Added failure analysis sections
- Added monitoring & SLO sections
- Added interview follow-up questions with expected answers

---

## 📖 Original DDIA Chapters Covered

| Chapter | Topic | Status |
|---------|-------|--------|
| 1 | Reliability, Scalability, Maintainability | ✅ Covered |
| 2 | Data Models and Query Languages | ✅ Covered |
| 3 | Storage and Retrieval | ✅ Covered + Deep Dive |
| 4 | Encoding and Evolution | ✅ Covered |
| 5 | Replication | ✅ Covered |
| 6 | Partitioning | ✅ Covered |
| 7 | Transactions | ✅ Covered |
| 8 | The Trouble with Distributed Systems | ✅ Covered |
| 9 | Consistency and Consensus | ✅ Covered |
| 10 | Batch Processing | ✅ Covered |
| 11 | Stream Processing | ✅ Covered |
| 12 | The Future of Data Systems | ✅ Covered |

---

## 🔧 2026 Must-Know Tools

| Category | Tools |
|----------|-------|
| **OLTP** | PostgreSQL, CockroachDB, DynamoDB, PlanetScale |
| **OLAP** | Snowflake, BigQuery, ClickHouse, Redshift |
| **Streaming** | Kafka, Flink, Redpanda, Kinesis |
| **Cache** | Redis, Memcached, CDN edge |
| **Search** | Elasticsearch, Meilisearch |
| **Vector** | Pinecone, pgvector, Weaviate |
| **Lakehouse** | Iceberg, Delta Lake, Hudi |
| **Observability** | OpenTelemetry, Grafana, Datadog |

---

*Based on "Designing Data-Intensive Applications" by Martin Kleppmann (2017). Modernized for 2026 industry practice.*
