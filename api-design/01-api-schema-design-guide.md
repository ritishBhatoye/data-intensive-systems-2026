# API & Schema Design Guide

> Best practices for designing APIs and schemas that are robust, maintainable, and scale well.

---

## 1. API Design Principles

### 1.1 REST vs gRPC vs GraphQL

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API PARADIGM COMPARISON                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                    REST                    gRPC                    GraphQL│
│  ═══════════════════════   ═══════════════════   ════════════════════  │
│                                                                          │
│  Communication     HTTP/1.1              HTTP/2               HTTP/1.1    │
│                                                                          │
│  Format           JSON                   Protobuf              JSON        │
│                                                                          │
│  Schema           OpenAPI/Swagger       .proto file           SDL         │
│                                                                          │
│  Type System      None (implicit)       Strong (proto)        Strong      │
│                                                                          │
│  Client Stub      Many generators       Code gen             Code gen    │
│                                                                          │
│  Learning Curve   Low                   Medium               Medium      │
│                                                                          │
│  ────────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WHEN TO USE:                                                           │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ REST:                                                               │   │
│  │ • Public APIs (broad client ecosystem)                          │   │
│  │ • Simple CRUD operations                                         │   │
│  │ • When HTTP semantics are natural                               │   │
│  │ • Browser-friendly (no special client needed)                  │   │
│  │                                                                  │   │
│  │ Examples: Stripe API, GitHub API, Twitter API v2               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ gRPC:                                                               │   │
│  │ • High-performance internal services                            │   │
│  │ • Streaming support                                             │   │
│  │ • When you need to generate multiple language clients          │   │
│  │ • Strong typing is important                                    │   │
│  │                                                                  │   │
│  │ Examples: Microservice communication, Kubernetes API            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ GraphQL:                                                           │   │
│  │ • Mobile clients with varying data needs                       │   │
│  │ • Complex data relationships                                    │   │
│  │ • Clients control what data they get                           │   │
│  │ • Aggregating multiple backends                                 │   │
│  │                                                                  │   │
│  │ Examples: Facebook (invented it), GitHub v4, Shopify          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. REST API Design

### 2.1 Resource Naming

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REST RESOURCE NAMING                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PRINCIPLES:                                                            │
│  ────────────                                                           │
│  • Use nouns, not verbs (GET /users, not GET /getUsers)               │
│  • Use plural forms (/users, /tweets)                                 │
│  • Use kebab-case or snake_case (not camelCase)                       │
│  • Use hierarchy for relationships (/users/{id}/posts)               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ENDPOINT EXAMPLES:                                                     │
│  ─────────────────                                                      │
│                                                                          │
│  Collection:      GET    /users                    → List users         │
│  Single:          GET    /users/{id}              → Get user           │
│  Create:          POST   /users                   → Create user         │
│  Update (full):   PUT    /users/{id}             → Replace user       │
│  Update (partial):PATCH  /users/{id}             → Update user        │
│  Delete:          DELETE /users/{id}              → Delete user        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  RELATIONSHIPS:                                                         │
│  ─────────────                                                          │
│                                                                          │
│  User's posts:      GET    /users/{id}/posts                          │
│  Post's comments:   GET    /posts/{id}/comments                       │
│  Add comment:       POST   /posts/{id}/comments                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ACTIONS (when verbs are needed):                                       │
│  ────────────────────────────────                                       │
│                                                                          │
│  Like a tweet:     POST   /tweets/{id}/likes                          │
│  Unlike a tweet:   DELETE /tweets/{id}/likes                          │
│  Send message:     POST   /conversations/{id}/messages               │
│                                                                          │
│  Alternative (more RESTful):                                            │
│  Send message:     POST   /messages                                   │
│                    Body: { conversation_id, text }                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  COMMON PATTERNS:                                                      │
│  ─────────────────                                                      │
│                                                                          │
│  Search:        GET  /tweets?search=term                              │
│  Filter:        GET  /tweets?author=id&sort=-created_at               │
│  Pagination:    GET  /tweets?cursor=abc123&limit=20                  │
│  Batch:         POST /users/batch-delete                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 HTTP Status Codes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HTTP STATUS CODES                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SUCCESS:                                                               │
│  ────────                                                               │
│  200 OK                    Request succeeded                            │
│  201 Created             Resource created successfully                   │
│  202 Accepted            Request accepted for async processing          │
│  204 No Content          Successful deletion or update, no response    │
│                                                                          │
│  CLIENT ERROR:                                                          │
│  ────────────                                                           │
│  400 Bad Request        Malformed request                               │
│  401 Unauthorized       Authentication required                        │
│  403 Forbidden          Authenticated but not authorized               │
│  404 Not Found          Resource doesn't exist                          │
│  409 Conflict            Resource state conflict (e.g., duplicate)      │
│  422 Unprocessable      Valid syntax but semantic error               │
│  429 Too Many Requests  Rate limited                                    │
│                                                                          │
│  SERVER ERROR:                                                          │
│  ────────────                                                           │
│  500 Internal Error     Unexpected server error                         │
│  502 Bad Gateway        Upstream service error                         │
│  503 Service Unavailable Temporarily overloaded                         │
│  504 Gateway Timeout    Upstream service timed out                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  IDEMPOTENCY:                                                           │
│  ────────────                                                           │
│                                                                          │
│  Safe methods (GET, HEAD, OPTIONS): Always idempotent                  │
│                                                                          │
│  Idempotent:                                                            │
│  • PUT (replace) - same result no matter how many times              │
│  • DELETE - deleting twice is same as once                            │
│                                                                          │
│  NOT idempotent:                                                        │
│  • POST (create) - multiple calls = multiple resources                │
│  • PATCH (partial update) - may not be idempotent                     │
│                                                                          │
│  Best practice: Use Idempotency-Key header for POST/PATCH             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. API Versioning

