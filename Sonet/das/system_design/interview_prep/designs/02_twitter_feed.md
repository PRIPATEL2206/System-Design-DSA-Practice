# Design: Twitter / Instagram News Feed

## Problem Statement
Design a social media news feed system where users can post content and see a ranked, personalised feed of posts from people they follow.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "How many followers can a user have? Any celebrity accounts?" → Yes, up to millions (hotspot problem)
- "Is the feed ranked (ML) or chronological?" → Ranked by relevance
- "Should we support images/video or text only?" → Text + images
- "Does the feed update in real-time?" → Near real-time (not live push)
- "What's the DAU?" → 300 million

### Functional Requirements
- Users can post tweets/posts (text, image)
- Users follow other users
- Users see a feed of posts from accounts they follow
- Feed is ranked by recency + relevance

### Non-Functional Requirements
- Feed load time < 200ms
- Eventual consistency is OK (slight delay is fine)
- High availability (99.99%)
- Scale: 300M DAU, 100M posts/day

---

## Step 2: Capacity Estimation

```
DAU = 300 million
Posts/day = 100 million
Reads/day = 300M × 100 posts viewed = 30 billion
Read/Write ratio ≈ 300:1 (read-heavy)

Write QPS = 100M / 86,400 ≈ 1,157 RPS
Read QPS  = 30B  / 86,400 ≈ 347,000 RPS
Peak Read QPS ≈ 700,000 RPS

Storage:
  Text post = 500 bytes
  Daily text = 100M × 500 bytes = 50 GB/day
  Images (30% of posts, 1MB avg) = 30M × 1 MB = 30 TB/day
  5-year text only ≈ 90 TB
  5-year with media ≈ 54 PB
```

---

## Step 3: Two Feed Architectures

### Approach 1: Pull Model (Fan-out on Read)
```
User requests feed:
1. Get user's following list (e.g., 500 accounts)
2. For each account, fetch latest posts from DB
3. Merge, sort, rank → return top 20

Pro: Simple, data always fresh
Con: Very slow for users following 1000+ accounts
     High DB load on every feed request
```

### Approach 2: Push Model (Fan-out on Write) — Twitter's core approach
```
When user A posts:
1. Find all A's followers (e.g., 500 followers)
2. Push post into each follower's "pre-built feed" (Redis sorted set)
3. When follower requests feed → just read their pre-built list (O(1))

Pro: Feed reads are instant (pre-computed)
Con: Celebrity with 10M followers → 10M writes per post (hotspot)
```

### Approach 3: Hybrid (Best for interview)
```
Regular users (< 10K followers):  Use Push (fan-out on write)
Celebrities (> 10K followers):    Use Pull (lazy load on read, blend with feed)

On read:
1. Fetch pre-built feed from Redis (for regular users you follow)
2. Fetch recent posts from celebrities you follow (pull)
3. Merge, de-duplicate, rank
4. Cache result for 1 minute
```

---

## Step 4: High-Level Architecture

```
[User App]
    |
[API Gateway]
    |
+---------------+         +--------------------+
| Post Service  | ──pub──> | Kafka (post events)|
+---------------+          +--------------------+
       |                          |
[Object Store]            [Fan-out Service]
  (S3 for media)                  |
                          +-----------------+
                          | For each follow |
                          | ZADD feed:{uid} |
                          |  (Redis SortedSet)|
                          +-----------------+
                          
[Feed Service] ──reads──> [Redis: feed:{user_id}]
                          (pre-built feed, score = timestamp)
                          
                          ──celebrity fetch──> [Post DB]
                          
[Post DB (MySQL + sharding)] ── read replicas ──> [Feed Service]
[User Graph DB (follows)]
[Media CDN] ──serves images──> [User App]
```

---

## Step 5: Component Deep Dive

### Post Service
```
1. Accept post (text + optional image)
2. Save image to S3, get CDN URL
3. Write post to Post DB
4. Publish event to Kafka topic "new_post"
```

