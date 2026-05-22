# Chapter 1: How to Answer System Design Questions

## The 45-Minute Framework

System design interviews are **structured conversations**, not coding tests. The interviewer wants to see how you **think**, communicate, and make **trade-offs**.

---

## Step 1: Requirements Clarification (5 minutes)

**Never start designing immediately.** Ask questions to narrow scope.

### Functional Requirements (What does the system DO?)
- What are the core features?
- Who are the users?
- What inputs/outputs?
- Any specific use cases to prioritize?

### Non-Functional Requirements (How WELL does it do it?)
- **Scale:** How many users? DAU/MAU?
- **Performance:** Latency requirements? (< 200ms?)
- **Availability:** 99.9%? 99.99%?
- **Consistency:** Strong or eventual?
- **Storage:** How much data? Growth rate?

### Example (URL Shortener):
```
You: "Should we support custom short URLs?"
You: "What's the expected QPS? Reads vs writes?"
You: "How long should URLs live? Do they expire?"
You: "Any analytics requirements?"
```

---

## Step 2: Back-of-the-Envelope Estimation (3 minutes)

Quick math to understand the scale:

```
"If we have 100M DAU, and each user creates 1 URL/day:
- Write QPS: 100M / 86400 ≈ 1200 writes/sec
- Read:Write ratio 100:1 → 120K reads/sec
- Storage: 100M × 500 bytes × 365 days × 5 years ≈ 90 TB"
```

This tells you: need caching, sharding, and distributed storage.

---

## Step 3: High-Level Design (5 minutes)

Draw the **big picture** with boxes and arrows:

```
┌─────────┐     ┌──────────────┐     ┌──────────┐
│  Client │────▶│ Load Balancer │────▶│ App Server│
└─────────┘     └──────────────┘     └──────────┘
                                           │
                    ┌──────────────────────┼──────────────┐
                    │                      │              │
               ┌────▼───┐          ┌──────▼──┐    ┌─────▼────┐
               │  Cache  │          │ Database │    │  Storage  │
               └─────────┘          └──────────┘    └──────────┘
```

**Key components to consider:**
- Clients (web, mobile, API)
- Load Balancer
- Application servers
- Cache layer
- Database(s)
- Message queue (if async needed)
- CDN (if serving static content)
- Search service (if search required)

---

## Step 4: Detailed Design (20 minutes) ← The Meat

This is where you go deep. The interviewer will pick 1-2 areas to dive into.

### For each component, discuss:
1. **Data Model** — What tables/collections? Schema? Relationships?
2. **API Design** — RESTful endpoints or GraphQL queries
3. **Algorithm** — How does the core logic work?
4. **Scaling** — How does this component handle growth?

### What interviewers look for:
- Can you identify the **right database** for the use case?
- Do you understand **trade-offs** (SQL vs NoSQL, push vs pull)?
- Can you handle **concurrent access** (race conditions, locks)?
- Do you know when to use **caching** and what strategy?

---

## Step 5: Bottlenecks & Trade-offs (5 minutes)

Proactively identify weaknesses:

```
"The database is our bottleneck at 120K QPS. I'd address this with:
1. Read replicas for read-heavy workload
2. Redis cache with 80% hit ratio → DB sees only 24K QPS
3. Database sharding by user_id if we outgrow single master"
```

Discuss trade-offs you made:
- "I chose eventual consistency because users can tolerate 2-3 second delay"
- "I picked NoSQL for the feed because the access pattern is key-value"

---

## Step 6: Wrap-up (2 minutes)

- Summarize your design
- Mention future improvements (monitoring, analytics, ML ranking)
- Ask if the interviewer wants you to dive deeper into anything

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Jumping into design without requirements | Always ask 3-5 clarifying questions first |
| Over-engineering | Start simple, scale only what needs scaling |
| Ignoring trade-offs | Every decision has pros and cons — state them |
| Not doing estimation | Numbers guide your architecture decisions |
| Monologue | Make it a conversation, check in with interviewer |
| Being vague | "We'll use a cache" → "We'll use Redis with LRU, TTL of 1 hour, write-through" |
| Skipping failure handling | Always mention: what happens when X goes down? |

---

## Magic Phrases That Score Points

- "The **bottleneck** here is..."
- "The **trade-off** between X and Y is..."
- "At this scale, we'd need to..."
- "For **consistency**, I'd choose... because..."
- "To handle **failures**, we could..."
- "This gives us **horizontal scalability** because..."

---

## Quick Reference: Picking the Right Tool

```
Need                         → Technology
───────────────────────────────────────────────────
Fast key-value lookups       → Redis, Memcached
Complex queries + ACID       → PostgreSQL, MySQL
Flexible schema + scale      → MongoDB, DynamoDB
Search + text matching       → Elasticsearch
Time-series data             → InfluxDB, TimescaleDB
Graph relationships          → Neo4j
Message queue                → Kafka, RabbitMQ, SQS
Object storage               → S3, GCS
CDN                          → CloudFront, Cloudflare
Service discovery            → Consul, ZooKeeper
```
