# Caching — Deep Dive

## Part 1: Redis Architecture Internals

### Single-Threaded Event Loop
```
Redis uses a single-threaded event loop (like Node.js).

Why single-threaded?
  - No lock contention (huge win for in-memory ops)
  - Memory operations are nanoseconds → CPU not the bottleneck
  - Simpler implementation, easier reasoning about consistency
  
Implication: One slow command (O(n) operation on large list) blocks ALL other commands.
  KEYS *       → O(n) → NEVER use in production (blocks Redis)
  SMEMBERS set → O(n) → Use SSCAN instead (cursor-based, non-blocking)
  SORT         → O(n log n) → Expensive; avoid on large sets
  
Redis 6.0+: IO threads for network read/write (still single-threaded for commands)
```

### Memory Model
```
Redis uses jemalloc as memory allocator.
Every Redis object is allocated as a robj (Redis object) with reference counting.

Encoding selection (automatic, based on size):
  String < 44 bytes:     EMBSTR encoding (object + string in same memory block)
  String ≥ 44 bytes:     RAW (separate allocation)
  Integer 0–9999:        Shared integer objects (pre-allocated, no allocation)
  Small Hash (< 128):    ZIPLIST (compact encoding, cache-friendly)
  Large Hash:            HASHTABLE
  Small List (< 128):    QUICKLIST (linked list of ziplists)
  Large Sorted Set:      SKIPLIST + HASHTABLE
  
Memory savings: A hash of 100 small fields uses ZIPLIST → 10x less memory than HASHTABLE
```

---

## Part 2: Redis Data Structures — When to Use Each

### String
```
Use cases: Simple cache values, counters, session tokens, feature flags

Key operations:
  SET key value [EX seconds] [NX|XX]
  GET key
  INCR/DECR key            → atomic counter
  GETSET key newvalue      → get old value, set new
  SETNX key value          → distributed lock primitive
  
Time complexity: O(1) for all operations.
```

### Hash
```
Use cases: Object storage (user profiles, product data)

Instead of:
  user:123:name    → "Alice"
  user:123:email   → "alice@example.com"
  user:123:age     → "30"

Use:
  HSET user:123 name "Alice" email "alice@example.com" age 30
  HGET user:123 name
  HMGET user:123 name email
  HGETALL user:123

Advantage: Fetch all fields in one command; efficient memory (ZIPLIST for small hashes).
Disadvantage: Can't set individual field TTL (only key-level TTL).
```

### List
```
Use cases: Message queues, activity feeds, recent items

LPUSH / RPUSH  → add to left/right
LPOP / RPOP    → remove from left/right
LRANGE 0 -1    → read all
BLPOP          → blocking pop (wait for item, perfect for task queue)

Task queue pattern:
  Producer: RPUSH jobs "task_data"
  Consumer: BLPOP jobs 30  → blocks up to 30s for next job

Reliable queue:
  BRPOPLPUSH source_queue processing_queue
  → Atomically moves item to processing list
  → If consumer crashes, item in processing list can be recovered
```

### Set
```
Use cases: Unique visitor tracking, tag sets, friends list

SADD set member       → add member
SISMEMBER set member  → check membership O(1)
SMEMBERS set          → all members (avoid for large sets)
SINTERSTORE out a b   → intersection of a and b
SUNIONSTORE out a b   → union
SDIFFSTORE out a b    → difference

Use case: "Common friends between user A and user B"
  SINTERSTORE common_friends:A:B friends:A friends:B
```

### Sorted Set (ZSet)
```
Use cases: Leaderboards, rate limiting, priority queues, time-series

Each member has a floating-point score.
Members sorted by score, then lexicographically.

ZADD leaderboard 5000 "Alice"
ZADD leaderboard 7000 "Bob"
ZREVRANGE leaderboard 0 9 WITHSCORES  → top 10

Rate limiting:
  ZADD rate:{user} {timestamp} {timestamp}
  ZREMRANGEBYSCORE rate:{user} 0 {now - 60sec}
  count = ZCARD rate:{user}
  
Feed pagination:
  ZADD feed:{user} {timestamp} {post_id}
  ZREVRANGEBYSCORE feed:{user} +inf -inf LIMIT offset count
```

---

## Part 3: Redis Cluster — Internals

### Hash Slots
```
Redis Cluster divides keyspace into 16,384 hash slots.
Each master node owns a range of slots.

key → CRC16(key) % 16384 → slot → responsible node

Example with 3 nodes:
  Node A: slots 0–5460
  Node B: slots 5461–10922
  Node C: slots 10923–16383

Adding a node:
  Some slots migrated from existing nodes to new node.
  Other slots unchanged.
  
Slot migration:
  Atomic key migration: MIGRATE command
  During migration: clients get MOVED or ASK redirect → follow redirect
```

### Hash Tags — Force Keys to Same Slot
```
Problem: Multi-key commands (MGET, SUNIONSTORE) require all keys on same node.

Solution: Hash tags — {tag} in key name → only tag is hashed.
  user:{123}:profile  → hash "123" → slot N → node A
  user:{123}:settings → hash "123" → slot N → same node A!
  
  MGET user:{123}:profile user:{123}:settings → single-node command, works!

Without hash tag:
  user:123:profile  → different slot, different node
  user:123:settings → MGET fails (cross-slot not allowed)
```

