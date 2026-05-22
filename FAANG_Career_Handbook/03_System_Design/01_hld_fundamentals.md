# System Design — High-Level Design (HLD) Fundamentals

## Why System Design Matters

At Senior+ levels, system design is 50% of the interview weight. It tests:
- Can you architect scalable systems?
- Do you understand trade-offs?
- Can you communicate complex ideas clearly?
- Do you have real engineering experience?

---

## The System Design Interview Framework (Use Every Time)

### Phase 1: Requirements (3-5 min)
```
Functional Requirements (What the system does):
- "Users should be able to..."
- List 3-5 core features
- Clarify: Read-heavy or write-heavy? Real-time or eventually consistent?

Non-Functional Requirements (Quality attributes):
- Scale: How many users? QPS?
- Latency: What's acceptable? (< 200ms for API, < 1s for search)
- Availability: 99.9%? 99.99%?
- Consistency: Strong or eventual?
- Durability: Can we lose data?
```

### Phase 2: Capacity Estimation (2-3 min)
```
Back-of-envelope math:
- DAU → QPS → Peak QPS (usually 2-5x average)
- Storage per record × records per day × retention period
- Bandwidth = QPS × average response size

Key numbers to memorize:
- 1 day = 86,400 seconds ≈ 100K seconds
- 1 million requests/day ≈ 12 QPS
- 100 million DAU with 10 actions/day = 1 billion/day ≈ 12K QPS
- 1 KB × 1 billion records = 1 TB
- SSD read: ~100μs | Network within DC: ~1ms | Cross-DC: ~50-100ms
```

### Phase 3: High-Level Design (10-15 min)
Draw the big picture:
- Client → Load Balancer → Application Servers → Cache → Database
- Identify major components and their interactions
- Show data flow for key operations

### Phase 4: Deep Dive (10-15 min)
Interviewer picks areas to explore. Be ready for:
- Database schema design
- API design
- Scaling bottlenecks
- Failure handling
- Data partitioning

### Phase 5: Wrap-up (3-5 min)
- Summarize key decisions and trade-offs
- Mention what you'd add with more time (monitoring, analytics)
- Address failure scenarios

---

## Core Building Blocks

### Load Balancer
```
Purpose: Distribute traffic across servers
Algorithms: Round-robin, Least connections, IP hash, Weighted

Types:
- L4 (Transport): Routes based on IP/port, fast
- L7 (Application): Routes based on URL/headers, smart

Where: Between client↔server, server↔database, between services
```

### Caching
```
Purpose: Speed up reads, reduce database load

Strategies:
┌─────────────────┬────────────────────────────────────┐
│ Cache-Aside     │ App checks cache → miss → read DB → write cache │
│ Write-Through   │ Write to cache AND DB simultaneously              │
│ Write-Behind    │ Write to cache → async write to DB               │
│ Read-Through    │ Cache auto-fetches from DB on miss               │
└─────────────────┴────────────────────────────────────┘

Eviction: LRU (most common), LFU, TTL-based
Tools: Redis, Memcached

Cache invalidation (hardest problem in CS):
- TTL-based: Set expiry, accept staleness
- Event-based: Invalidate on write
- Version-based: Include version in cache key
```

### Databases

| Type | Use When | Examples |
|------|----------|---------|
| Relational (SQL) | Structured data, ACID needed, complex queries | PostgreSQL, MySQL |
| Document (NoSQL) | Flexible schema, rapid iteration | MongoDB, DynamoDB |
| Wide-Column | Huge write throughput, time-series | Cassandra, HBase |
| Key-Value | Simple lookup, caching, sessions | Redis, DynamoDB |
| Graph | Relationships are the data | Neo4j, Neptune |
| Search | Full-text search, autocomplete | Elasticsearch |

**SQL vs NoSQL Decision:**
```
Choose SQL when:
- Data has clear relationships (foreign keys)
- ACID transactions required
- Complex queries/joins needed
- Data structure is stable

Choose NoSQL when:
- Schema changes frequently
- Horizontal scaling is priority
- High write throughput needed
- Denormalized reads are OK
```

### Message Queues
```
Purpose: Decouple services, handle bursts, enable async processing

Patterns:
- Point-to-Point: One producer → one consumer (task queue)
- Pub/Sub: One producer → multiple consumers (notifications)

Tools: Kafka (high throughput, log), RabbitMQ (complex routing), SQS (AWS managed)

When to use:
- Spiky traffic that would overwhelm downstream
- Long-running tasks (video processing, ML inference)
- Cross-service communication
- Event sourcing
```

