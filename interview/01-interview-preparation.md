# Interview Preparation Guide

> Comprehensive system design interview preparation with scoring rubrics, mock interview templates, and level-specific expectations.

---

## 1. Interview Framework

### 1.1 The 5-Step Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM DESIGN INTERVIEW FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STEP 1: REQUIREMENTS CLARIFICATION (3-5 min)                         │
│  ════════════════════════════════════════                             │
│                                                                          │
│  Goal: Understand what interviewer cares about                          │
│                                                                          │
│  Ask:                                                                   │
│  • "What scale are we designing for?"                                 │
│  • "What's the most important: consistency, availability, latency?"   │
│  • "Any specific constraints I should know about?"                    │
│  • "What features are must-have vs nice-to-have?"                     │
│                                                                          │
│  Key insight: Clarify BEFORE jumping in!                               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 2: HIGH-LEVEL DESIGN (5-10 min)                                │
│  ════════════════════════════════════════                             │
│                                                                          │
│  Goal: Show broad understanding                                        │
│                                                                          │
│  Components:                                                            │
│  • API design (REST/gRPC)                                            │
│  • Data model (SQL/NoSQL)                                             │
│  • Core services architecture                                          │
│  • Data flow (read/write paths)                                        │
│  • Diagram: Mermaid or ASCII                                           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 3: DEEP DIVE (15-20 min)                                       │
│  ══════════════════════════════════                                   │
│                                                                          │
│  Goal: Demonstrate depth in key areas                                 │
│                                                                          │
│  Pick 2-3 areas to dive deep:                                          │
│  • Database choice and why                                            │
│  • Scaling strategy                                                   │
│  • Consistency model                                                   │
│  • Handling failures                                                  │
│  • Caching strategy                                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 4: BOTTLENECKS & TRADE-OFFS (5 min)                           │
│  ══════════════════════════════════════════                          │
│                                                                          │
│  Goal: Show you think critically                                       │
│                                                                          │
│  Discuss:                                                               │
│  • Single points of failure                                            │
│  • What would you optimize first?                                      │
│  • What's the biggest risk?                                            │
│  • What would you do differently with more time?                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 5: MONITORING & OPERATIONS (5 min)                             │
│  ════════════════════════════════════════                             │
│                                                                          │
│  Goal: Show production mindset                                         │
│                                                                          │
│  Include:                                                               │
│  • Key metrics to track                                                │
│  • How would you detect issues?                                        │
│  • What alerts would you set?                                          │
│  • SLOs and SLIs                                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Scoring Rubric

