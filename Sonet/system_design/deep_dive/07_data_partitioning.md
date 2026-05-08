# Data Partitioning (Sharding) — Deep Dive

## Part 1: Why Partition?

### The Limits of a Single Node
```
Single MySQL instance limits (2024):
  Storage:  ~16 TB (practical limit before performance degrades)
  Writes:   ~10,000 writes/second (sustained)
  Reads:    ~100,000 reads/second (with read replicas)
  
When you exceed these:
  → Must distribute data across multiple nodes (sharding)

Read replicas help with reads but NOT with writes.
Sharding is the only solution for write-heavy workloads beyond a single primary.
```

---

## Part 2: Partitioning Strategies — Detailed

### Strategy 1: Range Partitioning
```
Divide data into contiguous ranges based on a key.

Example: Users partitioned by user_id:
  Shard 1: user_id 1 – 10,000,000
  Shard 2: user_id 10,000,001 – 20,000,000
  Shard 3: user_id 20,000,001 – 30,000,000

Pros:
  Range scans efficient (all users in range on same shard)
  Easy to add new ranges (add new shard at end)

Cons:
  Hot spots: New users always go to the last shard (write-heavy)
  Uneven distribution if user activity is skewed
  
Real-world use:
  HBase, Cassandra, DynamoDB: scan ranges on sorted row keys
  Time-series data: partition by time range (2024-Q1, 2024-Q2)
```

### Strategy 2: Hash Partitioning
```
Apply hash function to partition key → determine shard.

shard_id = hash(user_id) % num_shards

Example with 4 shards:
  user_id=1001  → hash=57  → 57 % 4 = 1  → Shard 1
  user_id=1002  → hash=89  → 89 % 4 = 1  → Shard 1
  user_id=1003  → hash=23  → 23 % 4 = 3  → Shard 3

Pros:
  Even distribution (hash functions designed for uniform output)
  No hot spots (assuming key space is uniform)

Cons:
  Range queries span all shards (scatter-gather → expensive)
  Resharding = massive data movement (see consistent hashing below)
  
Real-world use:
  Simple sharding schemes, application-level sharding
```

### Strategy 3: Directory-Based Partitioning
```
Maintain a lookup table: partition_key → shard_id

┌──────────────┬─────────┐
│ user_id range│ shard   │
├──────────────┼─────────┤
│ 1-1000000    │ Shard A │
│ 1000001-2M   │ Shard B │
│ 2000001-5M   │ Shard C │  ← hot users can be split out here
└──────────────┴─────────┘

Pros:
  Flexible — can move individual rows or ranges between shards
  Can account for hot spots by splitting popular data

Cons:
  Lookup table is a bottleneck and SPOF
  Extra network hop per query
  
Real-world use:
  When you need surgical control over data placement
  VitessDB (MySQL at YouTube) uses this approach
```

---

## Part 3: Consistent Hashing — The Right Way

### The Modulo Problem
```
Hash-based: shard = hash(key) % N

Adding one shard (N=4 → N=5):
  hash(key) % 4 gave shard 3
  hash(key) % 5 gives shard 2
  
Nearly all keys change shards! Must migrate ~80% of data.
→ Massive downtime during any scaling operation.
```

### Consistent Hashing Ring
```
Hash both keys AND servers onto a ring [0, 2^32).

Initial state (3 servers):
  Server A → position 100
  Server B → position 200  
  Server C → position 300

Key assignment: hash(key) → position → travel clockwise → first server = owner

  hash("user:123") = 150 → clockwise → Server B
  hash("user:456") = 250 → clockwise → Server C
  hash("user:789") = 50  → clockwise → Server A

Adding Server D at position 175:
  Before: keys 100-200 → Server B
  After:  keys 100-175 → Server B   (unchanged)
          keys 175-200 → Server D   (migrated from B to D)
          keys 200-300 → Server C   (unchanged)
          
Only ~1/N keys remapped. Everything else untouched.
```

### Virtual Nodes — Making Distribution Even
```
Problem: With 3 servers at arbitrary ring positions, distribution may be uneven.

Solution: Each server maps to K virtual nodes (positions) on the ring.

Server A → positions [47, 183, 302, 441, 507, ...]  (K=150 virtual nodes)
Server B → positions [23, 156, 284, 403, 512, ...]
Server C → positions [91, 234, 371, 486, 523, ...]

With K=150:
  Statistical guarantee: each server gets ~1/N of keys
  Load varies < 2% from ideal (empirically)
  
When Server A fails:
  Its K virtual nodes distributed across B and C
  Load spread evenly → no single server gets overwhelmed

Used by:
  Cassandra: murmur3 hash, 256 virtual nodes per node
  Amazon DynamoDB: consistent hashing with virtual nodes
  Redis Cluster: 16,384 hash slots (with slot ranges per node)
```

---

## Part 4: Partitioning in Practice — Real Systems