### CDN (Content Delivery Network)
```
Purpose: Serve static content from edge servers close to users
What: Images, videos, CSS, JS, static HTML

Pull CDN: CDN fetches from origin on first request, caches
Push CDN: You upload to CDN directly (good for predictable content)
```

---

## Scaling Strategies

### Vertical Scaling (Scale Up)
Bigger machine. Simple but has limits. Good for databases initially.

### Horizontal Scaling (Scale Out)
More machines. Requires stateless services. The standard at scale.

### Database Scaling
```
1. Read Replicas: Master handles writes, replicas handle reads
   - Good for read-heavy workloads
   - Replication lag = eventual consistency

2. Sharding (Partitioning): Split data across multiple databases
   - By user_id, by geography, by time range
   - Challenge: Cross-shard queries, rebalancing
   
   Hash-based: hash(key) % num_shards → even distribution
   Range-based: A-M on shard 1, N-Z on shard 2 → range queries work
   
3. Consistent Hashing: Minimize data movement when adding/removing nodes
   - Virtual nodes for even distribution
   - Used by: DynamoDB, Cassandra, load balancers
```

---

## Key Trade-offs (What Interviewers Test)

### CAP Theorem
In a distributed system, you can only guarantee 2 of 3:
- **Consistency:** Every read gets the most recent write
- **Availability:** Every request gets a response
- **Partition tolerance:** System works despite network failures

In practice (since network partitions WILL happen):
- **CP systems:** Sacrifice availability during partitions (banking, inventory)
- **AP systems:** Sacrifice consistency during partitions (social media feeds)

### Consistency Models
```
Strong Consistency: Read always returns latest write
  → Simplest to reason about, hardest to scale
  
Eventual Consistency: Reads may return stale data, but will converge
  → Easy to scale, but app must handle staleness
  
Causal Consistency: Causally related writes seen in order
  → Middle ground

Read-after-write: User always sees their own writes immediately
  → Most practical for user-facing systems
```

### Latency vs Throughput
- Latency: Time for one request
- Throughput: Requests per second
- They're not always inversely correlated — batching increases throughput AND latency

### Availability Numbers
```
99% = 87.6 hours downtime/year (unacceptable for most)
99.9% = 8.76 hours/year (standard)
99.99% = 52.6 minutes/year (high availability)
99.999% = 5.26 minutes/year (critical systems)
```

---

## API Design Principles

```
REST Best Practices:
- Use nouns, not verbs: GET /users/123, not GET /getUser?id=123
- Use HTTP methods correctly: GET (read), POST (create), PUT (update), DELETE
- Version your API: /api/v1/users
- Pagination: ?limit=20&offset=0 or cursor-based
- Rate limiting: Return 429 with Retry-After header

GraphQL (when to consider):
- Multiple clients need different data shapes
- Mobile needs less data than web
- Complex nested resources
```

---

## Common Design Problems and Their Core Challenges

| System | Core Challenge | Key Components |
|--------|---------------|----------------|
| URL Shortener | Hash function, read-heavy | Counter/hash → DB → Cache → Redirect |
| Twitter Feed | Fan-out (push vs pull) | Write: fan-out to followers' feeds. Read: merge + rank |
| Chat (WhatsApp) | Real-time delivery, offline | WebSockets, message queue, acknowledgments |
| YouTube | Video storage + streaming | CDN, transcoding pipeline, metadata DB |
| Uber | Real-time location, matching | Geospatial index, matching algorithm, WebSockets |
| Google Drive | File sync, conflict resolution | Chunked upload, versioning, notification service |
| Rate Limiter | Distributed counting | Token bucket / sliding window, Redis |
| Notification | Multi-channel delivery | Priority queue, user preferences, retry logic |
| Search Autocomplete | Low latency suggestions | Trie, precomputed results, CDN |
| Payment System | Exactly-once processing | Idempotency keys, saga pattern, reconciliation |

---

## What Interviewers Actually Evaluate

1. **Requirements gathering** — Do you ask the right questions?
2. **Structured approach** — Can you organize your thoughts?
3. **Trade-off discussions** — Do you understand the "why" behind decisions?
4. **Depth of knowledge** — Can you dive deep when asked?
5. **Communication** — Can you explain clearly?
6. **Experience signals** — Have you built real systems?

**They do NOT evaluate:**
- Memorized architectures
- Exact numbers (estimates are fine)
- Knowing every tool (knowing categories and trade-offs matters more)
