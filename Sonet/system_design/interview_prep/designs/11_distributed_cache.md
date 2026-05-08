# Design: Distributed Cache (Redis at Scale)

## Problem Statement
Design a distributed in-memory caching system similar to Redis or Memcached that supports high read throughput, low latency, and horizontal scalability.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "What data structures? Just key-value, or also lists, sets?" → Key-value + sorted sets
- "Persistence (survive restart)?" → Yes, within configurable tolerance
- "Eviction when full?" → LRU
- "Consistency model: strong or eventual?" → Eventual (performance > consistency)
- "Replication for fault tolerance?" → Yes, primary-replica
- "Scale?" → 10 TB total data, 1 million RPS

### Functional Requirements
- GET(key) → value, O(1), < 1ms
- SET(key, value, ttl) — set with optional expiry
- DELETE(key)
- INCR/DECR for atomic counters
- LRU eviction when memory full

### Non-Functional Requirements
- Sub-millisecond latency (P99 < 1ms)
- 1 million RPS aggregate throughput
- 10 TB total data across cluster
- High availability: tolerate one node failure without data loss
- Eventual consistency across replicas

---

## Step 2: Capacity Estimation

```
10 TB total data
Average value size = 1 KB
Keys = 10 TB / 1 KB = 10 billion keys

Per node: 100 GB RAM → 100 million keys
Nodes needed: 10 TB / 100 GB = 100 nodes

Replication factor 2: 200 total node slots
(100 primary + 100 replica)

QPS:
  1 million RPS across 100 nodes = 10,000 RPS per node
  Redis: single thread handles ~100K RPS per node → well within limits
```

---

## Step 3: High-Level Architecture

```
[Client (App Server)]
        |
   [Cache Client Library]
   - Consistent hashing ring
   - Connection pool
        |
   determine shard by key hash
        |
  +-----+-----+
  |     |     |
[Shard1][Shard2][Shard3] ... [Shard N]
  |
[Primary Node]  ←── all writes
  |
[Replica Node]  ←── all reads (can also read from primary)
```

---

## Step 4: Consistent Hashing for Sharding

```
Map both cache nodes and keys onto a hash ring (0 to 2^32).

key → hash(key) → position on ring → nearest node clockwise = responsible node.

Adding a node:
  New node inserted into ring.
  Only keys between it and previous node migrate to new node.
  ~1/N keys remapped (much better than hash % N).

Virtual nodes (150 per server):
  Prevents uneven distribution.
  On failure, load spreads across many nodes (not just one).

Implementation in cache client:
  On startup: load ring from config (ZooKeeper / etcd)
  On SET/GET: hash key → route to correct node
  On node failure: ring updated → keys automatically route to next node
```

---

## Step 5: Replication

### Primary-Replica Replication
```
Each shard: 1 Primary + 1 (or 2) Replicas.

Write path:
  Client → Primary (write)
  Primary → Replica (async replication)
  Client gets ACK after Primary writes (not waiting for replica)

Read path:
  Client → Replica (reads) ← reduces load on primary
  Primary → for read-your-writes consistency scenarios

Replication lag: typically < 1ms (in same datacenter)
Risk: If Primary dies before replication → very small data loss window

Semi-sync option:
  Primary waits for at least one replica to ACK before returning.
  Slightly higher write latency, much stronger durability.
```

### Failover
```
Each shard has a Sentinel monitoring primary health.
Sentinel pings primary every 1 second.
3 consecutive missed pings → primary considered down.
Sentinel promotes replica to primary.
Other sentinels confirm (majority vote).
Clients notified of new primary via sentinel.
Failover time: typically < 30 seconds.
```

---

## Step 6: Memory Management

### Memory Layout
```
Hash table (dict):
  key → value pointer
  Separate chaining for collision resolution
  Load factor < 1.0 → rehash (double size)

Value storage:
  Redis uses multiple encodings per data type for memory efficiency:
    Short strings (< 44 chars): embedded in dict entry (avoids pointer overhead)
    Long strings: separate allocation
    Small integer lists: ziplist (compact array)
    Large lists: linked list
    Small hash: ziplist
    Large hash: hash table
```

