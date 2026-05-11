# DBMS (Database Management Systems) — Complete Deep Dive

## 1. ACID Properties

```
Atomicity:    All or nothing. Transaction either fully completes or fully rolls back.
              Example: Transfer ₹500: debit A AND credit B — both or neither.

Consistency:  DB moves from one valid state to another. All constraints satisfied.
              Example: Balance can never be negative (if constraint exists).

Isolation:    Concurrent transactions don't interfere with each other.
              Each sees a consistent snapshot as if running alone.

Durability:   Once committed, data survives crashes (written to disk/WAL).
              Even if power fails 1ms after COMMIT → data is safe.
```

---

## 2. Normalization

```
Purpose: Eliminate redundancy and anomalies (insert, update, delete anomalies)

1NF: No repeating groups. Each cell has atomic (single) value.
     ❌ name: "Prince, Patel"  →  ✅ first_name: "Prince", last_name: "Patel"

2NF: 1NF + No partial dependency (non-key depends on FULL primary key).
     If PK = (student_id, course_id):
     ❌ student_name depends only on student_id (partial) → move to separate table

3NF: 2NF + No transitive dependency (non-key depends only on PK, not other non-keys).
     ❌ city depends on zip_code, zip_code depends on student_id
     → Move city to zip_code table

BCNF: Every determinant is a candidate key.
      Stricter than 3NF. Used when multiple candidate keys exist.

When to DENORMALIZE:
  - Read-heavy workloads (avoid JOINs)
  - Analytics/reporting queries
  - Caching layer (pre-computed views)
  - When write performance is acceptable trade-off
```

---

## 3. Indexing

```
Without index: Sequential scan O(n) — check every row
With index: B-Tree lookup O(log n) — jump directly

B-Tree Index (default in PostgreSQL, MySQL):
─────────────────────────────────────────────
          [30 | 60]                    ← Root
         /    |    \
    [10|20] [40|50] [70|80|90]        ← Internal
    / | \   / | \    / | | \
   leaves with actual data pointers    ← Leaf nodes

Properties:
- Balanced: All leaves at same depth
- Ordered: Supports range queries (WHERE age > 25)
- Self-balancing: Insertions/deletions maintain balance

Types of Indexes:
─────────────────
B-Tree:          Default, range queries, equality
Hash:            Only equality (=), O(1) but no range
Composite:       (col1, col2) — leftmost prefix rule!
Covering:        Index includes all needed columns (no table lookup)
Partial:         Index only rows matching WHERE condition
Full-text:       For LIKE '%search%' (inverted index)
GiST/GIN:       For JSON, arrays, geo data (PostgreSQL)

Composite Index Rule:
  INDEX(a, b, c) supports:
  ✅ WHERE a = 1
  ✅ WHERE a = 1 AND b = 2
  ✅ WHERE a = 1 AND b = 2 AND c = 3
  ❌ WHERE b = 2 (leftmost prefix not satisfied)
  ❌ WHERE a = 1 AND c = 3 (b is skipped)
```

---

## 4. Transactions & Isolation Levels

```
Concurrency problems:
─────────────────────
Dirty Read:       Read uncommitted data from another transaction
Non-repeatable Read: Same query returns different results within one transaction
Phantom Read:     New rows appear matching a WHERE clause during transaction

Isolation Levels (weakest → strongest):
────────────────────────────────────────
┌──────────────────┬────────────┬─────────────────┬──────────────┐
│ Level            │ Dirty Read │ Non-repeatable   │ Phantom Read │
├──────────────────┼────────────┼─────────────────┼──────────────┤
│ Read Uncommitted │ Possible   │ Possible         │ Possible     │
│ Read Committed   │ Prevented  │ Possible         │ Possible     │
│ Repeatable Read  │ Prevented  │ Prevented        │ Possible     │
│ Serializable     │ Prevented  │ Prevented        │ Prevented    │
└──────────────────┴────────────┴─────────────────┴──────────────┘

Default: PostgreSQL = Read Committed, MySQL = Repeatable Read

Implementation:
  MVCC (Multi-Version Concurrency Control):
    Each transaction sees a snapshot of the DB
    Writers don't block readers, readers don't block writers
    Used by: PostgreSQL, MySQL InnoDB, Oracle
```

---

## 5. SQL Deep Dive

