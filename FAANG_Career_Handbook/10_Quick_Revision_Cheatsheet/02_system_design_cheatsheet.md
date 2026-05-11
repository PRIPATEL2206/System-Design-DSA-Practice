# System Design Quick Revision — One-Page Reference

## The Framework (Burn This Into Memory)
```
1. REQUIREMENTS (3 min)    → Functional + Non-functional + Scale
2. ESTIMATION (2 min)      → QPS, Storage, Bandwidth
3. HIGH-LEVEL DESIGN (12 min) → Components + Data Flow
4. DEEP DIVE (15 min)      → Database, Caching, Scaling decisions
5. WRAP-UP (3 min)         → Trade-offs, monitoring, what's missing
```

---

## Capacity Estimation Cheat

```
1 day ≈ 100K seconds
1M requests/day ≈ 12 QPS
100M DAU × 10 actions = 1B/day ≈ 12K QPS
Peak = 2-5× average

Storage:
1 tweet (280 chars + metadata) ≈ 500 bytes
1 image ≈ 300 KB
1 minute video ≈ 50 MB
1B records × 1KB = 1 TB
```

---

## Building Block Quick Reference

| Component | When to Use | Tool |
|-----------|-------------|------|
| Load Balancer | Multiple app servers | ALB/NLB, Nginx |
| Cache | Read-heavy, reduce DB load | Redis, Memcached |
| Message Queue | Async processing, decouple | Kafka, SQS, RabbitMQ |
| CDN | Static content, global users | CloudFront, Cloudflare |
| Object Storage | Files, images, videos | S3 |
| Search | Full-text, autocomplete | Elasticsearch |
| NoSQL | Flexible schema, massive scale | DynamoDB, Cassandra |
| SQL | ACID, complex queries, relations | PostgreSQL, MySQL |
| WebSocket | Real-time bidirectional | For chat, gaming, live |

---

## Database Selection

```
Need ACID + complex queries?        → PostgreSQL
Need massive write throughput?      → Cassandra / DynamoDB
Need flexible documents?            → MongoDB
Need fast key-value?                → Redis
Need full-text search?              → Elasticsearch
Need graph relationships?           → Neo4j
Need time-series?                   → TimescaleDB / InfluxDB
Need analytics/warehouse?           → Redshift / BigQuery
```

---

## Scaling Patterns

```
Read-heavy? → Cache + Read Replicas + CDN
Write-heavy? → Message Queue + Sharding + Async processing
Both? → CQRS (separate read/write models)

Sharding strategies:
- Hash-based: hash(key) % N → even distribution
- Range-based: A-M, N-Z → good for range queries
- Consistent hashing: minimal data movement on resize

Caching strategies:
- Cache-aside: App manages cache (most common)
- Write-through: Cache + DB together (consistency)
- Write-behind: Cache first, async DB (performance)
- Eviction: LRU (default choice)
```

---

## Key Trade-offs (One-liners)

| Trade-off | Option A | Option B |
|-----------|----------|----------|
| Consistency vs Availability | CP: Banking, inventory | AP: Social feed, cache |
| Latency vs Throughput | Low latency: Real-time chat | High throughput: Batch analytics |
| SQL vs NoSQL | SQL: Complex relations, ACID | NoSQL: Scale, flexible schema |
| Push vs Pull | Push: Chat (fan-out write) | Pull: Search (on-demand) |
| Monolith vs Microservices | Monolith: Small team, speed | Micro: Scale team, independent deploy |
| Strong vs Eventual | Strong: Payment, counter | Eventual: Like count, feed |

---

## Top 10 Systems — Core Insight Each

| System | One Key Insight |
|--------|----------------|
| URL Shortener | Base62 counter + cache hot URLs |
| Rate Limiter | Token bucket in Redis, distributed counter |
| News Feed | Hybrid fan-out (push for normal, pull for celebrities) |
| Chat | WebSocket + message queue + delivery acknowledgment |
| YouTube | CDN for video + transcoding pipeline + ABR streaming |
| Uber | Geospatial index (QuadTree/H3) + real-time matching |
| Google Drive | Chunked sync + conflict resolution + notifications |
| Notifications | Priority queue + multi-channel + user preferences |
| Autocomplete | Trie/prefix tree + pre-computed suggestions + CDN |
| Payment | Idempotency key + saga pattern + reconciliation |

---

## API Design Template
```
POST   /api/v1/resource       → Create
GET    /api/v1/resource/:id   → Read
PUT    /api/v1/resource/:id   → Update
DELETE /api/v1/resource/:id   → Delete
GET    /api/v1/resource?filter=X&sort=Y&page=Z  → List

Pagination: cursor-based (for infinite scroll) or offset (for pages)
Rate limit: Return 429 + Retry-After header
Versioning: /v1/ in URL path
Auth: JWT in Authorization: Bearer header
```

---

## Failure Handling
```
Single point of failure → Redundancy (replicas, multi-AZ)
Cascading failure → Circuit breaker (Hystrix pattern)
Data loss → Replication + Backups + WAL
Network partition → Retry with exponential backoff + idempotency
Overload → Rate limiting + load shedding + auto-scaling
Split brain → Consensus (Raft/Paxos) + fencing tokens
```

---

## Numbers You Must Know

```
QPS a single server handles: ~10K-100K (depends on complexity)
Redis QPS: 100K+
Network roundtrip (same DC): ~0.5ms
Network roundtrip (cross-region): ~100-150ms
SSD random read: ~100μs
RAM access: ~100ns
99.9% availability: 8.7 hours downtime/year
99.99% availability: 52 minutes downtime/year
```

---

## The "What Would You Add?" List (Wrap-up)
When interviewer asks "What else would you consider?"
- Monitoring & alerting (Prometheus, Grafana, PagerDuty)
- Logging & tracing (distributed tracing with request IDs)
- A/B testing framework
- Rate limiting & DDoS protection
- Data backup & disaster recovery
- CI/CD pipeline
- Security (encryption at rest & in transit, auth, audit logs)
- Internationalization (i18n, multi-region)
- Cost optimization (reserved instances, spot instances, tiered storage)