### 2.1 FAANG Scoring Rubric

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM DESIGN SCORING RUBRIC                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SCORING DIMENSIONS (1-4 scale per dimension):                         │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │ 1 - Poor: Major gaps, fundamentally wrong                      │   │
│  │                                                                  │   │
│  │ 2 - Basic: Shows understanding but incomplete                   │   │
│  │                                                                  │   │
│  │ 3 - Good: Solid design, minor gaps                             │   │
│  │                                                                  │   │
│  │ 4 - Excellent: Production-ready, considers trade-offs          │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  DIMENSIONS:                                                            │
│                                                                          │
│  ┌────────────────────┬────────────────────────────────────────────┐  │
│  │ Dimension          │ What Evaluators Look For                   │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Requirements      │ Asked clarifying questions, prioritized    │  │
│  │ Clarification     │ features, considered scale                 │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Architecture      │ Clear component diagram, appropriate      │  │
│  │                   │ services, proper data flow                  │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Data Model        │ SQL vs NoSQL justified, schema design,    │  │
│  │                   │ access patterns considered                 │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Scaling Strategy  │ Horizontal vs vertical, sharding,         │  │
│  │                   │ replication, caching                       │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Consistency       │ Chose appropriate model, understood       │  │
│  │                   │ trade-offs                                  │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Trade-offs        │ Identified bottlenecks, discussed         │  │
│  │                   │ alternatives                               │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Production        │ Monitoring, failure handling, SLOs        │  │
│  │ Mindset           │ considered                                 │  │
│  ├────────────────────┼────────────────────────────────────────────┤  │
│  │ Communication     │ Clear, organized, responsive to hints     │  │
│  │                   │                                            │  │
│  └────────────────────┴────────────────────────────────────────────┘  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  LEVEL-SPECIFIC EXPECTATIONS:                                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │ L4 (Senior):                                                    │   │
│  │ • 3-4 dimensions at "Good" level                              │   │
│  │ • Can complete a design with minimal guidance                  │   │
│  │ • Understands trade-offs                                       │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │ L5 (Staff):                                                     │   │
│  │ • 4 in most dimensions                                        │   │
│  │ • Anticipates follow-up questions                              │   │
│  │ • Mentions real-world production challenges                     │   │
│  │ • Goes beyond basic design to discuss optimization             │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │ L6 (Principal):                                                │   │
│  │ • 4 across all dimensions                                      │   │
│  │ • Shows industry context (what do FAANG companies do?)         │   │
│  │ • Discusses organizational/team considerations                │   │
│  │ • Can lead the interview, not just respond                     │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PASSING THRESHOLDS:                                                    │
│                                                                          │
│  • Strong Hire: 3.5+ average, no 1s                                  │
│  • Hire: 3.0+ average, max one 2                                     │
│  • No Hire: Below 3.0, or any 1                                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  COMMON FAILURE MODES:                                                 │
│                                                                          │
│  • Jumps in without clarifying requirements (1)                      │
│  • Gets stuck on one component, misses overall picture (1-2)         │
│  • Doesn't consider scale (1-2)                                      │
│  • Can't explain trade-offs (1-2)                                     │
│  • Ignores production concerns (1-2)                                 │
│  • Poor communication / unresponsive to hints (1-2)                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Mock Interview Templates

