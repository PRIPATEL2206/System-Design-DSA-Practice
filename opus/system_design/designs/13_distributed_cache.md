# Design: Distributed Cache (Redis at Scale)

**Frequency:** High | **Difficulty:** ⭐⭐⭐ | **Companies:** Amazon, Google, Meta, Microsoft

---

## 1. Requirements

### Functional
- GET/SET/DELETE operations with sub-millisecond latency
- TTL (time-to-live) per key
- Support multiple data types (string, list, hash, set)
- Eviction policies (LRU, LFU)
- Pub/Sub for cache invalidation

### Non-Functional
- **Scale:** 1M+ QPS across cluster
- **Latency:** < 1ms P99 (same DC)
- **Availability:** 99.99% (cache miss = slow, not broken)
- **Memory:** Efficient usage, predictable behavior

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Applications                       │
└────────────────────────────┬────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Client Library │  (consistent hashing)
                    │  (Jedis/Redis-py)│
                    └────────┬────────┘
                             │ route to correct shard
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ Shard 1 │        │ Shard 2 │        │ Shard 3 │
    │ Master  │        │ Master  │        │ Master  │
    │ (0-5460)│        │(5461-10922)│     │(10923-16383)│
    └────┬────┘        └────┬────┘        └────┬────┘
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ Replica │        │ Replica │        │ Replica │
    └─────────┘        └─────────┘        └─────────┘

Hash Slots: 16384 total, divided among shards
Key routing: slot = CRC16(key) % 16384 → find shard owning that slot
```

---

## 3. Sharding Strategies

```
Option 1: Client-side sharding (Redis Cluster)
  Client computes hash → routes to correct shard
  ✅ No proxy overhead
  ⚠️ Client must handle topology changes

Option 2: Proxy-based (Twemproxy, Codis)
  Client → Proxy → Shard
  ✅ Simple client
  ⚠️ Proxy is bottleneck and SPOF

Option 3: Redis Cluster (built-in)
  Nodes gossip topology
  Client gets MOVED redirect if wrong shard
  ✅ Official, battle-tested
  ✅ Auto-failover

Consistent Hashing:
  When adding/removing shard → only ~1/N keys need to move
  Virtual nodes ensure even distribution
```

---

## 4. Replication & Failover

```
Master-Replica (async replication):

  ┌────────┐   writes    ┌────────┐
  │ Client │────────────▶│ Master │
  └────────┘             └───┬────┘
                             │ async replicate
                        ┌────▼────┐
       reads ─────────▶│ Replica │ (read replicas for scale)
                        └─────────┘

Failover (Redis Sentinel or Cluster):
1. Sentinel monitors master health (heartbeat)
2. Master down → Sentinel promotes replica to master
3. Clients redirected to new master
4. Failover time: 5-15 seconds

⚠️ Async replication → may lose last few writes during failover
   (not suitable for data that MUST not be lost)
```

---

## 5. Eviction Policies

```
When memory is full, which keys to remove?

Policy          Description                        Use When
──────────────────────────────────────────────────────────────
noeviction      Return error on write              Safety-critical data
allkeys-lru     Remove least recently used (any)   General caching ← DEFAULT
volatile-lru    LRU only among keys with TTL       Mix of cache + permanent
allkeys-lfu     Remove least frequently used       Skewed access patterns
volatile-ttl    Remove keys closest to expiry      Time-sensitive data
allkeys-random  Random eviction                    Uniform access

Memory management:
  maxmemory 10gb                  (cap at 10GB)
  maxmemory-policy allkeys-lru    (evict LRU when full)
```

---

## 6. Cache Patterns in Practice

```
Cache Warming:
  On deploy → pre-populate cache with hot data
  Prevents thundering herd after restart

Multi-Level Cache:
  L1: In-process (Guava/Caffeine) — 1μs, small
  L2: Distributed (Redis) — 1ms, large
  L3: Database — 10ms, unlimited

  Check L1 → miss? → Check L2 → miss? → Check L3 → populate L2 → populate L1

Cache Stampede Prevention:
  Problem: Key expires → 1000 threads hit DB simultaneously
  
  Solution 1: Distributed lock
    First thread acquires lock → fetches from DB → writes cache
    Others wait → read from cache when ready
  
  Solution 2: Stale-while-revalidate
    Serve stale data → refresh in background
    User gets slightly stale (1-5 seconds) but no latency spike
  
  Solution 3: Probabilistic early expiration
    Each read: random chance to refresh BEFORE TTL
    Spreads refresh load over time
```

---

## 7. Hot Key Problem

```
Problem: One key gets 100K+ QPS (viral tweet, flash sale)
  Single Redis shard overwhelmed!

Solutions:
1. Local cache (L1): Cache hot keys in app server memory
   Reduces Redis load by 10-100x for hot keys

2. Key replication: Store same value under multiple keys
   "hot_key" → "hot_key:1", "hot_key:2", ..., "hot_key:10"
   Client randomly picks suffix → load spread across shards

3. Read replicas: Direct hot key reads to multiple replicas
```

---

## 8. Persistence Options

```
RDB (Snapshots):
  - Point-in-time snapshot every N minutes
  - fork() → child writes entire dataset to disk
  - Fast recovery, but lose data since last snapshot
  - Good for: backup, disaster recovery

AOF (Append Only File):
  - Log every write operation
  - On restart: replay all operations
  - More durable (can sync every second or every write)
  - Good for: minimal data loss tolerance

Hybrid (RDB + AOF):
  - RDB for fast bulk recovery
  - AOF for operations since last RDB snapshot
  - Best of both worlds ← RECOMMENDED
```

---

## 9. Monitoring

```
Key metrics:
  - Hit ratio: hits / (hits + misses) — target > 90%
  - Memory usage: used_memory vs maxmemory
  - Connected clients: watch for leaks
  - Evictions/sec: if high → need more memory or better TTLs
  - Latency: P50, P99, P999
  - Replication lag: bytes behind master
```
