# System Design Case Study: Twitter/X Home Timeline

> A comprehensive walkthrough of designing Twitter's Home Timeline at scale, with capacity planning, API design, failure analysis, and interview-ready depth.

---

## 1. Requirements Analysis

### 1.1 Functional Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FUNCTIONAL REQUIREMENTS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  F1: Home Timeline Display                                              │
│      • Display tweets from followed users in reverse chronological    │
│      • Support pagination (infinite scroll)                            │
│      • Maximum 1000 tweets per timeline                                 │
│                                                                          │
│  F2: Tweet Creation                                                     │
│      • Create text tweets (max 280 characters)                         │
│      • Support media attachments (images, videos)                     │
│      • Support quote tweets                                             │
│                                                                          │
│  F3: Social Actions                                                     │
│      • Like/Unlike tweets                                               │
│      • Retweet                                                          │
│      • Reply to tweets                                                  │
│                                                                          │
│  F4: User Following                                                     │
│      • Follow/unfollow users                                            │
│      • Follower/following counts                                        │
│                                                                          │
│  F5: Real-time Updates                                                 │
│      • New tweets appear within 5 seconds                               │
│      • Likes/retweets update in real-time                              │
│                                                                          │
│  F6: Content Filtering                                                 │
│      • Mute keywords/users                                              │
│      • Quality filtering (removal of low-quality content)             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NON-FUNCTIONAL REQUIREMENTS                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SCALABILITY:                                                           │
│  • 500M monthly active users                                           │
│  • 200M daily active users                                             │
│  • 500M tweets per day average (peak: 1B+)                             │
│  • 200B timeline queries per day                                        │
│                                                                          │
│  LATENCY:                                                               │
│  • Timeline load: p99 < 200ms                                          │
│  • Tweet creation: p99 < 100ms                                          │
│  • Real-time updates: < 5 seconds                                       │
│                                                                          │
│  AVAILABILITY:                                                           │
│  • 99.95% uptime (4.38 hours downtime/year)                            │
│  • Multi-region deployment                                              │
│  • Graceful degradation under load                                      │
│                                                                          │
│  CONSISTENCY:                                                           │
│  • Timeline: Eventual consistency (1-5 seconds)                       │
│  • Social actions: Strong consistency                                   │
│  • Tweet creation: Strong consistency                                   │
│                                                                          │
│  STORAGE:                                                               │
│  • 10+ years tweet retention                                           │
│  • Petabytes of media storage                                          │
│  • Cost optimization for cold data                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Capacity Estimation

### 2.1 QPS Calculations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CAPACITY ESTIMATION                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  USER METRICS:                                                          │
│  ─────────────                                                         │
│  • Monthly Active Users (MAU): 500M                                     │
│  • Daily Active Users (DAU): 200M (40% of MAU)                         │
│  • Average tweets per user per day: 5                                  │
│  • Average timeline views per user per day: 20                         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WRITE QPS:                                                              │
│  ─────────                                                              │
│  • Tweets: 200M users × 5 tweets / 86400 sec                          │
│           = ~12,000 tweets/second average                               │
│           = ~50,000 tweets/second peak                                  │
│                                                                          │
│  • Likes: Assume 10x tweets                                             │
│           = ~120,000 likes/second average                               │
│                                                                          │
│  READ QPS:                                                              │
│  ──────────                                                             │
│  • Timeline queries: 200M × 20 / 86400                                │
│           = ~46,000 queries/second average                              │
│           = ~200,000 queries/second peak                                │
│                                                                          │
│  PEAK MULTIPLIER: 3-5x average                                          │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STORAGE ESTIMATION:                                                    │
│  ─────────────────                                                      │
│  Tweet storage:                                                          │
│  • 500M tweets/day × 365 days × 10 years                              │
│  • = 1.825 trillion tweets total                                       │
│  • Average tweet: ~1KB (text + metadata)                               │
│  • = 1.825 PB text data                                                 │
│                                                                          │
│  Media storage:                                                          │
│  • 10% of tweets have media                                             │
│  • Average media: 100KB                                                 │
│  • = 500M × 10% × 100KB × 365 × 10 years                              │
│  • = 182.5 PB media storage                                            │
│                                                                          │
│  USER GRAPH:                                                            │
│  • 500M users × ~500 following average                                 │
│  • = 250B follow edges                                                  │
│  • Edge storage: ~2.5TB (proper indexing)                               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  NETWORK BANDWIDTH:                                                      │
│  ─────────────────                                                      │
│  • Timeline response: ~50KB average (100 tweets)                       │
│  • Timeline queries: 46,000 QPS × 50KB                                 │
│  • = 2.3 GB/s = 18 Tbps                                                │
│                                                                          │
│  CACHE:                                                                 │
│  • Hot timeline data: 1M users × 1MB each                              │
│  • = 1TB Redis cluster                                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. API Design