### 3.1 Versioning Strategies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API VERSIONING STRATEGIES                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STRATEGY 1: URL Path (Most Common)                                   │
│  ─────────────────────────────────                                     │
│                                                                          │
│  GET /v1/users                                                          │
│  GET /v2/users                                                          │
│                                                                          │
│  Pros: Clear, explicit, easy to route                                  │
│  Cons: URL pollution, requires code duplication                        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STRATEGY 2: Query Parameter                                           │
│  ────────────────────────                                              │
│                                                                          │
│  GET /users?version=2                                                  │
│                                                                          │
│  Pros: Single URL, cleaner                                             │
│  Cons: Default version required, harder to document                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STRATEGY 3: Header                                                    │
│  ────────────────                                                       │
│                                                                          │
│  GET /users                                                             │
│  Accept: application/vnd.myapi.v2+json                                │
│                                                                          │
│  Pros: URL stays clean                                                 │
│  Cons: Harder to test, less discoverable                               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STRATEGY 4: Media Type                                                │
│  ───────────────────                                                   │
│                                                                          │
│  GET /users                                                             │
│  Accept: application/vnd.myapi+json;version=2                         │
│                                                                          │
│  Same pros/cons as header versioning                                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  BEST PRACTICES:                                                        │
│  ────────────────                                                       │
│                                                                          │
│  1. Start with /v1 from day one                                       │
│                                                                          │
│  2. Support previous version for at least 6-12 months                │
│                                                                          │
│  3. Communicate deprecation via:                                        │
│     • Response headers: Deprecation, Sunset                             │
│     • Documentation                                                     │
│     • Email to API users                                               │
│                                                                          │
│  4. Response headers example:                                          │
│     Deprecation: true                                                   │
│     Sunset: Mon, 01 Jan 2024 00:00:00 GMT                              │
│     Link: <https://api.example.com/v2/users>; rel="successor-version"  │
│                                                                          │
│  5. Use API gateways for routing                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Pagination Patterns

