# Design: Twitter/X Feed

**Frequency:** Very High | **Difficulty:** ⭐⭐⭐ | **Companies:** Meta, Google, Amazon, Twitter

---

## 1. Requirements

### Functional
- Post tweets (text, images, video)
- Follow/unfollow users
- Home timeline (feed of tweets from followed users)
- Like, retweet, reply
- Search tweets

### Non-Functional
- **Scale:** 500M users, 300M DAU, 500M tweets/day
- **Latency:** Feed loads in < 200ms
- **Availability:** 99.99%
- **Consistency:** Eventual (slight delay in feed OK)

### Estimation
```
Write (tweets): 500M/day ÷ 86400 ≈ 6K tweets/sec
Read (timeline): 300M DAU × 10 loads/day = 3B → 35K QPS
Timeline fanout: Avg user has 200 followers → 500M × 200 = 100B fan-out/day
```

---

## 2. Core Challenge: Feed Generation

### Approach: Hybrid Fan-out

```
Regular users (< 500K followers): FAN-OUT ON WRITE
──────────────────────────────────────────────────
When user posts:
  → Push tweet to all followers' feed caches (pre-computed)

Celebrity users (> 500K followers): FAN-OUT ON READ
──────────────────────────────────────────────────
When follower opens feed:
  → Pull celebrity tweets at read time, merge with pre-computed feed

Why hybrid?
  - Regular users: 200 followers × 6K tweets/sec = 1.2M writes/sec (manageable)
  - Celebrities: 10M followers × 1 tweet = 10M writes (too expensive per tweet)
  - At read time: merge ~50 celebrity feeds (fast, done in memory)
```

---

## 3. Architecture

```
┌──────────┐                                   ┌──────────────────┐
│  Client  │                                   │  Post Service    │
└────┬─────┘                                   │  (write tweets)  │
     │                                         └────────┬─────────┘
     │ GET /feed                                        │
     │                                                  ▼
┌────▼──────────┐    ┌──────────────┐    ┌─────────────────────┐
│ Feed Service  │───▶│ Feed Cache   │    │   Fan-out Service   │
│(read timeline)│    │ (Redis)      │    │ (push to follower   │
└───────────────┘    └──────────────┘    │  feed caches)       │
                                          └──────────┬──────────┘
     │                                               │
     │ celebrity feeds                               │ push
     ▼                                               ▼
┌──────────────┐                          ┌──────────────────┐
│ Tweet Store  │                          │  Feed Cache      │
│ (user tweets)│                          │  (per-user feed) │
│  Cassandra   │                          │  Redis           │
└──────────────┘                          └──────────────────┘
```

---

## 4. Data Model

```
Tweets Table (Cassandra):
  PK: tweet_id (Snowflake ID — time-ordered)
  user_id, content, media_urls, created_at, like_count, retweet_count

User Timeline (tweets BY a user):
  PK: user_id, CK: tweet_id (desc)
  → "Show me all tweets by user X"

Home Feed Cache (Redis):
  Key: feed:{user_id}
  Value: Sorted Set of tweet_ids (scored by timestamp)
  → Top 800 tweet_ids per user
  → On feed request: get tweet_ids → batch fetch tweet details

Follow Graph (separate service):
  Following: user_id → [list of who they follow]
  Followers: user_id → [list of who follows them]
  Store in: Redis (fast lookup) + persistent DB (source of truth)
```

---

## 5. Write Path (Posting a Tweet)

```
1. User posts tweet → Post Service
2. Store tweet in Tweets table (Cassandra)
3. Add to user's timeline (their profile)
4. Trigger fan-out:
   a. Get follower list
   b. For each follower: ZADD feed:{follower_id} <timestamp> <tweet_id>
   c. For celebrities: skip fan-out (handled at read time)
5. Push notification to mentioned users
6. Index in Elasticsearch (for search)

Fan-out workers: 
  - Queue: Kafka (partitioned by user_id for ordering)
  - Workers: Process fan-out asynchronously
  - Throughput: 1.2M fan-out writes/sec across worker fleet
```

---

## 6. Read Path (Loading Feed)

```
1. User opens app → GET /feed?cursor=<last_seen_tweet_id>
2. Feed Service:
   a. Get pre-computed feed from Redis: ZREVRANGE feed:{user_id} 0 19
   b. Get followed celebrities list
   c. For each celebrity: get recent tweets from their timeline
   d. Merge & rank (by time, engagement, ML relevance)
   e. Return top 20 tweets
3. Hydrate: Batch fetch full tweet objects (content, media, counts)
4. Return enriched feed to client

Optimization:
  - Cache hydrated tweets (popular tweets accessed millions of times)
  - Client-side infinite scroll with cursor pagination
  - Prefetch next page while user reads current page
```

---

## 7. Ranking (Making the Feed Smart)

```
Simple: Reverse chronological (newest first)
Better: ML-ranked feed (engagement prediction)

Features for ranking:
  - Recency (time since posted)
  - Author affinity (how often user interacts with author)
  - Content type (user prefers videos? images?)
  - Engagement velocity (tweet gaining likes quickly)
  - Social proof (friends liked it)

Pipeline:
  Candidates (1000 tweets) → Scoring Model → Top 20 → Display

  Training: Positive = user liked/retweeted/replied
            Negative = user scrolled past
```

---

## 8. Scaling

```
Fan-out Service: 
  - The heaviest component
  - Horizontally scaled workers consuming from Kafka
  - Circuit breaker for celebrity detection

Feed Cache (Redis):
  - Sharded by user_id
  - ~800 tweet_ids × 8 bytes = 6.4 KB per user
  - 500M users × 6.4 KB = 3.2 TB total → Redis cluster

Tweet Store (Cassandra):
  - Sharded by tweet_id
  - High write throughput naturally

Search (Elasticsearch):
  - Separate cluster
  - Indexing pipeline via Kafka

Global:
  - CDN for media (images, video thumbnails)
  - Multi-region for low latency globally
  - Feed generation is regional (users + their feed cache in same region)
```
