# System Design Case Studies — Interview-Ready Walkthroughs

## How to Use This Guide
Each case study follows the exact structure you should use in interviews. Practice explaining these out loud in 25-30 minutes.

---

## Case Study 1: URL Shortener (TinyURL)

### Requirements
- Shorten long URLs → short links (e.g., tiny.url/abc123)
- Redirect short URL → original URL
- Custom aliases (optional)
- Analytics (click count)
- Scale: 100M new URLs/day, 10:1 read-to-write ratio

### Capacity Estimation
- Writes: 100M/day ≈ 1200 QPS
- Reads: 1B/day ≈ 12K QPS
- Storage: 100M × 500 bytes ≈ 50 GB/day → 18 TB over 1 year
- Short URL length: 7 chars (base62) = 62⁷ ≈ 3.5 trillion combinations

### High-Level Design
```
Client → Load Balancer → App Server → Cache (Redis) → Database

Write flow: POST /shorten {long_url}
1. Generate short code (hash or counter)
2. Store mapping: short_code → long_url
3. Return short URL

Read flow: GET /{short_code}
1. Check cache for mapping
2. If miss, check DB
3. Return 301/302 redirect
```

### Key Decisions
| Decision | Choice | Why |
|----------|--------|-----|
| ID generation | Counter (distributed) + base62 | No collisions, predictable length |
| Redirect code | 301 (permanent) vs 302 (temporary) | 302 if we need analytics |
| Database | NoSQL (DynamoDB) | Simple key-value, high throughput |
| Cache | Redis with LRU | Hot URLs get cached |

### Deep Dive: ID Generation
- **Hash-based:** MD5/SHA → take first 7 chars. Problem: collisions
- **Counter-based:** Distributed counter (Zookeeper or Twitter Snowflake). No collisions.
- **Pre-generated:** Generate IDs in advance, assign from pool. Fast writes.

---

## Case Study 2: Twitter/X News Feed

### Requirements
- Post tweets (text, images)
- Follow/unfollow users
- News feed: see tweets from people you follow (sorted by time)
- Scale: 500M users, 300M DAU

### The Core Challenge: Fan-Out

**Fan-out on Write (Push model):**
- When user tweets, push to all followers' feeds
- Fast read (feed is pre-computed)
- Slow write for celebrities (fan-out to millions)

**Fan-out on Read (Pull model):**
- When user opens feed, fetch tweets from all followed users + merge
- Slow read (need to merge many sources)
- Fast write

**Hybrid (What Twitter Actually Does):**
- Regular users: fan-out on write
- Celebrities (>100K followers): fan-out on read
- Merge both at read time

### Architecture
```
Tweet Service → Message Queue → Fan-out Service → Feed Cache (per user)

Read: User → Feed Service → Feed Cache → Merge with celebrity tweets → Rank → Return

Storage:
- Tweets: Distributed DB (sharded by tweet_id)
- Social Graph: Graph DB or adjacency list in cache
- Feed Cache: Redis (sorted set per user, tweet_ids sorted by time)
```

### Key Design Points
- **Feed Storage:** Redis sorted set (score = timestamp, value = tweet_id). Keep only top 800 entries.
- **Ranking:** Initially chronological, then ML-based relevance scoring
- **Media:** Store in object storage (S3), serve via CDN

---

## Case Study 3: Chat System (WhatsApp)

### Requirements
- 1:1 messaging and group chat (up to 500 members)
- Online/offline status
- Message delivery guarantees (sent, delivered, read)
- Push notifications for offline users
- Scale: 2B users, 60B messages/day

### Architecture
```
Client ←WebSocket→ Chat Server → Message Queue → Receiver's Chat Server → Client

Offline: Message Queue → Push Notification Service → Device

Components:
- Connection Manager: Track which user connected to which server
- Message Service: Store and relay messages
- Presence Service: Online/offline/last seen
- Group Service: Group membership and fan-out
- Push Notification: For offline users
```

### Key Decisions

**Protocol:** WebSocket for real-time bidirectional communication
- HTTP polling: Too much overhead
- Long polling: OK but WebSocket is cleaner
- WebSocket: Persistent connection, low latency

**Message Delivery:**
```
Sent (✓)     → Server received the message
Delivered (✓✓) → Recipient's device got it
Read (✓✓ blue) → Recipient opened the chat
```

