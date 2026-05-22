# Phase 10 — Indexes & Query Optimization

> **Curriculum:** DBMS A-to-Z | **File:** `10_indexes_and_optimization.md`  
> **Prerequisites:** Phases 1–9 (Foundations → Views)

---

## Table of Contents

1. [What Is an Index?](#1-what-is-an-index)
2. [B-Tree Index Internals](#2-b-tree-index-internals)
3. [Hash Indexes](#3-hash-indexes)
4. [Bitmap Indexes](#4-bitmap-indexes)
5. [GIN and GiST Indexes](#5-gin-and-gist-indexes)
6. [Composite Indexes](#6-composite-indexes)
7. [Covering Indexes](#7-covering-indexes)
8. [Partial Indexes](#8-partial-indexes)
9. [Index Scan vs Sequential Scan vs Index-Only Scan](#9-index-scan-vs-sequential-scan-vs-index-only-scan)
10. [Reading EXPLAIN / EXPLAIN ANALYZE](#10-reading-explain--explain-analyze)
11. [The Query Optimizer: Cost Model & Statistics](#11-the-query-optimizer-cost-model--statistics)
12. [Index Maintenance & Overhead](#12-index-maintenance--overhead)
13. [10 Slow Query Examples → Optimized Versions](#13-10-slow-query-examples--optimized-versions)
14. [Common Interview Questions](#14-common-interview-questions)
15. [Key Takeaways](#15-key-takeaways)

---

## 1. What Is an Index?

An **index** is a separate data structure maintained by the DBMS that maps column values to the physical location of the rows containing those values. It is built on one or more columns of a table and allows the engine to find rows **without scanning the entire table**.

### Real-World Analogy

A table without an index is like a textbook without a back-of-book index — to find "ACID properties" you must read every page. With an index you flip to "A", find the page number, and go directly there. The index costs extra paper (storage) and must be updated when the book changes (write overhead), but lookups become dramatically faster.

### The Core Trade-off

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="130" viewBox="0 0 560 130">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <!-- Read side -->
  <rect x="10" y="20" width="240" height="90" fill="#D5E8D4" stroke="#82B366" rx="8"/>
  <text x="130" y="44" text-anchor="middle" font-weight="bold" fill="#2D6A2D" font-size="13">Reads get FASTER</text>
  <text x="130" y="66" text-anchor="middle" fill="#2D6A2D">Fewer rows scanned</text>
  <text x="130" y="84" text-anchor="middle" fill="#2D6A2D">O(log N) vs O(N)</text>
  <text x="130" y="102" text-anchor="middle" fill="#2D6A2D">Point lookups become near-instant</text>

  <text x="280" y="70" text-anchor="middle" fill="#555" font-size="20">⇄</text>

  <!-- Write side -->
  <rect x="310" y="20" width="240" height="90" fill="#FDEDEC" stroke="#C0392B" rx="8"/>
  <text x="430" y="44" text-anchor="middle" font-weight="bold" fill="#7B241C" font-size="13">Writes get SLOWER</text>
  <text x="430" y="66" text-anchor="middle" fill="#7B241C">INSERT/UPDATE/DELETE</text>
  <text x="430" y="84" text-anchor="middle" fill="#7B241C">must also update the index</text>
  <text x="430" y="102" text-anchor="middle" fill="#7B241C">Extra storage consumed</text>
</svg>
```

### When Indexes Help vs Hurt

| Situation | Add index? |
|-----------|-----------|
| Column frequently in `WHERE`, `JOIN ON`, `ORDER BY` | ✅ Yes |
| Column has high cardinality (many distinct values) | ✅ Yes |
| Table has millions of rows | ✅ Yes |
| Query returns < ~10% of table rows | ✅ Yes |
| Table is tiny (< a few thousand rows) | ❌ No — seq scan is faster |
| Column has very low cardinality (boolean, status 3 values) | ❌ No (usually) |
| Table is write-heavy (bulk inserts every second) | ⚠️ Consider deferring/dropping |
| Column only used in `SELECT` list, never in `WHERE` | ❌ No |

### Sample Schema

```sql
CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(100),
    dept_id    INT,
    salary     NUMERIC(10,2),
    hire_date  DATE,
    is_active  BOOLEAN,
    email      VARCHAR(150)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    status      VARCHAR(20),
    total       NUMERIC(12,2)
);

CREATE TABLE order_items (
    item_id    INT PRIMARY KEY,
    order_id   INT,
    product_id INT,
    qty        INT,
    unit_price NUMERIC(10,2)
);
```

---

## 2. B-Tree Index Internals

### What Is a B-Tree?

A **B-Tree (Balanced Tree)** is the default index type in virtually every RDBMS (PostgreSQL, MySQL InnoDB, SQL Server, Oracle). It is a self-balancing search tree that keeps data sorted and guarantees O(log N) lookups, inserts, and deletes.

### Structure

A B-Tree has three node types:

- **Root node** — the top; always kept in memory by the buffer pool.
- **Internal nodes (branch pages)** — contain keys and pointers to child nodes.
- **Leaf nodes** — contain keys and pointers (TIDs / row locators) to actual heap rows.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="320" viewBox="0 0 680 320">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.kv{font-size:10px;fill:#333}</style>

  <!-- ROOT -->
  <rect x="240" y="10" width="200" height="40" fill="#2980B9" rx="6"/>
  <text x="340" y="26" text-anchor="middle" fill="white" font-weight="bold" font-size="12">ROOT NODE</text>
  <text x="340" y="44" text-anchor="middle" fill="white">[ 40 | 70 ]</text>

  <!-- Lines root → internal -->
  <line x1="280" y1="50" x2="130" y2="110" stroke="#555" stroke-width="1.5"/>
  <line x1="340" y1="50" x2="340" y2="110" stroke="#555" stroke-width="1.5"/>
  <line x1="400" y1="50" x2="550" y2="110" stroke="#555" stroke-width="1.5"/>

  <!-- INTERNAL nodes -->
  <rect x="40"  y="110" width="160" height="40" fill="#6C8EBF" rx="6"/>
  <text x="120" y="126" text-anchor="middle" fill="white" font-weight="bold">INTERNAL</text>
  <text x="120" y="144" text-anchor="middle" fill="white">[ 20 | 30 ]</text>

  <rect x="260" y="110" width="160" height="40" fill="#6C8EBF" rx="6"/>
  <text x="340" y="126" text-anchor="middle" fill="white" font-weight="bold">INTERNAL</text>
  <text x="340" y="144" text-anchor="middle" fill="white">[ 50 | 60 ]</text>

  <rect x="480" y="110" width="160" height="40" fill="#6C8EBF" rx="6"/>
  <text x="560" y="126" text-anchor="middle" fill="white" font-weight="bold">INTERNAL</text>
  <text x="560" y="144" text-anchor="middle" fill="white">[ 80 | 90 ]</text>

  <!-- Lines internal → leaf -->
  <line x1="70"  y1="150" x2="55"  y2="210" stroke="#888" stroke-width="1.2"/>
  <line x1="120" y1="150" x2="120" y2="210" stroke="#888" stroke-width="1.2"/>
  <line x1="170" y1="150" x2="185" y2="210" stroke="#888" stroke-width="1.2"/>

  <line x1="290" y1="150" x2="275" y2="210" stroke="#888" stroke-width="1.2"/>
  <line x1="340" y1="150" x2="340" y2="210" stroke="#888" stroke-width="1.2"/>
  <line x1="390" y1="150" x2="405" y2="210" stroke="#888" stroke-width="1.2"/>

  <line x1="510" y1="150" x2="495" y2="210" stroke="#888" stroke-width="1.2"/>
  <line x1="560" y1="150" x2="560" y2="210" stroke="#888" stroke-width="1.2"/>
  <line x1="610" y1="150" x2="625" y2="210" stroke="#888" stroke-width="1.2"/>

  <!-- LEAF nodes (row 1) -->
  <rect x="10"  y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="55" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="55" y="246" text-anchor="middle" class="kv">10→row ptr</text>
  <text x="55" y="258" text-anchor="middle" class="kv">15→row ptr</text>

  <rect x="105" y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="150" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="150" y="246" text-anchor="middle" class="kv">20→row ptr</text>
  <text x="150" y="258" text-anchor="middle" class="kv">25→row ptr</text>

  <rect x="200" y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="245" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="245" y="246" text-anchor="middle" class="kv">30→row ptr</text>
  <text x="245" y="258" text-anchor="middle" class="kv">35→row ptr</text>

  <rect x="295" y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="340" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="340" y="246" text-anchor="middle" class="kv">50→row ptr</text>
  <text x="340" y="258" text-anchor="middle" class="kv">55→row ptr</text>

  <rect x="390" y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="435" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="435" y="246" text-anchor="middle" class="kv">60→row ptr</text>
  <text x="435" y="258" text-anchor="middle" class="kv">65→row ptr</text>

  <rect x="485" y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="530" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="530" y="246" text-anchor="middle" class="kv">80→row ptr</text>
  <text x="530" y="258" text-anchor="middle" class="kv">85→row ptr</text>

  <rect x="580" y="210" width="90" height="50" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="625" y="228" text-anchor="middle" fill="#2D6A2D" font-weight="bold">LEAF</text>
  <text x="625" y="246" text-anchor="middle" class="kv">90→row ptr</text>
  <text x="625" y="258" text-anchor="middle" class="kv">95→row ptr</text>

  <!-- Linked-list arrows between leaves -->
  <line x1="100" y1="235" x2="105" y2="235" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#al)"/>
  <line x1="195" y1="235" x2="200" y2="235" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#al)"/>
  <line x1="290" y1="235" x2="295" y2="235" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#al)"/>
  <line x1="385" y1="235" x2="390" y2="235" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#al)"/>
  <line x1="480" y1="235" x2="485" y2="235" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#al)"/>
  <line x1="575" y1="235" x2="580" y2="235" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#al)"/>
  <defs><marker id="al" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto">
    <path d="M0,0 L0,6 L6,3z" fill="#E74C3C"/>
  </marker></defs>

  <text x="340" y="308" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">Leaf nodes linked in order → efficient range scans</text>
</svg>
```

### Key Properties

| Property | Value |
|----------|-------|
| Lookup complexity | O(log N) |
| Range scan | Efficient — leaf nodes are doubly linked |
| Tree height (1M rows) | ~3–4 levels |
| Supports | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'` |
| Does NOT support | `LIKE '%suffix'`, most functions on indexed column |
| Balanced? | Always — auto-balances on insert/delete |

### Creating a B-Tree Index (default)

```sql
-- All of these create B-Tree indexes (the default type):
CREATE INDEX idx_emp_dept    ON employees (dept_id);
CREATE INDEX idx_emp_salary  ON employees (salary);
CREATE INDEX idx_ord_date    ON orders (order_date);
CREATE INDEX idx_ord_custid  ON orders (customer_id);

-- Unique constraint automatically creates a B-Tree index
CREATE UNIQUE INDEX idx_emp_email ON employees (email);
```

---

## 3. Hash Indexes

### Concept

A **hash index** stores a hash value of each indexed key. Lookups use the hash function to find the bucket containing the matching row pointer. Extremely fast for exact equality lookups — but useless for ranges.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="520" height="180" viewBox="0 0 520 180">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <text x="80" y="20" text-anchor="middle" fill="#333" font-weight="bold">Key (email)</text>
  <rect x="10" y="30" width="140" height="30" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="80" y="50" text-anchor="middle" fill="#1A5276">alice@example.com</text>
  <rect x="10" y="70" width="140" height="30" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="80" y="90" text-anchor="middle" fill="#1A5276">bob@example.com</text>
  <rect x="10" y="110" width="140" height="30" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="80" y="130" text-anchor="middle" fill="#1A5276">carol@example.com</text>

  <text x="210" y="50"  text-anchor="middle" fill="#E74C3C" font-weight="bold">hash()</text>
  <text x="210" y="90"  text-anchor="middle" fill="#E74C3C" font-weight="bold">hash()</text>
  <text x="210" y="130" text-anchor="middle" fill="#E74C3C" font-weight="bold">hash()</text>
  <line x1="150" y1="45"  x2="250" y2="45"  stroke="#555" stroke-width="1.5" marker-end="url(#ah)"/>
  <line x1="150" y1="85"  x2="250" y2="85"  stroke="#555" stroke-width="1.5" marker-end="url(#ah)"/>
  <line x1="150" y1="125" x2="250" y2="125" stroke="#555" stroke-width="1.5" marker-end="url(#ah)"/>
  <defs><marker id="ah" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <text x="360" y="20" text-anchor="middle" fill="#333" font-weight="bold">Hash Buckets</text>
  <rect x="260" y="30" width="100" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="310" y="50" text-anchor="middle" fill="#784212">Bucket 2 → row</text>
  <rect x="260" y="70" width="100" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="310" y="90" text-anchor="middle" fill="#784212">Bucket 7 → row</text>
  <rect x="260" y="110" width="100" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="310" y="130" text-anchor="middle" fill="#784212">Bucket 2 → row</text>

  <text x="260" y="165" fill="#7D3C98" font-size="11" font-style="italic">O(1) equality lookup — cannot do range scans</text>
</svg>
```

### When to Use Hash Indexes

| Supports | Does NOT support |
|----------|----------------|
| `col = value` (exact equality) | `<`, `>`, `BETWEEN`, `LIKE` |
| Extremely fast point lookups | Range scans |
| In-memory hash joins (engine uses internally) | Ordering |

```sql
-- PostgreSQL: explicit hash index
CREATE INDEX idx_emp_email_hash ON employees USING HASH (email);

-- MySQL InnoDB: hash indexes are created internally by the Adaptive Hash Index (AHI)
-- You cannot create explicit hash indexes in MySQL InnoDB

-- SQL Server: hash indexes available on memory-optimized tables only
```

**Practical note:** In PostgreSQL, the planner almost always prefers B-Tree over a hash index even for equality. Hash indexes are niche — use them only after profiling confirms benefit.

---

## 4. Bitmap Indexes

### Concept

A **bitmap index** stores a bit array (bitmap) for each distinct value in the indexed column. Each bit position corresponds to a row; a `1` means the row has that value, `0` means it does not.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="200" viewBox="0 0 560 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <text x="280" y="16" text-anchor="middle" font-weight="bold" fill="#333" font-size="12">Bitmap Index on status column (5 rows)</text>

  <!-- Row IDs header -->
  <text x="130" y="36" text-anchor="middle" fill="#555">Row IDs →</text>
  <text x="230" y="36" text-anchor="middle" fill="#555">1</text>
  <text x="270" y="36" text-anchor="middle" fill="#555">2</text>
  <text x="310" y="36" text-anchor="middle" fill="#555">3</text>
  <text x="350" y="36" text-anchor="middle" fill="#555">4</text>
  <text x="390" y="36" text-anchor="middle" fill="#555">5</text>

  <!-- 'Active' bitmap -->
  <rect x="10" y="46" width="100" height="30" fill="#AED6F1" stroke="#2980B9" rx="4"/>
  <text x="60" y="66" text-anchor="middle" fill="#1A5276" font-weight="bold">Active</text>
  <text x="230" y="66" text-anchor="middle" fill="#27AE60">1</text>
  <text x="270" y="66" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="310" y="66" text-anchor="middle" fill="#27AE60">1</text>
  <text x="350" y="66" text-anchor="middle" fill="#27AE60">1</text>
  <text x="390" y="66" text-anchor="middle" fill="#E74C3C">0</text>

  <!-- 'Pending' bitmap -->
  <rect x="10" y="86" width="100" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="60" y="106" text-anchor="middle" fill="#784212" font-weight="bold">Pending</text>
  <text x="230" y="106" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="270" y="106" text-anchor="middle" fill="#27AE60">1</text>
  <text x="310" y="106" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="350" y="106" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="390" y="106" text-anchor="middle" fill="#E74C3C">0</text>

  <!-- 'Closed' bitmap -->
  <rect x="10" y="126" width="100" height="30" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="60" y="146" text-anchor="middle" fill="#2D6A2D" font-weight="bold">Closed</text>
  <text x="230" y="146" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="270" y="146" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="310" y="146" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="350" y="146" text-anchor="middle" fill="#E74C3C">0</text>
  <text x="390" y="146" text-anchor="middle" fill="#27AE60">1</text>

  <!-- AND operation note -->
  <text x="280" y="185" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">AND/OR/NOT on bitmaps = lightning-fast multi-condition filters (data warehouses)</text>
</svg>
```

### Characteristics

| Property | Value |
|----------|-------|
| Best for | Low-cardinality columns (status, region, boolean, gender) |
| Storage | Very compact for low cardinality |
| Operations | AND, OR, NOT in O(N/64) using CPU bitwise instructions |
| NOT suitable for | High-cardinality data (IDs, emails) — one bitmap per value |
| Concurrent writes | Poor — locking at bitmap level causes contention |
| Main use | Oracle, data warehouses, OLAP systems |

```sql
-- Oracle
CREATE BITMAP INDEX idx_order_status ON orders (status);

-- PostgreSQL does NOT have persistent bitmap indexes
-- It creates temporary bitmaps in-memory during query execution (Bitmap Index Scan)
```

---

## 5. GIN and GiST Indexes

### GIN — Generalized Inverted Index

Best for **full-text search**, array containment, JSONB queries, and multi-valued attributes. Think of it as a reverse lookup: for each element (word, array element, JSON key), it stores the set of rows containing that element.

```sql
-- Full-text search index (PostgreSQL)
CREATE INDEX idx_products_fts ON products
USING GIN (to_tsvector('english', product_name || ' ' || description));

-- Query using the GIN index
SELECT product_name
FROM   products
WHERE  to_tsvector('english', product_name || ' ' || description)
       @@ to_tsquery('english', 'wireless & headphone');

-- JSONB containment index
CREATE INDEX idx_orders_meta ON orders USING GIN (metadata);
SELECT * FROM orders WHERE metadata @> '{"channel": "mobile"}';

-- Array containment
CREATE INDEX idx_tags ON posts USING GIN (tags);
SELECT * FROM posts WHERE tags @> ARRAY['sql', 'database'];
```

### GiST — Generalized Search Tree

A framework for building custom index types. Used for **geometric/spatial data**, range types, and nearest-neighbour searches. PostGIS (geographic data) relies heavily on GiST.

```sql
-- Geometric/spatial index (PostGIS)
CREATE INDEX idx_locations ON stores USING GIST (geom);

-- Find stores within 5 km of a point
SELECT store_name
FROM   stores
WHERE  ST_DWithin(geom, ST_MakePoint(-73.9857, 40.7484)::geography, 5000);

-- Range type index
CREATE INDEX idx_booking_range ON bookings USING GIST (daterange(check_in, check_out));
SELECT * FROM bookings WHERE daterange(check_in, check_out) && '[2024-06-01, 2024-06-10)';
```

### GIN vs GiST Quick Reference

| Feature | GIN | GiST |
|---------|-----|------|
| Best for | Full-text, arrays, JSONB | Geometry, ranges, custom types |
| Lookup speed | Faster for exact/containment | Faster for nearest-neighbour |
| Build speed | Slower to build | Faster to build |
| Storage | Larger | Smaller |
| Update cost | Higher | Lower |

---

## 6. Composite Indexes

### Concept

A **composite index** (multi-column index) is built on two or more columns. The key insight is **column order matters** — the index is sorted left-to-right, so it supports queries that filter on a **leading prefix** of the indexed columns.

### The Left-Prefix Rule

```sql
CREATE INDEX idx_emp_dept_salary ON employees (dept_id, salary, hire_date);
```

| Query | Uses index? | Why |
|-------|-------------|-----|
| `WHERE dept_id = 10` | ✅ Yes | Leading column |
| `WHERE dept_id = 10 AND salary > 70000` | ✅ Yes | Leading prefix |
| `WHERE dept_id = 10 AND salary > 70000 AND hire_date > '2020-01-01'` | ✅ Yes | Full index |
| `WHERE salary > 70000` | ❌ No | Skips leading column |
| `WHERE salary > 70000 AND dept_id = 10` | ✅ Yes | Planner reorders |
| `WHERE hire_date > '2020-01-01'` | ❌ No | Not a leading prefix |

### Designing Composite Indexes

**Rule of thumb:** put **equality columns first**, then **range columns last**.

```sql
-- Query: find active employees in a department within a salary range
-- Columns: is_active (equality), dept_id (equality), salary (range)

-- Good order: equality first, range last
CREATE INDEX idx_active_dept_salary
ON employees (is_active, dept_id, salary);

-- Bad order: range in the middle — stops being useful after salary
CREATE INDEX idx_bad_order
ON employees (is_active, salary, dept_id);
-- dept_id can't be used as the planner can't skip a range scan column
```

### Example Queries

```sql
-- Efficiently uses idx_active_dept_salary:
SELECT name, salary
FROM   employees
WHERE  is_active = TRUE
  AND  dept_id   = 10
  AND  salary    BETWEEN 70000 AND 90000;

-- Creating the composite index:
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date);

-- Benefits both of these queries:
SELECT * FROM orders WHERE customer_id = 42;                         -- leading col only
SELECT * FROM orders WHERE customer_id = 42 AND order_date > '2024-01-01'; -- full prefix
```

---

## 7. Covering Indexes

### Concept

A **covering index** includes all columns needed by a query — not just the filter columns but also the `SELECT` list columns. When every column the query needs is in the index, the database can answer the query **entirely from the index** without ever touching the heap (base table). This is called an **index-only scan** and is the fastest possible access path.

### PostgreSQL: INCLUDE clause

```sql
-- Without INCLUDE: index on (dept_id, salary) — SELECT must also fetch name from heap
CREATE INDEX idx_dept_salary ON employees (dept_id, salary);

-- With INCLUDE: adds name as a non-key payload — enables index-only scan
CREATE INDEX idx_dept_salary_cover ON employees (dept_id, salary)
INCLUDE (name, hire_date);

-- This query can now be answered entirely from the index:
SELECT name, salary, hire_date
FROM   employees
WHERE  dept_id = 10
  AND  salary > 70000;
```

### SQL Server: INCLUDE in non-clustered indexes

```sql
CREATE NONCLUSTERED INDEX idx_orders_cover
ON orders (customer_id, order_date)
INCLUDE (status, total);

-- Query answered entirely from the index:
SELECT status, total
FROM   orders
WHERE  customer_id = 42
  AND  order_date  >= '2024-01-01';
```

### Covering Index vs Normal Index

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="600" height="200" viewBox="0 0 600 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Normal index flow -->
  <rect x="10" y="10" width="260" height="170" fill="#EBF5FB" stroke="#2980B9" rx="8"/>
  <text x="140" y="30" text-anchor="middle" font-weight="bold" fill="#1A5276" font-size="12">Normal Index Scan</text>
  <rect x="30" y="45" width="100" height="35" fill="#AED6F1" stroke="#6C8EBF" rx="4"/>
  <text x="80" y="68" text-anchor="middle" fill="#1A5276">Index</text>
  <text x="80" y="84" text-anchor="middle" fill="#1A5276">(filter cols)</text>
  <line x1="130" y1="62" x2="165" y2="62" stroke="#E74C3C" stroke-width="2" marker-end="url(#acov)"/>
  <defs><marker id="acov" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/>
  </marker></defs>
  <rect x="165" y="45" width="90" height="35" fill="#FDEDEC" stroke="#C0392B" rx="4"/>
  <text x="210" y="68" text-anchor="middle" fill="#7B241C">Heap</text>
  <text x="210" y="84" text-anchor="middle" fill="#7B241C">(fetch cols)</text>
  <text x="140" y="120" text-anchor="middle" fill="#E74C3C">→ Requires heap fetch</text>
  <text x="140" y="140" text-anchor="middle" fill="#555">Random I/O per row</text>
  <text x="140" y="160" text-anchor="middle" fill="#555">Slower for wide result sets</text>

  <!-- Covering index flow -->
  <rect x="330" y="10" width="260" height="170" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="460" y="30" text-anchor="middle" font-weight="bold" fill="#145A32" font-size="12">Index-Only Scan (Covering)</text>
  <rect x="350" y="45" width="220" height="35" fill="#A9DFBF" stroke="#1E8449" rx="4"/>
  <text x="460" y="63" text-anchor="middle" fill="#145A32">Index</text>
  <text x="460" y="79" text-anchor="middle" fill="#145A32">(filter cols + SELECT cols)</text>
  <text x="460" y="120" text-anchor="middle" fill="#27AE60">→ No heap access needed</text>
  <text x="460" y="140" text-anchor="middle" fill="#555">Sequential index reads</text>
  <text x="460" y="160" text-anchor="middle" fill="#555">Fastest access pattern</text>
</svg>
```

---

## 8. Partial Indexes

### Concept

A **partial index** is built only on rows that satisfy a `WHERE` condition. It is smaller, faster to maintain, and the query planner can use it only when the query's predicate matches the index predicate. Ideal for queries that always filter on a common value.

```sql
-- Index only active employees (ignores the large inactive population)
CREATE INDEX idx_active_employees
ON employees (dept_id, salary)
WHERE is_active = TRUE;

-- This query uses the partial index:
SELECT name, salary
FROM   employees
WHERE  is_active = TRUE
  AND  dept_id   = 10;

-- This query does NOT use it (no is_active filter):
SELECT name, salary FROM employees WHERE dept_id = 10;
```

### Example 2 — Index only pending orders

```sql
CREATE INDEX idx_pending_orders
ON orders (customer_id, order_date)
WHERE status = 'Pending';

-- Fast lookup for a customer's pending orders:
SELECT order_id, order_date
FROM   orders
WHERE  customer_id = 42
  AND  status      = 'Pending';
```

### Example 3 — Unique partial index (allow multiple NULLs)

```sql
-- Standard UNIQUE allows only one NULL in most RDBMS
-- Partial unique index: unique only on non-NULL values
CREATE UNIQUE INDEX idx_unique_email_active
ON employees (email)
WHERE email IS NOT NULL;
```

### Partial vs Full Index Comparison

| | Full Index | Partial Index |
|---|----------|--------------|
| Rows indexed | All rows | Matching rows only |
| Size | Large | Small |
| Maintenance cost | High | Low |
| When usable | Any matching query | Only when WHERE predicate matches |
| Best for | General-purpose | Highly skewed data, common filter |

---

## 9. Index Scan vs Sequential Scan vs Index-Only Scan

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="260" viewBox="0 0 680 260">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.h{font-size:12px;font-weight:bold}</style>

  <!-- Sequential Scan -->
  <rect x="10" y="10" width="200" height="220" fill="#EBF5FB" stroke="#2980B9" rx="8"/>
  <text x="110" y="32" text-anchor="middle" class="h" fill="#1A5276">Sequential Scan</text>
  <line x1="20" y1="40" x2="200" y2="40" stroke="#AED6F1"/>
  <text x="110" y="60" text-anchor="middle" fill="#333">Reads every page</text>
  <text x="110" y="76" text-anchor="middle" fill="#333">in the table in order</text>
  <text x="110" y="104" text-anchor="middle" fill="#27AE60">✓ Fast when returning</text>
  <text x="110" y="120" text-anchor="middle" fill="#27AE60">  many rows (&gt;~10%)</text>
  <text x="110" y="138" text-anchor="middle" fill="#27AE60">✓ No index overhead</text>
  <text x="110" y="158" text-anchor="middle" fill="#E74C3C">✗ Slow for point lookups</text>
  <text x="110" y="176" text-anchor="middle" fill="#E74C3C">  on large tables</text>
  <text x="110" y="208" text-anchor="middle" fill="#7D6608" font-style="italic">Cost: O(N pages)</text>

  <!-- Index Scan -->
  <rect x="240" y="10" width="200" height="220" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="340" y="32" text-anchor="middle" class="h" fill="#145A32">Index Scan</text>
  <line x1="250" y1="40" x2="430" y2="40" stroke="#A9DFBF"/>
  <text x="340" y="60" text-anchor="middle" fill="#333">Traverses index to find</text>
  <text x="340" y="76" text-anchor="middle" fill="#333">matching rows, then</text>
  <text x="340" y="92" text-anchor="middle" fill="#333">fetches each from heap</text>
  <text x="340" y="118" text-anchor="middle" fill="#27AE60">✓ Fast for selective</text>
  <text x="340" y="134" text-anchor="middle" fill="#27AE60">  queries (few rows)</text>
  <text x="340" y="154" text-anchor="middle" fill="#E74C3C">✗ Random I/O on heap</text>
  <text x="340" y="172" text-anchor="middle" fill="#E74C3C">  (expensive for HDD)</text>
  <text x="340" y="208" text-anchor="middle" fill="#7D6608" font-style="italic">Cost: O(log N + K)</text>

  <!-- Index-Only Scan -->
  <rect x="470" y="10" width="200" height="220" fill="#FEF9E7" stroke="#F39C12" rx="8"/>
  <text x="570" y="32" text-anchor="middle" class="h" fill="#7D6608">Index-Only Scan</text>
  <line x1="480" y1="40" x2="660" y2="40" stroke="#FAD7A0"/>
  <text x="570" y="60" text-anchor="middle" fill="#333">All needed columns are</text>
  <text x="570" y="76" text-anchor="middle" fill="#333">in the index itself</text>
  <text x="570" y="92" text-anchor="middle" fill="#333">— no heap access</text>
  <text x="570" y="118" text-anchor="middle" fill="#27AE60">✓ Fastest read path</text>
  <text x="570" y="134" text-anchor="middle" fill="#27AE60">✓ No random I/O</text>
  <text x="570" y="154" text-anchor="middle" fill="#E74C3C">✗ Requires covering</text>
  <text x="570" y="170" text-anchor="middle" fill="#E74C3C">  index design</text>
  <text x="570" y="208" text-anchor="middle" fill="#7D6608" font-style="italic">Cost: O(log N)</text>
</svg>
```

### Bitmap Index Scan (PostgreSQL specific)

When multiple indexes can each contribute to a query, PostgreSQL combines them with a **Bitmap Index Scan + BitmapAnd / BitmapOr**:

1. Scan index A → build bitmap of matching row locations.
2. Scan index B → build bitmap of matching row locations.
3. AND/OR the bitmaps.
4. Fetch heap pages in physical order (reduces random I/O).

```sql
-- Two separate indexes can be combined:
CREATE INDEX idx_dept ON employees (dept_id);
CREATE INDEX idx_salary ON employees (salary);

SELECT * FROM employees WHERE dept_id = 10 AND salary > 80000;
-- Plan: BitmapAnd(Bitmap Index Scan on idx_dept, Bitmap Index Scan on idx_salary)
--       → Bitmap Heap Scan on employees
```

---

## 10. Reading EXPLAIN / EXPLAIN ANALYZE

### EXPLAIN vs EXPLAIN ANALYZE

| Command | Executes query? | Shows | Use when |
|---------|----------------|-------|----------|
| `EXPLAIN` | ❌ No | Estimated plan and costs | Safe on any query; check the plan |
| `EXPLAIN ANALYZE` | ✅ Yes | Estimated + actual times and rows | Confirm plan and diagnose row estimates |
| `EXPLAIN (ANALYZE, BUFFERS)` | ✅ Yes | + cache hit/miss statistics | Deep I/O analysis |

### Anatomy of an EXPLAIN Output

```sql
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM   employees   e
JOIN   departments d ON e.dept_id = d.dept_id
WHERE  e.salary > 70000;

-- Output:
-- Hash Join  (cost=1.09..2.42 rows=3 width=104)
--             (actual time=0.05..0.08 rows=3 loops=1)
--   Hash Cond: (e.dept_id = d.dept_id)
--   ->  Seq Scan on employees e  (cost=0.00..1.06 rows=3 width=58)
--                                (actual time=0.01..0.02 rows=3 loops=1)
--         Filter: (salary > 70000)
--         Rows Removed by Filter: 2
--   ->  Hash  (cost=1.03..1.03 rows=3 width=54)
--             (actual time=0.02..0.02 rows=3 loops=1)
--         ->  Seq Scan on departments d  (cost=0.00..1.03 rows=3 width=54)
--                                       (actual time=0.01..0.02 rows=3 loops=1)
-- Planning Time: 0.14 ms
-- Execution Time: 0.13 ms
```

### Decoding Each Part

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="240" viewBox="0 0 680 240">
  <style>text{font-family:Arial,sans-serif;font-size:10px}</style>

  <rect x="10" y="10" width="660" height="40" fill="#2C3E50" rx="4"/>
  <text x="340" y="24" text-anchor="middle" fill="white" font-size="11" font-weight="bold">Hash Join  (cost=1.09..2.42 rows=3 width=104) (actual time=0.05..0.08 rows=3 loops=1)</text>
  <text x="340" y="42" text-anchor="middle" fill="#AED6F1" font-size="10">Node type     startup..total cost  est. rows  est. row width(bytes)  actual start..end  real rows  how many times</text>

  <!-- Annotation boxes -->
  <rect x="10" y="65" width="140" height="60" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="80" y="82" text-anchor="middle" fill="#2D6A2D" font-weight="bold">Node Type</text>
  <text x="80" y="98" text-anchor="middle" fill="#2D6A2D">How this step</text>
  <text x="80" y="112" text-anchor="middle" fill="#2D6A2D">was executed</text>

  <rect x="165" y="65" width="140" height="60" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="235" y="82" text-anchor="middle" fill="#1A5276" font-weight="bold">cost=S..T</text>
  <text x="235" y="98" text-anchor="middle" fill="#1A5276">S=startup cost</text>
  <text x="235" y="112" text-anchor="middle" fill="#1A5276">T=total cost (planner units)</text>

  <rect x="320" y="65" width="140" height="60" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="390" y="82" text-anchor="middle" fill="#784212" font-weight="bold">rows=N</text>
  <text x="390" y="98" text-anchor="middle" fill="#784212">Estimated rows</text>
  <text x="390" y="112" text-anchor="middle" fill="#784212">from planner stats</text>

  <rect x="475" y="65" width="180" height="60" fill="#FDEDEC" stroke="#C0392B" rx="4"/>
  <text x="565" y="82" text-anchor="middle" fill="#7B241C" font-weight="bold">actual rows vs est. rows</text>
  <text x="565" y="98" text-anchor="middle" fill="#7B241C">Large mismatch = stale stats</text>
  <text x="565" y="112" text-anchor="middle" fill="#7B241C">Run ANALYZE to fix</text>

  <!-- Red flags -->
  <rect x="10" y="150" width="660" height="80" fill="#FEF9E7" stroke="#F39C12" rx="6"/>
  <text x="340" y="170" text-anchor="middle" fill="#7D6608" font-weight="bold" font-size="11">🚩 Red Flags to Look For in EXPLAIN Output</text>
  <text x="340" y="188" text-anchor="middle" fill="#555">Seq Scan on a large table with few rows returned → missing index</text>
  <text x="340" y="204" text-anchor="middle" fill="#555">Estimated rows ≪ or ≫ actual rows → stale statistics (run ANALYZE / UPDATE STATISTICS)</text>
  <text x="340" y="220" text-anchor="middle" fill="#555">Nested Loop with large outer table and no index on inner → add index or rewrite as hash join</text>
</svg>
```

### PostgreSQL EXPLAIN tips

```sql
-- More detail: buffers shows cache hits/misses
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

-- JSON format for programmatic analysis
EXPLAIN (FORMAT JSON) SELECT ...;

-- Force a sequential scan to compare plans
SET enable_indexscan = OFF;
EXPLAIN SELECT ...;
SET enable_indexscan = ON;
```

---

## 11. The Query Optimizer: Cost Model & Statistics

### How the Optimizer Works

The **query optimizer** (also called the query planner) takes a SQL statement and determines the most efficient **execution plan** from potentially thousands of alternatives.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="640" height="180" viewBox="0 0 640 180">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <rect x="10"  y="40" width="100" height="50" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="60"  y="62" text-anchor="middle" fill="#2D6A2D" font-weight="bold">SQL Query</text>

  <line x1="110" y1="65" x2="150" y2="65" stroke="#555" stroke-width="1.5" marker-end="url(#aopt)"/>
  <defs><marker id="aopt" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <rect x="150" y="20" width="120" height="90" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="210" y="48" text-anchor="middle" fill="#1A5276" font-weight="bold">Parser /</text>
  <text x="210" y="64" text-anchor="middle" fill="#1A5276">Rewriter</text>
  <text x="210" y="90" text-anchor="middle" fill="#555" font-size="10">Parse tree</text>
  <text x="210" y="104" text-anchor="middle" fill="#555" font-size="10">+ rewrites</text>

  <line x1="270" y1="65" x2="310" y2="65" stroke="#555" stroke-width="1.5" marker-end="url(#aopt)"/>

  <rect x="310" y="10" width="140" height="110" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="380" y="34" text-anchor="middle" fill="#784212" font-weight="bold">Planner /</text>
  <text x="380" y="50" text-anchor="middle" fill="#784212">Optimizer</text>
  <text x="380" y="74" text-anchor="middle" fill="#555" font-size="10">Enumerate alternatives</text>
  <text x="380" y="90" text-anchor="middle" fill="#555" font-size="10">Estimate costs using</text>
  <text x="380" y="106" text-anchor="middle" fill="#555" font-size="10">stats + cost model</text>

  <line x1="450" y1="65" x2="490" y2="65" stroke="#555" stroke-width="1.5" marker-end="url(#aopt)"/>

  <rect x="490" y="30" width="130" height="70" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="555" y="56" text-anchor="middle" fill="#2D6A2D" font-weight="bold">Executor</text>
  <text x="555" y="76" text-anchor="middle" fill="#555" font-size="10">Runs the chosen</text>
  <text x="555" y="90" text-anchor="middle" fill="#555" font-size="10">execution plan</text>

  <text x="320" y="165" fill="#7D3C98" font-size="11" font-style="italic" text-anchor="middle">The planner picks the plan with the lowest estimated total cost.</text>
</svg>
```

### Statistics the Planner Uses

The optimizer relies on table statistics to estimate cardinality (row counts after filters):

```sql
-- PostgreSQL: view column statistics
SELECT attname, n_distinct, correlation
FROM   pg_stats
WHERE  tablename = 'employees';

-- Update statistics (PostgreSQL)
ANALYZE employees;
ANALYZE;                   -- analyze all tables

-- SQL Server
UPDATE STATISTICS employees;
UPDATE STATISTICS employees (idx_emp_dept);

-- MySQL
ANALYZE TABLE employees;
```

### Key Statistics

| Statistic | What it tells the planner |
|-----------|--------------------------|
| `n_distinct` | Number of distinct values → cardinality estimates |
| `most_common_vals` | Frequent values → skew-aware estimates |
| `histogram_bounds` | Distribution of values → range selectivity |
| `correlation` | Physical order vs logical order → index vs seq scan decision |

### Optimizer Hints (when you know better)

Most databases support hints to guide the optimizer when it makes poor choices (usually due to stale stats):

```sql
-- PostgreSQL: no official hints, but you can disable specific strategies
SET enable_nestloop = OFF;     -- discourage nested loop joins
SET enable_seqscan  = OFF;     -- force index usage for testing

-- MySQL: USE INDEX, FORCE INDEX
SELECT * FROM orders USE INDEX (idx_ord_date) WHERE order_date > '2024-01-01';
SELECT * FROM orders FORCE INDEX (idx_ord_date) WHERE order_date > '2024-01-01';

-- SQL Server: query hints
SELECT * FROM orders WITH (INDEX = idx_ord_date) WHERE order_date > '2024-01-01';
SELECT * FROM orders OPTION (HASH JOIN, LOOP JOIN);

-- Oracle: hints in comments
SELECT /*+ INDEX(o idx_ord_date) */ * FROM orders o WHERE order_date > '2024-01-01';
SELECT /*+ USE_HASH(e d) */ * FROM employees e JOIN departments d ON e.dept_id = d.dept_id;
```

---

## 12. Index Maintenance & Overhead

### Write Overhead

Every index must be updated on INSERT, UPDATE, and DELETE. A table with 8 indexes means every single INSERT writes to 9 data structures (1 heap + 8 indexes).

```sql
-- Monitor index usage (PostgreSQL) — identify unused indexes
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM   pg_stat_user_indexes
ORDER BY idx_scan ASC;   -- indexes with idx_scan = 0 are candidates for removal

-- Index bloat: B-Trees accumulate dead tuples after updates/deletes
-- Rebuild a bloated index (takes an exclusive lock)
REINDEX INDEX idx_emp_salary;

-- Rebuild without locking (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_emp_salary;
```

### Index Bloat

Updates and deletes leave **dead index entries**. Over time this bloat wastes storage and slows scans. Use `VACUUM` / `REINDEX` to reclaim space:

```sql
-- PostgreSQL: VACUUM removes dead tuples and reclaims bloat
VACUUM ANALYZE employees;
VACUUM (VERBOSE, ANALYZE) employees;

-- Check if autovacuum is keeping up
SELECT relname, n_dead_tup, last_autovacuum
FROM   pg_stat_user_tables
WHERE  n_dead_tup > 10000;
```

### Creating Indexes Without Locking (PostgreSQL)

```sql
-- Standard CREATE INDEX locks the table for writes during build
CREATE INDEX idx_emp_salary ON employees (salary);

-- CONCURRENTLY: allows reads AND writes during build (takes longer)
CREATE INDEX CONCURRENTLY idx_emp_salary ON employees (salary);
```

---

## 13. 10 Slow Query Examples → Optimized Versions

### Slow Q1 — Function on indexed column disables the index

```sql
-- SLOW: YEAR() prevents index use on order_date
SELECT * FROM orders
WHERE  YEAR(order_date) = 2024;

-- FAST: use a range predicate instead
SELECT * FROM orders
WHERE  order_date >= '2024-01-01'
  AND  order_date <  '2025-01-01';

-- Index to add:
CREATE INDEX idx_orders_date ON orders (order_date);
```

### Slow Q2 — Leading wildcard in LIKE

```sql
-- SLOW: leading % prevents B-Tree index use
SELECT * FROM employees WHERE name LIKE '%alice%';

-- FAST option 1: use full-text search (for large tables)
CREATE INDEX idx_emp_name_fts ON employees USING GIN (to_tsvector('english', name));
SELECT * FROM employees WHERE to_tsvector('english', name) @@ to_tsquery('alice');

-- FAST option 2: prefix search (anchored left) uses B-Tree
SELECT * FROM employees WHERE name LIKE 'Alice%';
CREATE INDEX idx_emp_name ON employees (name);
```

### Slow Q3 — NOT IN with potential NULLs + no index

```sql
-- SLOW: NOT IN scans subquery, NULLs cause problems
SELECT name FROM employees
WHERE  dept_id NOT IN (SELECT dept_id FROM departments);

-- FAST: NOT EXISTS + indexes on both join columns
SELECT e.name FROM employees e
WHERE  NOT EXISTS (
    SELECT 1 FROM departments d WHERE d.dept_id = e.dept_id
);

CREATE INDEX idx_dept_id       ON employees   (dept_id);
CREATE INDEX idx_dept_dept_id  ON departments (dept_id);
```

### Slow Q4 — Correlated subquery in SELECT (runs once per row)

```sql
-- SLOW: scalar correlated subquery — one query per employee row
SELECT name,
       (SELECT dept_name FROM departments d WHERE d.dept_id = e.dept_id) AS dept
FROM   employees e;

-- FAST: replace with a JOIN — single pass
SELECT e.name, d.dept_name
FROM   employees   e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

### Slow Q5 — Missing index on JOIN column (FK side)

```sql
-- SLOW: no index on order_items.order_id — forces seq scan of order_items
SELECT o.order_id, SUM(oi.qty * oi.unit_price)
FROM   orders      o
JOIN   order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id;

-- FIX: add index on the FK side
CREATE INDEX idx_oi_order_id ON order_items (order_id);

-- Now the join uses a nested loop with index scan on order_items
```

### Slow Q6 — SELECT * over a wide table with only a few columns needed

```sql
-- SLOW: fetches all columns (heap I/O), can't use index-only scan
SELECT * FROM employees WHERE dept_id = 10 AND is_active = TRUE;

-- FAST: select only needed columns; enable index-only scan
SELECT emp_id, name, salary
FROM   employees
WHERE  dept_id   = 10
  AND  is_active = TRUE;

CREATE INDEX idx_emp_dept_active ON employees (dept_id, is_active)
INCLUDE (emp_id, name, salary);
```

### Slow Q7 — Implicit type conversion on indexed column

```sql
-- SLOW: customer_id is INT but string literal causes type cast — index skipped
SELECT * FROM orders WHERE customer_id = '42';

-- FAST: match the data type exactly
SELECT * FROM orders WHERE customer_id = 42;

-- Same issue in JOINs:
-- SLOW: e.emp_id (INT) joined to o.manager_id (VARCHAR) — cast blocks index
-- FIX: ensure consistent data types, or add a function-based index
```

### Slow Q8 — Aggregation without an index + large table scan

```sql
-- SLOW: full seq scan of order_items to compute totals per product
SELECT product_id, SUM(qty * unit_price) AS revenue
FROM   order_items
GROUP BY product_id
ORDER BY revenue DESC
LIMIT 10;

-- FAST: use a covering composite index
CREATE INDEX idx_oi_product_cover ON order_items (product_id)
INCLUDE (qty, unit_price);

-- Even faster: use a materialized view refreshed nightly
CREATE MATERIALIZED VIEW product_revenue_mv AS
SELECT product_id, SUM(qty * unit_price) AS revenue
FROM   order_items
GROUP BY product_id
WITH DATA;
CREATE UNIQUE INDEX ON product_revenue_mv (product_id);
```

### Slow Q9 — OR condition prevents index use

```sql
-- SLOW: OR across two columns often forces seq scan
SELECT * FROM orders
WHERE  customer_id = 42 OR status = 'Pending';

-- FAST option 1: rewrite as UNION ALL (each branch uses its own index)
SELECT * FROM orders WHERE customer_id = 42
UNION ALL
SELECT * FROM orders WHERE status = 'Pending' AND customer_id <> 42;

-- FAST option 2: separate indexes; planner uses BitmapOr in PostgreSQL
CREATE INDEX idx_ord_custid ON orders (customer_id);
CREATE INDEX idx_ord_status  ON orders (status);
```

### Slow Q10 — ORDER BY + LIMIT without matching index (filesort)

```sql
-- SLOW: must sort entire result before applying LIMIT (filesort / quicksort)
SELECT * FROM orders
WHERE  customer_id = 42
ORDER BY order_date DESC
LIMIT 10;

-- FAST: composite index matches both filter and sort
CREATE INDEX idx_ord_cust_date ON orders (customer_id, order_date DESC);

-- Now: index scan in order → LIMIT stops after 10 rows — no full sort needed
```

---

## 14. Common Interview Questions

**Q1. What is an index and what are the trade-offs of adding one?**  
An index is a separate data structure that maps column values to row locations, enabling fast lookups without full-table scans. Trade-off: reads are faster (O(log N) vs O(N)), but writes (INSERT/UPDATE/DELETE) are slower because every index must be maintained. Indexes also consume storage.

**Q2. Why is the B-Tree the default index type?**  
B-Trees support equality (`=`), ranges (`<`, `>`, `BETWEEN`), prefix matches (`LIKE 'abc%'`), and sorted output — covering nearly all common query patterns. They stay balanced automatically and are efficient for both random lookups and range scans.

**Q3. What is the left-prefix rule for composite indexes?**  
A composite index on (A, B, C) can be used for queries filtering on A, A+B, or A+B+C, but NOT for queries filtering only on B or C (skipping A). Equality columns should come before range columns in the index definition.

**Q4. What is a covering index?**  
An index that includes all columns referenced by a query (both filter and SELECT-list columns). The database can answer the query entirely from the index without accessing the heap — called an index-only scan — which eliminates random I/O.

**Q5. When would the query planner choose a sequential scan over an index scan?**  
When the query returns a large fraction of rows (~10–20%+ depending on storage), a sequential scan is faster because random I/O (index scan heap fetches) is more expensive than sequential I/O. The planner estimates this using table statistics.

**Q6. What is a partial index?**  
An index built only on rows that satisfy a WHERE condition. It is smaller and cheaper to maintain than a full index. Use it for highly filtered queries (e.g., `WHERE status = 'Pending'`) where only a small fraction of rows are queried.

**Q7. How do stale statistics affect query performance?**  
Outdated statistics cause the planner to mis-estimate row counts (cardinality). A query expected to return 10 rows might actually return 10,000 — causing the planner to choose a nested-loop join instead of a hash join. Fix with `ANALYZE` (PostgreSQL) or `UPDATE STATISTICS` (SQL Server).

**Q8. Why does LIKE '%word%' not use a B-Tree index?**  
B-Trees are sorted prefix trees. A leading wildcard means there is no common prefix to traverse — the engine must examine every leaf node, which is equivalent to a sequential scan. Solutions: full-text GIN index, or restrict to prefix search (`LIKE 'word%'`).

**Q9. What is a Bitmap Index Scan in PostgreSQL?**  
PostgreSQL can combine multiple indexes using bitmap operations. Each index produces a bitmap of matching row locations. The bitmaps are ANDed/ORed in memory, then the heap is scanned in physical page order — reducing random I/O compared to a standard index scan with multiple conditions.

**Q10. How do you identify unused indexes?**  
In PostgreSQL: query `pg_stat_user_indexes` for entries where `idx_scan = 0` after a representative workload period. In SQL Server: query `sys.dm_db_index_usage_stats`. Unused indexes waste storage and slow down writes with no read benefit.

**Q11. What is the difference between a clustered and non-clustered index?**  
A **clustered index** determines the physical order of rows on disk — the table IS the index (SQL Server, MySQL InnoDB). There can be only one per table. A **non-clustered index** is a separate structure pointing back to the heap. PostgreSQL has no clustered index concept (CLUSTER command reorders once, but doesn't stay sorted on inserts).

**Q12. Write a query and show how you would diagnose and fix a slow query.**

```sql
-- Slow query: full seq scan
SELECT name, salary FROM employees WHERE dept_id = 10 AND salary > 80000;

-- Step 1: EXPLAIN ANALYZE to see the plan
EXPLAIN ANALYZE SELECT name, salary FROM employees WHERE dept_id = 10 AND salary > 80000;
-- Output: Seq Scan on employees (cost=0.00..1.07 rows=1 ...)

-- Step 2: Add a composite covering index
CREATE INDEX idx_emp_dept_sal_cover
ON employees (dept_id, salary) INCLUDE (name);

-- Step 3: Verify the plan changed
EXPLAIN ANALYZE SELECT name, salary FROM employees WHERE dept_id = 10 AND salary > 80000;
-- Output: Index Only Scan using idx_emp_dept_sal_cover on employees
```

---

## 15. Key Takeaways

| Concept | One-line rule |
|---------|--------------|
| **B-Tree** | Default for almost everything — equality, ranges, sorting, prefix LIKE |
| **Hash index** | O(1) equality only — niche; B-Tree usually preferred |
| **Bitmap index** | Low-cardinality columns in DW/OLAP; beware write contention |
| **GIN** | Full-text, arrays, JSONB containment |
| **GiST** | Geometry, ranges, nearest-neighbour |
| **Composite index** | Left-prefix rule: equality cols first, range col last |
| **Covering index** | Include all SELECT cols in index → index-only scan (fastest) |
| **Partial index** | Index only the rows you query; smaller and cheaper to maintain |
| **Seq Scan** | Wins when returning many rows (>~10%); random I/O is expensive |
| **Index-Only Scan** | No heap access — fastest possible read path |
| **EXPLAIN ANALYZE** | Always verify the planner chose the right plan after adding an index |
| **ANALYZE** | Keep statistics fresh — stale stats = bad plans |

### Index Design Checklist

```
1. Identify the query's WHERE, JOIN, and ORDER BY columns.
2. Are there equality conditions? → Place them first in a composite index.
3. Is there a range condition? → Place it last.
4. Does SELECT list add extra columns? → Add INCLUDE() for a covering index.
5. Is there a common filter (e.g., is_active = TRUE)? → Consider a partial index.
6. Will this index be used? Run EXPLAIN ANALYZE to confirm.
7. Monitor pg_stat_user_indexes — drop indexes with idx_scan = 0.
```

---

*Phase 10 complete. Phase 11 covers Transactions & ACID (`11_transactions_acid.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="10_indexes_and_optimization.md" download="10_indexes_and_optimization.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 10_indexes_and_optimization.md
  </button>
</a>
