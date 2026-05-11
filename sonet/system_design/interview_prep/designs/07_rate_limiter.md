# Design: Rate Limiter

## Problem Statement
Design a rate limiter that restricts the number of requests a client can make to an API within a time window, to protect services from overload and abuse.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Client-side, server-side, or middleware rate limiter?" → Server-side / API Gateway
- "Per user, per IP, per API key, or all?" → Per user and per IP
- "Different limits for different endpoints?" → Yes (e.g., /login stricter than /feed)
- "What happens when rate limit is exceeded?" → HTTP 429 Too Many Requests
- "Distributed system (multiple servers)?" → Yes, must share state across servers
- "Hard vs soft limits?" → Hard (strictly enforce, no overflow)

### Functional Requirements
- Allow N requests per time window per client identifier (user ID, IP, API key)
- Return 429 with Retry-After header when limit exceeded
- Different rules per endpoint
- Rate limit headers in every response (X-RateLimit-Remaining, X-RateLimit-Reset)

### Non-Functional Requirements
- Low latency: rate limit check must add < 1ms to request
- High availability: if rate limiter is down, requests should pass (fail open) or block (fail closed) — configurable
- Accurate: in distributed environment, counts must be consistent
- Scale: handle millions of requests/sec

---

## Step 2: High-Level Architecture

```
[Client]
   |
   | HTTP Request
   ↓
[API Gateway / Rate Limiter Middleware]
   |
   | Check: has client exceeded limit?
   ↓
[Redis] ──hit limit──> 429 Too Many Requests
   |
   | within limit
   ↓
[Origin Service]
   |
   ↓
[Response + Rate Limit Headers]
```

---

## Step 3: Algorithms

### Algorithm 1: Token Bucket (Most Common in Practice)
```
Each client has a bucket with capacity N tokens.
Tokens refill at rate R per second.
Each request consumes 1 token.
No tokens left → reject request.

Example: 100 requests/min = 100 tokens capacity, refill 100/60 ≈ 1.67 tokens/sec

Properties:
  + Allows burst traffic (consume accumulated tokens)
  + Smooth average rate
  - Two parameters to tune (capacity, refill rate)

Redis implementation:
  Lua script (atomic):
    tokens = GET bucket:{user_id}
    if tokens == nil: tokens = max_tokens
    now = current_time
    elapsed = now - GETSET last_refill:{user_id} now
    tokens = min(max_tokens, tokens + elapsed × refill_rate)
    if tokens >= 1: tokens -= 1; allow; SET bucket:{user_id} tokens
    else: deny
```

### Algorithm 2: Fixed Window Counter (Simplest)
```
Count requests per client per time window (e.g., 1 minute).
Window resets at :00 of each minute.

Redis: 
  INCR rate_limit:{user_id}:{current_minute}
  EXPIRE rate_limit:{user_id}:{current_minute} 60
  if count > limit: deny

Problem — Boundary burst attack:
  Limit = 100/min
  Client sends 100 at 11:59:59 (valid)
  Client sends 100 at 12:00:00 (valid — new window)
  = 200 requests in 2 seconds!
```

### Algorithm 3: Sliding Window Log
```
Store timestamp of every request in a sorted set.
Count entries in [now - 60sec, now].
If count > limit → deny.

Redis (sorted set by timestamp):
  ZADD rate_log:{user_id} now now     ← add current request
  ZREMRANGEBYSCORE rate_log:{user_id} 0 (now - 60sec)  ← remove old entries
  count = ZCARD rate_log:{user_id}
  if count > limit: deny

Properties:
  + Very accurate
  - Memory intensive (store every timestamp)
  - For 1M users × 100 entries = 100M entries in Redis
```

### Algorithm 4: Sliding Window Counter (Best Balance)
```
Blend of fixed window + sliding calculation:

count = current_window_count + previous_window_count × (overlap_ratio)

Example:
  Window = 1 min, limit = 100
  At 12:00:45 (45 sec into current window):
  Previous window had 80 requests.
  Current window has 40 requests.
  Overlap = (60 - 45) / 60 = 25% of previous window in range.
  Estimated count = 40 + 80 × 0.25 = 60 ← allow

  + Accurate, memory-efficient
  + Used by Cloudflare
```