### Fan-out Service (Kafka Consumer)
```
Consumes "new_post" event:
1. Query User Graph DB for poster's followers
2. For regular users: ZADD feed:{follower_id} {timestamp} {post_id}
   (Redis Sorted Set — sorted by score = timestamp)
3. For celebrities (> 10K followers): skip fan-out
   Their posts are fetched on read.
4. Evict old posts from sorted set (keep last 1000 posts per user)
```

### Feed Service (Read Path)
```
GET /feed?user_id=123&page=1:
1. ZREVRANGE feed:123 0 49  → top 50 post IDs from Redis
2. Batch fetch post details from Post DB (or Post Cache)
3. Fetch top 5 posts from celebrities user follows (Post DB, cached)
4. Merge, deduplicate, apply ranking model
5. Return enriched posts
```

---

## Step 6: Data Model

### posts table
```sql
CREATE TABLE posts (
  id          BIGINT PRIMARY KEY,  -- snowflake ID
  user_id     BIGINT NOT NULL,
  content     TEXT,
  media_url   VARCHAR(255),
  created_at  DATETIME,
  like_count  INT DEFAULT 0,
  reply_count INT DEFAULT 0
);
-- Shard by user_id
-- Index: (user_id, created_at DESC)
```

### follows table
```sql
CREATE TABLE follows (
  follower_id   BIGINT NOT NULL,
  followee_id   BIGINT NOT NULL,
  created_at    DATETIME,
  PRIMARY KEY (follower_id, followee_id)
);
-- Index: (followee_id) for fan-out lookups
```

### Redis Feed Structure
```
Key: "feed:{user_id}"
Type: Sorted Set
Score: Unix timestamp
Member: post_id

ZADD feed:123 1697000000 "post:456"
ZREVRANGE feed:123 0 19  → top 20 most recent post IDs
```

---

## Step 7: API Design

```
POST /api/v1/posts
Body: { "content": "Hello world!", "media": "base64..." }
Response: { "post_id": "789", "url": "..." }

GET /api/v1/feed?cursor=abc&limit=20
Response: { "posts": [...], "next_cursor": "xyz" }

GET /api/v1/users/{id}/posts?cursor=abc
Response: { "posts": [...], "next_cursor": "xyz" }

POST /api/v1/posts/{id}/like
POST /api/v1/users/{id}/follow
```

---

## Step 8: Scaling & Edge Cases

### Hot Celebrity Problem
```
Justin Bieber (100M followers) posts.
Push approach: 100M Redis writes in < 1 second → impossible.

Solution: Skip fan-out for celebrities.
Tag accounts with > 10K followers as "hot users".
On feed read: blend regular pre-built feed with celebrity posts fetched on demand.
```

### Timeline Inconsistency
```
User sees post in feed, clicks profile, post not visible yet.
Solution: Short TTL cache (< 1 min). Eventual consistency is acceptable for social feeds.
```

### Sharding the Post DB
```
Shard by user_id (all posts from one user on same shard)
Pro: User's post page is single-shard query
Con: Hot user (celebrity) = hot shard

Alternative: Shard by post_id (hash-based)
Pro: Even distribution
Con: User's posts scattered across shards (page = scatter-gather)
```

---

## Interview Tips

- Lead with: "This is extremely read-heavy (300:1). The key challenge is serving feed reads fast."
- Hybrid fan-out model is the key insight — mention regular vs celebrity accounts.
- Mention cursor-based pagination (not offset) for infinite scroll.
- Discuss: pre-computed feed trade-off (fast reads, complex writes) vs pull (simple, slow reads).

## Common Follow-up Questions
- "What if a user has 1 billion followers?" → Only pull, no fan-out; CDN for posts
- "How to add trending posts to the feed?" → ML ranking service merges with feed
- "How to rank the feed by ML?" → Feature store + ranking service re-scores before returning
- "What if Redis is down?" → Fall back to DB query (slower but functional)
