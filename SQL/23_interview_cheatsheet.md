# Phase 23 — Interview Prep & Cheat Sheet

> This is the capstone of the entire DBMS curriculum. It consolidates everything from Phases 1–22 into a battle-ready reference: **50 interview questions**, **SQL puzzles**, **system design walkthroughs**, and a **one-page cheat sheet** covering every major concept.

---

## Table of Contents

1. [Top 50 DBMS Interview Questions](#1-top-50-interview-questions)
2. [SQL Puzzle Questions](#2-sql-puzzles)
3. [System Design — Twitter DB Schema](#3-design-twitter)
4. [System Design — Uber DB Schema](#4-design-uber)
5. [System Design — Amazon DB Schema](#5-design-amazon)
6. [Quick-Reference Cheat Sheet](#6-cheat-sheet)
7. [Download](#7-download)

---

## 1. Top 50 DBMS Interview Questions

### Fundamentals (Q1–Q10)

**Q1. What is a DBMS and how does it differ from a file system?**

A DBMS provides structured storage with ACID transactions, concurrency control, crash recovery, access control, and a query language (SQL). A file system stores raw bytes with no schema enforcement, no query optimiser, no transactional guarantees, and no built-in concurrency control. The DBMS trades raw I/O speed for data integrity and developer productivity.

---

**Q2. Explain the three-schema architecture (external, conceptual, internal).**

The **external schema** defines user-specific views (what each application or role sees). The **conceptual schema** defines the logical structure (tables, relationships, constraints) shared across all users. The **internal schema** defines physical storage (file organisation, indexes, page layout). This separation provides **logical data independence** (change conceptual without breaking external) and **physical data independence** (change storage without breaking conceptual).

---

**Q3. What are the ACID properties? Give an example of each.**

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All or nothing | Bank transfer: debit + credit both succeed or both rollback |
| **Consistency** | DB moves from one valid state to another | CHECK constraint prevents negative balance |
| **Isolation** | Concurrent transactions don't interfere | Two simultaneous transfers don't lose money |
| **Durability** | Committed data survives crashes | After COMMIT, power failure doesn't lose the transfer |

---

**Q4. What is normalization? Walk through 1NF to BCNF.**

Normalization eliminates redundancy by decomposing tables based on functional dependencies.

| Normal Form | Rule | Violation Example |
|---|---|---|
| **1NF** | Atomic values, no repeating groups | `phone: "555-1234, 555-5678"` → split into rows |
| **2NF** | 1NF + no partial dependencies on composite key | `(student_id, course_id) → student_name` — partial dep |
| **3NF** | 2NF + no transitive dependencies | `student_id → dept_id → dept_name` — transitive |
| **BCNF** | Every determinant is a candidate key | A non-key column determines part of a candidate key |

---

**Q5. When would you denormalize?**

Denormalize when: (1) read-heavy workloads with frequent JOINs across many tables, (2) reporting / OLAP queries needing aggregated data, (3) caching derived values to avoid recomputation (e.g., storing `order_total` instead of computing `SUM(line_items)`), (4) data warehouse star schemas where dimensional redundancy speeds queries. Always measure — premature denormalization causes update anomalies.

---

**Q6. Explain the different types of keys.**

| Key Type | Description |
|---|---|
| **Super key** | Any set of columns that uniquely identifies a row |
| **Candidate key** | Minimal super key (no column can be removed) |
| **Primary key** | The chosen candidate key for the table |
| **Foreign key** | Column referencing another table's primary key |
| **Composite key** | PK or candidate key spanning multiple columns |
| **Surrogate key** | System-generated (auto-increment, UUID) |
| **Natural key** | Real-world identifier (SSN, email) |

---

**Q7. What are the different types of JOINs?**

| JOIN | Returns |
|---|---|
| **INNER JOIN** | Only matching rows from both tables |
| **LEFT JOIN** | All left rows + matching right (NULL if no match) |
| **RIGHT JOIN** | All right rows + matching left (NULL if no match) |
| **FULL OUTER JOIN** | All rows from both (NULL where no match) |
| **CROSS JOIN** | Cartesian product (every combination) |
| **SELF JOIN** | Table joined with itself (e.g., employee → manager) |
| **LATERAL JOIN** | Subquery that references columns from preceding tables |

---

**Q8. Explain the difference between WHERE and HAVING.**

`WHERE` filters rows **before** grouping (`GROUP BY`). `HAVING` filters groups **after** aggregation. You cannot use aggregate functions in `WHERE` (they haven't been computed yet), but you can in `HAVING`.

```sql
SELECT department, AVG(salary) AS avg_sal
FROM employees
WHERE status = 'active'           -- filters individual rows
GROUP BY department
HAVING AVG(salary) > 80000;       -- filters groups
```

---

**Q9. What is a view? What is a materialized view?**

A **view** is a named query — a virtual table computed on the fly each time it's referenced. A **materialized view** stores the query result physically on disk and must be refreshed (manually or on schedule) to reflect changes. Views provide security and abstraction; materialized views provide performance for expensive aggregations.

```sql
-- View (virtual)
CREATE VIEW active_orders AS
SELECT * FROM orders WHERE status = 'active';

-- Materialized view (physical)
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT DATE_TRUNC('month', order_date) AS month, SUM(total) AS revenue
FROM orders GROUP BY 1;

REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

---

**Q10. Explain referential integrity and CASCADE options.**

Referential integrity ensures foreign keys always point to existing rows. CASCADE options define what happens when the referenced row changes:

| Option | On DELETE | On UPDATE |
|---|---|---|
| `CASCADE` | Delete child rows too | Update child FK values |
| `SET NULL` | Set child FK to NULL | Set child FK to NULL |
| `SET DEFAULT` | Set child FK to default | Set child FK to default |
| `RESTRICT` | Block if children exist | Block if children exist |
| `NO ACTION` | Check at end of statement | Check at end of statement |

---

### Indexes & Optimization (Q11–Q20)

**Q11. What is a B-Tree index and how does it work?**

A B-Tree (balanced tree) index organises keys in sorted order across a tree structure. Internal nodes contain keys and pointers to child nodes; leaf nodes contain keys and pointers to data rows (CTIDs). Lookup is O(log N) — for 100M rows with branching factor 500, only 3–4 levels need to be traversed. B-Trees support equality (`=`), range (`<`, `>`, `BETWEEN`), and prefix matching (`LIKE 'abc%'`).

---

**Q12. Compare B-Tree, Hash, GIN, and GiST indexes.**

| Index Type | Best For | Supports Range? | Notes |
|---|---|---|---|
| **B-Tree** | General equality + range | Yes | Default, most versatile |
| **Hash** | Exact equality only | No | Smaller than B-Tree for equality |
| **GIN** | Full-text search, JSONB, arrays | Special | Inverted index; one entry per value |
| **GiST** | Geometry, range types, similarity | Special | Lossy; supports nearest-neighbor |
| **BRIN** | Large sorted tables (time-series) | Yes | Tiny; stores min/max per page range |

---

**Q13. What is a covering index?**

A covering index includes all columns needed by a query, so the query can be answered entirely from the index without visiting the heap (index-only scan). In PostgreSQL, use `INCLUDE`:

```sql
CREATE INDEX idx_orders_cover ON orders (customer_id)
    INCLUDE (order_date, total);

-- This query uses index-only scan:
SELECT order_date, total FROM orders WHERE customer_id = 42;
```

---

**Q14. How do you read an EXPLAIN plan?**

Key things to look for:
- **Seq Scan** vs **Index Scan** vs **Index Only Scan** — how data is accessed
- **Nested Loop** vs **Hash Join** vs **Merge Join** — join strategy
- **Rows** — estimated vs actual (if using `ANALYZE`)
- **Cost** — startup cost..total cost (in arbitrary units)
- **Buffers** — shared hit (cache) vs read (disk)
- **Sort** — in-memory vs on-disk (external sort)

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ... FROM ... WHERE ...;
```

---

**Q15. What is a composite index and how does column order matter?**

A composite index indexes multiple columns in a specific order. The index can be used for queries that filter on a **leftmost prefix** of the indexed columns:

```sql
CREATE INDEX idx ON orders (customer_id, order_date, status);
-- Used for: WHERE customer_id = 42
-- Used for: WHERE customer_id = 42 AND order_date > '2025-01-01'
-- Used for: WHERE customer_id = 42 AND order_date > '2025-01-01' AND status = 'shipped'
-- NOT used: WHERE order_date > '2025-01-01' (skips first column)
-- NOT used: WHERE status = 'shipped' (skips first two columns)
```

---

**Q16. When would a sequential scan beat an index scan?**

When the query returns a large fraction of the table (>5-10%), the cost of random I/O from index lookups exceeds the cost of reading the entire table sequentially. The planner compares: `N_pages × seq_page_cost` vs `M_pages × random_page_cost`. Other cases: very small tables (a few pages), no suitable index, or `WHERE` clause doesn't match any index.

---

**Q17. What is index bloat and how do you fix it?**

Index bloat occurs when dead tuples (from UPDATEs/DELETEs) leave empty space in index pages that VACUUM cannot fully reclaim. Fix with: (1) `REINDEX INDEX idx_name;` (locks the index), (2) `REINDEX CONCURRENTLY` (PostgreSQL 12+, online), (3) `pg_repack` (online, no locks), (4) tune `autovacuum` to prevent bloat accumulating.

---

**Q18. Explain partial and expression indexes.**

```sql
-- Partial index: only indexes rows matching a condition
CREATE INDEX idx_active_orders ON orders (customer_id)
    WHERE status = 'active';
-- Smaller index, faster lookups for active orders

-- Expression index: indexes a computed value
CREATE INDEX idx_lower_email ON users (LOWER(email));
-- Supports: WHERE LOWER(email) = 'alice@example.com'
```

---

**Q19. What is query plan caching and when can it hurt?**

PostgreSQL caches plans for prepared statements. A generic plan may be suboptimal when parameter values have very different selectivities (e.g., `status = 'active'` matches 90% of rows vs `status = 'cancelled'` matches 0.1%). PostgreSQL auto-switches between custom and generic plans based on cost estimates. Force custom plans with `PREPARE ... / EXECUTE ...` or application-level hints.

---

**Q20. How do you identify and fix slow queries?**

1. `pg_stat_statements` — find top queries by total time, calls, mean time
2. `EXPLAIN (ANALYZE, BUFFERS)` — identify full scans, disk reads, bad joins
3. Check for missing indexes (`seq_scan` count in `pg_stat_user_tables`)
4. Check for bloat (`pgstattuple`, `pg_stat_user_tables.n_dead_tup`)
5. Check for lock contention (`pg_stat_activity`, `pg_locks`)
6. Fix: add indexes, rewrite queries, increase `work_mem`, partition tables

---

### Transactions & Concurrency (Q21–Q30)

**Q21. What are the SQL isolation levels?**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Implementation |
|---|---|---|---|---|
| **READ UNCOMMITTED** | Possible | Possible | Possible | Rare in practice |
| **READ COMMITTED** | Prevented | Possible | Possible | PostgreSQL default |
| **REPEATABLE READ** | Prevented | Prevented | Possible* | *PG prevents phantoms too via MVCC |
| **SERIALIZABLE** | Prevented | Prevented | Prevented | SSI in PostgreSQL |

---

**Q22. Explain MVCC (Multi-Version Concurrency Control).**

MVCC keeps multiple versions of each row (identified by `xmin`/`xmax` transaction IDs). Readers see a snapshot of data as of their transaction start — they never block writers, and writers never block readers. Old versions are cleaned up by VACUUM. PostgreSQL stores versions in the heap; InnoDB stores them in the undo log.

---

**Q23. What is a deadlock and how do databases handle it?**

A deadlock occurs when two transactions each hold a lock the other needs. The database detects deadlock cycles (via a wait-for graph) and kills one transaction (the victim), rolling it back. Prevention strategies: (1) acquire locks in consistent order, (2) use `SELECT ... FOR UPDATE NOWAIT`, (3) keep transactions short.

```
T1: locks row A, waits for row B
T2: locks row B, waits for row A
→ Deadlock! DB kills T2 → T2 rolls back → T1 proceeds
```

---

**Q24. Explain pessimistic vs optimistic concurrency.**

**Pessimistic**: Acquire locks before accessing data — prevents conflicts but reduces concurrency. `SELECT ... FOR UPDATE`.

**Optimistic**: Read without locking, check for conflicts at commit time (using version numbers or timestamps). If conflict detected, abort and retry. Better for low-contention workloads.

```sql
-- Optimistic with version column
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE product_id = 42 AND version = 7;
-- If 0 rows affected → conflict, retry
```

---

**Q25. What is the difference between shared and exclusive locks?**

| Lock | Also Called | Allows Concurrent | Acquired By |
|---|---|---|---|
| **Shared (S)** | Read lock | Other shared locks | SELECT ... FOR SHARE |
| **Exclusive (X)** | Write lock | Nothing | UPDATE, DELETE, SELECT ... FOR UPDATE |

Multiple readers can hold shared locks simultaneously. An exclusive lock blocks all other locks.

---

**Q26. Explain Two-Phase Locking (2PL).**

2PL guarantees serializability: (1) **Growing phase** — acquire locks, never release. (2) **Shrinking phase** — release locks, never acquire new ones. **Strict 2PL** holds all exclusive locks until commit/abort, preventing cascading rollbacks. The downside: reduced concurrency and potential for deadlocks.

---

**Q27. What is a phantom read?**

A phantom read occurs when a transaction re-executes a range query and finds new rows that were inserted by another committed transaction. Example: T1 reads `WHERE salary > 100000` (finds 10 rows), T2 inserts a new employee with salary 150000 and commits, T1 re-reads and finds 11 rows. Prevented at SERIALIZABLE isolation.

---

**Q28. What is Write-Ahead Logging (WAL)?**

WAL ensures durability: every change is first written to a sequential log file before modifying data pages. On crash, the database replays WAL records to reconstruct committed changes and undo uncommitted ones. WAL writes are sequential (fast) while data page writes are random (slow), so WAL absorbs the write burst.

---

**Q29. Explain the ARIES recovery algorithm.**

ARIES has three phases: (1) **Analysis** — scan WAL from last checkpoint to identify dirty pages and active transactions. (2) **Redo** — replay all WAL records from the earliest dirty page LSN to reconstruct the state at crash time. (3) **Undo** — roll back uncommitted transactions by processing their WAL records in reverse.

---

**Q30. What are savepoints?**

Savepoints create named rollback points within a transaction. You can roll back to a savepoint without aborting the entire transaction.

```sql
BEGIN;
INSERT INTO orders VALUES (1, 'pending', 100);
SAVEPOINT sp1;
INSERT INTO orders VALUES (2, 'pending', 200);
-- Oops, rollback just the second insert
ROLLBACK TO sp1;
COMMIT; -- Only order 1 is committed
```

---

### Advanced Topics (Q31–Q40)

**Q31. Explain stored procedures vs functions.**

| Aspect | Stored Procedure | Function |
|---|---|---|
| Return value | Optional OUT params | Must return a value |
| Usable in SELECT | No | Yes (`SELECT my_func()`) |
| Transaction control | Can COMMIT/ROLLBACK inside | Cannot (runs in caller's txn) |
| Side effects | Allowed | Should be side-effect-free |

---

**Q32. What are window functions?**

Window functions compute a value across a set of rows related to the current row, without collapsing them into groups (unlike `GROUP BY`). Key clause: `OVER(PARTITION BY ... ORDER BY ...)`.

```sql
SELECT
    employee_id,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    salary - LAG(salary) OVER (ORDER BY salary) AS diff_from_prev
FROM employees;
```

---

**Q33. Explain the CAP theorem.**

A distributed system can provide at most two of three guarantees simultaneously: **Consistency** (every read returns the latest write), **Availability** (every request gets a response), **Partition tolerance** (system functions despite network splits). Since partitions are inevitable in distributed systems, the real choice is CP (sacrifice availability during partitions — e.g., traditional RDBMS, HBase) vs AP (sacrifice consistency — e.g., Cassandra, DynamoDB).

---

**Q34. What is the difference between ACID and BASE?**

| ACID | BASE |
|---|---|
| Strong consistency | Eventual consistency |
| Pessimistic (lock-based) | Optimistic (conflict resolution) |
| Strict transactional guarantees | Soft state, best-effort |
| Vertical scaling | Horizontal scaling |
| Traditional RDBMS | NoSQL, distributed systems |

---

**Q35. Compare star schema and snowflake schema.**

**Star**: Central fact table + flat denormalized dimension tables. Fewer JOINs, simpler queries, preferred for BI/OLAP.

**Snowflake**: Dimension tables are normalized into sub-tables. Less redundancy, more JOINs. Use when dimensions are very large or hierarchies change frequently.

---

**Q36. What are slowly changing dimensions (SCD)?**

SCD handles dimension attribute changes over time. **Type 1**: overwrite (no history). **Type 2**: add new row with effective dates (full history, most common). **Type 3**: add column for previous value (limited history). **Type 6**: hybrid of 1+2+3.

---

**Q37. Explain partitioning vs sharding.**

**Partitioning**: splits a table into sub-tables within one database instance. Benefits: partition pruning, easier maintenance, O(1) archival via DROP. **Sharding**: distributes data across multiple independent database nodes. Benefits: horizontal scaling of writes/storage. Sharding is far more complex (cross-shard joins, distributed transactions, rebalancing). Use partitioning first; shard only when one machine isn't enough.

---

**Q38. What is a columnar storage engine?**

Columnar engines store data column-by-column instead of row-by-row. Benefits for analytics: (1) read only needed columns (less I/O), (2) same-type values compress 5-10x better, (3) vectorized CPU processing. Examples: Redshift, BigQuery, ClickHouse, Parquet, DuckDB. Bad for OLTP (row lookups need all columns).

---

**Q39. Explain the buffer pool and its replacement policy.**

The buffer pool caches disk pages in RAM. On a hit, data is served from memory (~100 ns). On a miss, the page is loaded from disk (~20 µs). When full, a replacement policy decides which page to evict. PostgreSQL uses **clock-sweep** with usage counts (0–5): frequently accessed pages have higher counts and survive more eviction sweeps. A ring buffer protects against sequential scan flooding.

---

**Q40. What is TOAST in PostgreSQL?**

TOAST (The Oversized-Attribute Storage Technique) handles values > ~2 KB. PostgreSQL compresses the value; if still too large, slices it into chunks stored in a separate TOAST table. The main row stores a small pointer. Strategies: EXTENDED (compress + externalize), EXTERNAL (externalize without compression), MAIN (prefer inline), PLAIN (no TOAST).

---

### NoSQL & Distributed (Q41–Q50)

**Q41. When should you use NoSQL over SQL?**

| Use NoSQL When | Use SQL When |
|---|---|---|
| Schema evolves rapidly | Schema is stable and well-defined |
| Massive write throughput needed | Complex transactions with ACID |
| Data is hierarchical/nested (JSON) | Complex multi-table JOINs |
| Horizontal scaling is critical | Strong consistency required |
| Simple access patterns (key-based) | Ad-hoc analytical queries |

---

**Q42. Compare MongoDB, Redis, Cassandra, and Neo4j.**

| Database | Type | Data Model | Best For |
|---|---|---|---|
| **MongoDB** | Document | JSON/BSON documents | Flexible schemas, catalogs, CMS |
| **Redis** | Key-Value | Strings, Hashes, Lists, Sets | Caching, sessions, leaderboards |
| **Cassandra** | Column-Family | Wide rows, partition+clustering key | IoT, time-series, messaging |
| **Neo4j** | Graph | Nodes + edges + properties | Social networks, recommendations |

---

**Q43. What is eventual consistency?**

After a write, replicas may temporarily return stale data, but given enough time without new writes, all replicas will converge to the same value. The **consistency window** is the time between write and full convergence. Techniques to speed convergence: anti-entropy (Merkle trees), read repair, gossip protocol.

---

**Q44. Explain vector clocks.**

Vector clocks track causality in distributed systems. Each node maintains a vector of counters `[N1:3, N2:5, N3:2]`. On local event: increment own counter. On send: attach vector. On receive: take element-wise max and increment own counter. Two events are concurrent if neither vector dominates the other — conflict resolution needed (LWW, app-level merge, CRDTs).

---

**Q45. What is consistent hashing?**

Consistent hashing places nodes and keys on a hash ring. Each key maps to the next node clockwise. When a node is added/removed, only ~1/N of keys need to move (vs rehashing everything with `hash % N`). Virtual nodes (each physical node has ~100+ positions on the ring) ensure even distribution.

---

**Q46. Explain the Saga pattern for distributed transactions.**

A Saga breaks a distributed transaction into a sequence of local transactions, each with a **compensating action** for rollback. If step 3 fails, compensating actions for steps 2 and 1 are executed in reverse. Two coordination styles: **choreography** (events trigger next step) and **orchestration** (central coordinator directs steps).

---

**Q47. What are CRDTs?**

Conflict-free Replicated Data Types are data structures that can be independently updated on different replicas and automatically merged without conflicts. Examples: G-Counter (grow-only counter), PN-Counter (increment/decrement), OR-Set (observed-remove set), LWW-Register. Used in eventually consistent systems where coordination is expensive.

---

**Q48. Explain ETL vs ELT.**

**ETL** (Extract-Transform-Load): Transform data in an external engine before loading into the warehouse. Traditional, on-premise approach.

**ELT** (Extract-Load-Transform): Load raw data into the warehouse, transform there using SQL/dbt. Modern cloud approach — leverages elastic compute, preserves raw data, enables iterative transformation.

---

**Q49. What is polyglot persistence?**

Using multiple database technologies in a single application, each chosen for its strengths: PostgreSQL for transactions, Redis for caching, Elasticsearch for search, Neo4j for recommendations, S3 for blob storage. The challenge is keeping data consistent across systems — solved with CDC (Change Data Capture), event sourcing, or dual writes with idempotency.

---

**Q50. How do you design a schema for a system that needs both OLTP and OLAP?**

Separate concerns: (1) OLTP database (PostgreSQL/MySQL) handles transactions with normalized schema. (2) ETL/ELT pipeline (Fivetran + dbt) moves data to a warehouse. (3) OLAP warehouse (Snowflake/BigQuery/Redshift) with star schema for analytics. (4) Optional: materialized views or read replicas for light analytics on the OLTP DB. Never run heavy analytical queries on the OLTP database — it will degrade transactional performance.

---

## 2. SQL Puzzle Questions

### Puzzle 1 — Second Highest Salary

```sql
-- Find the second highest salary (handle ties correctly)
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
OFFSET 1 LIMIT 1;

-- Alternative using DENSE_RANK:
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;
```

---

### Puzzle 2 — Employees Earning More Than Their Manager

```sql
SELECT
    e.name   AS employee,
    e.salary AS employee_salary,
    m.name   AS manager,
    m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

---

### Puzzle 3 — Consecutive Login Days (Island Detection)

```sql
-- Find users who logged in for 3+ consecutive days
WITH numbered AS (
    SELECT
        user_id,
        login_date,
        login_date - ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        )::INT AS grp
    FROM (SELECT DISTINCT user_id, login_date FROM logins) t
),
streaks AS (
    SELECT user_id, grp, COUNT(*) AS streak_len,
           MIN(login_date) AS start_date,
           MAX(login_date) AS end_date
    FROM numbered
    GROUP BY user_id, grp
    HAVING COUNT(*) >= 3
)
SELECT * FROM streaks ORDER BY user_id, start_date;
```

---

### Puzzle 4 — Department with Highest Average Salary

```sql
SELECT department, avg_salary
FROM (
    SELECT
        department,
        AVG(salary) AS avg_salary,
        RANK() OVER (ORDER BY AVG(salary) DESC) AS rnk
    FROM employees
    GROUP BY department
) t
WHERE rnk = 1;
```

---

### Puzzle 5 — Running Total That Resets

```sql
-- Running total of daily sales that resets each month
SELECT
    sale_date,
    daily_amount,
    SUM(daily_amount) OVER (
        PARTITION BY DATE_TRUNC('month', sale_date)
        ORDER BY sale_date
        ROWS UNBOUNDED PRECEDING
    ) AS mtd_total
FROM daily_sales
ORDER BY sale_date;
```

---

### Puzzle 6 — Find Duplicates

```sql
-- Find all duplicate emails
SELECT email, COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Delete duplicates, keeping the row with the lowest ID
DELETE FROM users
WHERE id NOT IN (
    SELECT MIN(id) FROM users GROUP BY email
);
```

---

### Puzzle 7 — Pivot / Cross-Tab Without PIVOT

```sql
-- Monthly revenue by category as columns
SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(CASE WHEN category = 'Electronics' THEN total ELSE 0 END) AS electronics,
    SUM(CASE WHEN category = 'Clothing'    THEN total ELSE 0 END) AS clothing,
    SUM(CASE WHEN category = 'Food'        THEN total ELSE 0 END) AS food
FROM orders o
JOIN products p ON o.product_id = p.product_id
GROUP BY 1
ORDER BY 1;
```

---

### Puzzle 8 — Median Salary

```sql
-- PostgreSQL: using percentile_cont
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;

-- ANSI SQL approach:
WITH ranked AS (
    SELECT salary,
           ROW_NUMBER() OVER (ORDER BY salary) AS rn,
           COUNT(*) OVER () AS total
    FROM employees
)
SELECT AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((total + 1) / 2.0), CEIL((total + 1) / 2.0));
```

---

### Puzzle 9 — Recursive Org Chart

```sql
-- Find all reports (direct and indirect) under a manager
WITH RECURSIVE org_tree AS (
    -- Base: the manager
    SELECT employee_id, name, manager_id, 1 AS depth
    FROM employees
    WHERE employee_id = 100

    UNION ALL

    -- Recurse: their reports
    SELECT e.employee_id, e.name, e.manager_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.employee_id
)
SELECT * FROM org_tree ORDER BY depth, name;
```

---

### Puzzle 10 — Gaps and Islands

```sql
-- Find gaps in a sequence of IDs
WITH boundaries AS (
    SELECT
        id + 1 AS gap_start,
        LEAD(id) OVER (ORDER BY id) - 1 AS gap_end
    FROM (SELECT DISTINCT id FROM bookings) t
)
SELECT gap_start, gap_end
FROM boundaries
WHERE gap_start <= gap_end
ORDER BY gap_start;
```

---

## 3. System Design — Twitter DB Schema

### Requirements

- Users can post tweets (280 chars)
- Users can follow other users
- Home timeline: see tweets from people you follow
- User can like and retweet
- Search tweets by keyword

### Schema

<svg viewBox="0 0 800 500" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Twitter — Core DB Schema</text>

  <!-- users -->
  <rect x="30" y="50" width="200" height="130" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <rect x="30" y="50" width="200" height="26" rx="8" fill="#1565c0"/>
  <text x="130" y="68" text-anchor="middle" font-size="11" font-weight="bold" fill="white">users</text>
  <text x="40" y="90" font-size="9" fill="#333">user_id        BIGSERIAL PK</text>
  <text x="40" y="106" font-size="9" fill="#333">username       VARCHAR(15) UNIQUE</text>
  <text x="40" y="122" font-size="9" fill="#333">display_name   VARCHAR(50)</text>
  <text x="40" y="138" font-size="9" fill="#333">bio            VARCHAR(160)</text>
  <text x="40" y="154" font-size="9" fill="#333">created_at     TIMESTAMPTZ</text>
  <text x="40" y="170" font-size="9" fill="#333">followers_count INT DEFAULT 0</text>

  <!-- tweets -->
  <rect x="300" y="50" width="220" height="150" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <rect x="300" y="50" width="220" height="26" rx="8" fill="#2e7d32"/>
  <text x="410" y="68" text-anchor="middle" font-size="11" font-weight="bold" fill="white">tweets</text>
  <text x="310" y="90" font-size="9" fill="#333">tweet_id       BIGSERIAL PK</text>
  <text x="310" y="106" font-size="9" fill="#1565c0">user_id        BIGINT FK → users</text>
  <text x="310" y="122" font-size="9" fill="#333">content        VARCHAR(280)</text>
  <text x="310" y="138" font-size="9" fill="#333">created_at     TIMESTAMPTZ</text>
  <text x="310" y="154" font-size="9" fill="#6a1b9a">reply_to_id    BIGINT FK → tweets</text>
  <text x="310" y="170" font-size="9" fill="#6a1b9a">retweet_of_id  BIGINT FK → tweets</text>
  <text x="310" y="186" font-size="9" fill="#333">like_count     INT DEFAULT 0</text>
  <text x="310" y="196" font-size="9" fill="#333">retweet_count  INT DEFAULT 0</text>

  <!-- follows -->
  <rect x="30" y="220" width="200" height="90" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <rect x="30" y="220" width="200" height="26" rx="8" fill="#e65100"/>
  <text x="130" y="238" text-anchor="middle" font-size="11" font-weight="bold" fill="white">follows</text>
  <text x="40" y="260" font-size="9" fill="#1565c0">follower_id    BIGINT FK → users</text>
  <text x="40" y="276" font-size="9" fill="#1565c0">following_id   BIGINT FK → users</text>
  <text x="40" y="292" font-size="9" fill="#333">created_at     TIMESTAMPTZ</text>
  <text x="40" y="304" font-size="9" fill="#888">PK (follower_id, following_id)</text>

  <!-- likes -->
  <rect x="300" y="250" width="220" height="85" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <rect x="300" y="250" width="220" height="26" rx="8" fill="#c62828"/>
  <text x="410" y="268" text-anchor="middle" font-size="11" font-weight="bold" fill="white">likes</text>
  <text x="310" y="290" font-size="9" fill="#1565c0">user_id        BIGINT FK → users</text>
  <text x="310" y="306" font-size="9" fill="#2e7d32">tweet_id       BIGINT FK → tweets</text>
  <text x="310" y="322" font-size="9" fill="#333">created_at     TIMESTAMPTZ</text>
  <text x="310" y="332" font-size="9" fill="#888">PK (user_id, tweet_id)</text>

  <!-- timeline cache -->
  <rect x="570" y="50" width="210" height="110" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <rect x="570" y="50" width="210" height="26" rx="8" fill="#6a1b9a"/>
  <text x="675" y="68" text-anchor="middle" font-size="11" font-weight="bold" fill="white">home_timeline (Redis)</text>
  <text x="580" y="90" font-size="9" fill="#333">Key: timeline:{user_id}</text>
  <text x="580" y="106" font-size="9" fill="#333">Value: Sorted Set of tweet_ids</text>
  <text x="580" y="122" font-size="9" fill="#333">Score: created_at timestamp</text>
  <text x="580" y="142" font-size="9" fill="#888">Fan-out on write for users</text>
  <text x="580" y="154" font-size="9" fill="#888">with &lt; 1M followers</text>

  <!-- hashtags -->
  <rect x="570" y="190" width="210" height="85" rx="8" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <rect x="570" y="190" width="210" height="26" rx="8" fill="#00695c"/>
  <text x="675" y="208" text-anchor="middle" font-size="11" font-weight="bold" fill="white">tweet_hashtags</text>
  <text x="580" y="232" font-size="9" fill="#2e7d32">tweet_id       BIGINT FK → tweets</text>
  <text x="580" y="248" font-size="9" fill="#333">hashtag        VARCHAR(100)</text>
  <text x="580" y="264" font-size="9" fill="#888">Index on hashtag for search</text>
</svg>

### Key Design Decisions

```sql
-- DDL
CREATE TABLE users (
    user_id         BIGSERIAL PRIMARY KEY,
    username        VARCHAR(15) UNIQUE NOT NULL,
    display_name    VARCHAR(50),
    bio             VARCHAR(160),
    followers_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tweets (
    tweet_id        BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(user_id),
    content         VARCHAR(280) NOT NULL,
    reply_to_id     BIGINT REFERENCES tweets(tweet_id),
    retweet_of_id   BIGINT REFERENCES tweets(tweet_id),
    like_count      INT DEFAULT 0,
    retweet_count   INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE follows (
    follower_id  BIGINT REFERENCES users(user_id),
    following_id BIGINT REFERENCES users(user_id),
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, following_id)
);

CREATE TABLE likes (
    user_id   BIGINT REFERENCES users(user_id),
    tweet_id  BIGINT REFERENCES tweets(tweet_id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, tweet_id)
);

-- Indexes
CREATE INDEX idx_tweets_user    ON tweets (user_id, created_at DESC);
CREATE INDEX idx_tweets_reply   ON tweets (reply_to_id) WHERE reply_to_id IS NOT NULL;
CREATE INDEX idx_follows_target ON follows (following_id);
CREATE INDEX idx_likes_tweet    ON likes (tweet_id);

-- Home timeline query (fan-out on read for celebrities)
SELECT t.tweet_id, t.content, t.created_at, u.username
FROM tweets t
JOIN follows f ON t.user_id = f.following_id
JOIN users u   ON t.user_id = u.user_id
WHERE f.follower_id = :current_user_id
ORDER BY t.created_at DESC
LIMIT 50;
```

### Scaling Strategy

| Component | Technology | Why |
|---|---|---|
| Tweets | PostgreSQL, sharded by user_id | Write-heavy, user-scoped queries |
| Timeline cache | Redis Sorted Sets | Sub-ms reads, fan-out on write |
| Search | Elasticsearch | Full-text inverted index |
| Media | S3 + CDN | BLOBs don't belong in the DB |
| Counters | Redis + async flush to PG | High-frequency increments |

---

## 4. System Design — Uber DB Schema

### Requirements

- Riders request rides; drivers accept
- Real-time driver location tracking
- Fare calculation (distance, time, surge)
- Trip history and rating

### Schema

```sql
CREATE TABLE users (
    user_id    BIGSERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(200) UNIQUE NOT NULL,
    phone      VARCHAR(20) UNIQUE NOT NULL,
    role       VARCHAR(10) NOT NULL CHECK (role IN ('rider', 'driver')),
    rating     NUMERIC(3,2) DEFAULT 5.00,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE driver_profiles (
    user_id         BIGINT PRIMARY KEY REFERENCES users(user_id),
    vehicle_make    VARCHAR(50),
    vehicle_model   VARCHAR(50),
    license_plate   VARCHAR(20),
    vehicle_type    VARCHAR(20),  -- 'economy', 'premium', 'xl'
    is_active       BOOLEAN DEFAULT FALSE,
    current_lat     DOUBLE PRECISION,
    current_lng     DOUBLE PRECISION,
    last_location_at TIMESTAMPTZ
);

CREATE TABLE trips (
    trip_id          BIGSERIAL PRIMARY KEY,
    rider_id         BIGINT NOT NULL REFERENCES users(user_id),
    driver_id        BIGINT REFERENCES users(user_id),
    status           VARCHAR(20) DEFAULT 'requested',
    -- requested → accepted → in_progress → completed → cancelled
    pickup_lat       DOUBLE PRECISION NOT NULL,
    pickup_lng       DOUBLE PRECISION NOT NULL,
    dropoff_lat      DOUBLE PRECISION,
    dropoff_lng      DOUBLE PRECISION,
    pickup_address   TEXT,
    dropoff_address  TEXT,
    requested_at     TIMESTAMPTZ DEFAULT NOW(),
    accepted_at      TIMESTAMPTZ,
    started_at       TIMESTAMPTZ,
    completed_at     TIMESTAMPTZ,
    distance_km      NUMERIC(8,2),
    duration_minutes NUMERIC(8,2),
    surge_multiplier NUMERIC(3,2) DEFAULT 1.00,
    fare_amount      NUMERIC(10,2),
    payment_method   VARCHAR(20)
);

CREATE TABLE trip_locations (
    location_id BIGSERIAL PRIMARY KEY,
    trip_id     BIGINT NOT NULL REFERENCES trips(trip_id),
    lat         DOUBLE PRECISION NOT NULL,
    lng         DOUBLE PRECISION NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (recorded_at);

CREATE TABLE ratings (
    rating_id   BIGSERIAL PRIMARY KEY,
    trip_id     BIGINT UNIQUE NOT NULL REFERENCES trips(trip_id),
    from_user   BIGINT NOT NULL REFERENCES users(user_id),
    to_user     BIGINT NOT NULL REFERENCES users(user_id),
    score       SMALLINT CHECK (score BETWEEN 1 AND 5),
    comment     TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Geospatial index for finding nearby drivers
CREATE EXTENSION postgis;
CREATE INDEX idx_driver_location ON driver_profiles
    USING GIST (ST_SetSRID(ST_MakePoint(current_lng, current_lat), 4326));

-- Find nearest available drivers
SELECT
    u.user_id,
    u.name,
    dp.vehicle_type,
    ST_Distance(
        ST_SetSRID(ST_MakePoint(dp.current_lng, dp.current_lat), 4326)::geography,
        ST_SetSRID(ST_MakePoint(:pickup_lng, :pickup_lat), 4326)::geography
    ) AS distance_meters
FROM driver_profiles dp
JOIN users u ON dp.user_id = u.user_id
WHERE dp.is_active = TRUE
  AND ST_DWithin(
        ST_SetSRID(ST_MakePoint(dp.current_lng, dp.current_lat), 4326)::geography,
        ST_SetSRID(ST_MakePoint(:pickup_lng, :pickup_lat), 4326)::geography,
        5000  -- within 5 km
      )
ORDER BY distance_meters
LIMIT 10;
```

### Scaling Strategy

| Component | Technology | Why |
|---|---|---|
| Trips | PostgreSQL, partitioned by date | Time-series pattern, archivable |
| Driver locations | Redis GEO + PostGIS | Real-time geo queries, sub-ms |
| Trip locations | TimescaleDB or partitioned PG | High-frequency GPS pings |
| Surge pricing | Redis (pre-computed grids) | Geohash-based grid cells |
| Analytics | Snowflake / BigQuery | Trip history, driver performance |

---

## 5. System Design — Amazon DB Schema

### Requirements

- Product catalog with categories
- Shopping cart
- Order processing with payments
- Inventory tracking
- Product reviews and ratings

### Schema

```sql
-- Product catalog
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    parent_id     INT REFERENCES categories(category_id),
    path          LTREE  -- PostgreSQL ltree for hierarchical queries
);

CREATE TABLE products (
    product_id    BIGSERIAL PRIMARY KEY,
    sku           VARCHAR(50) UNIQUE NOT NULL,
    name          VARCHAR(300) NOT NULL,
    description   TEXT,
    category_id   INT REFERENCES categories(category_id),
    brand         VARCHAR(100),
    price         NUMERIC(12,2) NOT NULL,
    weight_kg     NUMERIC(8,3),
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE product_images (
    image_id    BIGSERIAL PRIMARY KEY,
    product_id  BIGINT REFERENCES products(product_id),
    url         TEXT NOT NULL,
    sort_order  SMALLINT DEFAULT 0
);

-- Inventory (per warehouse)
CREATE TABLE warehouses (
    warehouse_id  SERIAL PRIMARY KEY,
    name          VARCHAR(100),
    city          VARCHAR(100),
    region        VARCHAR(50)
);

CREATE TABLE inventory (
    product_id    BIGINT REFERENCES products(product_id),
    warehouse_id  INT REFERENCES warehouses(warehouse_id),
    quantity      INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    reserved      INT NOT NULL DEFAULT 0,
    updated_at    TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (product_id, warehouse_id)
);

-- Shopping cart (session-based, could also use Redis)
CREATE TABLE cart_items (
    cart_id       UUID DEFAULT gen_random_uuid(),
    user_id       BIGINT REFERENCES users(user_id),
    product_id    BIGINT REFERENCES products(product_id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    added_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, product_id)
);

-- Orders
CREATE TABLE orders (
    order_id        BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(user_id),
    status          VARCHAR(20) DEFAULT 'pending',
    -- pending → paid → processing → shipped → delivered → returned
    shipping_address TEXT NOT NULL,
    subtotal        NUMERIC(12,2),
    tax             NUMERIC(10,2),
    shipping_cost   NUMERIC(10,2),
    total           NUMERIC(12,2),
    payment_method  VARCHAR(30),
    payment_txn_id  VARCHAR(100),
    ordered_at      TIMESTAMPTZ DEFAULT NOW(),
    shipped_at      TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ
) PARTITION BY RANGE (ordered_at);

CREATE TABLE order_items (
    order_item_id  BIGSERIAL PRIMARY KEY,
    order_id       BIGINT NOT NULL REFERENCES orders(order_id),
    product_id     BIGINT NOT NULL REFERENCES products(product_id),
    quantity       INT NOT NULL,
    unit_price     NUMERIC(12,2) NOT NULL,
    total_price    NUMERIC(12,2) NOT NULL
);

-- Reviews
CREATE TABLE reviews (
    review_id   BIGSERIAL PRIMARY KEY,
    product_id  BIGINT NOT NULL REFERENCES products(product_id),
    user_id     BIGINT NOT NULL REFERENCES users(user_id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title       VARCHAR(200),
    body        TEXT,
    helpful_count INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (product_id, user_id)
);

-- Key indexes
CREATE INDEX idx_products_category ON products (category_id, is_active);
CREATE INDEX idx_products_search   ON products USING GIN (
    to_tsvector('english', name || ' ' || COALESCE(description, ''))
);
CREATE INDEX idx_orders_user       ON orders (user_id, ordered_at DESC);
CREATE INDEX idx_reviews_product   ON reviews (product_id, rating);
CREATE INDEX idx_inventory_qty     ON inventory (product_id)
    WHERE quantity - reserved > 0;  -- partial index: only in-stock

-- Place an order (transaction)
BEGIN;

-- Reserve inventory
UPDATE inventory
SET reserved = reserved + 1
WHERE product_id = :pid AND warehouse_id = :wid
  AND quantity - reserved >= 1;

-- Create order
INSERT INTO orders (user_id, shipping_address, subtotal, tax, shipping_cost, total)
VALUES (:uid, :addr, :sub, :tax, :ship, :total)
RETURNING order_id;

-- Create order items
INSERT INTO order_items (order_id, product_id, quantity, unit_price, total_price)
VALUES (:oid, :pid, 1, :price, :price);

-- Clear cart
DELETE FROM cart_items WHERE user_id = :uid;

COMMIT;
```

### Scaling Strategy

| Component | Technology | Why |
|---|---|---|
| Product catalog | PostgreSQL + Elasticsearch | SQL for ACID; ES for full-text search |
| Cart | Redis Hash | Session-speed, TTL expiry |
| Orders | PostgreSQL, partitioned by month | Transaction integrity, archivable |
| Inventory | PostgreSQL with row-level locks | `SELECT ... FOR UPDATE` prevents overselling |
| Reviews | PostgreSQL + read replicas | Write-once, read-heavy |
| Images | S3 + CloudFront CDN | BLOB storage + global delivery |
| Recommendations | Neo4j or collaborative filtering | "Customers who bought X also bought Y" |

---

## 6. Quick-Reference Cheat Sheet

<svg viewBox="0 0 820 1650" xmlns="http://www.w3.org/2000/svg" style="max-width:820px; font-family:Arial,sans-serif;">
  <text x="410" y="28" text-anchor="middle" font-size="18" font-weight="bold" fill="#1a1a2e">DBMS — Complete Cheat Sheet</text>

  <!-- Section 1: SQL Fundamentals -->
  <rect x="15" y="45" width="390" height="240" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="25" y="68" font-size="13" font-weight="bold" fill="#1565c0">SQL Fundamentals</text>
  <text x="25" y="88" font-size="9" fill="#333">DDL: CREATE, ALTER, DROP, TRUNCATE</text>
  <text x="25" y="103" font-size="9" fill="#333">DML: INSERT, UPDATE, DELETE, MERGE</text>
  <text x="25" y="118" font-size="9" fill="#333">DQL: SELECT...FROM...WHERE...GROUP BY...HAVING...ORDER BY</text>
  <text x="25" y="138" font-size="9" fill="#555">Execution order: FROM → WHERE → GROUP BY → HAVING</text>
  <text x="25" y="153" font-size="9" fill="#555">  → SELECT → DISTINCT → ORDER BY → LIMIT</text>
  <text x="25" y="173" font-size="9" fill="#333">JOINs: INNER | LEFT | RIGHT | FULL | CROSS | SELF</text>
  <text x="25" y="188" font-size="9" fill="#333">Set ops: UNION [ALL] | INTERSECT | EXCEPT</text>
  <text x="25" y="208" font-size="9" fill="#333">Subqueries: scalar, row, table, correlated</text>
  <text x="25" y="223" font-size="9" fill="#333">CTEs: WITH name AS (...) SELECT ...</text>
  <text x="25" y="238" font-size="9" fill="#333">Recursive: WITH RECURSIVE ... UNION ALL ...</text>
  <text x="25" y="258" font-size="9" fill="#333">Window: func() OVER (PARTITION BY ... ORDER BY ...)</text>
  <text x="25" y="273" font-size="9" fill="#333">Agg: ROLLUP | CUBE | GROUPING SETS</text>

  <!-- Section 2: Keys & Constraints -->
  <rect x="415" y="45" width="390" height="240" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="425" y="68" font-size="13" font-weight="bold" fill="#2e7d32">Keys & Constraints</text>
  <text x="425" y="88" font-size="9" fill="#333">PRIMARY KEY = NOT NULL + UNIQUE</text>
  <text x="425" y="103" font-size="9" fill="#333">FOREIGN KEY → referential integrity</text>
  <text x="425" y="118" font-size="9" fill="#333">UNIQUE, NOT NULL, CHECK, DEFAULT</text>
  <text x="425" y="138" font-size="9" fill="#555">CASCADE options: CASCADE | SET NULL | RESTRICT</text>
  <text x="425" y="158" font-size="9" font-weight="bold" fill="#333">Normalization:</text>
  <text x="425" y="173" font-size="9" fill="#333">1NF: atomic values, no repeating groups</text>
  <text x="425" y="188" font-size="9" fill="#333">2NF: no partial dependencies (composite keys)</text>
  <text x="425" y="203" font-size="9" fill="#333">3NF: no transitive dependencies</text>
  <text x="425" y="218" font-size="9" fill="#333">BCNF: every determinant is a candidate key</text>
  <text x="425" y="238" font-size="9" fill="#555">Surrogate keys (INT, SERIAL) for warehouses</text>
  <text x="425" y="253" font-size="9" fill="#555">Natural keys for operational lookups</text>
  <text x="425" y="273" font-size="9" fill="#555">Denormalize only with measured justification</text>

  <!-- Section 3: Indexes -->
  <rect x="15" y="295" width="390" height="195" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="25" y="318" font-size="13" font-weight="bold" fill="#e65100">Indexes & Optimization</text>
  <text x="25" y="338" font-size="9" fill="#333">B-Tree: equality + range (DEFAULT)</text>
  <text x="25" y="353" font-size="9" fill="#333">Hash: equality only, smaller</text>
  <text x="25" y="368" font-size="9" fill="#333">GIN: full-text, JSONB, arrays (inverted)</text>
  <text x="25" y="383" font-size="9" fill="#333">GiST: geometry, ranges, nearest-neighbor</text>
  <text x="25" y="398" font-size="9" fill="#333">BRIN: large sorted tables (min/max per range)</text>
  <text x="25" y="418" font-size="9" fill="#555">Composite: leftmost prefix rule</text>
  <text x="25" y="433" font-size="9" fill="#555">Covering: INCLUDE cols for index-only scan</text>
  <text x="25" y="448" font-size="9" fill="#555">Partial: WHERE clause filters index rows</text>
  <text x="25" y="463" font-size="9" fill="#555">EXPLAIN ANALYZE BUFFERS — always verify plans</text>
  <text x="25" y="480" font-size="9" fill="#c62828">Seq scan beats index at > 5-10% selectivity</text>

  <!-- Section 4: Transactions -->
  <rect x="415" y="295" width="390" height="195" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="425" y="318" font-size="13" font-weight="bold" fill="#c62828">Transactions & Concurrency</text>
  <text x="425" y="338" font-size="9" fill="#333">ACID: Atomicity, Consistency, Isolation, Durability</text>
  <text x="425" y="358" font-size="9" fill="#333">Isolation: READ UNCOMMITTED → READ COMMITTED</text>
  <text x="425" y="373" font-size="9" fill="#333">  → REPEATABLE READ → SERIALIZABLE</text>
  <text x="425" y="393" font-size="9" fill="#333">MVCC: multiple row versions, readers ≠ block writers</text>
  <text x="425" y="408" font-size="9" fill="#333">2PL: growing phase (lock) → shrinking phase (unlock)</text>
  <text x="425" y="428" font-size="9" fill="#555">Deadlock: cycle in wait-for graph → victim rollback</text>
  <text x="425" y="443" font-size="9" fill="#555">Pessimistic: SELECT ... FOR UPDATE</text>
  <text x="425" y="458" font-size="9" fill="#555">Optimistic: version column + retry on conflict</text>
  <text x="425" y="478" font-size="9" fill="#555">WAL: log before data; ARIES: Analysis → Redo → Undo</text>

  <!-- Section 5: Window Functions -->
  <rect x="15" y="500" width="390" height="165" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="25" y="523" font-size="13" font-weight="bold" fill="#6a1b9a">Window Functions</text>
  <text x="25" y="543" font-size="9" fill="#333">Ranking: ROW_NUMBER, RANK, DENSE_RANK, NTILE</text>
  <text x="25" y="558" font-size="9" fill="#333">Offset: LAG, LEAD, FIRST_VALUE, LAST_VALUE</text>
  <text x="25" y="573" font-size="9" fill="#333">Aggregate: SUM, AVG, COUNT as window functions</text>
  <text x="25" y="593" font-size="9" fill="#555">Frame: ROWS | RANGE | GROUPS</text>
  <text x="25" y="608" font-size="9" fill="#555">  BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW</text>
  <text x="25" y="628" font-size="9" fill="#555">Running total: SUM() OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)</text>
  <text x="25" y="648" font-size="9" fill="#555">Top-N per group: ROW_NUMBER() PARTITION BY ... = 1</text>

  <!-- Section 6: Security -->
  <rect x="415" y="500" width="390" height="165" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="425" y="523" font-size="13" font-weight="bold" fill="#00695c">Security & Access Control</text>
  <text x="425" y="543" font-size="9" fill="#333">GRANT privilege ON object TO role;</text>
  <text x="425" y="558" font-size="9" fill="#333">REVOKE privilege ON object FROM role;</text>
  <text x="425" y="573" font-size="9" fill="#333">RBAC: CREATE ROLE → GRANT role TO user</text>
  <text x="425" y="593" font-size="9" fill="#333">RLS: ALTER TABLE ENABLE ROW LEVEL SECURITY;</text>
  <text x="425" y="608" font-size="9" fill="#555">SQL Injection prevention: parameterized queries ALWAYS</text>
  <text x="425" y="628" font-size="9" fill="#555">Encryption: TDE (at rest), TLS/SSL (in transit)</text>
  <text x="425" y="648" font-size="9" fill="#555">Audit: pgAudit (PG), SQL Server Audit, MySQL audit log</text>

  <!-- Section 7: Distributed -->
  <rect x="15" y="675" width="390" height="195" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="25" y="698" font-size="13" font-weight="bold" fill="#f57f17">Distributed Systems</text>
  <text x="25" y="718" font-size="9" fill="#333">CAP: Consistency + Availability + Partition tolerance</text>
  <text x="25" y="733" font-size="9" fill="#333">  → pick CP or AP (partitions are inevitable)</text>
  <text x="25" y="748" font-size="9" fill="#333">PACELC: if Partition → A/C; Else → Latency/Consistency</text>
  <text x="25" y="768" font-size="9" fill="#333">Replication: leader-follower | multi-leader | leaderless</text>
  <text x="25" y="783" font-size="9" fill="#333">Quorum: W + R > N for consistency</text>
  <text x="25" y="803" font-size="9" fill="#555">Consensus: Raft (leader election + log replication)</text>
  <text x="25" y="818" font-size="9" fill="#555">Distributed txns: 2PC, 3PC, Saga pattern</text>
  <text x="25" y="838" font-size="9" fill="#555">Consistent hashing: add/remove nodes → move ~1/N keys</text>
  <text x="25" y="858" font-size="9" fill="#555">CRDTs: G-Counter, PN-Counter, OR-Set (conflict-free)</text>

  <!-- Section 8: NoSQL -->
  <rect x="415" y="675" width="390" height="195" rx="10" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="425" y="698" font-size="13" font-weight="bold" fill="#3949ab">NoSQL Databases</text>
  <text x="425" y="718" font-size="9" fill="#333">Document: MongoDB (JSON/BSON, flexible schema)</text>
  <text x="425" y="733" font-size="9" fill="#333">Key-Value: Redis (in-memory, sub-ms, data structures)</text>
  <text x="425" y="748" font-size="9" fill="#333">Column-Family: Cassandra (wide rows, write-heavy)</text>
  <text x="425" y="763" font-size="9" fill="#333">Graph: Neo4j (nodes, edges, Cypher traversals)</text>
  <text x="425" y="778" font-size="9" fill="#333">Time-Series: InfluxDB, TimescaleDB</text>
  <text x="425" y="798" font-size="9" fill="#555">ACID vs BASE: strong consistency vs eventual</text>
  <text x="425" y="813" font-size="9" fill="#555">Vector clocks: detect concurrent writes</text>
  <text x="425" y="833" font-size="9" fill="#555">Polyglot persistence: right DB for each workload</text>
  <text x="425" y="853" font-size="9" fill="#555">SQL vs NoSQL: not either/or — use both where appropriate</text>

  <!-- Section 9: Data Warehousing -->
  <rect x="15" y="880" width="390" height="195" rx="10" fill="#fbe9e7" stroke="#bf360c" stroke-width="2"/>
  <text x="25" y="903" font-size="13" font-weight="bold" fill="#bf360c">Data Warehousing & OLAP</text>
  <text x="25" y="923" font-size="9" fill="#333">OLTP: run the business | OLAP: analyse the business</text>
  <text x="25" y="938" font-size="9" fill="#333">Star schema: fact table + denormalized dimensions</text>
  <text x="25" y="953" font-size="9" fill="#333">Snowflake: normalized dimensions (sub-tables)</text>
  <text x="25" y="968" font-size="9" fill="#333">Galaxy: multiple facts sharing conformed dimensions</text>
  <text x="25" y="988" font-size="9" fill="#555">Fact types: transaction | periodic | accumulating | factless</text>
  <text x="25" y="1003" font-size="9" fill="#555">SCD Type 2: new row + effective dates (full history)</text>
  <text x="25" y="1023" font-size="9" fill="#555">ETL: transform outside | ELT: transform inside (dbt)</text>
  <text x="25" y="1038" font-size="9" fill="#555">Columnar: 5-10x compression, vectorized scan</text>
  <text x="25" y="1053" font-size="9" fill="#555">ROLLUP: hierarchy | CUBE: all combos | GROUPING SETS: pick</text>
  <text x="25" y="1068" font-size="9" fill="#555">Additive | Semi-additive | Non-additive measures</text>

  <!-- Section 10: Partitioning & Sharding -->
  <rect x="415" y="880" width="390" height="195" rx="10" fill="#e0f7fa" stroke="#006064" stroke-width="2"/>
  <text x="425" y="903" font-size="13" font-weight="bold" fill="#006064">Partitioning & Sharding</text>
  <text x="425" y="923" font-size="9" fill="#333">Range: date ranges (time-series, logs)</text>
  <text x="425" y="938" font-size="9" fill="#333">List: discrete values (region, status)</text>
  <text x="425" y="953" font-size="9" fill="#333">Hash: even distribution (user_id % N)</text>
  <text x="425" y="968" font-size="9" fill="#333">Composite: range + hash for multi-dimensional</text>
  <text x="425" y="988" font-size="9" fill="#555">Pruning: WHERE on partition key → skip irrelevant</text>
  <text x="425" y="1003" font-size="9" fill="#c62828">Never wrap partition key in functions!</text>
  <text x="425" y="1023" font-size="9" fill="#555">DROP PARTITION is O(1) — use instead of DELETE</text>
  <text x="425" y="1043" font-size="9" fill="#555">Sharding: distribute across nodes (horizontal scale)</text>
  <text x="425" y="1058" font-size="9" fill="#555">Shard key: high cardinality, query-aligned, stable</text>
  <text x="425" y="1073" font-size="9" fill="#555">Scale ladder: indexes → replicas → partition → shard</text>

  <!-- Section 11: Storage Internals -->
  <rect x="15" y="1085" width="390" height="195" rx="10" fill="#f5f5f5" stroke="#616161" stroke-width="2"/>
  <text x="25" y="1108" font-size="13" font-weight="bold" fill="#616161">Storage Internals</text>
  <text x="25" y="1128" font-size="9" fill="#333">Page: 8 KB (PG/SQL Server) or 16 KB (InnoDB)</text>
  <text x="25" y="1143" font-size="9" fill="#333">Slotted page: header → line ptrs → free → tuples</text>
  <text x="25" y="1158" font-size="9" fill="#333">Heap: unordered, O(1) insert, O(N) scan</text>
  <text x="25" y="1173" font-size="9" fill="#333">Clustered: sorted by key, O(log N + K) range scan</text>
  <text x="25" y="1188" font-size="9" fill="#333">Hashed: O(1) exact match, no range support</text>
  <text x="25" y="1208" font-size="9" fill="#555">Buffer pool: pages cached in RAM (target > 99% hit)</text>
  <text x="25" y="1223" font-size="9" fill="#555">Clock-sweep: usage count 0-5, evict at 0</text>
  <text x="25" y="1238" font-size="9" fill="#555">Sequential I/O >> random I/O (6-200x faster)</text>
  <text x="25" y="1253" font-size="9" fill="#555">TOAST: compress/chunk values > 2 KB</text>
  <text x="25" y="1268" font-size="9" fill="#555">InnoDB: PK is clustered B+Tree; secondary → PK lookup</text>

  <!-- Section 12: Programmability -->
  <rect x="415" y="1085" width="390" height="195" rx="10" fill="#efebe9" stroke="#795548" stroke-width="2"/>
  <text x="425" y="1108" font-size="13" font-weight="bold" fill="#795548">Programmability</text>
  <text x="425" y="1128" font-size="9" fill="#333">Stored procedure: server-side logic, txn control</text>
  <text x="425" y="1143" font-size="9" fill="#333">Function: returns value, usable in SELECT</text>
  <text x="425" y="1158" font-size="9" fill="#333">Trigger: BEFORE/AFTER INSERT/UPDATE/DELETE</text>
  <text x="425" y="1173" font-size="9" fill="#333">Cursor: row-by-row processing (avoid if possible)</text>
  <text x="425" y="1193" font-size="9" fill="#555">Error handling: EXCEPTION (PG), TRY/CATCH (MSSQL)</text>
  <text x="425" y="1208" font-size="9" fill="#555">Views: virtual table = named query</text>
  <text x="425" y="1223" font-size="9" fill="#555">Materialized views: cached query result, refresh needed</text>
  <text x="425" y="1243" font-size="9" fill="#c62828">Prefer set-based SQL over cursors — always</text>
  <text x="425" y="1258" font-size="9" fill="#c62828">Parameterized queries — never concatenate user input</text>

  <!-- Bottom: Decision Quick-Ref -->
  <rect x="15" y="1295" width="790" height="340" rx="12" fill="#fafafa" stroke="#333" stroke-width="2"/>
  <text x="410" y="1320" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a1a2e">Quick Decision Reference</text>

  <text x="30" y="1345" font-size="10" font-weight="bold" fill="#1565c0">Need ACID transactions?</text>
  <text x="280" y="1345" font-size="10" fill="#555">→ PostgreSQL, MySQL, SQL Server</text>

  <text x="30" y="1365" font-size="10" font-weight="bold" fill="#1565c0">Sub-ms caching / sessions?</text>
  <text x="280" y="1365" font-size="10" fill="#555">→ Redis (key-value, in-memory)</text>

  <text x="30" y="1385" font-size="10" font-weight="bold" fill="#1565c0">Flexible JSON documents?</text>
  <text x="280" y="1385" font-size="10" fill="#555">→ MongoDB (or PostgreSQL JSONB)</text>

  <text x="30" y="1405" font-size="10" font-weight="bold" fill="#1565c0">Massive write throughput?</text>
  <text x="280" y="1405" font-size="10" fill="#555">→ Cassandra, ScyllaDB (AP, tunable consistency)</text>

  <text x="30" y="1425" font-size="10" font-weight="bold" fill="#1565c0">Graph traversals?</text>
  <text x="280" y="1425" font-size="10" fill="#555">→ Neo4j (Cypher), Amazon Neptune</text>

  <text x="30" y="1445" font-size="10" font-weight="bold" fill="#1565c0">Full-text search?</text>
  <text x="280" y="1445" font-size="10" fill="#555">→ Elasticsearch (or PostgreSQL GIN + tsvector)</text>

  <text x="30" y="1465" font-size="10" font-weight="bold" fill="#1565c0">Data warehouse / analytics?</text>
  <text x="280" y="1465" font-size="10" fill="#555">→ Snowflake, BigQuery, Redshift (columnar)</text>

  <text x="30" y="1485" font-size="10" font-weight="bold" fill="#1565c0">Time-series / IoT?</text>
  <text x="280" y="1485" font-size="10" fill="#555">→ TimescaleDB, InfluxDB, QuestDB</text>

  <text x="30" y="1505" font-size="10" font-weight="bold" fill="#1565c0">Embedded analytics?</text>
  <text x="280" y="1505" font-size="10" fill="#555">→ DuckDB ("SQLite for OLAP")</text>

  <text x="30" y="1525" font-size="10" font-weight="bold" fill="#1565c0">Global distributed SQL?</text>
  <text x="280" y="1525" font-size="10" fill="#555">→ CockroachDB, TiDB, Spanner (NewSQL)</text>

  <rect x="30" y="1545" width="760" height="75" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="410" y="1568" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">The Golden Rules</text>
  <text x="50" y="1588" font-size="10" fill="#333">1. Measure before optimizing  2. Normalize first, denormalize with evidence  3. Index WHERE/JOIN/ORDER BY columns</text>
  <text x="50" y="1608" font-size="10" fill="#333">4. Use EXPLAIN always  5. Parameterize every query  6. Partition before sharding  7. Right tool for the job</text>
</svg>

---

## 7. Download

<a href="23_interview_cheatsheet.md" download="23_interview_cheatsheet.md" style="display:inline-block;padding:14px 28px;background:#1a1a2e;color:white;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;cursor:pointer;">Download 23_interview_cheatsheet.md</a>