### 3.1 REST API Endpoints

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API DESIGN - REST                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  TIMELINE:                                                              │
│  ─────────                                                              │
│  GET    /api/v2/timeline/home                                          │
│          Query params:                                                  │
│            • count: number of tweets (default 20, max 100)            │
│            • cursor: pagination cursor                                 │
│            • with_tweet_payloads: include full tweet objects           │
│                                                                          │
│          Response:                                                      │
│          {                                                             │
│            "data": [tweet_objects],                                    │
│            "meta": {                                                   │
│              "result_count": 20,                                       │
│              "next_cursor": "pagination_token",                       │
│              "previous_cursor": "..."                                  │
│            }                                                           │
│          }                                                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  TWEET OPERATIONS:                                                      │
│  ────────────────                                                      │
│  POST   /api/v2/tweets                                                  │
│          Body:                                                         │
│          {                                                             │
│            "text": "Hello world!",                                     │
│            "media": { "media_ids": ["123"] },                         │
│            "reply": { "in_reply_to_tweet_id": "456" }                │
│          }                                                             │
│                                                                          │
│  DELETE /api/v2/tweets/:id                                              │
│                                                                          │
│  POST   /api/v2/tweets/:id/like                                         │
│  POST   /api/v2/tweets/:id/unlike                                       │
│  POST   /api/v2/tweets/:id/retweet                                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  USER OPERATIONS:                                                       │
│  ────────────────                                                      │
│  GET    /api/v2/users/:id                                               │
│  POST   /api/v2/users/:id/follow                                        │
│  DELETE /api/v2/users/:id/follow                                        │
│  GET    /api/v2/users/:id/followers                                     │
│  GET    /api/v2/users/:id/following                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PAGINATION:                                                            │
│  ────────────                                                           │
│  • Cursor-based pagination (not offset)                                │
│  • Cursor is opaque, generated by server                                │
│  • Prevents "skipping" and is stable for inserts                       │
│  • Example: /timeline?cursor=eyJpZCI6MTAwfQ==                         │
│                                                                          │
│  RATE LIMITING:                                                         │
│  ─────────────                                                          │
│  • Per-user: 300 tweets/day, 1000 likes/day                            │
│  • Per-IP: 300 requests/15 minutes                                     │
│  • Headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 gRPC Service Definition (Optional)

```protobuf
// twitter.proto
syntax = "proto3";

package twitter.v2;

service TimelineService {
  rpc GetHomeTimeline(GetTimelineRequest) returns (GetTimelineResponse);
  rpc StreamTimelineUpdates(StreamRequest) returns (stream Tweet);
}

service TweetService {
  rpc CreateTweet(CreateTweetRequest) returns (Tweet);
  rpc DeleteTweet(DeleteTweetRequest) returns (Empty);
  rpc LikeTweet(LikeRequest) returns (Empty);
}

message GetTimelineRequest {
  string user_id = 1;
  int32 count = 2;
  string cursor = 3;
}

message GetTimelineResponse {
  repeated Tweet tweets = 1;
  string next_cursor = 2;
}

message Tweet {
  string id = 1;
  string text = 2;
  User author = 3;
  int64 created_at = 4;
  Metrics metrics = 5;
  bool liked = 6;
  bool retweeted = 7;
}

message User {
  string id = 1;
  string username = 2;
  string display_name = 3;
  string avatar_url = 4;
}
```

