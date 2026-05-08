# Caching

## Why Cache?

**Core idea:** Store frequently-read data in fast memory (RAM) to avoid slow operations (DB queries, API calls, computation).

```
Without cache:  Client → App Server → Database → return (10–100ms)
With cache:     Client → App Server → Cache hit → return (< 1ms)
Cache miss:     Client → App Server → Cache miss → DB → update cache → return
```

**Cache hit ratio** = hits / (hits + misses). Target > 80% for production.

---

## Layers of Caching

```
[Browser Cache]           L1: HTML, CSS, JS, images in browser
      ↓ miss
[CDN Cache]               L2: Static assets at edge servers globally
      ↓ miss
[Load Balancer Cache]     L3: Some LBs cache responses (rare)
      ↓ miss
[Application Cache]       L4: Redis/Memcached — most common in design interviews
      ↓ miss
[Database Query Cache]    L5: DB-level cache (deprecated in MySQL 8.0+)
      ↓ miss
[Disk / Storage]          Source of truth
```

---

## Cache Write Strategies

### 1. Cache-Aside (Lazy Loading) — Most Common
```
READ:  App checks cache → hit → return | miss → read DB → write to cache → return
WRITE: App writes to DB directly. Cache entry invalidated or not updated.

Pro: Only cache data that's actually read. Cache failure doesn't break writes.
Con: Cache miss on first read (cold start). Potential stale data.
Use: User profiles, product pages, anything read-heavy.
```

### 2. Write-Through
```
WRITE: App writes to cache AND DB synchronously. Both updated together.
READ:  App reads from cache (always fresh).

Pro: No stale data.
Con: Write latency doubles. Cache fills with data that may never be read.
Use: When read-after-write consistency is critical.
```

### 3. Write-Back (Write-Behind)
```
WRITE: App writes to cache only. Cache asynchronously flushes to DB later.
READ:  App reads from cache.

Pro: Very fast writes.
Con: Data loss risk if cache fails before flush.
Use: Write-heavy workloads where minor data loss is tolerable (gaming scores, analytics).
```

### 4. Read-Through
```
READ: App asks cache. On miss, cache itself queries DB and updates itself.
App never talks to DB directly.

Pro: App logic is simple.
Con: Cache library must support this pattern.
Use: When you want caching transparent to application code.
```

---

## Cache Eviction Policies

When the cache is full, which entry to remove?

| Policy | Algorithm | Use When |
|--------|-----------|---------|
| **LRU** (Least Recently Used) | Remove the item not used for the longest time | General purpose, most common |
| **LFU** (Least Frequently Used) | Remove the item with the fewest accesses | When access frequency matters more than recency |
| **FIFO** (First In First Out) | Remove the oldest item | Simple, but poor cache efficiency |
| **TTL** (Time To Live) | Remove items older than X seconds | Time-sensitive data (session tokens, rate limits) |
| **Random** | Remove a random item | Simple, sometimes surprisingly effective |

**Redis default:** LRU when maxmemory is reached.

---

## Cache Invalidation — The Hard Problem

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Strategies
| Strategy | How | Pro | Con |
|----------|-----|-----|-----|
| **TTL expiry** | Cache entries auto-expire after N seconds | Simple, automatic | Stale data until TTL |
| **Event-driven invalidation** | When DB updates, explicitly delete cache key | Always fresh | Complex, coupling |
| **Write-through** | Update cache on every DB write | Always fresh | Slower writes |
| **Versioned keys** | Change key on every update (`user:123:v4`) | No stale reads | Key proliferation |

---

## Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Strings, Hash, List, Set, Sorted Set, Stream | Strings only |
| Persistence | Yes (RDB snapshots, AOF log) | No |
| Replication | Yes (master-replica) | No |
| Clustering | Yes (Redis Cluster) | Limited (client-side sharding) |
| Atomic operations | Yes (INCR, LPUSH, etc.) | Limited |
| Pub/Sub | Yes | No |
| Memory efficiency | Slightly lower | Higher (simpler) |
| Use when | Rich data types, persistence, pub/sub needed | Pure key-value, simple, max performance |

**Interview tip:** Almost always say Redis. Justify Memcached only if you need multi-threaded simplicity and don't need persistence.

---

## Cache Stampede (Thundering Herd)

**Problem:** Cache key expires. Thousands of requests hit the DB simultaneously to regenerate the same cache entry.

```
Time 0: key "top_posts" expires
Time 1: 10,000 simultaneous cache misses
Time 1: 10,000 simultaneous DB queries for same data
         → DB overwhelmed → slow or crash
```

### Solutions

#### 1. Mutex / Lock
```
On cache miss, acquire a lock. Only one thread queries DB.
Others wait. Lock holder updates cache. Others read from cache.
Con: Requests wait (increased latency during refresh).
```

#### 2. Probabilistic Early Expiry (XFetch)
```
Before TTL expires, probabilistically refresh the cache early.
As expiry approaches, probability of early refresh increases.
Smooths out the refresh, no single expiry event.
```

#### 3. Stale-While-Revalidate
```
Serve stale data immediately (low latency).
Trigger background refresh async.
Next request gets the fresh data.
```

---

## What to Cache (Common Patterns)

| Data | Cache? | TTL | Rationale |
|------|--------|-----|-----------|
| User session tokens | Yes | 30 min | Frequently checked, fast lookup |
| User profile data | Yes | 5 min | Read much more than written |
| Product catalogue | Yes | 10 min | Read-heavy, rarely changes |
| Social feed | Yes | 1 min | High read traffic, eventual consistency OK |
| Rate limit counters | Yes (Redis INCR) | 1 min | Need atomic increment, fast |
| Financial balances | No | — | Must be strongly consistent |
| OTP / verification codes | Yes | 5 min | Short-lived by nature |
| Search results | Yes | 1-5 min | Expensive to compute |

---

## Quick Reference

| Concept | One-line |
|---------|---------|
| Cache-aside | App manages cache manually; most common pattern |
| Write-through | Write to cache and DB together; no stale reads |
| Write-back | Write to cache only; async flush to DB; fastest writes |
| LRU | Evict least recently used; best general-purpose policy |
| TTL | Expiry time for cache entries |
| Cache stampede | Thundering herd on cache expiry; use mutex/early expiry |
| Redis | In-memory, persistent, rich data types |
| Cache hit ratio | Target > 80%; lower means too many DB hits |