---

## Step 4: Distributed Rate Limiting

The hard part: multiple app servers must share rate limit counts.

### Centralised Counter (Redis)
```
All servers talk to same Redis instance for counts.
  Pros: Accurate, single source of truth
  Cons: Redis is a bottleneck; network latency per request

Optimisation: Lua script for atomicity
  All check-and-increment done as single atomic operation
  Prevents race conditions

Redis Cluster: Shard by user_id for horizontal scale
```

### Local Counter + Sync (Facebook/Twitter approach)
```
Each server maintains local counters in memory.
Every N seconds, sync to central Redis:
  - Read current count from Redis
  - Add local increment
  - Write back
  
Pros: Very fast (no Redis per request)
Cons: Slight inaccuracy (can overshoot limit by local_count amount between syncs)
Use when: Small inaccuracy is acceptable (most commercial APIs)
```

### Race Condition Handling
```
Without atomic operation:
  Server 1 reads count = 99 → allows → writes 100
  Server 2 reads count = 99 (before Server 1 writes!) → allows → writes 100
  = 101 requests allowed

Fix: Redis Lua script (single atomic operation)
  EVAL """
    local count = redis.call('INCR', KEYS[1])
    if count == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end
    if count > tonumber(ARGV[2]) then return 0 end
    return 1
  """ 1 rate:{user_id} 60 100
```

---

## Step 5: Rate Limit Headers

```
Every API response should include:
  X-RateLimit-Limit:     100        ← max requests allowed
  X-RateLimit-Remaining: 47         ← remaining in current window
  X-RateLimit-Reset:     1697000060 ← Unix timestamp when window resets
  Retry-After:           13         ← seconds until retry (only on 429)
```

---

## Step 6: Configuration System

```
Rules stored in DB / config service:
  endpoint       | limit | window | scope
  POST /login    |   5   | 1 min  | per_ip
  GET  /feed     | 100   | 1 min  | per_user
  POST /tweet    |  30   | 1 min  | per_user
  GET  /search   | 200   | 1 min  | per_user
  *              | 500   | 1 min  | per_api_key

Rate limiter reads rules from Redis (cached from DB).
Config changes propagated via pub/sub.
```

---

## Step 7: Where to Deploy

| Location | Pros | Cons |
|----------|------|------|
| **API Gateway** (recommended) | Centralised, before any business logic | Gateway is SPOF if misconfigured |
| **Middleware in each service** | Decentralised, no gateway dependency | Duplicate logic, harder to manage |
| **Client-side** | Reduces requests before they hit server | Can be bypassed by malicious clients |

---

## Step 8: Edge Cases

```
Fail Open vs Fail Closed:
  Redis down → should requests be allowed or blocked?
  
  Fail Open: Allow requests (high availability, but unprotected during outage)
  Fail Closed: Block requests (safe, but degrades service)
  
  Recommendation: Fail open with alerting — availability > protection for most APIs.
  For security-critical (login, payments): Fail closed.

Multi-datacenter:
  Two regions both have Redis → counts drift → can exceed limit globally.
  Solution: 
    A. Accept slight overshoot (practical — 2x limit for brief period)
    B. Single global Redis with active-active replication (consistent, higher latency)
    C. Use region-specific limits (each region gets N/2 of total limit)
```

---

## Interview Tips

- Start with algorithm comparison: "Token Bucket for burst-friendly APIs; Sliding Window Counter for accuracy."
- Key distributed challenge: "Multiple servers need shared state → Redis as centralised counter."
- Atomic operations: "Use Redis Lua script to prevent race conditions in INCR + check."
- "Always include rate limit headers — good API practice."

## Common Follow-up Questions
- "How to rate limit at the network layer?" → Use iptables / cloud WAF (not just app layer)
- "How to handle API keys vs user IDs?" → Multiple dimensions; check all that apply, use strictest
- "How to implement for GraphQL?" → Rate by query complexity score, not just request count
- "What about DDoS?" → Rate limiting alone is insufficient; need WAF + CDN-layer blocking (Cloudflare)