### 3.1 Template 1: URL Shortener

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MOCK INTERVIEW: URL SHORTENER                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  INTERVIEWER: "Design a URL shortener like bit.ly"                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  [STEP 1: Clarify - 3 minutes]                                         │
│                                                                          │
│  CANDIDATE:                                                             │
│  • "What scale are we talking about?"                                 │
│  • "How long do short URLs need to persist?"                          │
│  • "What are the most important properties - availability,            │
│    consistency, latency?"                                               │
│                                                                          │
│  INTERVIEWER: "100M users, 1B URLs. Availability is key.              │
│               99.99% uptime. Can accept eventual consistency."         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  [STEP 2: High-Level Design - 5 minutes]                               │
│                                                                          │
│  CANDIDATE:                                                            │
│  "Let me sketch out the high-level design..."                         │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  ┌─────────┐     ┌──────────────┐     ┌─────────────┐        │   │
│  │  │ Client  │────▶│  API Server  │────▶│  Database   │        │   │
│  │  └─────────┘     │  (stateless) │     │  (Redis +   │        │   │
│  │                   └──────────────┘     │   Postgres) │        │   │
│  │                        │               └─────────────┘        │   │
│  │                        ▼                                    │   │
│  │                 ┌──────────────┐                            │   │
│  │                 │  Redirect    │                            │   │
│  │                 │  Service     │                            │   │
│  │                 └──────────────┘                            │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  "Core components:                                                     │
│  - API for creating/reading URLs                                       │
│  - Hash generator for short codes                                     │
│  - Redirect service                                                   │
│  - Database: Redis for caching, PostgreSQL for persistence"           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  [STEP 3: Deep Dive - 15 minutes]                                      │
│                                                                          │
│  INTERVIEWER: "Let's dive deeper into the hash generation"            │
│                                                                          │
│  CANDIDATE:                                                            │
│  "For URL shorteners, we have two main approaches:..."               │
│                                                                          │
│  1. Random ID generation:                                              │
│     - Generate random 6-char alphanumeric                             │
│     - Check if exists, retry if collision                             │
│     - Pros: Simple, no predictability                                 │
│     - Cons: Need collision checking                                   │
│                                                                          │
│  2. Custom aliases:                                                    │
│     - Allow users to choose their own                                │
│     - Need validation and conflict checking                          │
│                                                                          │
│  "I'd use approach 1 with base62 encoding (a-z, A-Z, 0-9)"           │
│  "62^6 = 56 billion possible codes - with good hashing,             │
│   collision probability is negligible"                                │
│                                                                          │
│  INTERVIEWER: "What about the database design?"                       │
│                                                                          │
│  CANDIDATE:                                                            │
│  "CREATE TABLE urls (                                                 │
│      id BIGSERIAL PRIMARY KEY,                                        │
│      short_code VARCHAR(10) UNIQUE NOT NULL,                         │
│      original_url TEXT NOT NULL,                                      │
│      created_at TIMESTAMP DEFAULT NOW(),                               │
│      clicks BIGINT DEFAULT 0                                           │
│  );                                                                    │
│                                                                          │
│  "Index on short_code for O(1) lookups"                               │
│                                                                          │
│  INTERVIEWER: "How would you handle 10x traffic spike?"              │
│                                                                          │
│  CANDIDATE:                                                            │
│  "Multiple strategies:                                                  │
│  1. Read replicas - scale reads                                       │
│  2. Redis cache - hot URLs cached                                     │
│  3. CDN for redirect - static-like pattern                          │
│  4. Horizontal scaling - stateless API servers"                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  [STEP 4: Trade-offs - 5 minutes]                                     │
│                                                                          │
│  INTERVIEWER: "What are the main trade-offs in your design?"         │
│                                                                          │
│  CANDIDATE:                                                            │
│  "1. Consistency vs Availability: I chose eventual for redirects    │
│     because 99.99% availability matters more than seeing               │
│     the exact click count instantly."                                 │
│                                                                          │
│  "2. Memory vs Compute: Heavy Redis caching costs money but           │
│     gives sub-millisecond reads."                                     │
│                                                                          │
│  "3. Security: Need rate limiting to prevent abuse. Would use         │
│     API gateway with rate limits per user."                           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  [STEP 5: Operations - 5 minutes]                                    │
│                                                                          │
│  INTERVIEWER: "How would you monitor this service?"                  │
│                                                                          │
│  CANDIDATE:                                                            │
│  "Key metrics:                                                         │
│  - Redirect latency (p50, p99)                                        │
│  - Cache hit rate                                                     │
│  - Error rate (5xx)                                                   │
│  - URL creation rate                                                  │
│                                                                          │
│  "SLO: p99 < 50ms for 99.9% of redirects"                          │
│                                                                          │
│  "Alerts:                                                              │
│  - p99 > 100ms for 5 minutes                                        │
│  - Error rate > 1%                                                   │
│  - Cache hit rate < 80%"                                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Expected Answer Patterns

