# Database Internals — Deep Dive

## Part 1: How B-Trees Work (The Foundation of Relational DBs)

### Why B-Trees?
Most relational databases (MySQL InnoDB, PostgreSQL, Oracle) use B-Trees for indexes.

```
Problem: Finding a record among 1 billion rows.
  Linear scan: O(n) = 1 billion comparisons = minutes
  B-Tree: O(log n) = ~30 comparisons = microseconds
```

### B-Tree Structure
```
                    [50 | 100]                     ← Root (Internal node)
                   /    |      \
         [20|35]  [70|85]   [110|150]              ← Internal nodes
        /  |   \    ...
[1-19] [21-34] [36-49]  ...                        ← Leaf nodes (actual data)

Properties:
  - All leaf nodes at same depth (balanced)
  - Each node = one disk page (typically 4KB or 16KB)
  - B-Tree of order m: each node has up to m-1 keys, m children
  - MySQL InnoDB: order ≈ 1200 (a 3-level tree holds ~1.7 billion rows)
  
Node search: binary search within node → fast
Tree traversal: 3-4 levels for billions of rows → very fast
```

### B+ Tree (Used in Practice)
```
Variant used by MySQL, PostgreSQL:
  - All values stored at leaf nodes only (internal nodes only hold keys)
  - Leaf nodes linked together as a doubly linked list
  
Advantage: Range queries! 
  SELECT * WHERE age BETWEEN 20 AND 30:
    1. Find node with age=20 (O(log n))
    2. Walk linked list forward until age > 30
    → Much faster than B-Tree (no jumping between levels)
```

---

## Part 2: LSM-Trees (How Cassandra, RocksDB, LevelDB Work)

### Problem with B-Trees for Writes
```
B-Tree insert: find correct leaf node → modify page → write back to disk
Random disk write: ~10ms
Sequential disk write: ~100μs

At 10,000 inserts/sec → 10,000 random disk writes/sec → disk I/O saturated!
```

### LSM-Tree: Optimise for Writes
```
Log-Structured Merge Tree (LSM-Tree):

Write path:
  1. Write to in-memory buffer (MemTable) — very fast
  2. When MemTable full → flush to disk as immutable SSTable file
  3. Periodically: merge (compact) SSTables on disk

  All disk writes are SEQUENTIAL (much faster than random)!
  
Read path:
  1. Check MemTable (in memory) → O(1)
  2. Check recent SSTables (in memory cache) 
  3. Check older SSTables (disk read) — may need multiple file reads

SSTable (Sorted String Table):
  Sorted key-value pairs in a file
  Bloom filter per SSTable → "is key X in this SSTable?" → O(1) check
  If Bloom says yes → binary search SSTable
  If Bloom says no (definitely not) → skip file
```

### Compaction — The Key Operation
```
Problem: Many SSTables → many files to check on read → slow reads

Compaction: Merge multiple SSTables → one sorted SSTable
  Read all SSTables sorted order
  Merge (like merge sort)
  Write to new SSTable
  Delete old SSTables

This is like a background garbage collector for the storage engine.
Compaction runs continuously in background.

Trade-off:
  Reads: potentially check many SSTables before finding key
  Writes: very fast (sequential)
  Compaction: uses I/O bandwidth in background

Best for: Write-heavy workloads (logs, metrics, events)
Used by: Cassandra, HBase, RocksDB (which powers WiredTiger in MongoDB)
```

---

## Part 3: WAL (Write-Ahead Log)

Every major database uses WAL for durability.

```
Without WAL:
  Server writes data to disk page.
  Power failure mid-write → partially written page → corrupted DB.

With WAL:
  1. First, write the INTENT to a log file (WAL) — sequential write
  2. Then, modify the actual data page
  
  On restart after crash:
    Scan WAL for committed transactions not reflected in data files.
    "Redo" those operations.
    Database is always consistent.

WAL properties:
  - Sequential writes → fast
  - Durable after fsync()
  - Foundation of replication: replicas replay the WAL stream
```

---

## Part 4: MVCC (Multi-Version Concurrency Control)

How databases allow concurrent reads and writes without locking.

### The Problem
```
Transaction A: SELECT balance FROM accounts WHERE id=1  ← takes 100ms
Transaction B: UPDATE accounts SET balance=0 WHERE id=1 ← completes during A's read

Without MVCC: A gets either the old value or new value depending on timing.
With MVCC:    A sees a consistent snapshot of the DB at start of transaction.
```

### How MVCC Works (PostgreSQL)
```
Every row has hidden columns:
  xmin = transaction ID that created this row version
  xmax = transaction ID that deleted this row version (0 = not deleted)

When row is updated:
  Old version: xmin=100, xmax=200 (visible to transactions < 200)
  New version: xmin=200, xmax=0  (visible to transactions >= 200)

A transaction with ID 150:
  Sees old version (xmax=200 > 150 → old version still "alive" for this transaction)
  
A transaction with ID 300:
  Sees new version (xmax=0 → not deleted from its perspective)

No locks needed for reads! Readers never block writers, writers never block readers.
```

### Vacuum — Cleaning Up Old Versions
```
Problem: Old row versions accumulate → table bloat.

PostgreSQL VACUUM:
  Removes dead tuples (old versions with xmax < oldest active transaction)
  Reclaims space
  Must run regularly (AUTOVACUUM does this automatically)

Failure to vacuum → table grows unbounded → xid wraparound (catastrophic)
```

---

## Part 5: Query Execution — Under the Hood