### Cassandra Partitioning
```
Data model:
  Partition key → determines which node owns the data
  Clustering key → determines sort order within the partition

CREATE TABLE messages (
  conversation_id UUID,     ← partition key (hash → node)
  message_id      TIMEUUID, ← clustering key (sort order within partition)
  content         TEXT,
  PRIMARY KEY (conversation_id, message_id)
);

All messages for one conversation → same partition → same node → fast reads.
Queries must always include partition key (no cross-partition queries without scatter-gather).
```

### DynamoDB Partitioning
```
Partition key (PK) → hash → internal partition
Sort key (SK) → ordering within partition

Good PK: high cardinality, even access distribution
  Good:  user_id (millions of distinct values, even access)
  Bad:   status ('active'/'inactive') → only 2 partitions, hot spots

Adaptive Capacity (DynamoDB feature):
  If one partition is "hot" (more throughput), DynamoDB automatically
  rebalances by splitting the partition.
  
Hot partition example and fix:
  order_id as PK → good (UUID, uniform)
  date as PK → bad (all writes on Dec 25 → hot partition)
  Fix: PK = date#shard (date + random suffix 0-9) → 10 partitions per date
  Query: must scatter-gather across 10 shards, but writes are balanced
```

### Vitess (MySQL Sharding at YouTube/Slack)
```
Vitess is a sharding middleware for MySQL.
Application sees one logical MySQL → Vitess routes to correct shard.

Vindex (Virtual Index):
  Lookup vindex: user_id → shard_id (separate lookup table in DB)
  Hash vindex: crc32(user_id) % num_shards (computed, no lookup needed)
  
Cross-shard transactions:
  2PC for ACID (expensive, avoid)
  Or: redesign queries to avoid cross-shard (preferred)
  
Resharding workflow:
  1. Add new shards
  2. Copy data from old shards to new (online, no downtime)
  3. Verify consistency
  4. Cut over: switch routing to new shards
  5. Drop data from old shards
```

---

## Part 5: The Hotspot Problem

### Celebrity/Hot Key Problem
```
Normal distribution: 1 million users, each with 1000 followers.
Celebrity: 1 user with 100 million followers.

Impact: Justin Bieber posts → 100M fan-out operations → one shard handles all writes.
Others: spread across N shards.

Solutions:
  1. Detect hot keys (monitor request rate per shard key)
  2. Split hot key across sub-shards: 
     user:bieber:0, user:bieber:1, ... user:bieber:9
     Fan-out writes to all 10; reads aggregate from all 10.
     
  3. Separate "celebrity" tier:
     Normal users: standard sharding
     Celebrities: dedicated partition/node with extra resources
     
  4. Different algorithm for celebrities:
     Standard users: push model (fan-out writes)
     Celebrities: pull model (fans fetch on read)
```

---

## Part 6: Secondary Indexes on Sharded Data

### The Challenge
```
Shard by user_id.

Query: "Find all orders with status='shipped' in the last 7 days"
  → status is not the shard key
  → Must query ALL shards simultaneously (scatter-gather)
  → N shards × latency = higher latency + more load

Options:
  1. Scatter-gather: query all shards in parallel, merge results
     Acceptable for admin/batch queries; bad for user-facing (latency)
     
  2. Global secondary index:
     Maintain separate index shard: status → [order_ids across all shards]
     On status change: update main shard AND index shard (dual write)
     Risk: inconsistency between index and main data
     
  3. Denormalization:
     Pre-compute the query result
     Materialised view in a separate "shipped_orders" table
     Updated by event-driven processing
     Stale reads possible (eventual consistency)
```

---

## Part 7: Choosing the Right Partition Key

### Key Selection Criteria
```
Good partition key:
  ✓ High cardinality (many distinct values)
  ✓ Uniform distribution (no hot spots)
  ✓ Matches your most common query pattern
  ✓ Immutable (don't shard by something that changes)
  
Bad partition key:
  ✗ Low cardinality (boolean, status, country)
  ✗ Monotonic (auto-increment IDs → always writing to last shard)
  ✗ Too predictable (allows targeted DoS attacks)
```

### Common Patterns
```
User data → user_id (UUID or random)
Messages → conversation_id (all messages in a thread together)
Events → user_id or event_id (depends on query pattern)
Orders → customer_id (if queries are per-customer)
         OR order_id (if queries are per-order, lookup by ID)
Logs → service_id + time_bucket (one partition per service per day)
```

---

## Key Takeaways

1. **Range sharding**: great for scans, bad for new-data hotspots.
2. **Hash sharding**: even distribution, terrible for range queries, resharding is expensive.
3. **Consistent hashing**: the solution to resharding pain — only 1/N keys move.
4. **Virtual nodes**: ensure even load distribution with consistent hashing.
5. **Partition key selection**: high cardinality + uniform access pattern = good key.
6. **Hot spots**: detect, then split hot keys across sub-shards or change algorithm.
7. **Secondary indexes on sharded data**: always expensive; denormalise or scatter-gather.
8. **Never use monotonically increasing IDs** (auto-increment) as shard key — always writes to last shard.
