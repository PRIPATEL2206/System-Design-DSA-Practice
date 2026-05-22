# Data-Intensive Patterns

## 1. Fan-out on Write vs Fan-out on Read

The fundamental choice for **feed/timeline** systems.

```
Fan-out on Write (Push Model):
──────────────────────────────
When user posts → push to ALL followers' feeds immediately

User A posts → write to:
  - Follower 1's feed cache
  - Follower 2's feed cache
  - ... Follower N's feed cache

✅ Fast reads (feed is pre-computed)
❌ Slow writes (celebrity with 10M followers = 10M writes)
❌ Wasted work if follower is inactive

Fan-out on Read (Pull Model):
─────────────────────────────
When user opens feed → pull from ALL followees at read time

User opens feed → query:
  - Get posts from Person 1 they follow
  - Get posts from Person 2 they follow
  - ... merge and rank

✅ No wasted writes
✅ Handles celebrities easily
❌ Slow reads (many queries at read time)

Hybrid (What Twitter/Instagram actually does):
──────────────────────────────────────────────
- Regular users: Fan-out on Write (push to followers)
- Celebrities (>500K followers): Fan-out on Read (pull at read time)
- Mix both in the feed
```

---

## 2. Write-Ahead Log (WAL)

```
Problem: Crash between deciding to write and actually writing → data loss

Solution: Write to sequential log FIRST, then apply to main storage

┌────────┐  1.write  ┌──────┐  2.apply  ┌──────────┐
│ Client │──────────▶│ WAL  │──────────▶│ Database │
└────────┘           │(disk)│           │ (memory) │
                     └──────┘           └──────────┘

On crash recovery: Replay WAL to restore uncommitted changes

Used by: PostgreSQL, MySQL, Kafka, Redis (AOF), LevelDB
```

---

## 3. LSM Trees (Log-Structured Merge Trees)

```
Optimized for WRITE-HEAVY workloads.

Write path:
1. Write to in-memory buffer (MemTable) → very fast!
2. When MemTable full → flush to disk as sorted SSTable
3. Background compaction: merge SSTables periodically

Read path:
1. Check MemTable
2. Check each SSTable (newest first)
3. Use Bloom filters to skip SSTables without the key

┌──────────┐
│ MemTable │  (in-memory, sorted)
└────┬─────┘
     │ flush
     ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ SSTable 1│ │ SSTable 2│ │ SSTable 3│  (on disk, immutable)
└──────────┘ └──────────┘ └──────────┘
         \         |         /
          \___ compact ____/
                  │
            ┌─────▼─────┐
            │  Merged    │  (fewer, larger files)
            │  SSTable   │
            └────────────┘

Used by: Cassandra, RocksDB, LevelDB, HBase
Compare: B-Tree (read-optimized) vs LSM (write-optimized)
```

---

## 4. Bloom Filters

```
Probabilistic data structure:
- "Is X in the set?" 
- "Definitely NOT" (100% accurate)
- "Probably YES" (small false positive rate)
- No false negatives!
- Very space efficient (10 bits per element for 1% FP rate)

Use cases in system design:
- Cache: Check if key exists before hitting DB (avoid cache penetration)
- Cassandra: Skip SSTables that don't contain the key
- Web crawler: Check if URL already visited
- CDN: Check if content is cached without full lookup

┌─────────────────────────────────────────┐
│ Bloom Filter: bit array [0,1,0,1,1,0,1]│
│                                          │
│ Insert "hello": hash1("hello")=2        │
│                  hash2("hello")=4        │
│                  hash3("hello")=6        │
│ Set bits 2, 4, 6 → [0,0,1,0,1,0,1]    │
│                                          │
│ Check "world": hash1=1, hash2=3, hash3=5│
│ Bit 1 = 0 → DEFINITELY NOT in set      │
└─────────────────────────────────────────┘
```

---

## 5. Consistent Hashing

```
Problem with simple hash:  hash(key) % N
  Adding/removing server → ALL keys remapped! (catastrophic cache miss)

Consistent Hashing:
  Servers and keys on a circular ring (0 to 2^32)
  Key maps to first server clockwise

         Server A
           │
    ┌──────┼──────┐
    │      ●      │
    │    /   \    │
   S_D ●     ● S_B
    │    \   /    │
    │      ●      │
    └──────┼──────┘
         Server C

Adding/removing server → only K/N keys remapped (K=keys, N=servers)

Virtual nodes: Each physical server → multiple points on ring
  → Better distribution (avoid hotspots)
  → Server with more capacity → more virtual nodes

Used by: DynamoDB, Cassandra, Memcached, Nginx (upstream)
```

---

## 6. Merkle Trees (Hash Trees)

```
Used to efficiently detect differences between replicas.

         Root Hash
        /          \
   Hash(AB)      Hash(CD)
   /      \      /      \
Hash(A) Hash(B) Hash(C) Hash(D)
  |       |       |       |
Data A  Data B  Data C  Data D

Comparison:
- If root hashes match → data is identical (one comparison!)
- If root differs → drill down to find exactly which blocks differ
- O(log n) comparisons instead of O(n)

Used by:
- Cassandra (anti-entropy repair between replicas)
- Git (content-addressable storage)
- Blockchain (transaction verification)
- S3 (data integrity verification)
```

---

## 7. Sharding Strategies Deep Dive

```
1. Hash Sharding
   key → hash(key) % num_shards
   ✅ Even distribution
   ❌ Range queries need scatter-gather

2. Range Sharding  
   A-M → Shard 1, N-Z → Shard 2
   ✅ Range queries within shard
   ❌ Hotspots (some ranges busier)

3. Geographic Sharding
   US users → US shard, EU users → EU shard
   ✅ Data locality, compliance
   ❌ Cross-region queries expensive

4. Entity-based Sharding
   All data for user_123 → same shard
   ✅ JOINs within entity are local
   ❌ Celebrity users create hotspots

5. Time-based Sharding
   Jan data → Shard 1, Feb → Shard 2
   ✅ Old shards can be archived
   ❌ Current month shard is hot
```

---

## 8. Backpressure

```
Problem: Producer is faster than consumer → memory explodes

Solutions:
┌───────────────────────────────────────────────────────────────┐
│ Strategy          │ How it works                              │
├───────────────────┼───────────────────────────────────────────┤
│ Blocking          │ Producer waits when buffer full           │
│ Drop newest       │ Discard new messages (lossy)              │
│ Drop oldest       │ Discard old messages (lossy)              │
│ Rate limiting     │ Reject above threshold (429)             │
│ Adaptive          │ Signal producer to slow down             │
│ Buffering         │ Expand buffer (queue) temporarily         │
└───────────────────┴───────────────────────────────────────────┘

In interviews: "If our consumer falls behind, the message queue acts as a buffer.
We'd set up alerts on queue depth and auto-scale consumers."
```