### Joins
```
INNER JOIN:     Only matching rows from both tables
LEFT JOIN:      All rows from left + matching from right (NULL if no match)
RIGHT JOIN:     All rows from right + matching from left
FULL OUTER:     All rows from both (NULL where no match)
CROSS JOIN:     Cartesian product (every row × every row)
SELF JOIN:      Table joined with itself (hierarchies, comparisons)

  Table A        Table B
  ┌───┐          ┌───┐
  │ 1 │          │ 2 │
  │ 2 │          │ 3 │
  │ 3 │          │ 4 │
  └───┘          └───┘

  INNER: {2, 3}
  LEFT:  {1(null), 2, 3}
  RIGHT: {2, 3, 4(null)}
  FULL:  {1(null), 2, 3, 4(null)}
```

### Window Functions (Asked frequently!)
```sql
-- Rank within partition
SELECT name, department, salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num
FROM employees;

-- Running total
SELECT date, amount,
  SUM(amount) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) as running_total
FROM transactions;

-- Moving average
SELECT date, price,
  AVG(price) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg_7d
FROM stock_prices;

-- Compare with previous row
SELECT date, revenue,
  LAG(revenue, 1) OVER (ORDER BY date) as prev_day,
  revenue - LAG(revenue, 1) OVER (ORDER BY date) as daily_change
FROM daily_revenue;
```

### Query Optimization
```
EXPLAIN ANALYZE tells you:
  - Seq Scan vs Index Scan (index = good)
  - Estimated vs actual rows
  - Sort method (in-memory vs disk)
  - Join method (nested loop, hash join, merge join)

Optimization tips:
1. Add indexes on WHERE, JOIN, ORDER BY columns
2. Avoid SELECT * — only select needed columns
3. Use LIMIT for pagination
4. Avoid functions on indexed columns: WHERE YEAR(date) = 2024
   → Better: WHERE date >= '2024-01-01' AND date < '2025-01-01'
5. Use EXISTS instead of IN for subqueries (often faster)
6. Denormalize for read-heavy queries (materialized views)
```

---

## 6. SQL vs NoSQL

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│                    │ SQL (Relational)     │ NoSQL                │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Schema             │ Fixed, predefined    │ Flexible/dynamic     │
│ Scaling            │ Vertical (mostly)    │ Horizontal           │
│ Transactions       │ Full ACID            │ Limited/eventual     │
│ Queries            │ Complex JOINs, SQL   │ Simple key-value     │
│ Consistency        │ Strong               │ Eventual (mostly)    │
│ Best for           │ Structured data,     │ Unstructured, high   │
│                    │ complex relations    │ scale, flexible      │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Examples           │ PostgreSQL, MySQL    │ MongoDB, Cassandra,  │
│                    │ Oracle, SQL Server   │ DynamoDB, Redis      │
└────────────────────┴──────────────────────┴──────────────────────┘

Choose SQL when: ACID needed, complex queries, data integrity critical
Choose NoSQL when: High scale, flexible schema, simple access patterns
```

---

## 7. Database Internals

### How a Query Executes
```
SQL Query → Parser → Query Optimizer → Execution Engine → Storage Engine → Result

Query Optimizer chooses:
  - Which indexes to use
  - Join order (most selective first)
  - Join algorithm (nested loop, hash, merge)
  - Whether to use sequential or index scan

Storage Engine (e.g., InnoDB):
  - Buffer Pool: Cache pages in memory
  - WAL (Write-Ahead Log): Write log before data (crash recovery)
  - Page: Basic unit of I/O (8KB in PostgreSQL, 16KB in MySQL)
  - MVCC: Multiple versions for concurrent access
```

### Write Path
```
1. Write to WAL (sequential, fast) → DURABILITY guaranteed
2. Update buffer pool (in-memory page)
3. Background: Flush dirty pages to disk (checkpoint)

If crash after step 1: Replay WAL on restart → consistent state
```

---

## 8. Interview Questions (Top 20)

1. Explain ACID properties with examples.
2. What are isolation levels? When to use each?
3. Explain indexing. B-Tree vs Hash index.
4. What is normalization? Explain up to 3NF with examples.
5. When would you denormalize a database?
6. Explain MVCC (Multi-Version Concurrency Control).
7. SQL vs NoSQL — how to choose?
8. What is a deadlock in databases? How to handle?
9. Explain query optimization — how would you speed up a slow query?
10. What are window functions? Give examples.
11. Explain sharding vs partitioning.
12. What is a materialized view? When to use?
13. Explain CAP theorem.
14. How does a database handle concurrent writes?
15. What is connection pooling? Why is it important?
16. Explain different types of JOINs with examples.
17. What is a clustered vs non-clustered index?
18. How does PostgreSQL's VACUUM work?
19. Explain Write-Ahead Logging (WAL).
20. Design a schema for [Twitter / E-commerce / Chat] — normalize it.
