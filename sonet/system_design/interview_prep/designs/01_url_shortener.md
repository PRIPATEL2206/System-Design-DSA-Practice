# Design: URL Shortener (TinyURL / Bit.ly)

## Problem Statement
Build a service that takes a long URL and returns a short URL. Clicking the short URL redirects to the original.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Do we need custom aliases (e.g., tinyurl.com/my-brand)?"
- "Do short URLs expire? If so, after how long?"
- "Should we track analytics (click counts, geo, device)?"
- "Is this read-heavy or write-heavy?" → Read-heavy (redirects >> creation)
- "What's the expected scale?" → 100M URLs created/day?

### Functional Requirements
- User submits a long URL → get a short URL (7 chars)
- User clicks short URL → redirect to original long URL
- Optional: Custom aliases, expiry, analytics

### Non-Functional Requirements
- High availability (99.99% — downtime = broken links everywhere)
- Low latency for redirects (< 10ms)
- Short URLs must be unique and non-guessable
- Durable: once created, never lost

### Out of Scope
- User authentication, paid plans, UI

---

## Step 2: Capacity Estimation

```
Scale:
  100M new URLs created/day
  Read/Write ratio = 10:1 (redirects are more common)
  1 billion redirects/day

QPS:
  Write QPS = 100M / 86,400 ≈ 1,160 writes/sec
  Read QPS  = 1B   / 86,400 ≈ 11,574 reads/sec
  Peak Read QPS ≈ 50,000/sec

Storage:
  Avg URL row size = 500 bytes (short URL + long URL + metadata)
  Daily  = 100M × 500 bytes = 50 GB/day
  5-year = 50 GB × 365 × 5 ≈ 90 TB

Cache:
  Cache the top 20% of hot URLs (80% of traffic — Pareto rule)
  20% of daily URLs = 20M URLs × 500 bytes = 10 GB RAM
```

---

## Step 3: High-Level Architecture

```
[User]
  |
  | POST /shorten  { url: "https://long.com/..." }
  v
[API Gateway / LB]
  |
[Short URL Service] ──── writes ────> [Database (MySQL/Postgres)]
       |                                       |
       |──── read short code ──> [Cache (Redis)] ──miss──> [DB]
       |
[Redirect Service]  ──── GET /{code} ────> 301/302 Redirect

[Analytics Service] ──── async ──────> [Kafka] ──> [ClickStream DB]
```

---

## Step 4: Short URL Generation

### Option 1: Hash-Based (MD5/SHA-256)
```
hash = md5(long_url)  → 128-bit hex
take first 7 chars    → "a3f8b2c"

Problem: Collisions! Two different URLs → same 7 chars.
Solution: On collision, append salt and re-hash. Or check DB.
```

### Option 2: Base62 Counter (Recommended)
```
Maintain a global auto-incrementing ID in DB.
Convert ID to Base62 (a-z, A-Z, 0-9 = 62 chars).

ID = 1         → "000001"
ID = 100       → "00001c"
ID = 3.5B      → "aaaaaa"  (6 chars supports 3.5 billion URLs)
ID = 218B      → "aaaaaaa" (7 chars supports 218 billion URLs)

Pro: Guaranteed unique, predictable length
Con: Reveals sequence (guessable); DB counter is a bottleneck
```

### Avoiding Counter Bottleneck
```
Option A: Token Service — dedicated service pre-generates IDs in batches.
  TokenService holds batch of 1000 IDs in memory.
  Each App Server claims a batch from Token Service.
  No DB hit per URL creation.

Option B: Multiple counter ranges
  Server 1: IDs 1–1M
  Server 2: IDs 1M–2M
  Each server increments its own range.
```

### Option 3: Random + Collision Check
```
Generate random 7-char Base62 string.
Check if it exists in DB. If yes, regenerate.
At 100M URLs, collision probability is low but not zero.
```

---

## Step 5: Redirect Flow

```
User clicks tinyurl.com/abc1234

1. DNS resolves tinyurl.com → Load Balancer
2. GET /abc1234 → Redirect Service
3. Check Redis cache for key "abc1234"
   HIT:  return 301/302 to original URL
   MISS: query DB for short_code = "abc1234"
         → update Redis cache
         → return 301/302
```

### 301 vs 302 Redirect
| Type | Meaning | Browser Caches? | Use When |
|------|---------|----------------|---------|
| 301 Permanent | Resource moved forever | Yes | Saves server bandwidth (browser doesn't call again) |
| 302 Temporary | Resource temporarily here | No | Analytics tracking (every redirect hits your server) |

**Interview tip:** For analytics, use 302. For pure performance, use 301.

---

## Step 6: Data Model

### URL Table
```sql
CREATE TABLE urls (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  short_code   VARCHAR(7) UNIQUE NOT NULL,
  original_url TEXT NOT NULL,
  user_id      BIGINT,                    -- NULL if anonymous
  created_at   DATETIME DEFAULT NOW(),
  expires_at   DATETIME,                  -- NULL = no expiry
  click_count  BIGINT DEFAULT 0
);

Indexes:
  INDEX on short_code (primary lookup)
  INDEX on user_id (list user's URLs)
```

### Click Analytics Table
```sql
CREATE TABLE clicks (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  short_code   VARCHAR(7) NOT NULL,
  clicked_at   DATETIME NOT NULL,
  ip_address   VARCHAR(45),
  country      VARCHAR(2),
  user_agent   TEXT
);
-- Write to this via Kafka consumer, not inline
```

---

## Step 7: API Design

```
POST   /api/v1/urls
Body:  { "original_url": "https://...", "alias": "mylink", "expires_in": 3600 }
Response: { "short_url": "https://tinyurl.com/abc1234" }

GET    /{short_code}
Response: HTTP 302 Location: https://original-long-url.com

DELETE /api/v1/urls/{short_code}
Response: HTTP 204 No Content

GET    /api/v1/urls/{short_code}/stats
Response: { "clicks": 1234, "top_countries": [...] }
```

---

## Step 8: Scaling & Edge Cases

### Scaling Reads (Redirect is the bottleneck)
```
1. Redis cache — serves 99% of redirects in < 1ms
2. Read replicas — DB reads spread across replicas
3. CDN — for most popular short URLs, cache redirect at CDN edge
```

### Single Points of Failure
```
1. Token Service — run 2 instances; each gets a range
2. Redis — use Redis Sentinel or Redis Cluster
3. DB — master-replica with auto-failover (RDS Multi-AZ)
```

### URL Cleanup
```
Background job runs daily:
- Delete expired URLs from DB
- Evict from cache
- Free short_code for reuse
```

---

## Interview Tips

- Start with: "This is a read-heavy system (10:1 redirects to creates), so I'll optimize the read path."
- Immediately mention: cache for redirects, Base62 encoding, counter service.
- Discuss: 301 vs 302 trade-off proactively — shows you understand analytics.
- Mention: URL sanitisation (validate the long URL format before storing).
- Avoid: Saying "just use a hash" without addressing collisions.

## Common Follow-up Questions
- "How would you handle 10x the scale?" → CDN for hot URLs, Redis Cluster
- "How do you prevent abuse?" → Rate limiting per user/IP, URL scanning
- "How do custom aliases work?" → Check availability, store as short_code
- "How would you add analytics?" → Kafka + ClickHouse or BigQuery