### The Query Planner
```
SELECT u.name, COUNT(o.id) 
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.country = 'IN'
GROUP BY u.name
HAVING COUNT(o.id) > 5
ORDER BY COUNT(o.id) DESC
LIMIT 10;

Query planner steps:
1. Parse SQL → AST (Abstract Syntax Tree)
2. Rewrite → logical plan
3. Optimise → physical plan (this is the smart part)
4. Execute → return rows

EXPLAIN ANALYZE shows the actual plan + execution stats.
```

### Join Algorithms
```
Nested Loop Join:
  FOR each row in users:
    FOR each matching row in orders:
      yield result
  Complexity: O(n × m) — bad for large tables, good for small outer table

Hash Join (most common for large tables):
  Phase 1: Build hash table from smaller table (users → HashMap<user_id, user>)
  Phase 2: Scan larger table (orders), probe hash table
  Complexity: O(n + m) — fast, but requires memory for hash table
  
Merge Join (for pre-sorted inputs):
  Both tables sorted on join key → merge like merge sort
  Complexity: O(n log n + m log n) → good when index already provides ordering
  
PostgreSQL/MySQL chooses automatically based on statistics and cost estimates.
```

---

## Part 6: Storage Engines

### InnoDB (MySQL Default)
```
Architecture:
  Buffer Pool (RAM cache for pages): 60-80% of available RAM
  B+ Tree for indexes (clustered + secondary)
  WAL (redo log)
  MVCC for concurrency
  Row-level locking
  Foreign key support

Clustered Index:
  Primary key → determines physical row order on disk
  Secondary indexes store primary key value → lookup is primary key lookup
  
  Impact: Table is sorted by PK. Range queries on PK are fast.
  Bad PK choice (random UUID) → random writes → fragmentation
```

### PostgreSQL
```
Architecture:
  Shared Buffers (page cache)
  MVCC with heap storage (rows not clustered by PK)
  WAL
  Multiple index types: B-Tree, Hash, GiST, GIN, BRIN
  Table partitioning native
  
Advantages over InnoDB:
  Better MVCC implementation (no gap locks)
  Richer data types (JSON, arrays, ranges, UUIDs, PostGIS)
  Parallel query execution
  More advanced query planner
```

---

## Part 7: Database Replication — Mechanics

### Physical (Binary) Replication
```
Primary sends raw binary WAL stream to replica.
Replica applies WAL byte-by-byte.

Advantages:
  - Exact copy of primary (byte-for-byte)
  - Very low replication lag
  
Disadvantages:
  - Same OS, same DB version required
  - Cannot replicate to different DB engine (MySQL → PostgreSQL impossible)
  
Used by: PostgreSQL streaming replication, MySQL binary log replication
```

### Logical Replication
```
Primary sends logical changes (INSERT/UPDATE/DELETE rows) as a stream.
Replica interprets the logical operations.

Advantages:
  - Can replicate to different DB version
  - Can replicate subset of tables
  - Transformation possible (column subsets, type changes)
  
Disadvantages:
  - Higher overhead (decode WAL to logical operations)
  - Schema changes can break replication

Used by: PostgreSQL logical replication, Debezium CDC
```

### Replication Lag — Causes and Solutions
```
Sources of lag:
  1. Network latency between primary and replica
  2. Replica can't apply WAL as fast as primary generates it (under-provisioned replica)
  3. Long-running queries on replica blocking WAL application (read-write conflict)

Monitoring:
  SHOW SLAVE STATUS; → Seconds_Behind_Master
  pg_stat_replication → write_lag, flush_lag, replay_lag

Solutions:
  - Provision replica to match primary I/O performance
  - Use sync_standby (semi-sync) to limit lag
  - Route time-sensitive reads to primary
```

---

## Part 8: Database Indexing — Advanced

### Index Design Principles

```
Index Selectivity:
  High selectivity = few rows match (good index)
  Low selectivity = many rows match (poor index)
  
  INDEX on gender ('M'/'F') → 50% of rows match → poor
  INDEX on email → 1 in 1M rows match → excellent

Composite Index Column Order:
  Index (a, b, c) can answer queries on:
    WHERE a = ?                     ← yes
    WHERE a = ? AND b = ?           ← yes
    WHERE a = ? AND b = ? AND c = ? ← yes
    WHERE b = ?                     ← NO (leading column must be present)
    WHERE a = ? AND c = ?           ← partial (only a used for index scan)
    
  Rule: Most selective column first for equality predicates.
        Range predicate column goes last (stops prefix matching).

Covering Index:
  All columns in SELECT clause exist in the index itself.
  No table lookup needed ("index only scan").
  
  Query: SELECT id, email FROM users WHERE country='IN';
  Index: (country, id, email)  ← covers all needed columns
  → Zero table reads; extremely fast
```

### Index Types
```
B-Tree:   Most common; equality and range queries; sorted
Hash:     Equality only (=, !=); no range; faster for point lookups
GIN:      Full-text search, array containment (PostgreSQL)
GiST:     Geometric, geographic data (PostGIS), text similarity
BRIN:     Block Range Index; huge tables where physical order correlates with query
          (e.g., append-only logs ordered by created_at)
          Very small index size, less accurate than B-Tree
```

---

## Key Takeaways

1. **B-Trees** are optimal for range queries and sorted access (most relational DBs).
2. **LSM-Trees** trade read performance for write performance (Cassandra, RocksDB).
3. **WAL** is the foundation of durability — every write goes here first.
4. **MVCC** enables high read concurrency without locks.
5. **Clustered index** (InnoDB): choose PK that supports your most common access pattern.
6. **Composite indexes**: put equality columns first, range column last.
7. **Replication** works by shipping WAL or logical changes to replicas.
