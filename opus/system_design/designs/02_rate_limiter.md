# Design: Rate Limiter

**Frequency:** Very High | **Difficulty:** ⭐⭐ | **Companies:** All FAANG, Stripe, Cloudflare

---

## 1. Requirements

### Functional
- Limit requests per user/IP/API key within a time window
- Return 429 Too Many Requests when exceeded
- Support different limits per API endpoint
- Provide rate limit headers in response

### Non-Functional
- **Latency:** < 1ms added to request path
- **Accuracy:** No false positives (don't reject valid requests)
- **Scale:** 10M+ RPM across distributed servers
- **Fault tolerance:** If rate limiter is down, allow traffic (fail open)

---

## 2. Algorithms

### Token Bucket (Most Popular)
```
┌─────────────────────────────────────┐
│  Bucket capacity: 10 tokens         │
│  Refill rate: 2 tokens/second       │
│                                      │
│  Request arrives:                    │
│    tokens > 0? → Allow, tokens -= 1 │
│    tokens == 0? → Reject (429)      │
│                                      │
│  ✅ Allows bursts (up to capacity)  │
│  ✅ Smooth long-term rate           │
└─────────────────────────────────────┘

Implementation (Redis):
  Key: rate_limit:{user_id}
  Value: {tokens: 8, last_refill: 1640000000}
```

### Sliding Window Counter (Best Balance)
```
Current window count × weight + Previous window count × (1 - weight)

Example: 100 requests/minute limit
  Previous minute (00:00-01:00): 80 requests
  Current minute (01:00-02:00): 30 requests so far
  Current time: 01:15 (25% into current window)

  Weighted count = 30 + 80 × (1 - 0.25) = 30 + 60 = 90
  90 < 100 → ALLOW

  ✅ Smooth (no boundary burst)
  ✅ Low memory (2 counters per window)
```

### Fixed Window Counter (Simplest)
```
Count requests per fixed time window.
Reset counter at window boundary.

⚠️ Problem: Burst at boundaries
  Window 1 (00:00-01:00): 0 requests... then 100 at 00:59
  Window 2 (01:00-02:00): 100 at 01:01... then blocked
  Result: 200 requests in 2 seconds (burst!)
```

---

## 3. Distributed Rate Limiting Architecture

```
┌─────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────┐
│ Client  │───▶│ API Gateway  │───▶│  Rate Limiter   │───▶│ Backend  │
└─────────┘    │  (Nginx/Kong)│    │  (Redis-based)  │    │ Service  │
               └──────────────┘    └────────┬────────┘    └──────────┘
                                            │
                                     ┌──────▼──────┐
                                     │   Redis     │
                                     │  Cluster    │
                                     └─────────────┘
```

### Redis Implementation (Token Bucket via Lua)
```lua
-- Atomic token bucket in Redis (Lua script)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])    -- max tokens
local refill_rate = tonumber(ARGV[2]) -- tokens per second
local now = tonumber(ARGV[3])         -- current timestamp

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens based on elapsed time
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

if new_tokens >= 1 then
    redis.call('HMSET', key, 'tokens', new_tokens - 1, 'last_refill', now)
    redis.call('EXPIRE', key, capacity / refill_rate * 2)
    return 1  -- ALLOWED
else
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    return 0  -- REJECTED
end
```

---

## 4. Rate Limit Rules

```yaml
rules:
  - endpoint: "/api/v1/messages"
    limits:
      - key: "user_id"
        requests: 100
        window: "1m"
      - key: "ip"
        requests: 1000
        window: "1h"
  
  - endpoint: "/api/v1/login"
    limits:
      - key: "ip"
        requests: 5
        window: "5m"  # Brute force protection
  
  - endpoint: "/api/v1/search"
    limits:
      - key: "user_id"
        requests: 30
        window: "1m"
      - key: "org_id"
        requests: 1000
        window: "1m"
```

---

## 5. Response Headers
```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 67
X-RateLimit-Reset: 1640000060

HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000060
```

---

## 6. Race Conditions & Solutions

```
Problem: Two requests arrive simultaneously, both read tokens=1

Request A: read tokens=1, tokens >= 1? YES → allow, write tokens=0
Request B: read tokens=1, tokens >= 1? YES → allow, write tokens=0
Result: 2 requests allowed when only 1 token available!

Solutions:
1. Redis Lua script (atomic) ← BEST
2. Redis MULTI/EXEC (optimistic locking)
3. INCR + check (may over-count slightly)
```

---

## 7. Scaling Considerations

```
Single-node Redis: ~500K operations/sec (enough for most)
Redis Cluster: Shard by rate limit key → linear scale

Multi-region:
  Option A: Local rate limiting (each region independent)
    ✅ Fast (no cross-region calls)
    ⚠️ User can get 100 req/min per region

  Option B: Global rate limiting (sync across regions)
    ✅ Accurate global limit
    ⚠️ Cross-region latency (50-200ms)
    
  Option C: Local with async sync (hybrid)
    ✅ Fast + eventually accurate
    Slight over-allowance during sync window
```
