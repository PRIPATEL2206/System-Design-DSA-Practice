# Databases

## SQL vs NoSQL — Decision Framework

```
Use SQL when:                          Use NoSQL when:
- Data is structured & relational      - Schema is flexible / evolves fast
- Need ACID transactions               - Need massive scale (millions of writes/sec)
- Complex queries with JOINs           - Data is hierarchical (JSON/document)
- Strong consistency required          - Eventually consistent is acceptable
- Data < few TB                        - Global distribution needed
```

---

## SQL (Relational Databases)

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server, Amazon Aurora

### Strengths
- ACID guarantees
- Rich query language (JOINs, aggregations, window functions)
- Mature ecosystem, well-understood
- Strong consistency

### Indexing
```
Without index: Full table scan O(n)
With B-tree index: O(log n) lookup

Types:
- Primary key index (auto-created)
- Secondary index (on frequently queried columns)
- Composite index (multiple columns, order matters!)
- Covering index (index includes all columns in query — no table lookup)
- Full-text index (for LIKE '%search%' queries)
```

**Rule:** Index columns you filter/sort on. Too many indexes slow down writes.

### Scaling SQL
```
1. Connection Pooling (PgBouncer) — reuse DB connections
2. Read Replicas — route SELECT queries to replicas
3. Query Optimisation — indexes, EXPLAIN ANALYZE
4. Caching layer (Redis) — cache frequent read results
5. Vertical scaling — bigger instance
6. Sharding — last resort (breaks JOINs)
```

---

## NoSQL Database Types

### 1. Key-Value Store
```
Structure: HashMap<key, value>
Operations: GET, SET, DELETE (O(1))
Use cases: Session storage, caching, user preferences, shopping carts
Examples: Redis, DynamoDB, Memcached
```

### 2. Document Store
```
Structure: Collection of JSON/BSON documents
Operations: Find by field, nested queries
Use cases: User profiles, product catalogues, CMS, event logs
Examples: MongoDB, CouchDB, Firestore
Schema: Flexible — each document can have different fields
```

### 3. Wide-Column Store (Column-Family)
```
Structure: Rows identified by row key, columns grouped in families
Optimised: Write-heavy, time-series, append-only workloads
Use cases: IoT sensor data, analytics, activity logs, Cassandra at Netflix
Examples: Apache Cassandra, HBase, Google Bigtable
Key design: Choose partition key wisely — determines which node stores data
```

### 4. Graph Database
```
Structure: Nodes (entities) + Edges (relationships)
Use cases: Social networks, recommendation engines, fraud detection
Query: "Friends of friends", "Shortest path between A and B"
Examples: Neo4j, Amazon Neptune, JanusGraph
```

---

## Database Replication

**Why:** High availability, fault tolerance, read scaling.

### Master-Slave (Primary-Replica)
```
[Primary]  ←──── all writes
    |
    |──replication log──>  [Replica 1]  ← reads
    |──replication log──>  [Replica 2]  ← reads
    |──replication log──>  [Replica 3]  ← reads

Failover: Replica promoted to Primary if Primary fails.
```

**Trade-off:** Replication lag — replicas may be slightly behind. Reads from replicas may return stale data.

### Multi-Master (Active-Active)
```
[Master 1] ←──────→ [Master 2]  (both accept writes, sync bidirectionally)

Use case: Multi-region writes (allow writes in both US and EU)
Challenge: Conflict resolution when same record updated simultaneously
```

### Synchronous vs Asynchronous Replication
| Type | Guarantee | Latency |
|------|-----------|---------|
| **Synchronous** | Write confirmed only after replica acknowledges | Higher write latency |
| **Asynchronous** | Write confirmed immediately, replica may lag | Lower latency, risk of data loss on failover |
| **Semi-synchronous** | At least one replica must acknowledge | Balance of both |

---

## Database Sharding (Horizontal Partitioning)

**Why:** When a single DB server can't handle the data volume or write throughput.

**Sharding** = split data across multiple DB servers (shards), each shard holds a subset of the data.

### Sharding Strategies

#### 1. Range-Based Sharding
```
User IDs 1–1M       → Shard 1
User IDs 1M–2M      → Shard 2
User IDs 2M–3M      → Shard 3

Pro: Simple, range queries are easy
Con: Hot spots (new users always go to last shard)
```

#### 2. Hash-Based Sharding
```
shard = hash(user_id) % num_shards

Pro: Even distribution
Con: Range queries span all shards. Resharding is painful.
```

#### 3. Directory-Based Sharding
```
[Lookup Table]
user_id 123 → Shard 2
user_id 456 → Shard 1

Pro: Flexible, can move data between shards
Con: Lookup table becomes a bottleneck / SPOF
```

#### 4. Consistent Hashing (Best for dynamic shards)
```
Hash shard keys and servers onto a ring.
Each key goes to the first server clockwise on the ring.
Adding/removing servers only remaps ~1/N keys.
```

### Sharding Challenges
```
- Cross-shard JOINs are expensive (or impossible)
- Distributed transactions across shards are complex
- Resharding live data without downtime is hard
- Schema changes must apply across all shards
```

**Interview tip:** Mention sharding only when you've exhausted: caching, read replicas, query optimisation. Sharding is complex and should be a last resort.

---

## Choosing the Right Database

| Use Case | Recommended DB | Why |
|----------|---------------|-----|
| User accounts, auth | PostgreSQL / MySQL | ACID, relational |
| Session data, caching | Redis | Key-value, in-memory, fast |
| Product catalogue | MongoDB | Flexible schema, nested docs |
| Time-series metrics | Cassandra / InfluxDB | Write-heavy, append-only |
| Social graph | Neo4j / DynamoDB | Graph traversal / key-value |
| Full-text search | Elasticsearch | Inverted index |
| Data warehouse | Redshift / BigQuery | Columnar, analytical queries |
| Message offsets | Kafka (log) | Ordered, durable, high-throughput |
| File metadata | MySQL + S3 | Relational meta + object store |

---

## Quick Reference

| Concept | One-line |
|---------|---------|
| Index | Speeds up reads, slows down writes |
| Replication | Copies of data on multiple nodes for availability |
| Sharding | Split data across nodes for write scale |
| ACID | Strict guarantees — use for financial/critical data |
| BASE | Relaxed guarantees — use for scale/speed |
| Read replica | Serve read traffic from copies of the primary |
| Master-slave | One writer, many readers |
| Consistent hashing | Distribute shards so adding nodes remaps minimal keys |