### 4.1 Cursor vs Offset

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PAGINATION PATTERNS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  OFFSET PAGINATION:                                                     │
│  ─────────────────                                                     │
│                                                                          │
│  GET /users?offset=100&limit=20                                        │
│                                                                          │
│  {                                                                     │
│    "data": [...],                                                     │
│    "pagination": {                                                     │
│      "offset": 100,                                                   │
│      "limit": 20,                                                     │
│      "total": 1000                                                    │
│    }                                                                   │
│  }                                                                     │
│                                                                          │
│  PROS:                                                                 │
│  • Can jump to specific page                                          │
│  • Total count available                                               │
│                                                                          │
│  CONS:                                                                 │
│  • Performance degrades with high offset                              │
│  • Results can shift if data changes between requests                  │
│  • Not suitable for real-time data                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  CURSOR PAGINATION (Recommended):                                      │
│  ────────────────────────────────                                      │
│                                                                          │
│  GET /users?cursor=eyJpZCI6MTBwfQ&limit=20                            │
│                                                                          │
│  {                                                                     │
│    "data": [...],                                                     │
│    "pagination": {                                                     │
│      "next_cursor": "eyJpZCI6MzBwfQ",                                 │
│      "previous_cursor": "eyJpZCI6MTBwfQ",                              │
│      "has_more": true                                                 │
│    }                                                                   │
│  }                                                                     │
│                                                                          │
│  IMPLEMENTATION:                                                        │
│  • Cursor = base64-encoded, opaque value                               │
│  •_se Contains: lasten_id + sort criteria                             │
│  • Never expose internal IDs directly                                  │
│                                                                          │
│  PROS:                                                                 │
│  • Consistent performance regardless of position                       │
│  • Stable even with inserts/deletes                                    │
│  • Works with infinite scroll                                          │
│                                                                          │
│  CONS:                                                                 │
│  • Can't jump to specific position                                    │
│  • No total count (estimate possible)                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  KEYSET PAGINATION (Alternative):                                      │
│  ────────────────────────────                                          │
│                                                                          │
│  GET /users?since_id=100&limit=20                                     │
│  GET /users?created_at.lt=2024-01-01&limit=20                        │
│                                                                          │
│  Pros: Very efficient, works with indexed columns                      │
│  Cons: Only works with single column sort                              │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WHEN TO USE WHAT:                                                     │
│  ───────────────────                                                   │
│                                                                          │
│  Offset: When user needs to jump to specific page (e.g., search results)│
│  Cursor: When data changes frequently (e.g., social feeds)            │
│  Keyset: When sorted by indexed field (e.g., chronological feeds)      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Schema Design Patterns

### 5.1 Normalization vs Denormalization

