# Chapter 4: Databases Deep Dive

## Database Selection Guide

```
┌───────────────────────────────────────────────────────────────────────┐
│                    HOW TO PICK YOUR DATABASE                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Need ACID transactions?  ──YES──▶  Relational (PostgreSQL, MySQL)   │
│          │                                                            │
│         NO                                                            │
│          │                                                            │
│  Flexible schema + horizontal scale?  ──YES──▶  Document (MongoDB)   │
│          │                                                            │
│         NO                                                            │
│          │                                                            │
│  Simple key-value at massive scale?  ──YES──▶  KV Store (DynamoDB)   │
│          │                                                            │
│         NO                                                            │
│          │                                                            │
│  Complex relationships?  ──YES──▶  Graph DB (Neo4j)                  │
│          │                                                            │
│         NO                                                            │
│          │                                                            │
│  Time-series data?  ──YES──▶  TimescaleDB, InfluxDB                  │
│          │                                                            │
│         NO                                                            │
│          │                                                            │
│  Full-text search?  ──YES──▶  Elasticsearch                          │
│          │                                                            │
│         NO                                                            │
│          │                                                            │
│  Wide-column analytics?  ──YES──▶  Cassandra, HBase                  │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Relational Databases (SQL)

### When to Use
- Structured data with clear relationships
- Need ACID guarantees (banking, inventory)
- Complex queries with JOINs
- Data integrity is critical

### ACID Properties
```
Atomicity    → All or nothing (transaction either completes fully or rolls back)
Consistency  → Data always valid (constraints, foreign keys enforced)
Isolation    → Concurrent transactions don't interfere
Durability   → Committed data survives crashes (written to disk)
```

### Indexing (Critical for Performance!)

```
Without index: Full table scan O(n) → 1M rows = slow
With B-tree index: O(log n) → 1M rows = ~20 disk reads

Types:
──────
B-tree index        → Default, good for range queries and equality
Hash index          → O(1) equality only, no range
Composite index     → (col1, col2) — order matters! (leftmost prefix)
Covering index      → Includes all columns needed (avoids table lookup)
Full-text index     → For LIKE '%text%' searches
```

**Index Tips:**
- Index columns in WHERE, JOIN, ORDER BY
- Don't over-index (writes become slow)
- Composite index (a, b) works for queries on (a) or (a, b) but NOT (b) alone

### Scaling Relational DBs

```
Step 1: Vertical Scaling (bigger machine)
         ↓ limits reached
Step 2: Read Replicas (separate reads from writes)
         ┌──────────────┐
         │   Primary    │──writes──▶ single master
         │  (Leader)    │
         └──────┬───────┘
           replication
         ┌──────┼───────┐
         │      │       │
      ┌──▼──┐ ┌─▼──┐ ┌──▼──┐
      │Rep 1│ │Rep 2│ │Rep 3│  ← reads distributed
      └─────┘ └────┘ └─────┘

Step 3: Sharding (split data across multiple DBs)
         See "Sharding" section below
```

---

## NoSQL Databases

### Document Store (MongoDB, DynamoDB)
```
Best for:
- Flexible/evolving schema
- Hierarchical data (nested JSON)
- Microservices (each owns its data)
- Catalog, user profiles, content management

Example document:
{
  "_id": "user_123",
  "name": "Prince",
  "orders": [
    {"id": "ord_1", "total": 99.99, "items": [...]}
  ]
}
```

### Key-Value Store (Redis, DynamoDB, Memcached)
```
Best for:
- Caching (session data, API responses)
- Simple lookups by key
- Counters, rate limiting
- Leaderboards (sorted sets)

Operations: GET, SET, DELETE → all O(1)
```

### Wide-Column (Cassandra, HBase)
```
Best for:
- Write-heavy workloads
- Time-series data
- Event logging
- IoT data at scale

Think of it as: 2D key-value store
Row key → {column_family: {column: value, ...}}
```

### Graph Database (Neo4j)
```
Best for:
- Social networks (friends of friends)
- Recommendation engines
- Fraud detection
- Knowledge graphs

Query: "Find friends of friends who like Python"
→ 1 query in Graph DB vs complex JOINs in SQL
```

---

## Sharding (Partitioning)

Splitting data across multiple database instances.

### Sharding Strategies

```
1. Range-based Sharding
   ────────────────────
   Users A-M → Shard 1
   Users N-Z → Shard 2
   ⚠️ Problem: Uneven distribution (hotspots)

2. Hash-based Sharding (Most Common)
   ──────────────────────────────────
   shard = hash(user_id) % num_shards
   ✅ Even distribution
   ⚠️ Problem: Resharding when adding nodes

3. Consistent Hashing
   ───────────────────
   Nodes on a ring, data maps to nearest clockwise node
   ✅ Adding/removing nodes only affects neighbors
   Used by: DynamoDB, Cassandra, memcached

4. Directory-based Sharding
   ─────────────────────────
   Lookup table: key → shard mapping
   ✅ Flexible, can move data
   ⚠️ Single point of failure (the directory)
```

### Sharding Challenges
```
Cross-shard queries    → Expensive (scatter-gather)
Transactions           → Distributed transactions are hard
Joins                  → Can't JOIN across shards easily
Rebalancing            → Moving data is expensive
Hotspots               → Celebrity problem (one shard overloaded)
```

---

## Replication

### Single-Leader (Master-Slave)
```
All writes → Leader → replicates to → Followers
Reads can go to any replica

Pros: Simple, strong consistency for writes
Cons: Leader is SPOF, replication lag for reads
```

### Multi-Leader
```
Multiple leaders accept writes, sync with each other
Used for: Multi-datacenter setups

Pros: Write availability in each DC
Cons: Conflict resolution needed (last-write-wins, CRDTs)
```

### Leaderless (Dynamo-style)
```
Write to W replicas, Read from R replicas
If W + R > N: guaranteed to read latest write

Used by: Cassandra, DynamoDB
Pros: High availability, no SPOF
Cons: Eventual consistency, conflict resolution
```

---

## CAP Theorem

```
You can only guarantee 2 of 3:

    Consistency ───── Availability
         \              /
          \            /
           \          /
            \        /
        Partition Tolerance
        (this ALWAYS happens in distributed systems)

So real choice is:
CP → Consistent + Partition Tolerant (sacrifice availability)
     Examples: HBase, MongoDB (in default mode), ZooKeeper

AP → Available + Partition Tolerant (sacrifice consistency)
     Examples: Cassandra, DynamoDB, CouchDB
```

**In interviews say:**
"During a network partition, I'd choose [consistency/availability] because [reason based on use case]"

- Banking → CP (can't show wrong balance)
- Social feed → AP (showing slightly stale data is OK)

---

## Database Patterns for Interviews

### Read-Heavy (100:1 ratio)
```
Solution: Cache + Read Replicas
Client → Cache (Redis) → miss? → Read Replica → miss? → Primary
```

### Write-Heavy
```
Solution: Write-ahead log + Async processing
Client → Message Queue → Worker → Database
```

### Mixed with High Scale
```
Solution: CQRS (Command Query Responsibility Segregation)
Writes → Write DB (normalized, ACID)
Reads  → Read DB (denormalized, fast)
Sync via event stream (Kafka)
```