### LRU Eviction
```
Approximate LRU (Redis approach):
  True LRU requires maintaining a global LRU list → expensive.
  Approximation: On eviction event, sample N random keys.
  Evict the one with oldest last-access time.
  N=10 samples gives 97% accuracy vs true LRU.
  
Eviction policies:
  noeviction     → return error when full (use for critical data)
  allkeys-lru    → evict any key by LRU (general purpose)
  volatile-lru   → evict only keys with TTL, by LRU
  allkeys-lfu    → evict by least-frequently-used
  volatile-ttl   → evict key with soonest TTL expiry
```

---

## Step 7: Persistence Options

### RDB (Redis Database Snapshots)
```
Point-in-time snapshot of all data.
Written to disk every N minutes or M writes (configurable).

Process:
  fork() creates child process (copy-on-write, no blocking)
  Child serialises data to .rdb file
  Atomic rename replaces old file

Pros: Fast restore, compact file, minimal performance impact
Cons: Data loss up to N minutes if crash between snapshots
```

### AOF (Append-Only File)
```
Log of every write command.
On restart: replay commands to rebuild state.

fsync options:
  always  — write + sync every command (safest, slowest)
  everysec — write every command, sync every second (balanced)
  no      — write but let OS decide when to sync (fastest, risky)

AOF Rewrite:
  Periodically compact AOF (remove redundant commands).
  SET x 1; SET x 2; SET x 3 → compacted to: SET x 3

Pros: Up to 1-second data durability
Cons: Larger file, slower than RDB
```

### Combined (Best of Both)
```
On startup: Load RDB snapshot, then replay AOF from that point.
Use both in production for durability + fast recovery.
```

---

## Step 8: Cache Client Design

```python
class CacheClient:
    def __init__(self, nodes):
        self.ring = ConsistentHashRing(nodes)
        self.connections = ConnectionPool(nodes, max_per_node=10)
    
    def get(self, key):
        node = self.ring.get_node(key)
        conn = self.connections.get(node)
        return conn.GET(key)
    
    def set(self, key, value, ttl=None):
        node = self.ring.get_node(key)
        conn = self.connections.get(node)
        if ttl:
            return conn.SETEX(key, ttl, value)
        return conn.SET(key, value)
    
    def handle_node_failure(self, failed_node):
        self.ring.remove_node(failed_node)
        # Keys automatically reroute to next node on ring
```

---

## Step 9: Cluster Data Rebalancing

```
Adding a new node to the cluster:
1. New node added to ring (between two existing nodes)
2. Keys that should now live on new node are migrated
3. Migration is gradual (no all-at-once downtime)
4. Old node keeps data until migration confirmed
5. Remove old keys from origin node

Zero-downtime migration:
  During migration, both old and new node serve the key.
  Reads: check new node first, fall back to old.
  Writes: write to both.
  After migration: stop writing to old node.
```

---

## Interview Tips

- Key insight: "Consistent hashing is the foundation — it's what allows adding/removing nodes without remapping all keys."
- Memory efficiency: "Redis uses different internal encodings for small vs large collections — massive memory savings."
- Persistence trade-off: "RDB for fast recovery, AOF for durability. Use both in production."
- LRU: "Redis uses approximate LRU (sample N keys) — much faster than maintaining a full LRU list."

## Common Follow-up Questions
- "How to handle hot keys (one key gets 20% of all traffic)?" → Replicate hot keys with suffix (key:1, key:2); client randomises which replica to read
- "How to implement distributed locks?" → SETNX (SET if Not eXists) with TTL; Redlock algorithm for multi-node
- "How to implement pub/sub?" → Redis pub/sub; SUBSCRIBE and PUBLISH commands
- "What if cache and DB are inconsistent?" → Cache-aside pattern + TTL; accept eventual consistency or use write-through