### 4.1 Database Selection

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EXPECTED ANSWER: DATABASE SELECTION                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  GOOD ANSWER:                                                          │
│  ────────────                                                           │
│  "I would use PostgreSQL because:                                     │
│  - It handles our relational data well                                │
│  - ACID transactions ensure data integrity                            │
│  - Strong ecosystem and tooling                                        │
│  - But I'd also add Redis for caching hot URLs"                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  BETTER ANSWER:                                                        │
│  ──────────────                                                         │
│  "The choice depends on access patterns:                              │
│                                                                          │
│  For user data (relational, complex queries):                         │
│  → PostgreSQL                                                          │
│                                                                          │
│  For session data (key-value, high throughput):                       │
│  → Redis                                                               │
│                                                                          │
│  For analytics (aggregations, time-series):                           │
│  → ClickHouse or time-series DB                                       │
│                                                                          │
│  The key is: don't use one database for everything."                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  BEST ANSWER:                                                          │
│  ─────────────                                                          │
│  "Let me walk through the decision:                                   │
│                                                                          │
│  1. What's the dominant access pattern?                               │
│     - If point lookups by ID: key-value store                        │
│     - If range queries: B-tree (PostgreSQL)                          │
│     - If full-text search: search engine                              │
│                                                                          │
│  2. Do we need ACID?                                                  │
│     - Financial data: Yes → PostgreSQL                                │
│     - User preferences: No → DynamoDB                                 │
│                                                                          │
│  3. What's the scale?                                                 │
│     - <1TB: Single PostgreSQL                                         │
│     - >1TB with sharding: CockroachDB or sharded PostgreSQL         │
│                                                                          │
│  4. Trade-off: consistency vs availability                            │
│     - CAP: must pick during partition                                │
│     - Most apps: eventual consistency is fine                        │
│                                                                          │
│  In practice: polyglot persistence is common."                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Scaling Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EXPECTED ANSWER: SCALING STRATEGY                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  GOOD ANSWER:                                                          │
│  ────────────                                                           │
│  "We'll add more servers and use a load balancer."                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  BETTER ANSWER:                                                        │
│  ──────────────                                                         │
│  "I'll scale in layers:                                               │
│                                                                          │
│  1. Read replicas: First add read replicas to scale reads            │
│     - Works if read/write ratio is high                              │
│     - Adds replication lag complexity                                  │
│                                                                          │
│  2. Caching layer: Add Redis for hot data                           │
│     - Cache-aside pattern                                             │
│     - 80/20 rule: cache 20% hot data                                 │
│     - Need cache invalidation strategy                               │
│                                                                          │
│  3. Horizontal scaling: Add more API servers                         │
│     - Stateless design                                                │
│     - Load balancer distributes traffic                               │
│                                                                          │
│  4. Database sharding: If single DB can't keep up                    │
│     - Shard by user_id or other key                                  │
│     - Cross-shard joins are complex"                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  BEST ANSWER:                                                         │
│  ─────────────                                                          │
│  "Let me break this down by bottleneck:                               │
│                                                                          │
│  If CPU-bound:                                                        │
│  → Horizontal scaling with more instances                             │
│  → Consider async processing                                          │
│                                                                          │
│  If I/O-bound:                                                       │
│  → Caching (Redis, CDN)                                              │
│  → Database indexing                                                  │
│  → Read replicas                                                      │
│                                                                          │
│  If storage-bound:                                                    │
│  → Archiving old data                                                │
│  →│  → Tier Sharding                                                          │
ed storage                                                    │
│                                                                          │
│  The key: measure first, optimize the bottleneck."                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Common Pitfalls by Level