---

## 4. Data Model

### 4.1 Database Schema

```sql
-- Core Tweet table (PostgreSQL)
CREATE TABLE tweets (
    id              BIGSERIAL PRIMARY KEY,
    tweet_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    text            VARCHAR(280) NOT NULL,
    media_urls      TEXT[],  -- Array of media URLs
    reply_to_id     UUID,
    retweet_of_id   UUID,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Denormalized for performance
    like_count      BIGINT DEFAULT 0,
    retweet_count   BIGINT DEFAULT 0,
    reply_count     BIGINT DEFAULT 0,
    
    -- Indexes
    CONSTRAINT tweet_user_id_fk FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_tweets_user_time ON tweets(user_id, created_at DESC);
CREATE INDEX idx_tweets_created_at ON tweets(created_at DESC);

-- User table
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50) UNIQUE NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    bio             TEXT,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Follow graph (adjacency list)
CREATE TABLE follows (
    follower_id     UUID NOT NULL,
    following_id    UUID NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(id),
    FOREIGN KEY (following_id) REFERENCES users(id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);

-- Like records
CREATE TABLE likes (
    user_id         UUID NOT NULL,
    tweet_id        UUID NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (user_id, tweet_id)
);

-- Timeline cache (Redis)
-- Key: timeline:{user_id}
-- Value: Sorted set of tweet_ids by score (timestamp)
-- TTL: 7 days
```

### 4.2 Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATA FLOW - TWEET CREATION                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────┐     ┌──────────────┐     ┌─────────────────┐            │
│  │  User   │────▶│ API Gateway  │────▶│ Tweet Service   │            │
│  │         │     │ (Rate Limit, │     │ (Validation)   │            │
│  │ POST    │     │  Auth)       │     └────────┬────────┘            │
│  │ /tweets │     └──────────────┘              │                     │
│  └─────────┘                                   │                     │
│                                                ▼                     │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │              POSTGRES (System of Record)                    │       │
│  │  1. INSERT into tweets table                               │       │
│  │  2. UPDATE user's tweet_count                              │       │
│  │  3. Create like/retweet counts (init to 0)                │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                │                     │
│                                   ┌────────────┴───────────┐           │
│                                   ▼                        ▼           │
│  ┌──────────────────────────────────────┐  ┌────────────────────────┐ │
│  │ CDC (Debezium)                       │  │ Kafka (TweetCreated)   │ │
│  │ PostgreSQL → Kafka                   │  │ Partition by tweet_id │ │
│  └──────────────────────────────────────┘  └───────────┬────────────┘ │
│                                                        │              │
│                          ┌─────────────────────────────┼───────────┐  │
│                          ▼                             ▼           ▼  │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  ┌─────────────┐  │
│  │ Elastic-    │  │   Fan-out    │  │ Analytics │  │ Notification│  │
│  │ search      │  │   Service    │  │   (Snow-  │  │  Service    │  │
│  │ (searchable)│  │ (timeline    │  │   flake)  │  │ (email,     │  │
│  └─────────────┘  │  update)     │  └───────────┘  │  push)       │  │
│                   └──────────────┘                  └─────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Architecture