```
┌─────────────────────────────────────────────────────────────────────────┐
│            NORMALIZATION vs DENORMALIZATION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  NORMALIZED (OLTP):                                                    │
│  ══════════════                                                        │
│                                                                          │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    │
│  │    users        │    │     posts       │    │   comments      │    │
│  ├─────────────────┤    ├─────────────────┤    ├─────────────────┤    │
│  │ id (PK)        │───▶│ id (PK)         │    │ id (PK)         │    │
│  │ name           │    │ user_id (FK)    │───▶│ post_id (FK)    │    │
│  │ email          │    │ title           │    │ user_id (FK)    │    │
│  │ created_at     │    │ body            │    │ body            │    │
│  └─────────────────┘    │ created_at     │    │ created_at     │    │
│                         └─────────────────┘    └─────────────────┘    │
│                                                                          │
│  TO FIND USER'S POSTS:                                                 │
│  SELECT * FROM posts WHERE user_id = ?                                │
│                                                                          │
│  PROS:                                                                │
│  • No data duplication                                                │
│  • Updates only in one place                                           │
│  • Clean, logical structure                                           │
│                                                                          │
│  CONS:                                                                │
│  • More joins = slower reads                                          │
│  • Complex queries                                                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  DENORMALIZED (OLAP / READ-OPTIMIZED):                               │
│  ════════════════════════                                              │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ posts_with_authors                                                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ id                │                                              │   │
│  │ user_id           │                                              │   │
│  │ user_name         │ ←─ DENORMALIZED (from users table)        │   │
│  │ user_email        │ ←─ DENORMALIZED (from users table)        │   │
│  │ title            │                                              │   │
│  │ body             │                                              │   │
│  │ created_at       │                                              │   │
│  │ comment_count    │ ←─ DENORMALIZED (aggregated)               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  TO FIND USER'S POSTS:                                                 │
│  SELECT * FROM posts_with_authors WHERE user_id = ?                  │
│                                                                          │
│  PROS:                                                                │
│  • Fewer joins = faster reads                                         │
│  • Simpler queries                                                   │
│                                                                          │
│  CONS:                                                                 │
│  • Data duplication                                                  │
│  • Updates must sync multiple places                                   │
│  • More storage                                                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  DECISION FRAMEWORK:                                                   │
│  ───────────────────                                                   │
│                                                                          │
│  Use NORMALIZED when:                                                  │
│  • Writing frequency >= reading frequency                              │
│  • Data changes frequently                                            │
│  • ACID transactions are critical                                      │
│  • Storage is expensive                                                │
│                                                                          │
│  Use DENORMALIZED when:                                                │
│  • Reading frequency >> writing frequency                             │
│  • Data is relatively static                                          │
│  • Query performance is critical                                      │
│  • Using read replicas (replication lag)                               │
│                                                                          │
│  COMMON HYBRID APPROACH:                                              │
│  • Normalized database as source of truth                             │
│  • Denormalized views for read-heavy workloads                       │
│  • Materialized views, cached data, read replicas                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Common Schema Patterns

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEMA DESIGN PATTERNS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PATTERN 1: ENUM FIELDS                                               │
│  ──────────────────────                                               │
│                                                                          │
│  Instead of:  status VARCHAR (allowed: 'pending', 'active', 'done')  │
│  Use:          status ENUM ('pending', 'active', 'done')             │
│                                                                          │
│  Benefits:                                                              │
│  • Database enforces valid values                                       │
│  • Self-documenting                                                    │
│  • More efficient storage                                              │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PATTERN 2: SOFT DELETES                                              │
│  ─────────────────────                                                 │
│                                                                          │
│  Instead of:  DELETE FROM users WHERE id = ?                          │
│  Use:          UPDATE users SET deleted_at = NOW() WHERE id = ?       │
│                                                                          │
│  Benefits:                                                             │
│  • Preserves data integrity                                           │
│  • Audit trail maintained                                              │
│  • Can "undelete"                                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PATTERN 3: AUDIT COLUMNS                                             │
│  ──────────────────────                                               │
│                                                                          │
│  Every table:                                                          │
│  • created_at TIMESTAMP DEFAULT NOW()                                  │
│  • created_by UUID                                                     │
│  • updated_at TIMESTAMP                                                │
│  • updated_by UUID                                                     │
│                                                                          │
│  Benefits:                                                             │
│  • Track who did what                                                 │
│  • Debug data issues                                                  │
│  • Compliance requirements                                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PATTERN 4: POLYMORPHIC RELATIONSHIPS                                 │
│  ────────────────────────────────────                                  │
│                                                                          │
│  Instead of:  likes table with (user_id, tweet_id)                    │
│               likes table with (user_id, comment_id)                  │
│                                                                          │
│  Use:          likes table with (user_id, object_id, object_type)      │
│                                                                          │
│  object_type: 'tweet' | 'comment' | 'post'                            │
│                                                                          │
│  Alternative: Separate tables for each type                            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PATTERN 5: STORAGE COLUMNS                                           │
│  ────────────────────────                                              │
│                                                                          │
│  JSON/JSONB columns for semi-structured data:                          │
│                                                                          │
│  CREATE TABLE events (                                                │
│      id UUID PRIMARY KEY,                                             │
│      event_type VARCHAR NOT NULL,                                      │
│      payload JSONB,                                                    │
│      created_at TIMESTAMP DEFAULT NOW()                               │
│  );                                                                    │
│                                                                          │
│  Benefits:                                                             │
│  • Schema flexibility                                                 │
│  • Handle new fields without migrations                                │
│  • Good for event data                                                │
│                                                                          │
│  Trade-offs:                                                           │
│  • Can't index inside JSON easily (use partial indexes)               │
│  • Less validation                                                    │
│  • Query performance varies                                            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PATTERN 6: INHERITANCE (Table Partitioning)                          │
│  ─────────────────────────────────────────                             │
│                                                                          │
│  orders                                                                 │
│  ├── orders_2024_q1 (inherits from orders)                            │
│  ├── orders_2024_q2 (inherits from orders)                            │
│  └── orders_2024_q3 (inherits from orders)                            │
│                                                                          │
│  Benefits:                                                             │
│  • Efficient archival                                                 │
│  • Partition pruning for queries                                      │
│  • Independent maintenance                                            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PATTERN 7: SNAPSHOT / IMMUTABLE TABLES                               │
│  ─────────────────────────────────────────                             │
│                                                                          │
│  For financial/audit data:                                            │
│                                                                          │
│  balances_snapshot (immutable)                                        │
│  ├── id                                                               │
│  ├── user_id                                                          │
│  ├── balance_at (immutable snapshot)                                  │
│  ├── snapshot_date                                                    │
│  ├── created_at                                                       │
│                                                                          │
│  Never UPDATE - always INSERT new row                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Error Handling

### 6.1 Error Response Format

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ERROR RESPONSE FORMAT                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STANDARD ERROR RESPONSE:                                              │
│  ─────────────────────────                                              │
│                                                                          │
│  {                                                                     │
│    "error": {                                                          │
│      "code": "VALIDATION_ERROR",                                      │
│      "message": "The request body contains invalid fields",          │
│      "details": [                                                      │
│        {                                                              │
│          "field": "email",                                            │
│          "message": "Invalid email format"                             │
│        },                                                              │
│        {                                                              │
│          "field": "password",                                         │
│          "message": "Password must be at least 8 characters"          │
│        }                                                              │
│      ],                                                                │
│      "request_id": "req_abc123"                                       │
│    }                                                                   │
│  }                                                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  COMMON ERROR CODES:                                                   │
│  ─────────────────                                                      │
│                                                                          │
│  {                                                                     │
│    "VALIDATION_ERROR": "Request validation failed",                   │
│    "NOT_FOUND": "Requested resource not found",                       │
│    "UNAUTHORIZED": "Authentication required",                          │
│    "FORBIDDEN": "Insufficient permissions",                           │
│    "RATE_LIMITED": "Too many requests",                               │
│    "INTERNAL_ERROR": "Unexpected server error",                       │
│    "SERVICE_UNAVAILABLE": "Service temporarily unavailable",           │
│    "CONFLICT": "Resource state conflict"                              │
│  }                                                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HTTP STATUS + ERROR CODE MAPPING:                                    │
│  ──────────────────────────────────────                                │
│                                                                          │
│  400 + VALIDATION_ERROR                                                │
│  400 + INVALID_REQUEST                                                 │
│  400 + CONFLICT                                                        │
│                                                                          │
│  401 + UNAUTHORIZED                                                    │
│  401 + INVALID_TOKEN                                                   │
│                                                                          │
│  403 + FORBIDDEN                                                       │
│  403 + INSUFFICIENT_PERMISSIONS                                       │
│                                                                          │
│  404 + NOT_FOUND                                                       │
│                                                                          │
│  429 + RATE_LIMITED                                                    │
│                                                                          │
│  500 + INTERNAL_ERROR                                                  │
│  502 + BAD_GATEWAY                                                     │
│  503 + SERVICE_UNAVAILABLE                                             │
│  504 + GATEWAY_TIMEOUT                                                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  BEST PRACTICES:                                                        │
│  ────────────────                                                       │
│                                                                          │
│  1. Always include request_id for debugging                           │
│                                                                          │
│  2. Don't expose internal error details in production                 │
│                                                                          │
│  3. Include helpful error message for developers                       │
│                                                                          │
│  4. Use consistent error codes across API                             │
│                                                                          │
│  5. Document error responses in OpenAPI                                │
│                                                                          │
│  6. Rate limit errors should include retry-after header              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

### 7.1 API Design Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API DESIGN CHECKLIST                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  URL DESIGN:                                                            │
│  • Use nouns, not verbs                                               │
│  • Plural resource names                                               │
│  • Proper HTTP methods                                                 │
│  • Nested relationships only when necessary                             │
│                                                                          │
│  VERSIONING:                                                            │
│  • Version in URL (recommended)                                        │
│  • Support previous version for 6-12 months                            │
│  • Communicate deprecation clearly                                     │
│                                                                          │
│  PAGINATION:                                                            │
│  • Use cursor-based pagination for feeds                               │
│  • Use offset for search results                                       │
│  • Never return unbounded results                                       │
│                                                                          │
│  ERROR HANDLING:                                                        │
│  • Consistent error format                                             │
│  • Appropriate HTTP status codes                                        │
│  • Include request_id                                                  │
│  • Don't expose internals in production                                │
│                                                                          │
│  SCHEMA:                                                                │
│  • Use proper types (not just VARCHAR)                                │
│  • Add audit columns                                                   │
│  • Consider soft deletes                                                │
│  • Normalize for OLTP, denormalize for OLAP                            │
│                                                                          │
│  DOCUMENTATION:                                                         │
│  • OpenAPI spec                                                        │
│  • Example requests/responses                                          │
│  • Error codes documented                                               │
│  • Changelog                                                           │
│                                                                          │
│  SECURITY:                                                              │
│  • TLS required                                                        │
│  • Rate limiting                                                       │
│  • Input validation                                                    │
│  • No sensitive data in URLs                                           │
│                                                                          │
│  PERFORMANCE:                                                           │
│  • Pagination                                                          │
│  • Field selection (allow ?fields=)                                    │
│  • Compression                                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
