# DBMS — Database Concepts That Matter for Interviews

## Why DBMS Knowledge Matters
Every system design question involves a database decision. You need to understand WHY you're choosing PostgreSQL over MongoDB, or when to add an index.

---

## ACID Properties

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All or nothing — transaction fully completes or fully rolls back | Bank transfer: debit AND credit both happen or neither |
| **Consistency** | DB moves from one valid state to another | Balance can't go negative if CHECK constraint exists |
| **Isolation** | Concurrent transactions don't interfere | Two users booking last seat — only one succeeds |
| **Durability** | Once committed, data survives crashes | Write-ahead log ensures recovery |

---

## Indexing (Most Important Topic)

### What an Index Is
A separate data structure (usually B+ tree) that makes lookups fast. Like a book's index — jump to the page instead of reading every page.

### B+ Tree Index (Default in PostgreSQL, MySQL)
```
Properties:
- Balanced tree (all leaves at same depth)
- Internal nodes store keys (routing)
- Leaf nodes store keys + pointers to rows
- Leaves linked (easy range scans)

Performance:
- Point lookup: O(log n) — typically 3-4 disk reads for millions of rows
- Range scan: O(log n + k) where k = results
- Insert/Delete: O(log n)
```

### Types of Indexes
| Type | Use Case | Example |
|------|----------|---------|
| Primary (clustered) | Row ordering on disk | PRIMARY KEY |
| Secondary | Additional lookup paths | INDEX on email |
| Composite | Multi-column queries | INDEX (country, city) |
| Unique | Enforce uniqueness | UNIQUE INDEX on username |
| Hash | Exact match only (no ranges) | Hash index on session_id |
| Full-text | Text search | GIN index for search |

### Composite Index — Order Matters!
```sql
-- Index on (country, city, zip)
-- Works for:
WHERE country = 'India'                        ✓ (leftmost prefix)
WHERE country = 'India' AND city = 'Pune'     ✓
WHERE country = 'India' AND city = 'Pune' AND zip = '411001' ✓

-- Does NOT work for:
WHERE city = 'Pune'                            ✗ (skips leftmost)
WHERE zip = '411001'                           ✗
```

### When NOT to Index
- Small tables (full scan is faster)
- Columns with low cardinality (e.g., boolean gender)
- Write-heavy tables where read performance isn't critical (indexes slow writes)
- Columns rarely used in WHERE/JOIN/ORDER BY

---

## Normalization

### Normal Forms (Know First 3)

**1NF:** Each cell contains a single value (no arrays, no repeating groups)
```
BAD:  | name  | phones          |
      | Prince| 123, 456        |

GOOD: | name  | phone |
      | Prince| 123   |
      | Prince| 456   |
```

**2NF:** 1NF + no partial dependencies (every non-key attribute depends on the WHOLE primary key)
```
BAD (composite key: student_id + course_id):
| student_id | course_id | student_name | grade |
student_name depends only on student_id, not the full key → violation

FIX: Separate into Students(id, name) and Enrollments(student_id, course_id, grade)
```

**3NF:** 2NF + no transitive dependencies (non-key depends on non-key)
```
BAD: | emp_id | dept_id | dept_name |
dept_name depends on dept_id, not directly on emp_id → violation

FIX: Employees(emp_id, dept_id) and Departments(dept_id, dept_name)
```

### When to Denormalize
- Read-heavy systems where JOIN cost matters
- Caching derived data (pre-computed counters)
- NoSQL systems (no joins available)

**Interview answer:** "I'd normalize for data integrity but consider selective denormalization for read performance on hot paths."

---

## Transactions & Isolation Levels

### Concurrency Problems
| Problem | Description |
|---------|-------------|
| Dirty Read | Read uncommitted data from another transaction |
| Non-repeatable Read | Same query returns different results within a transaction |
| Phantom Read | New rows appear between two reads in same transaction |

### Isolation Levels (from weakest to strongest)
| Level | Dirty Read | Non-repeatable | Phantom | Performance |
|-------|-----------|----------------|---------|------------|
| Read Uncommitted | ✗ | ✗ | ✗ | Fastest |
| Read Committed | ✓ | ✗ | ✗ | Good (PostgreSQL default) |
| Repeatable Read | ✓ | ✓ | ✗ | Moderate (MySQL default) |
| Serializable | ✓ | ✓ | ✓ | Slowest |

### Locks
- **Shared Lock (S):** Multiple readers, no writers
- **Exclusive Lock (X):** One writer, no readers
- **Row-level vs Table-level:** Row = better concurrency, Table = simpler
- **Optimistic Locking:** No lock, check version on write. Good for low-contention.
- **Pessimistic Locking:** Lock upfront. Good for high-contention.

---

## SQL Essentials for Interviews

### JOINs
```sql
-- INNER JOIN: Only matching rows from both tables
SELECT e.name, d.name FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;

-- LEFT JOIN: All from left + matching from right (NULL if no match)
SELECT e.name, d.name FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;

-- Self JOIN: Table joined with itself (org hierarchy, flights)
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### Window Functions (Asked at Amazon, Google)
```sql
-- Rank within a group
SELECT name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;

-- Running total
SELECT date, amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;

-- Moving average
SELECT date, value,
    AVG(value) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg
FROM metrics;
```

### Common Query Patterns
```sql
-- Find duplicates
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Second highest salary
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);
-- Or: SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;

-- Employees earning more than their manager
SELECT e.name FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

---

## Database Scaling Concepts

### Replication
- **Master-Slave:** Master handles writes, slaves handle reads
- **Master-Master:** Both handle writes (conflict resolution needed)
- **Replication Lag:** Slave may be behind master → eventual consistency

### Sharding (Horizontal Partitioning)
- Split rows across multiple databases
- **Shard key selection is critical** — determines data distribution and query routing
- Challenges: Cross-shard joins, rebalancing, hotspots

### Vertical Partitioning
Split columns — frequently accessed columns in one table, rarely accessed in another.

### Connection Pooling
Reuse database connections instead of creating new ones per request. Tools: PgBouncer, HikariCP.

---

## SQL vs NoSQL Decision Framework

| Factor | Choose SQL | Choose NoSQL |
|--------|-----------|-------------|
| Data structure | Well-defined, relational | Flexible, evolving |
| Queries | Complex joins, aggregations | Simple key-value or document lookup |
| Consistency | ACID required | Eventual consistency OK |
| Scale | Moderate (with read replicas) | Massive horizontal scale |
| Schema changes | Infrequent | Frequent |
| Examples | User profiles, financial data | Session store, product catalog, logs |

---

## Interview Must-Know

1. **"Why did you choose PostgreSQL for this?"** — ACID guarantees, complex queries, proven reliability
2. **"How would you handle slow queries?"** — EXPLAIN ANALYZE, add indexes, denormalize hot paths
3. **"How do you handle concurrent writes to same row?"** — Optimistic locking (version column) or SELECT FOR UPDATE
4. **"What's your sharding strategy?"** — Shard by user_id for user-centric data, consistent hashing for even distribution
