# System Design One-Pager — Read 15 Min Before Interview

## Framework (Memorize This Flow)

```
[0-5]  REQUIREMENTS: Ask "users, scale, latency, consistency?"
[5-8]  ESTIMATION: QPS, storage, bandwidth (quick math)
[8-15] HIGH-LEVEL: Draw boxes and arrows (LB → App → Cache → DB)
[15-35] DEEP DIVE: Data model, API, core algorithm, scaling
[35-42] TRADE-OFFS: "I chose X over Y because..."
[42-45] WRAP-UP: Monitoring, future improvements
```

---

## Numbers to Know

```
1 day = 86,400 sec ≈ 100K sec
1M requests/day ≈ 12 QPS
1B requests/day ≈ 12K QPS

Server capacity:
  App server: 1K-10K QPS
  Redis: 100K-500K QPS  
  PostgreSQL: 10K-30K QPS (with indexes)
  Kafka partition: 100K msg/sec

Storage:
  1 tweet: ~300 bytes
  1 image: ~200KB-2MB
  1 video (1 min): ~50MB
  1M users × 1KB = 1GB
  1B users × 1KB = 1TB

Latency:
  Redis: < 1ms
  Same DC network: 0.5ms
  Database query: 5-50ms
  Cross-region: 100-300ms
```

---

## Building Blocks (Quick Reference)

```
Component               When to Use
─────────────────────────────────────────────────
Load Balancer           Always (distribute traffic)
CDN                     Static content, global users
Cache (Redis)           Read-heavy, reduce DB load
Message Queue (Kafka)   Async processing, decouple services
Search (Elasticsearch)  Full-text search, autocomplete
Object Storage (S3)     Files, images, videos (NEVER in DB)
Rate Limiter            API protection, prevent abuse
API Gateway             Auth, routing, rate limiting
```

---

## Database Selection

```
Need ACID + complex queries?     → PostgreSQL
Simple key-value at scale?       → DynamoDB / Redis
Write-heavy + time-series?       → Cassandra
Search + text matching?          → Elasticsearch
Relationships/graph?             → Neo4j
Caching + sub-ms latency?       → Redis
```

---

## Scaling Checklist

```
□ Horizontal scaling (stateless app servers)
□ Database read replicas (for read-heavy)
□ Caching layer (Redis: 80% cache hit → 5x less DB load)
□ Database sharding (when single DB can't handle)
□ CDN (for static content + reduce origin load)
□ Async processing (queue non-critical work)
□ Rate limiting (protect from abuse)
□ Multi-region (for global latency)
```

---

## Common Trade-offs

```
Consistency vs Availability:  "For [banking], I choose consistency.
                               For [social feed], eventual consistency is fine."

Push vs Pull:                "Push for real-time (WebSocket).
                              Pull for feed (fan-out-on-write for non-celebrities)."

SQL vs NoSQL:                "SQL for ACID and complex queries.
                              NoSQL for horizontal scale and flexible schema."

Cache vs Fresh:              "Cache-aside with 5 min TTL. Acceptable staleness
                              for [use case]. Invalidate on write for critical data."

Sync vs Async:               "User-facing → sync. Background work → async queue."
```

---

## Magic Sentences

```
"The bottleneck here is [X]. I'd solve it with [Y]."
"At this scale of [N QPS], we need [sharding/caching/queue]."
"The trade-off is [consistency vs latency], and for this use case I'd choose..."
"If this service goes down, the fallback is [X]."
"I'd monitor [metric] and alert when [threshold]."
"For data integrity, I'd use [exactly-once/idempotency keys/WAL]."
```

---

## Top 5 Designs — Quick Recall

```
URL Shortener:     Counter → Base62 → DynamoDB → Redis cache → redirect
Rate Limiter:      Token bucket in Redis (Lua for atomicity) → 429
Chat (WhatsApp):   WebSocket → Kafka → Cassandra → push notification if offline
Twitter Feed:      Hybrid fan-out (push for normal, pull for celebrities)
Uber:              Redis GEO for drivers → match nearest → trip state machine
```