### 5.1 High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        Mobile[Mobile App]
        Web[Web Client]
    end

    subgraph "Edge Layer"
        CDN[CDN / CloudFlare]
        LB[Load Balancer]
    end

    subgraph "API Layer"
        Gateway[API Gateway]
        Auth[Auth Service]
        Rate[Rate Limiter]
    end

    subgraph "Service Layer"
        Tweet[T User[User Service]
       weet Service]
        Timeline[Timeline Service]
        Follow[Follow Service]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL)]
        Redis[(Redis Cache)]
        Kafka[Kafka]
    end

    Mobile --> CDN
    Web --> CDN
    CDN --> LB
    LB --> Gateway
    Gateway --> Auth
    Gateway --> Rate
    Auth --> Tweet
    Auth --> User
    Auth --> Timeline
    Auth --> Follow
    
    Tweet --> PG
    Timeline --> Redis
    Timeline --> PG
    Follow --> PG
    
    Tweet --> Kafka
    Kafka --> Timeline
```

### 5.2 Timeline Generation Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│              TIMELINE GENERATION - FAN-OUT                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  TWO APPROACHES:                                                         │
│                                                                          │
│  APPROACH 1: PUSH (FAN-OUT ON WRITE)                                    │
│  ───────────────────────────────────                                    │
│                                                                          │
│  When user tweets:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. Get all followers of user (from follows table)              │    │
│  │    • If follower count < 10,000: push to each timeline cache  │    │
│  │    • If follower count >= 10,000: use pull-on-read             │    │
│  │                                                                  │    │
│  │ 2. For each follower:                                           │    │
│  │    ZADD timeline:{follower_id} tweet_id score (timestamp)    │    │
│  │                                                                  │    │
│  │ 3. Trim each timeline to max 1000 tweets                       │    │
│  │    ZREMRANGEBYRANK timeline:{id} 0 -1001                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  PROS:                                                                  │
│  • Read is fast (just read from cache)                                 │
│  • Good for users with many followers                                  │
│                                                                          │
│  CONS:                                                                  │
│  • Write latency is high for viral tweets                              │
│  • If user has 50M followers, that's 50M writes!                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  APPROACH 2: PULL (FAN-OUT ON READ)                                     │
│  ───────────────────────────────────                                    │
│                                                                          │
│  When user loads timeline:                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. Get list of followed users (cache in Redis)                │    │
│  │                                                                  │    │
│  │ 2. For each followed user:                                     │    │
│  │    Fetch their recent tweets (from tweet cache or DB)          │    │
│  │                                                                  │    │
│  │ 3. Merge all tweets, sort by timestamp                         │    │
│  │ 4. Return top N                                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  PROS:                                                                  │
│  • Write is fast (only write tweet to DB)                               │
│  • Works for any follower count                                         │
│                                                                          │
│  CONS:                                                                   │
│  • Read is slow (must fetch from many sources)                         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HYBRID APPROACH (Twitter's actual approach):                          │
│  ─────────────────────────────────────────────                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │   If followed_by_count < 10,000:                                │    │
│  │       → Use PUSH (fan-out on write)                            │    │
│  │                                                                  │    │
│  │   If followed_by_count >= 10,000:                               │    │
│  │       → Use PULL (fan-out on read)                             │    │
│  │       → Maintain "celebrity list" for pull optimization        │    │
│  │                                                                  │    │
│  │   Celebrity tweets are indexed differently                      │    │
│  │   Timeline pulls from celebrity cache at read time             │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Bottleneck Analysis

### 6.1 Identified Bottlenecks

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BOTTLENECK ANALYSIS                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. FAN-OUT WRITE AMPLIFICATION                                         │
│  ────────────────────────────                                           │
│  Problem: Celebrity tweets (10M+ followers) cause massive fan-out       │
│  Solution: Hybrid push/pull, read-time celebrity aggregation            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  2. TIMELINE STORAGE                                                    │
│  ───────────────────                                                     │
│  Problem: 200M timelines × 1000 tweets = massive Redis/storage        │
│  Solution:                                                                 │
│    • LRU eviction of old timelines                                      │
│    • Generate on-demand, cache hot timelines                            │
│    • Use tiered storage (hot: Redis, warm: Cassandra, cold: S3)         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  3. DATABASE CONNECTIONS                                                 │
│  ─────────────────────                                                   │
│  Problem: High QPS can exhaust connection pool                          │
│  Solution:                                                                 │
│    • Read replicas for timeline queries                                  │
│    • Connection pooling (PgBouncer)                                     │
│    • Circuit breakers                                                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  4. NETWORK BANDWIDTH                                                   │
│  ───────────────────                                                    │
│  Problem: 18 Tbps for timeline delivery                                 │
│  Solution:                                                                 │
│    • Edge caching (CloudFlare, Fastly)                                  │
│    • Compression (gRPC, Brotli)                                        │
│    • Pagination (load older on demand)                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  5. CONSISTENCY vs LATENCY                                              │
│  ──────────────────────────                                             │
│  Problem: Timeline reads should be fast but consistent                  │
│  Solution:                                                                 │
│    • Eventual consistency for timeline (1-5 sec lag acceptable)         │
│    • Strong consistency for user actions (like, follow)                │
│    • Read-your-writes consistency for own timeline                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Failure Analysis

### 7.1 Failure Modes and Mitigations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FAILURE ANALYSIS                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  FAILURE: Database Primary Down                                         │
│  ─────────────────────────────                                          │
│  Impact: No new tweets can be created                                   │
│  Mitigation:                                                               │
│    • Automatic failover to standby (30 seconds)                        │
│    • Read replicas available for reads                                  │
│    • Queue tweets for retry during failover                              │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  FAILURE: Redis Timeline Cache Down                                      │
│  ────────────────────────────────────                                    │
│  Impact: Timeline loads slowly (must query DB)                          │
│  Mitigation:                                                               │
│    • Fall back to pull-based timeline generation                        │
│    • Reduced functionality acceptable                                    │
│    • Circuit breaker on cache client                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  FAILURE: Fan-out Service Overloaded                                     │
│  ──────────────────────────────────────                                  │
│  Impact: Delayed timeline updates for followers                         │
│  Mitigation:                                                               │
│    • Async message queue (Kafka)                                         │
│    • Backpressure handling                                               │
│    • Priority queue for verified users                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  FAILURE: Celebrity Account Viral Tweet                                  │
│  ─────────────────────────────────────────                              │
│  Impact: Massive fan-out spike, potential system overload               │
│  Mitigation:                                                               │
│    • Rate limiting on tweet creation                                    │
│    • Circuit breakers                                                    │
│    • Gradual fan-out (not all at once)                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  FAILURE: Region Outage                                                  │
│  ───────────────────                                                    │
│  Impact: Users in region lose service                                   │
│  Mitigation:                                                               │
│    • Multi-region active-active (read and write)                        │
│    • DNS failover (Route 53 health checks)                              │
│    • Data replication between regions                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  FAILURE: Cache Invalidation Race                                        │
│  ─────────────────────────────                                          │
│  Impact: User sees stale data after tweet delete                        │
│  Mitigation:                                                               │
│    • Delete events invalidate cache                                      │
│    • TTL as safety net                                                   │
│    • Eventual consistency acceptable                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Monitoring and SLOs

### 8.1 Key Metrics

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MONITORING & SLOs                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SERVICE LEVEL INDICATORS (SLIs):                                        │
│  ─────────────────────────────────                                      │
│                                                                          │
│  | Metric                | Target        | Measurement                  |  │
│  |-----------------------|---------------|------------------------------|  │
│  | Timeline API latency | p99 < 200ms   | 1-minute average            |  │
│  | Tweet creation p99   | < 100ms       | 1-minute average            |  │
│  | API availability     | 99.95%        | 5-minute windows            |  │
│  | Fan-out latency       | p99 < 5 sec   | End-to-end                  |  │
│  | Cache hit rate       | > 95%         | For timeline queries         |  │
│  | Error rate           | < 0.1%        | 5xx errors                  |  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  SLOs:                                                                   │
│  ────                                                                    │
│                                                                          │
│  SLO 1: "Timeline loads in under 200ms for 99% of requests"           │
│          • Burn budget: 4.38 hours/year                                 │
│          • Error budget: 0.05%                                          │
│                                                                          │
│  SLO 2: "Tweets appear in followers' timelines within 5 seconds"       │
│          • Burn budget: 4.38 hours/year                                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  CRITICAL ALERTS:                                                       │
│  ─────────────────                                                      │
│                                                                          │
│  • Timeline API p99 > 500ms for 5 minutes                              │
│  • Error rate > 1% for 2 minutes                                        │
│  • Fan-out queue backlog > 1M messages                                  │
│  • Database CPU > 80% for 10 minutes                                   │
│  • Cache hit rate < 80%                                                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  DASHBOARD PANELS:                                                      │
│  ────────────────                                                        │
│                                                                          │
│  • Request rate (QPS) by endpoint                                       │
│  • Latency by percentile (p50, p95, p99)                               │
│  • Error rate by type (4xx, 5xx)                                        │
│  • Timeline cache size and hit rate                                     │
│  • Fan-out queue depth                                                  │
│  • Database connection pool usage                                       │
│  • Active users (DAU, MAU trends)                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Interview Follow-Up Questions

### Q1: "How would you handle a celebrity with 50 million followers?"

**Expected Answer Structure:**
1. **Detection**: Identify accounts with >10K followers
2. **Strategy**: Hybrid approach - use pull-on-read for high-follower accounts
3. **Optimization**: Cache celebrity tweets separately
4. **Tradeoff**: Accept slightly higher read latency for celeb tweets

### Q2: "How do you handle deleted tweets in cached timelines?"

**Expected Answer:**
1. **Tombstones**: Mark as deleted in cache
2. **TTL**: Short TTL handles eventually
3. **Invalidation**: Delete event triggers cache update
4. **Acceptance**: Some users may see briefly, acceptable

### Q3: "What happens if Redis cluster fails?"

**Expected Answer:**
1. **Detection**: Health checks, circuit breaker
2. **Fallback**: Pull-from-DB timeline generation
3. **Degradation**: Reduced functionality, slower reads
4. **Recovery**: Alert on cache miss spike
5. **Acceptability**: Users can still see tweets, just slower

### Q4: "How would you implement retweets efficiently?"

**Expected Answer:**
1. **Don't copy tweet**: Just store reference
2. **Denormalization**: Increment retweet_count atomically
3. **Timeline**: Treat retweet as new entry with original reference
4. **Display**: Resolve original tweet at read time

### Q5: "Design the pagination for infinite scroll."

**Expected Answer:**
1. **Cursor-based**: Never use offset (problematic with inserts)
2. **Token**: Opaque cursor = timestamp + tweet_id
3. **Pre-fetch**: Load next page before user scrolls
4. **Race conditions**: Handle new tweets between loads

---

## 10. Summary

| Component | Technology | Key Decision |
|-----------|------------|--------------|
| **Primary DB** | PostgreSQL | System of record |
| **Cache** | Redis | Timeline storage |
| **Message Queue** | Kafka | Event streaming |
| **Search** | Elasticsearch | Tweet search |
| **Media** | S3 + CloudFront | CDN delivery |
| **Fan-out** | Hybrid | Push < 10K, Pull >= 10K |
| **Consistency** | Eventual | 1-5 second lag acceptable |
| **Availability** | 99.95% | Multi-region |

**Key Insights:**
1. Timeline generation is the hardest problem
2. Hybrid push/pull optimizes for both normal and celebrity users
3. Caching is critical for read performance
4. Eventual consistency is acceptable for social feeds
5. Monitor fan-out latency as primary SLO