---

## Part 4: CDN Architecture — Deep Dive

### Anycast Routing
```
CDN uses anycast: same IP address announced from multiple locations.
User's DNS/BGP routing sends them to nearest CDN PoP (Point of Presence).

Traditional unicast:
  IP 93.184.216.34 → one server in Virginia
  User in India → always routes to Virginia (300ms RTT)

Anycast:
  IP 93.184.216.34 → announced by 150 PoPs globally
  User in India → routed to nearest PoP in Mumbai (5ms RTT)
  
Used by: Cloudflare (270+ PoPs), Fastly, Akamai, AWS CloudFront
```

### Cache Hierarchy
```
L1 (Browser cache):
  Cache-Control: max-age=31536000 (1 year for static assets with hash in filename)
  ETag: "abc123" (conditional revalidation)
  
L2 (CDN Edge, nearest PoP):
  Serves most traffic
  Cache TTL set by origin (Cache-Control header)
  
L3 (CDN Shield, regional PoP):
  Reduces load on origin for cache misses at L2
  "Origin Shield": only one CDN node fetches from origin per region
  
L4 (Origin):
  S3 / your servers
  Only receives requests that miss all CDN layers
```

### Cache Invalidation in CDN
```
TTL-based (simple):
  Cache-Control: max-age=3600 → CDN caches for 1 hour
  Con: 1 hour delay between deploy and new content being served

Tag-based purge (advanced):
  Tag responses with content identifiers:
  Surrogate-Key: product-123 category-sports
  
  On product 123 update:
  POST /purge → { "tags": ["product-123"] }
  CDN immediately invalidates all responses tagged "product-123"
  
  Used by: Fastly, Cloudflare

Versioned URLs (best for static assets):
  /static/app-a3f8b2.js → never expires
  On new deploy: filename hash changes → old URL still valid (no cache flush needed)
  New URL auto-fetched on first request
```

---

## Part 5: Cache Patterns — Advanced

### Read-Through with Fallback Strategy
```python
def get_user(user_id):
    # Try L1: local in-process cache
    if user_id in local_cache:
        return local_cache[user_id]
    
    # Try L2: Redis
    data = redis.get(f"user:{user_id}")
    if data:
        local_cache[user_id] = data
        return data
    
    # L3: Database
    data = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # Warm both caches
    redis.setex(f"user:{user_id}", 300, data)  # 5-min TTL
    local_cache[user_id] = data
    
    return data
```

### Cache Stampede Prevention — Mutex Lock
```python
def get_with_lock(key, ttl, compute_func):
    value = redis.get(key)
    if value:
        return value
    
    # Try to acquire lock (SETNX = SET if Not eXists)
    lock_key = f"lock:{key}"
    acquired = redis.set(lock_key, "1", nx=True, ex=10)  # 10s TTL
    
    if acquired:
        try:
            value = compute_func()        # fetch from DB
            redis.setex(key, ttl, value)  # populate cache
            return value
        finally:
            redis.delete(lock_key)
    else:
        # Another thread is computing; wait briefly and retry
        time.sleep(0.1)
        return get_with_lock(key, ttl, compute_func)
```

### Cache-Aside with Version Numbers
```
Problem: Multiple caches (L1 in-process + L2 Redis) → hard to keep consistent.

Solution: Version in cache key:
  On each write: atomically increment version counter
  Cache key includes version: user:{id}:v{version}
  
  Read: get current version from Redis → construct key → fetch
  Write: increment version → old keys become unreachable → garbage collected by TTL
  
  No explicit cache invalidation needed!
```

---

## Part 6: Memcached vs Redis — When to Actually Use Memcached

```
Real cases where Memcached wins:

1. Pure string caching, very high hit rate
   Multi-threaded Memcached can use all CPU cores for network I/O.
   Redis 6+ closed this gap with IO threading.

2. Simplicity requirement
   Ops team knows Memcached well, doesn't want to learn Redis persistence/replication.

3. Very large values (> 1 MB)
   Redis stores everything in single-threaded memory; large values block others.
   
In practice (2024): Redis wins >90% of the time.
   Redis 7.x is extremely fast and feature-rich.
   Memcached's multi-threading advantage largely gone with Redis IO threads.
```

---

## Key Takeaways

1. **Redis single-threaded model**: never run O(n) commands in production on large datasets.
2. **Encoding optimisation**: keep hashes/sets small to benefit from ZIPLIST encoding.
3. **Hash tags in Redis Cluster**: mandatory for multi-key atomic operations.
4. **CDN anycast**: users route to nearest PoP; understand L2/L3/origin hierarchy.
5. **Cache stampede**: use mutex lock or probabilistic early expiry for hot, expensive keys.
6. **Versioned cache keys**: self-invalidating; no explicit purge needed.
7. **Choose Sorted Sets** for: leaderboards, rate limiting, time-series, feed pagination.