**Message Storage:**
- Use message queue (Kafka) for delivery
- Store messages in Cassandra (write-heavy, time-series nature)
- Key: chat_id + timestamp → natural ordering

**Group Messages:**
- Small groups: Fan-out on write (push to all members)
- Large groups: Fan-out on read (pull when user opens group)

---

## Case Study 4: YouTube / Video Streaming

### Requirements
- Upload videos
- Stream videos (adaptive quality)
- Search and recommendations
- Scale: 2B MAU, 500 hours uploaded per minute

### Architecture
```
Upload Pipeline:
Client → Upload Service → Object Storage (raw) → Transcoding Service → 
CDN-ready formats → Metadata DB + Search Index

Watch Pipeline:
Client → CDN (edge server) → Origin Storage

Components:
- Upload Service: Chunked upload, resumable
- Transcoding: Convert to multiple formats/resolutions (360p, 720p, 1080p, 4K)
- CDN: Serve videos from edge nodes closest to viewer
- Recommendation Engine: Based on watch history, collaborative filtering
- Search: Elasticsearch for title, description, tags
```

### Key Design Points
- **Adaptive Bitrate Streaming (ABR):** Client switches quality based on bandwidth (HLS/DASH)
- **Chunked Upload:** Upload in 5MB chunks, resume on failure
- **Transcoding:** DAG pipeline (split → transcode → merge → thumbnails → metadata)
- **CDN Strategy:** Popular videos cached at edge, long-tail served from origin
- **Storage:** Raw video in blob storage, metadata in SQL, search in Elasticsearch

---

## Case Study 5: Distributed Cache (Redis-like)

### Requirements
- Key-value store with GET/SET/DELETE
- Sub-millisecond latency
- High availability (replicated)
- Eviction when full (LRU)
- Scale: Millions of QPS

### Architecture
```
Client → Consistent Hash Ring → Cache Node (primary) → Replica Nodes

Each Node:
- In-memory hash table
- LRU eviction (doubly linked list + hash map)
- Write-ahead log (optional, for durability)
- Replication: Async to replicas
```

### Key Decisions
| Decision | Choice | Why |
|----------|--------|-----|
| Partitioning | Consistent hashing with vnodes | Minimal data movement on node add/remove |
| Replication | Async, 3 replicas | Availability > consistency for cache |
| Eviction | LRU | Most practical for general-purpose cache |
| Concurrency | Sharded locks or lock-free | Avoid single lock bottleneck |
| Failure detection | Gossip protocol | Decentralized, scalable |

---

## System Design Interview Rubric

### What Gets You a "Strong Hire"
1. ✅ Asked the right clarifying questions
2. ✅ Drove the conversation (didn't wait for prompts)
3. ✅ Made reasonable capacity estimates quickly
4. ✅ Drew clean, logical architecture
5. ✅ Discussed trade-offs without being asked
6. ✅ Went deep on 1-2 areas with confidence
7. ✅ Acknowledged limitations and mentioned monitoring/alerting

### What Gets You a "No Hire"
1. ❌ Jumped into drawing without understanding requirements
2. ❌ Couldn't explain WHY they chose specific components
3. ❌ Gave a textbook answer without adapting to specific constraints
4. ❌ Got stuck on one component and couldn't move on
5. ❌ No mention of failure scenarios
6. ❌ Couldn't do back-of-envelope math

---

## Systems to Practice (Priority Order)

| # | System | Core Concept Tested |
|---|--------|-------------------|
| 1 | URL Shortener | Hashing, caching, database |
| 2 | Rate Limiter | Distributed algorithms, Redis |
| 3 | Twitter Feed | Fan-out, caching, trade-offs |
| 4 | Chat System | WebSockets, queues, delivery |
| 5 | YouTube | CDN, transcoding, storage |
| 6 | Uber/Lyft | Geospatial, real-time, matching |
| 7 | Google Drive | Sync, conflict resolution |
| 8 | Notification System | Priority, multi-channel, templating |
| 9 | Search Autocomplete | Trie, pre-computation, CDN |
| 10 | Payment System | Exactly-once, reconciliation, saga |
| 11 | Distributed Cache | Consistent hashing, replication |
| 12 | Web Crawler | Politeness, dedup, queue |
| 13 | Stock Exchange | Order matching, low-latency |
| 14 | Hotel Booking | Concurrency, double-booking prevention |
| 15 | E-commerce (Amazon) | Inventory, checkout, recommendations |
