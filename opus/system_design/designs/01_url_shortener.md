# Design: URL Shortener (TinyURL)

**Frequency:** Very High | **Difficulty:** ⭐⭐ | **Companies:** All FAANG

---

## 1. Requirements

### Functional
- Given a long URL, generate a short URL
- Redirect short URL → original long URL
- Optional: custom short URLs, expiration, analytics

### Non-Functional
- **Scale:** 100M new URLs/day, read-heavy (100:1 read:write)
- **Latency:** < 100ms for redirect
- **Availability:** 99.99% (redirect must always work)
- **Durability:** URLs should not be lost

### Estimation
```
Write: 100M/day ÷ 86400 ≈ 1200 QPS
Read: 1200 × 100 = 120K QPS
Storage: 100M × 500 bytes × 365 × 5 years ≈ 90 TB
Short URL length: 62^7 = 3.5 trillion combinations (enough for decades)
```

---

## 2. High-Level Design

```
┌────────┐         ┌─────────────┐         ┌──────────────┐
│ Client │────────▶│ Load Balancer│────────▶│ App Servers  │
└────────┘         └─────────────┘         └──────┬───────┘
                                                   │
                                      ┌────────────┼────────────┐
                                      │            │            │
                                 ┌────▼───┐  ┌────▼────┐  ┌───▼────┐
                                 │ Cache  │  │   DB    │  │ Counter│
                                 │(Redis) │  │(NoSQL)  │  │Service │
                                 └────────┘  └─────────┘  └────────┘
```

---

## 3. Core Algorithm: Short URL Generation

### Option A: Base62 Encoding of Counter (Recommended)
```
1. Distributed counter generates unique ID (e.g., 123456789)
2. Convert to base62: 123456789 → "8M0kX"
3. Store mapping: "8M0kX" → "https://original-url.com/very/long/path"

Base62 = [a-z, A-Z, 0-9] = 62 characters
7 chars → 62^7 = 3.5 trillion unique URLs

Counter options:
- Auto-increment DB with range allocation
- Snowflake ID generator (Twitter's approach)
- Zookeeper for distributed coordination
```

### Option B: Hash + Collision Resolution
```
1. MD5/SHA256(long_url) → take first 7 chars of base62
2. Check DB for collision
3. If collision → append random char or re-hash
⚠️ Slower due to collision checks
```

### Option C: Pre-generated Keys
```
1. Pre-generate all 7-char keys, store in "key pool" DB
2. On request: take one key from pool, mark as used
✅ Very fast (no computation)
✅ No collisions
⚠️ Need to manage key pool
```

---

## 4. Detailed Design

### Data Model
```
Table: url_mappings
┌───────────────┬──────────────────────────────────────────────────┐
│ short_url (PK)│ VARCHAR(7) — the short code                     │
│ long_url      │ VARCHAR(2048) — original URL                     │
│ created_at    │ TIMESTAMP                                        │
│ expires_at    │ TIMESTAMP (nullable)                             │
│ user_id       │ VARCHAR (nullable, for analytics)                │
└───────────────┴──────────────────────────────────────────────────┘

Database choice: DynamoDB or Cassandra
- Access pattern is pure key-value (short_url → long_url)
- Need high write throughput
- No complex queries needed
```

### API Design
```
POST /api/v1/shorten
  Body: { "long_url": "https://...", "custom_alias": "my-link", "ttl": 3600 }
  Response: { "short_url": "https://tny.url/8M0kX" }

GET /{short_url}
  Response: 301 Redirect to long_url
  (301 = permanent redirect, browser caches. 302 = temporary, no cache)
```

### Read Path (Redirect) — Optimized
```
1. User hits GET /8M0kX
2. Check Redis cache → HIT? → 301 redirect
3. MISS → Query DynamoDB → Store in Redis (TTL: 1 hour) → 301 redirect

Cache hit ratio: ~80% (popular URLs accessed repeatedly)
Effective DB QPS: 120K × 0.2 = 24K (manageable)
```

### Write Path (Shorten)
```
1. Validate URL format
2. Check if URL already shortened (optional dedup)
3. Generate short code via counter service
4. Write to DynamoDB
5. Optionally warm cache
6. Return short URL
```

---

## 5. Scaling & Bottlenecks

```
Bottleneck 1: Counter service (single point of failure)
Solution: Range-based allocation
  - App Server 1 gets range [1, 1000000]
  - App Server 2 gets range [1000001, 2000000]
  - Each generates locally until range exhausted
  - No coordination needed for individual writes!

Bottleneck 2: Database at 120K read QPS
Solution: Redis cache (80% hit rate → 24K DB QPS) + read replicas

Bottleneck 3: Single region latency for global users
Solution: Multi-region deployment with geo-routing
  - Each region has its own counter range
  - Local DynamoDB table with global tables (replication)
```

---

## 6. Additional Features

### Analytics
```
Async pipeline (don't slow down redirects):
Redirect → Kafka event → Analytics Worker → ClickHouse/BigQuery

Track: timestamp, user agent, IP/geo, referrer
Aggregate: clicks per day, top URLs, geographic distribution
```

### URL Expiration
```
- TTL field in DB
- Background cleanup job (scan expired, delete)
- Or: DynamoDB TTL feature (auto-deletes expired items)
- Cache: set Redis TTL = min(1 hour, time_until_expiry)
```

---

## 7. Final Architecture

```
                         ┌──── CDN (cache 301 redirects) ────┐
                         │                                     │
┌────────┐        ┌─────▼──────┐        ┌─────────────────┐  │
│ Client │───────▶│    Nginx   │───────▶│   App Servers   │  │
└────────┘        │  (L7 LB)  │        │   (stateless)   │  │
                  └────────────┘        └────────┬────────┘  │
                                                  │           │
                                    ┌─────────────┼───────────┤
                                    │             │           │
                              ┌─────▼────┐  ┌────▼─────┐  ┌─▼──────────┐
                              │  Redis   │  │ DynamoDB │  │  Counter   │
                              │  Cache   │  │ (sharded)│  │  Service   │
                              └──────────┘  └──────────┘  │(Zookeeper)│
                                                          └────────────┘
                                    │
                              ┌─────▼─────┐
                              │   Kafka   │──▶ Analytics Pipeline
                              └───────────┘
```
