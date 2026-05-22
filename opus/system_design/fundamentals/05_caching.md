# Chapter 5: Caching Strategies

## Why Cache?

```
Without cache: Every request hits database → 10ms response, DB overloaded at 10K QPS
With cache:    80% requests served from cache → 1ms response, DB sees only 2K QPS

Cache = Trade memory (cheap) for latency (expensive) and throughput
```

---

## Cache Placement

```
┌────────┐   ┌─────────────┐   ┌──────────────┐   ┌──────────┐   ┌──────────┐
│ Browser │──▶│ CDN/Edge    │──▶│ App Server   │──▶│ Cache    │──▶│ Database │
│ Cache   │   │ Cache       │   │ (local cache)│   │ (Redis)  │   │          │
└────────┘   └─────────────┘   └──────────────┘   └──────────┘   └──────────┘
   L1              L2                 L3                L4              L5

L1: Browser cache (HTTP headers: Cache-Control, ETag)
L2: CDN (CloudFront, Cloudflare) — static assets
L3: Application-level (in-memory HashMap, Guava)
L4: Distributed cache (Redis, Memcached)
L5: Database query cache (usually disabled — too many invalidation issues)
```

---

## Caching Strategies

### 1. Cache-Aside (Lazy Loading) ← Most Common

```
Read:
1. App checks cache
2. Cache HIT → return data
3. Cache MISS → read from DB → write to cache → return

Write:
1. Write to DB
2. Invalidate (delete) cache entry

┌─────┐  1.get  ┌───────┐
│ App │─────────▶│ Cache │
│     │◀─────────│       │
└──┬──┘  2.data  └───────┘
   │                  │
   │ 3.miss?     ┌────▼───┐
   └────────────▶│   DB   │
                 └────────┘
```

**Pros:** Only caches what's actually read, simple to implement
**Cons:** Cache miss = 3 trips (cache + DB + write cache), stale data possible

### 2. Write-Through

```
Write:
1. Write to cache
2. Cache synchronously writes to DB
3. Return success

Read:
1. Always read from cache (always fresh)
```

**Pros:** Cache always consistent with DB, simple reads
**Cons:** Write latency higher (2 writes), caches data that may never be read

### 3. Write-Behind (Write-Back)

```
Write:
1. Write to cache
2. Return success immediately
3. Cache asynchronously writes to DB (batched)
```

**Pros:** Very fast writes, batch DB operations
**Cons:** Data loss risk if cache crashes before DB write

### 4. Read-Through

```
Read:
1. App reads from cache
2. On miss, CACHE fetches from DB (not app)
3. Cache stores and returns

(Like cache-aside but cache manages DB reads)
```

### 5. Refresh-Ahead

```
- Cache proactively refreshes entries BEFORE they expire
- Based on access patterns (predict what will be needed)
- Reduces cache miss latency
```

---

## Comparison Table

```
┌──────────────────┬──────────────┬────────────────┬──────────────────┐
│ Strategy         │ Best for     │ Consistency    │ Write Speed      │
├──────────────────┼──────────────┼────────────────┼──────────────────┤
│ Cache-Aside      │ Read-heavy   │ Eventual       │ Fast (DB only)   │
│ Write-Through    │ Read-heavy   │ Strong         │ Slow (2 writes)  │
│ Write-Behind     │ Write-heavy  │ Eventual       │ Very Fast        │
│ Read-Through     │ Read-heavy   │ Eventual       │ Fast             │
└──────────────────┴──────────────┴────────────────┴──────────────────┘
```

---

## Eviction Policies

When cache is full, what to remove?

```
LRU (Least Recently Used)    ← Most common, usually the right choice
LFU (Least Frequently Used)  ← Good for skewed access patterns
FIFO (First In, First Out)   ← Simple but not ideal
TTL (Time To Live)           ← Expire after fixed time
Random                       ← Surprisingly effective for uniform access
```

### LRU Implementation
```
HashMap + Doubly Linked List
- GET: O(1) lookup in map, move to front of list
- PUT: O(1) insert at front, if full remove from tail
- EVICT: Remove tail node
```

---

## Cache Invalidation

The **hardest problem in computer science** (along with naming things).

### Strategies
```
1. TTL-based        → Set expiry time, accept staleness
2. Event-based      → Publish event on write, subscribers invalidate
3. Version-based    → Each cache entry has version, compare on read
4. Write-invalidate → Delete cache key on write (most common)
5. Write-update     → Update cache on write (consistency but complexity)
```

### Common Pitfalls
```
Problem: Thundering Herd
─────────────────────────
Cache expires → 1000 requests simultaneously hit DB

Solutions:
- Lock: First requester fetches, others wait
- Stale-while-revalidate: Serve stale, refresh in background
- Pre-warming: Refresh before TTL expires
```

```
Problem: Cache Penetration
───────────────────────────
Queries for keys that DON'T EXIST → always miss → hit DB

Solutions:
- Cache null results (short TTL)
- Bloom filter: check if key possibly exists before DB query
```

```
Problem: Cache Avalanche
─────────────────────────
Many keys expire at same time → DB overwhelmed

Solutions:
- Add random jitter to TTL (TTL + random(0, 60s))
- Never expire hot keys (refresh in background)
```

---

## Redis vs Memcached

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│ Feature            │ Redis                │ Memcached            │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Data structures    │ String, List, Set,   │ String only          │
│                    │ Hash, Sorted Set,    │                      │
│                    │ Stream, HyperLogLog  │                      │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Persistence        │ RDB + AOF            │ None                 │
│ Replication        │ Built-in             │ None (client-side)   │
│ Clustering         │ Redis Cluster        │ Client-side sharding │
│ Lua scripting      │ Yes                  │ No                   │
│ Pub/Sub            │ Yes                  │ No                   │
│ Memory efficiency  │ Higher overhead      │ Better for simple KV │
│ Throughput         │ ~100K-500K QPS       │ ~100K-1M QPS         │
└────────────────────┴──────────────────────┴──────────────────────┘

Use Redis when: Need data structures, persistence, or pub/sub
Use Memcached when: Simple caching, multi-threaded, memory efficiency
```

---

## Interview Tips

1. **Always mention caching** when system is read-heavy
2. **State the strategy** — "I'd use cache-aside with Redis and TTL of 5 minutes"
3. **Acknowledge trade-offs** — "This means data could be up to 5 minutes stale, which is acceptable for a social feed but not for inventory"
4. **Address invalidation** — "On write, I'd delete the cache key so the next read fetches fresh data"
5. **Know the numbers** — Redis: ~100K QPS, sub-millisecond latency