### 5.1 L4 (Senior Engineer) Pitfalls

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    L4 COMMON PITFALLS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. NOT ASKING CLARIFYING QUESTIONS                                    │
│  ─────────────────────────────────────                                  │
│  Problem: Jumps straight into design without understanding scale       │
│  Fix: Always clarify: "What scale? What's most important?"            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  2. IGNORING TRADE-OFFS                                                │
│  ─────────────────────                                                  │
│  Problem: Presents one solution as "the right answer"                  │
│  Fix: Always discuss trade-offs and alternatives                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  3. OVER-ENGINEERING                                                   │
│  ─────────────────                                                     │
│  Problem: Uses Kafka + Redis + Elasticsearch for simple problems      │
│  Fix: Start simple, add complexity only when needed                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  4. MISSING OPERATIONS                                                 │
│  ───────────────────                                                   │
│  Problem: Doesn't discuss monitoring, failure handling                │
│  Fix: Always include production considerations                        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  5. POOR COMMUNICATION                                                 │
│  ─────────────────                                                     │
│  Problem: Mumbles, doesn't check in with interviewer                  │
│  Fix: Think out loud, ask if interviewer wants to dive somewhere      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 L5 (Staff Engineer) Pitfalls

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    L5 COMMON PITFALLS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. NOT GOING DEEP ENOUGH                                              │
│  ─────────────────────────                                              │
│  Problem: Stays at high-level, doesn't show depth                       │
│  Fix: Expect to dive deep in at least 2-3 areas                        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  2. MISSING REAL-WORLD CONTEXT                                        │
│  ───────────────────────────────────                                   │
│  Problem: Theoretical but not practical                                 │
│  Fix: Mention real tools (Kafka, Redis, PostgreSQL, etc.)             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  3. IGNORING ORGANIZATIONAL FACTORS                                    │
│  ─────────────────────────────────────                                 │
│  Problem: Only technical, not considering team/product                  │
│  Fix: Mention: "This would take 2 quarters to build"                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  4. NOT HANDLING FOLLOW-UPS                                            │
│  ────────────────────────                                              │
│  Problem: Crumbles when interviewer pushes on a point                   │
│  Fix: It's OK to say "That's a good point, let me think..."           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  5. NOT ANTICIPATING NEXT QUESTIONS                                    │
│  ────────────────────────────────────                                  │
│  Problem: Stops after first design, doesn't think ahead                  │
│  Fix: Proactively mention what you'd discuss next                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 L6 (Principal Engineer) Pitfalls

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    L6 COMMON PITFALLS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. NOT LEADING THE INTERVIEW                                          │
│  ───────────────────────────────────                                   │
│  Problem: Waits to be led, doesn't take ownership                       │
│  Fix: Guide the conversation, suggest directions                        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  2. MISSING BUSINESS CONTEXT                                           │
│  ───────────────────────────────                                       │
│  Problem: Purely technical, ignores business impact                     │
│  Fix: Discuss ROI, what success looks like                            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  3. NOT MENTIONING INDUSTRY PRACTICES                                   │
│  ──────────────────────────────────────                                │
│  Problem: Reinvents wheel, doesn't know industry standards              │
│  Fix: Reference: "This is how Twitter/Netflix/etc. does it"           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  4. OVERCOMPLICATING                                                   │
│  ────────────────                                                      │
│  Problem: Too many components, too complex                              │
│  Fix: Simple is better - start minimal, expand only if needed          │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  5. NOT CONSIDERING TEAM/ECOSYSTEM                                     │
│  ───────────────────────────────────────                               │
│  Problem: Only thinks about the system, not team building it            │
│  Fix: Mention: "Team skills needed, training, documentation"          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Quick Reference

### 6.1 Key Metrics to Calculate

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BACK-OF-ENVELOPE NUMBERS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  USER ESTIMATES:                                                        │
│  ───────────────                                                        │
│  • MAU = DAU × 30%                                                     │
│  • DAU = MAU × 30-40%                                                 │
│  • Peak DAU = DAU × 3-5                                               │
│                                                                          │
│  QPS ESTIMATES:                                                        │
│  ──────────────                                                        │
│  • Write QPS = DAU × actions_per_day / 86400                         │
│  • Read QPS = Write QPS × read_write_ratio                           │
│  • Peak QPS = Average QPS × 3-5                                       │
│                                                                          │
│  STORAGE ESTIMATES:                                                   │
│  ─────────────────                                                      │
│  • Daily storage = users × actions × avg_size                        │
│  • Yearly storage = daily × 365                                       │
│                                                                          │
│  EXAMPLE:                                                              │
│  ─────────                                                             │
│  100M DAU, 10 tweets/user/day, 1KB/tweet:                            │
│  • Write QPS = 100M × 10 / 86400 ≈ 12K/sec                          │
│  • Storage = 100M × 10 × 1KB × 365 ≈ 365TB/year                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  LATENCY BUDGET:                                                       │
│  ───────────────                                                       │
│  • Total latency = serialization + network + processing + DB          │
│  • p99 = p50 × 3-5 (typical ratio)                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
