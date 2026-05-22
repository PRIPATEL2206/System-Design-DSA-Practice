# Phase 21 — Partitioning & Sharding

> As tables grow from millions to billions of rows, a single monolithic table becomes a bottleneck — queries slow down, maintenance windows balloon, and storage limits loom. **Partitioning** splits a table into smaller, manageable pieces within a single database instance. **Sharding** distributes those pieces across multiple database instances. Both are divide-and-conquer strategies, but they operate at very different levels.

---

## Table of Contents

1. [Why Partition?](#1-why-partition)
2. [Table Partitioning Overview](#2-table-partitioning-overview)
3. [Range Partitioning](#3-range-partitioning)
4. [List Partitioning](#4-list-partitioning)
5. [Hash Partitioning](#5-hash-partitioning)
6. [Composite (Sub-)Partitioning](#6-composite-partitioning)
7. [Partition Pruning & Performance](#7-partition-pruning)
8. [Partition Management Operations](#8-partition-management)
9. [Horizontal vs Vertical Partitioning](#9-horizontal-vs-vertical)
10. [Sharding — Distributing Across Nodes](#10-sharding)
11. [Sharding vs Partitioning](#11-sharding-vs-partitioning)
12. [Worked Examples](#12-worked-examples)
13. [Common Interview Questions](#13-interview-questions)
14. [Key Takeaways](#14-key-takeaways)
15. [Download](#15-download)

---

## 1. Why Partition?

### The Problem with Large Tables

| Problem | Impact |
|---|---|
| **Full table scans** | A query touching 1 billion rows when only 1 month of data is needed |
| **Index bloat** | B-Tree indexes grow deep, increasing lookup time and memory pressure |
| **Maintenance windows** | `VACUUM`, `ANALYZE`, `REINDEX` on a 2 TB table takes hours |
| **Bulk data loading** | Loading / archiving monthly data requires touching the entire table |
| **Lock contention** | DDL operations (e.g., adding a column) lock the whole table |
| **Backup granularity** | Can't back up just the "hot" recent data separately |

### Real-World Analogy

Imagine a library with **one enormous bookshelf** holding every book ever. Finding a specific book means scanning the entire shelf. Now imagine the library is reorganised into **rooms by genre** (list partitioning), **shelves by publication year** (range partitioning), or **sections by the first letter of the author's last name** (hash partitioning). Each approach makes it far faster to find what you need, because you only search the relevant section.

<svg viewBox="0 0 780 280" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Why Partition? — Before vs After</text>

  <!-- Before -->
  <rect x="30" y="50" width="320" height="200" rx="12" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="190" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">Before: Monolithic Table</text>

  <rect x="55" y="92" width="270" height="140" rx="6" fill="#ef9a9a"/>
  <text x="190" y="130" text-anchor="middle" font-size="11" fill="#333">orders (2 billion rows)</text>
  <text x="190" y="152" text-anchor="middle" font-size="10" fill="#555">Full scan for any date range query</text>
  <text x="190" y="170" text-anchor="middle" font-size="10" fill="#555">VACUUM takes 6+ hours</text>
  <text x="190" y="188" text-anchor="middle" font-size="10" fill="#555">Index rebuild locks the table</text>
  <text x="190" y="206" text-anchor="middle" font-size="10" fill="#555">Can't archive old data efficiently</text>

  <!-- Arrow -->
  <text x="390" y="155" text-anchor="middle" font-size="22" fill="#666">→</text>

  <!-- After -->
  <rect x="430" y="50" width="320" height="200" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="590" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">After: Partitioned Table</text>

  <rect x="450" y="90" width="120" height="32" rx="4" fill="#a5d6a7"/>
  <text x="510" y="111" text-anchor="middle" font-size="9" fill="#333">orders_2025_q1</text>
  <rect x="580" y="90" width="120" height="32" rx="4" fill="#a5d6a7"/>
  <text x="640" y="111" text-anchor="middle" font-size="9" fill="#333">orders_2025_q2</text>
  <rect x="450" y="130" width="120" height="32" rx="4" fill="#c8e6c9"/>
  <text x="510" y="151" text-anchor="middle" font-size="9" fill="#333">orders_2024_q3</text>
  <rect x="580" y="130" width="120" height="32" rx="4" fill="#c8e6c9"/>
  <text x="640" y="151" text-anchor="middle" font-size="9" fill="#333">orders_2024_q4</text>
  <rect x="450" y="170" width="120" height="32" rx="4" fill="#e8f5e9"/>
  <text x="510" y="191" text-anchor="middle" font-size="9" fill="#888">orders_2024_q1</text>
  <rect x="580" y="170" width="120" height="32" rx="4" fill="#e8f5e9"/>
  <text x="640" y="191" text-anchor="middle" font-size="9" fill="#888">orders_2024_q2</text>

  <text x="590" y="222" text-anchor="middle" font-size="10" fill="#2e7d32">Query scans only relevant partitions</text>
  <text x="590" y="238" text-anchor="middle" font-size="10" fill="#2e7d32">VACUUM per partition — minutes, not hours</text>
</svg>

---

## 2. Table Partitioning Overview

Table partitioning divides a logical table into **physical sub-tables** (partitions) based on a **partition key**. The database engine routes queries and DML to the correct partition(s) transparently — the application sees a single table.

### Partitioning Strategies

| Strategy | Partition Key | How Rows Are Routed | Best For |
|---|---|---|---|
| **Range** | Continuous value (date, id range) | Value falls within a defined range | Time-series, logs, orders by date |
| **List** | Discrete value (country, status) | Value matches an explicit list | Regional data, categorical data |
| **Hash** | Any column | `hash(value) % N` | Even distribution when no natural range |
| **Composite** | Two strategies combined | Range + Hash, List + Range, etc. | Large-scale multi-dimensional partitioning |

### RDBMS Support

| Feature | PostgreSQL | MySQL | Oracle | SQL Server |
|---|---|---|---|---|
| Range | Declarative (10+) | Native | Native | Partition function |
| List | Declarative (10+) | Native | Native | Partition function |
| Hash | Declarative (11+) | Native | Native | — |
| Composite | Range + List/Hash (11+) | Range + Hash | Any combo | Limited |
| Auto-partition | pg_partman extension | AUTO for range (8.0.13+) | Interval partitioning | — |
| Partition pruning | Automatic | Automatic | Automatic | Automatic |

<svg viewBox="0 0 780 320" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Partitioning Strategies Overview</text>

  <!-- Range -->
  <rect x="20" y="50" width="230" height="250" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="135" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Range Partitioning</text>

  <rect x="40" y="90" width="190" height="28" rx="4" fill="#bbdefb"/>
  <text x="135" y="109" text-anchor="middle" font-size="10" fill="#333">2025-01 to 2025-03</text>
  <rect x="40" y="124" width="190" height="28" rx="4" fill="#90caf9"/>
  <text x="135" y="143" text-anchor="middle" font-size="10" fill="#333">2025-04 to 2025-06</text>
  <rect x="40" y="158" width="190" height="28" rx="4" fill="#64b5f6"/>
  <text x="135" y="177" text-anchor="middle" font-size="10" fill="#fff">2025-07 to 2025-09</text>
  <rect x="40" y="192" width="190" height="28" rx="4" fill="#42a5f5"/>
  <text x="135" y="211" text-anchor="middle" font-size="10" fill="#fff">2025-10 to 2025-12</text>

  <text x="135" y="240" text-anchor="middle" font-size="9" fill="#555">Ordered, continuous ranges</text>
  <text x="135" y="255" text-anchor="middle" font-size="9" fill="#555">Best for: time-series data</text>
  <text x="135" y="270" text-anchor="middle" font-size="9" fill="#555">Risk: hot partition (current period)</text>

  <!-- List -->
  <rect x="275" y="50" width="230" height="250" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="390" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">List Partitioning</text>

  <rect x="295" y="90" width="190" height="28" rx="4" fill="#c8e6c9"/>
  <text x="390" y="109" text-anchor="middle" font-size="10" fill="#333">region = 'NA'</text>
  <rect x="295" y="124" width="190" height="28" rx="4" fill="#a5d6a7"/>
  <text x="390" y="143" text-anchor="middle" font-size="10" fill="#333">region = 'EU'</text>
  <rect x="295" y="158" width="190" height="28" rx="4" fill="#81c784"/>
  <text x="390" y="177" text-anchor="middle" font-size="10" fill="#fff">region = 'APAC'</text>
  <rect x="295" y="192" width="190" height="28" rx="4" fill="#66bb6a"/>
  <text x="390" y="211" text-anchor="middle" font-size="10" fill="#fff">region = 'LATAM'</text>

  <text x="390" y="240" text-anchor="middle" font-size="9" fill="#555">Discrete, enumerated values</text>
  <text x="390" y="255" text-anchor="middle" font-size="9" fill="#555">Best for: categorical / regional</text>
  <text x="390" y="270" text-anchor="middle" font-size="9" fill="#555">Risk: uneven distribution</text>

  <!-- Hash -->
  <rect x="530" y="50" width="230" height="250" rx="12" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="645" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Hash Partitioning</text>

  <rect x="550" y="90" width="190" height="28" rx="4" fill="#ffe0b2"/>
  <text x="645" y="109" text-anchor="middle" font-size="10" fill="#333">hash(user_id) % 4 = 0</text>
  <rect x="550" y="124" width="190" height="28" rx="4" fill="#ffcc80"/>
  <text x="645" y="143" text-anchor="middle" font-size="10" fill="#333">hash(user_id) % 4 = 1</text>
  <rect x="550" y="158" width="190" height="28" rx="4" fill="#ffb74d"/>
  <text x="645" y="177" text-anchor="middle" font-size="10" fill="#fff">hash(user_id) % 4 = 2</text>
  <rect x="550" y="192" width="190" height="28" rx="4" fill="#ffa726"/>
  <text x="645" y="211" text-anchor="middle" font-size="10" fill="#fff">hash(user_id) % 4 = 3</text>

  <text x="645" y="240" text-anchor="middle" font-size="9" fill="#555">Even distribution guaranteed</text>
  <text x="645" y="255" text-anchor="middle" font-size="9" fill="#555">Best for: no natural range key</text>
  <text x="645" y="270" text-anchor="middle" font-size="9" fill="#555">Risk: no range-scan pruning</text>
</svg>

---

## 3. Range Partitioning

Range partitioning divides rows based on **value ranges** of the partition key. Typically used with dates, timestamps, or sequential IDs.

### PostgreSQL — Declarative Range Partitioning

```sql
-- Parent table (no data stored here directly)
CREATE TABLE orders (
    order_id    BIGSERIAL,
    customer_id INT         NOT NULL,
    order_date  DATE        NOT NULL,
    total       NUMERIC(12,2),
    status      VARCHAR(20)
) PARTITION BY RANGE (order_date);

-- Quarterly partitions for 2025
CREATE TABLE orders_2025_q1 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

CREATE TABLE orders_2025_q2 PARTITION OF orders
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

CREATE TABLE orders_2025_q3 PARTITION OF orders
    FOR VALUES FROM ('2025-07-01') TO ('2025-10-01');

CREATE TABLE orders_2025_q4 PARTITION OF orders
    FOR VALUES FROM ('2025-10-01') TO ('2026-01-01');

-- Default partition for anything outside defined ranges
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- Index (created on parent, auto-propagated to partitions)
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_date     ON orders (order_date);
```

> **Note**: `FROM` is inclusive, `TO` is exclusive. `FROM ('2025-01-01') TO ('2025-04-01')` includes Jan 1 through Mar 31.

### MySQL — Range Partitioning

```sql
CREATE TABLE orders (
    order_id    BIGINT AUTO_INCREMENT,
    customer_id INT         NOT NULL,
    order_date  DATE        NOT NULL,
    total       DECIMAL(12,2),
    status      VARCHAR(20),
    PRIMARY KEY (order_id, order_date)  -- partition key must be in PK
) PARTITION BY RANGE (YEAR(order_date) * 100 + MONTH(order_date)) (
    PARTITION p2025_q1 VALUES LESS THAN (202504),
    PARTITION p2025_q2 VALUES LESS THAN (202507),
    PARTITION p2025_q3 VALUES LESS THAN (202510),
    PARTITION p2025_q4 VALUES LESS THAN (202601),
    PARTITION p_future  VALUES LESS THAN MAXVALUE
);
```

### SQL Server — Range Partitioning

```sql
-- Step 1: Create a partition function
CREATE PARTITION FUNCTION pf_order_date (DATE)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01'
);

-- Step 2: Create a partition scheme (maps function → filegroups)
CREATE PARTITION SCHEME ps_order_date
AS PARTITION pf_order_date
TO (fg_archive, fg_q1, fg_q2, fg_q3, fg_q4);

-- Step 3: Create the table on the partition scheme
CREATE TABLE orders (
    order_id    BIGINT IDENTITY(1,1),
    customer_id INT         NOT NULL,
    order_date  DATE        NOT NULL,
    total       DECIMAL(12,2),
    status      VARCHAR(20)
) ON ps_order_date (order_date);

-- Step 4: Create aligned index
CREATE INDEX idx_orders_date ON orders (order_date)
ON ps_order_date (order_date);
```

---

## 4. List Partitioning

List partitioning routes rows based on a **discrete set of values**. Ideal for categorical data like region, country, status, or department.

### PostgreSQL

```sql
CREATE TABLE sales (
    sale_id     BIGSERIAL,
    region      VARCHAR(10) NOT NULL,
    sale_date   DATE        NOT NULL,
    amount      NUMERIC(12,2)
) PARTITION BY LIST (region);

CREATE TABLE sales_na    PARTITION OF sales FOR VALUES IN ('NA');
CREATE TABLE sales_eu    PARTITION OF sales FOR VALUES IN ('EU');
CREATE TABLE sales_apac  PARTITION OF sales FOR VALUES IN ('APAC');
CREATE TABLE sales_latam PARTITION OF sales FOR VALUES IN ('LATAM');
CREATE TABLE sales_other PARTITION OF sales DEFAULT;
```

### MySQL

```sql
CREATE TABLE sales (
    sale_id   BIGINT AUTO_INCREMENT,
    region    VARCHAR(10) NOT NULL,
    sale_date DATE        NOT NULL,
    amount    DECIMAL(12,2),
    PRIMARY KEY (sale_id, region)
) PARTITION BY LIST COLUMNS (region) (
    PARTITION p_na    VALUES IN ('NA'),
    PARTITION p_eu    VALUES IN ('EU'),
    PARTITION p_apac  VALUES IN ('APAC'),
    PARTITION p_latam VALUES IN ('LATAM')
);
```

### Oracle — Automatic List Partitioning

```sql
CREATE TABLE sales (
    sale_id   NUMBER GENERATED ALWAYS AS IDENTITY,
    region    VARCHAR2(10) NOT NULL,
    sale_date DATE         NOT NULL,
    amount    NUMBER(12,2)
) PARTITION BY LIST (region) AUTOMATIC (
    PARTITION p_na    VALUES ('NA'),
    PARTITION p_eu    VALUES ('EU')
    -- New regions get auto-created partitions
);
```

---

## 5. Hash Partitioning

Hash partitioning applies a **hash function** to the partition key and distributes rows evenly across N partitions. Useful when there is no natural range or list and the goal is even distribution.

### PostgreSQL

```sql
CREATE TABLE user_sessions (
    session_id  UUID        DEFAULT gen_random_uuid(),
    user_id     INT         NOT NULL,
    created_at  TIMESTAMP   NOT NULL,
    data        JSONB
) PARTITION BY HASH (user_id);

-- 8 hash partitions (power of 2 recommended for even split)
CREATE TABLE user_sessions_p0 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE user_sessions_p1 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);
CREATE TABLE user_sessions_p2 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 2);
CREATE TABLE user_sessions_p3 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 3);
CREATE TABLE user_sessions_p4 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 4);
CREATE TABLE user_sessions_p5 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 5);
CREATE TABLE user_sessions_p6 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 6);
CREATE TABLE user_sessions_p7 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 7);
```

### MySQL

```sql
CREATE TABLE user_sessions (
    session_id  CHAR(36)    NOT NULL,
    user_id     INT         NOT NULL,
    created_at  DATETIME    NOT NULL,
    data        JSON,
    PRIMARY KEY (session_id, user_id)
) PARTITION BY HASH (user_id) PARTITIONS 8;
```

### Hash Partitioning Trade-offs

| Advantage | Disadvantage |
|---|---|
| Guarantees even row distribution | No range-scan pruning (must scan all partitions for range queries) |
| No hot partition problem | Adding/removing partitions requires rehashing and data movement |
| Simple to configure | Partition key must be in queries for point-lookup pruning |
| Good for high-concurrency writes | Cross-partition queries still touch all partitions |

---

## 6. Composite (Sub-)Partitioning

Composite partitioning applies **two levels** of partitioning — e.g., range by date at the first level, then hash by user_id at the second. This addresses scenarios where a single strategy doesn't balance both query patterns and data distribution.

### PostgreSQL — Range + Hash

```sql
-- Level 1: Range by order_date
CREATE TABLE orders (
    order_id    BIGSERIAL,
    customer_id INT         NOT NULL,
    order_date  DATE        NOT NULL,
    total       NUMERIC(12,2)
) PARTITION BY RANGE (order_date);

-- Level 2: Each range partition is sub-partitioned by hash
CREATE TABLE orders_2025_q1 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01')
    PARTITION BY HASH (customer_id);

CREATE TABLE orders_2025_q1_p0 PARTITION OF orders_2025_q1
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_2025_q1_p1 PARTITION OF orders_2025_q1
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_2025_q1_p2 PARTITION OF orders_2025_q1
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_2025_q1_p3 PARTITION OF orders_2025_q1
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Repeat for Q2, Q3, Q4...
```

### Oracle — Range + List

```sql
CREATE TABLE sales (
    sale_id     NUMBER,
    region      VARCHAR2(10),
    sale_date   DATE,
    amount      NUMBER(12,2)
)
PARTITION BY RANGE (sale_date)
SUBPARTITION BY LIST (region) (
    PARTITION p_2025_q1 VALUES LESS THAN (DATE '2025-04-01') (
        SUBPARTITION p_2025_q1_na    VALUES ('NA'),
        SUBPARTITION p_2025_q1_eu    VALUES ('EU'),
        SUBPARTITION p_2025_q1_apac  VALUES ('APAC')
    ),
    PARTITION p_2025_q2 VALUES LESS THAN (DATE '2025-07-01') (
        SUBPARTITION p_2025_q2_na    VALUES ('NA'),
        SUBPARTITION p_2025_q2_eu    VALUES ('EU'),
        SUBPARTITION p_2025_q2_apac  VALUES ('APAC')
    )
);
```

<svg viewBox="0 0 780 340" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Composite Partitioning — Range + Hash</text>

  <!-- Parent -->
  <rect x="280" y="48" width="220" height="40" rx="10" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="390" y="73" text-anchor="middle" font-size="12" font-weight="bold" fill="#3949ab">orders (parent)</text>

  <!-- Lines down -->
  <line x1="290" y1="88" x2="130" y2="120" stroke="#666" stroke-width="1.5"/>
  <line x1="390" y1="88" x2="390" y2="120" stroke="#666" stroke-width="1.5"/>
  <line x1="490" y1="88" x2="650" y2="120" stroke="#666" stroke-width="1.5"/>

  <!-- Range partitions -->
  <rect x="40" y="120" width="180" height="35" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="130" y="143" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">Q1 (Jan–Mar)</text>

  <rect x="300" y="120" width="180" height="35" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="390" y="143" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">Q2 (Apr–Jun)</text>

  <rect x="560" y="120" width="180" height="35" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="650" y="143" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">Q3 (Jul–Sep)</text>

  <text x="60" y="113" font-size="9" fill="#1565c0">RANGE by order_date</text>

  <!-- Hash sub-partitions for Q1 -->
  <line x1="85" y1="155" x2="55" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="115" y1="155" x2="100" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="145" y1="155" x2="155" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="175" y1="155" x2="210" y2="185" stroke="#e65100" stroke-width="1"/>

  <rect x="20" y="185" width="55" height="30" rx="4" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="47" y="205" text-anchor="middle" font-size="8" fill="#333">h0</text>
  <rect x="80" y="185" width="55" height="30" rx="4" fill="#ffe0b2" stroke="#e65100" stroke-width="1.5"/>
  <text x="107" y="205" text-anchor="middle" font-size="8" fill="#333">h1</text>
  <rect x="140" y="185" width="55" height="30" rx="4" fill="#ffcc80" stroke="#e65100" stroke-width="1.5"/>
  <text x="167" y="205" text-anchor="middle" font-size="8" fill="#333">h2</text>
  <rect x="200" y="185" width="55" height="30" rx="4" fill="#ffb74d" stroke="#e65100" stroke-width="1.5"/>
  <text x="227" y="205" text-anchor="middle" font-size="8" fill="#fff">h3</text>

  <!-- Hash sub-partitions for Q2 -->
  <line x1="345" y1="155" x2="315" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="375" y1="155" x2="360" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="405" y1="155" x2="415" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="435" y1="155" x2="470" y2="185" stroke="#e65100" stroke-width="1"/>

  <rect x="280" y="185" width="55" height="30" rx="4" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="307" y="205" text-anchor="middle" font-size="8" fill="#333">h0</text>
  <rect x="340" y="185" width="55" height="30" rx="4" fill="#ffe0b2" stroke="#e65100" stroke-width="1.5"/>
  <text x="367" y="205" text-anchor="middle" font-size="8" fill="#333">h1</text>
  <rect x="400" y="185" width="55" height="30" rx="4" fill="#ffcc80" stroke="#e65100" stroke-width="1.5"/>
  <text x="427" y="205" text-anchor="middle" font-size="8" fill="#333">h2</text>
  <rect x="460" y="185" width="55" height="30" rx="4" fill="#ffb74d" stroke="#e65100" stroke-width="1.5"/>
  <text x="487" y="205" text-anchor="middle" font-size="8" fill="#fff">h3</text>

  <!-- Hash sub-partitions for Q3 -->
  <line x1="605" y1="155" x2="575" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="635" y1="155" x2="620" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="665" y1="155" x2="675" y2="185" stroke="#e65100" stroke-width="1"/>
  <line x1="695" y1="155" x2="730" y2="185" stroke="#e65100" stroke-width="1"/>

  <rect x="540" y="185" width="55" height="30" rx="4" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="567" y="205" text-anchor="middle" font-size="8" fill="#333">h0</text>
  <rect x="600" y="185" width="55" height="30" rx="4" fill="#ffe0b2" stroke="#e65100" stroke-width="1.5"/>
  <text x="627" y="205" text-anchor="middle" font-size="8" fill="#333">h1</text>
  <rect x="660" y="185" width="55" height="30" rx="4" fill="#ffcc80" stroke="#e65100" stroke-width="1.5"/>
  <text x="687" y="205" text-anchor="middle" font-size="8" fill="#333">h2</text>
  <rect x="720" y="185" width="55" height="30" rx="4" fill="#ffb74d" stroke="#e65100" stroke-width="1.5"/>
  <text x="747" y="205" text-anchor="middle" font-size="8" fill="#fff">h3</text>

  <text x="40" y="235" font-size="9" fill="#e65100">HASH by customer_id (4 buckets per quarter)</text>

  <!-- Benefits -->
  <rect x="40" y="255" width="700" height="70" rx="8" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="390" y="278" text-anchor="middle" font-size="11" font-weight="bold" fill="#333">Benefits of Composite Partitioning</text>
  <text x="60" y="298" font-size="9" fill="#555">Range pruning: WHERE order_date = '2025-02-15' → scans only Q1 sub-partitions</text>
  <text x="60" y="314" font-size="9" fill="#555">Hash distribution: Within Q1, rows are evenly spread across h0–h3, reducing per-partition scan size by 4x</text>
</svg>

---

## 7. Partition Pruning & Performance

**Partition pruning** (also called partition elimination) is the optimiser's ability to skip partitions that cannot contain matching rows. This is the single biggest performance benefit of partitioning.

### How Pruning Works

```sql
-- The query references the partition key with a constant filter:
SELECT * FROM orders WHERE order_date = '2025-05-15';
-- Optimiser knows: May 15 falls in Q2 → scan ONLY orders_2025_q2

-- Range scan also prunes:
SELECT * FROM orders WHERE order_date BETWEEN '2025-01-01' AND '2025-03-31';
-- Scans only orders_2025_q1

-- NO pruning if partition key is wrapped in a function:
SELECT * FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2025;
-- ❌ Cannot prune — scans ALL partitions
```

### Verifying Pruning with EXPLAIN

```sql
-- PostgreSQL
EXPLAIN (COSTS OFF)
SELECT * FROM orders WHERE order_date = '2025-05-15';
```

```
Append
  ->  Seq Scan on orders_2025_q2 orders_1
        Filter: (order_date = '2025-05-15'::date)
```

Only `orders_2025_q2` is scanned — all other partitions pruned.

### Pruning Rules

| Scenario | Pruning? | Why |
|---|---|---|
| `WHERE order_date = '2025-05-15'` | Yes | Direct match on partition key |
| `WHERE order_date BETWEEN '2025-01-01' AND '2025-06-30'` | Yes | Range overlaps Q1 + Q2 only |
| `WHERE order_date >= '2025-10-01'` | Yes | Only Q4 matches |
| `WHERE YEAR(order_date) = 2025` | **No** | Function wraps partition key |
| `WHERE order_date = some_column` | Partial | Runtime pruning (PG 11+) if join pushdown |
| `WHERE customer_id = 42` (non-partition key) | **No** | Must scan all partitions |
| Hash: `WHERE user_id = 42` | Yes | Hash computed at query time |
| Hash: `WHERE user_id BETWEEN 1 AND 100` | **No** | Hash doesn't support range pruning |

### Performance Impact — Benchmark Pattern

| Table Size | Partitioned (pruned) | Non-Partitioned | Speedup |
|---|---|---|---|
| 100M rows, query hits 1 quarter | ~25M rows scanned | 100M rows scanned | ~4x |
| 1B rows, query hits 1 month | ~83M rows scanned | 1B rows scanned | ~12x |
| 1B rows with composite (range+hash×8) | ~10M rows scanned | 1B rows scanned | ~100x |

### Partition-Wise Joins (PostgreSQL 12+)

When two tables are partitioned on the same key, the optimiser can join them **partition by partition**, reducing memory and enabling parallel execution:

```sql
SET enable_partitionwise_join = on;

-- Both tables partitioned by order_date with matching boundaries
SELECT o.order_id, li.product_id, li.quantity
FROM orders o
JOIN line_items li ON o.order_id = li.order_id
                  AND o.order_date = li.order_date
WHERE o.order_date BETWEEN '2025-01-01' AND '2025-03-31';
-- Joins only Q1 partitions from both tables
```

### Partition-Wise Aggregation

```sql
SET enable_partitionwise_aggregate = on;

SELECT DATE_TRUNC('month', order_date) AS month,
       SUM(total) AS revenue
FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY 1;
-- Each partition aggregated independently, results merged
```

---

## 8. Partition Management Operations

### Adding New Partitions

```sql
-- PostgreSQL: Create future partitions ahead of time
CREATE TABLE orders_2026_q1 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');

-- MySQL: Add a new partition before MAXVALUE
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2026_q1 VALUES LESS THAN (202604),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- SQL Server: Split a range
ALTER PARTITION FUNCTION pf_order_date()
SPLIT RANGE ('2026-01-01');
```

### Detaching / Dropping Old Partitions (Archiving)

```sql
-- PostgreSQL: Detach (keeps data, removes from partition tree)
ALTER TABLE orders DETACH PARTITION orders_2023_q1;
-- Now orders_2023_q1 is a standalone table — archive or drop it

-- PostgreSQL: Drop (instant — no vacuum needed)
DROP TABLE orders_2023_q1;

-- MySQL: Drop partition (instant)
ALTER TABLE orders DROP PARTITION p2023_q1;

-- SQL Server: Merge ranges (empties the partition, then merge boundary)
ALTER PARTITION FUNCTION pf_order_date()
MERGE RANGE ('2023-04-01');
```

> **Key Insight**: Dropping a partition is **O(1)** — instantaneous compared to `DELETE FROM orders WHERE order_date < '2023-04-01'` which would generate billions of WAL records and leave dead tuples requiring vacuum.

### Automatic Partition Creation

```sql
-- PostgreSQL with pg_partman extension
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
    p_parent_table  := 'public.orders',
    p_control       := 'order_date',
    p_type          := 'range',
    p_interval      := '1 month',
    p_premake       := 3   -- create 3 future partitions ahead
);

-- Runs periodically (cron job or pg_cron)
SELECT partman.run_maintenance();
```

---

## 9. Horizontal vs Vertical Partitioning

<svg viewBox="0 0 800 420" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Horizontal vs Vertical Partitioning</text>

  <!-- Original table -->
  <text x="400" y="52" text-anchor="middle" font-size="12" font-weight="bold" fill="#555">Original Table</text>
  <rect x="200" y="60" width="400" height="28" rx="4" fill="#e8eaf6" stroke="#3949ab" stroke-width="1.5"/>
  <text x="240" y="79" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">id</text>
  <text x="300" y="79" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">name</text>
  <text x="370" y="79" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">email</text>
  <text x="450" y="79" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">bio</text>
  <text x="540" y="79" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">avatar_blob</text>

  <rect x="200" y="88" width="400" height="20" fill="#f5f5f5" stroke="#ccc" stroke-width="0.5"/>
  <text x="240" y="102" text-anchor="middle" font-size="8" fill="#333">1</text>
  <text x="300" y="102" text-anchor="middle" font-size="8" fill="#333">Alice</text>
  <text x="370" y="102" text-anchor="middle" font-size="8" fill="#333">alice@co</text>
  <text x="450" y="102" text-anchor="middle" font-size="8" fill="#333">Long text...</text>
  <text x="540" y="102" text-anchor="middle" font-size="8" fill="#333">10 MB blob</text>

  <rect x="200" y="108" width="400" height="20" fill="#fafafa" stroke="#ccc" stroke-width="0.5"/>
  <text x="240" y="122" text-anchor="middle" font-size="8" fill="#333">2</text>
  <text x="300" y="122" text-anchor="middle" font-size="8" fill="#333">Bob</text>
  <text x="370" y="122" text-anchor="middle" font-size="8" fill="#333">bob@co</text>
  <text x="450" y="122" text-anchor="middle" font-size="8" fill="#333">Long text...</text>
  <text x="540" y="122" text-anchor="middle" font-size="8" fill="#333">8 MB blob</text>

  <rect x="200" y="128" width="400" height="20" fill="#f5f5f5" stroke="#ccc" stroke-width="0.5"/>
  <text x="240" y="142" text-anchor="middle" font-size="8" fill="#333">3</text>
  <text x="300" y="142" text-anchor="middle" font-size="8" fill="#333">Carol</text>
  <text x="370" y="142" text-anchor="middle" font-size="8" fill="#333">carol@co</text>
  <text x="450" y="142" text-anchor="middle" font-size="8" fill="#333">Long text...</text>
  <text x="540" y="142" text-anchor="middle" font-size="8" fill="#333">12 MB blob</text>

  <!-- Horizontal (left) -->
  <text x="160" y="185" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Horizontal Partitioning</text>
  <text x="160" y="200" text-anchor="middle" font-size="9" fill="#555">Split by ROWS</text>

  <rect x="20" y="212" width="280" height="18" rx="3" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="160" y="226" text-anchor="middle" font-size="8" fill="#333">Partition A: rows where id % 2 = 0</text>
  <rect x="20" y="232" width="280" height="18" fill="#bbdefb" stroke="#ccc" stroke-width="0.5"/>
  <text x="160" y="245" text-anchor="middle" font-size="8" fill="#333">2 | Bob | bob@co | Long text... | 8 MB</text>

  <rect x="20" y="262" width="280" height="18" rx="3" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="160" y="276" text-anchor="middle" font-size="8" fill="#333">Partition B: rows where id % 2 = 1</text>
  <rect x="20" y="282" width="280" height="18" fill="#bbdefb" stroke="#ccc" stroke-width="0.5"/>
  <text x="160" y="295" text-anchor="middle" font-size="8" fill="#333">1 | Alice | alice@co | Long text... | 10 MB</text>
  <rect x="20" y="300" width="280" height="18" fill="#bbdefb" stroke="#ccc" stroke-width="0.5"/>
  <text x="160" y="313" text-anchor="middle" font-size="8" fill="#333">3 | Carol | carol@co | Long text... | 12 MB</text>

  <rect x="20" y="332" width="280" height="55" rx="6" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="30" y="350" font-size="9" fill="#555">Each partition has ALL columns</text>
  <text x="30" y="365" font-size="9" fill="#555">but only a SUBSET of rows.</text>
  <text x="30" y="380" font-size="9" fill="#1565c0">This is what "partitioning" usually means.</text>

  <!-- Vertical (right) -->
  <text x="610" y="185" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Vertical Partitioning</text>
  <text x="610" y="200" text-anchor="middle" font-size="9" fill="#555">Split by COLUMNS</text>

  <rect x="430" y="212" width="160" height="18" rx="3" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="510" y="226" text-anchor="middle" font-size="8" fill="#333">users_core (hot columns)</text>
  <rect x="430" y="232" width="160" height="54" fill="#ffe0b2" stroke="#ccc" stroke-width="0.5"/>
  <text x="440" y="246" font-size="8" fill="#333">id | name    | email</text>
  <text x="440" y="259" font-size="8" fill="#333">1  | Alice   | alice@co</text>
  <text x="440" y="272" font-size="8" fill="#333">2  | Bob     | bob@co</text>

  <rect x="610" y="212" width="170" height="18" rx="3" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="695" y="226" text-anchor="middle" font-size="8" fill="#333">users_extended (cold columns)</text>
  <rect x="610" y="232" width="170" height="54" fill="#ffe0b2" stroke="#ccc" stroke-width="0.5"/>
  <text x="620" y="246" font-size="8" fill="#333">id | bio           | avatar_blob</text>
  <text x="620" y="259" font-size="8" fill="#333">1  | Long text...  | 10 MB</text>
  <text x="620" y="272" font-size="8" fill="#333">2  | Long text...  | 8 MB</text>

  <rect x="430" y="332" width="350" height="55" rx="6" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="440" y="350" font-size="9" fill="#555">Each partition has ALL rows</text>
  <text x="440" y="365" font-size="9" fill="#555">but only a SUBSET of columns.</text>
  <text x="440" y="380" font-size="9" fill="#e65100">Joined by PK when full row is needed.</text>
</svg>

### Horizontal Partitioning

| Aspect | Details |
|---|---|
| **What it does** | Splits rows into separate physical tables |
| **Same as** | Traditional "partitioning" (range, list, hash) |
| **Use case** | Large row counts — query only the relevant subset |
| **Application change** | None (DB routes transparently) |

### Vertical Partitioning

| Aspect | Details |
|---|---|
| **What it does** | Splits columns into separate tables, joined by PK |
| **Same as** | Schema decomposition, "cold/hot column split" |
| **Use case** | Wide tables with seldom-accessed BLOBs or text columns |
| **Application change** | May need JOIN to reassemble full row |

### Example — Vertical Partitioning in Practice

```sql
-- Original: one wide table
CREATE TABLE users (
    user_id    INT PRIMARY KEY,
    name       VARCHAR(100),
    email      VARCHAR(200),
    bio        TEXT,              -- rarely queried, large
    avatar     BYTEA              -- rarely queried, very large
);

-- After vertical partitioning:
CREATE TABLE users_core (
    user_id    INT PRIMARY KEY,
    name       VARCHAR(100),
    email      VARCHAR(200)
);

CREATE TABLE users_profile (
    user_id    INT PRIMARY KEY REFERENCES users_core(user_id),
    bio        TEXT,
    avatar     BYTEA
);

-- Hot-path query (fast — small rows, fits in cache):
SELECT name, email FROM users_core WHERE user_id = 42;

-- Full profile (only when needed):
SELECT c.name, c.email, p.bio
FROM users_core c
JOIN users_profile p ON c.user_id = p.user_id
WHERE c.user_id = 42;
```

---

## 10. Sharding — Distributing Across Nodes

**Sharding** (also called horizontal scaling or "shared-nothing" partitioning) distributes data across **multiple independent database instances** (nodes), each holding a subset of the total data.

### Sharding vs Partitioning

<svg viewBox="0 0 800 320" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Partitioning vs Sharding</text>

  <!-- Partitioning (left) -->
  <rect x="30" y="50" width="330" height="250" rx="14" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="195" y="78" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Partitioning</text>
  <text x="195" y="96" text-anchor="middle" font-size="10" fill="#555">Single database instance</text>

  <rect x="55" y="110" width="280" height="170" rx="8" fill="white" stroke="#999" stroke-width="1.5"/>
  <text x="195" y="130" text-anchor="middle" font-size="10" font-weight="bold" fill="#333">PostgreSQL Instance</text>
  <rect x="70" y="142" width="120" height="30" rx="4" fill="#bbdefb"/>
  <text x="130" y="162" text-anchor="middle" font-size="9" fill="#333">orders_q1</text>
  <rect x="200" y="142" width="120" height="30" rx="4" fill="#90caf9"/>
  <text x="260" y="162" text-anchor="middle" font-size="9" fill="#333">orders_q2</text>
  <rect x="70" y="180" width="120" height="30" rx="4" fill="#64b5f6"/>
  <text x="130" y="200" text-anchor="middle" font-size="9" fill="#fff">orders_q3</text>
  <rect x="200" y="180" width="120" height="30" rx="4" fill="#42a5f5"/>
  <text x="260" y="200" text-anchor="middle" font-size="9" fill="#fff">orders_q4</text>
  <text x="195" y="236" text-anchor="middle" font-size="9" fill="#555">All partitions share one CPU, RAM, disk</text>
  <text x="195" y="252" text-anchor="middle" font-size="9" fill="#555">Single point of failure</text>
  <text x="195" y="268" text-anchor="middle" font-size="9" fill="#555">Cross-partition queries are local</text>

  <!-- Arrow -->
  <text x="400" y="185" text-anchor="middle" font-size="16" fill="#666">vs</text>

  <!-- Sharding (right) -->
  <rect x="440" y="50" width="340" height="250" rx="14" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="610" y="78" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Sharding</text>
  <text x="610" y="96" text-anchor="middle" font-size="10" fill="#555">Multiple database instances</text>

  <rect x="460" y="110" width="130" height="55" rx="6" fill="white" stroke="#999" stroke-width="1.5"/>
  <text x="525" y="128" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">Node 1</text>
  <rect x="470" y="137" width="110" height="22" rx="4" fill="#ffe0b2"/>
  <text x="525" y="153" text-anchor="middle" font-size="8" fill="#333">users A–F</text>

  <rect x="610" y="110" width="130" height="55" rx="6" fill="white" stroke="#999" stroke-width="1.5"/>
  <text x="675" y="128" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">Node 2</text>
  <rect x="620" y="137" width="110" height="22" rx="4" fill="#ffcc80"/>
  <text x="675" y="153" text-anchor="middle" font-size="8" fill="#333">users G–N</text>

  <rect x="460" y="180" width="130" height="55" rx="6" fill="white" stroke="#999" stroke-width="1.5"/>
  <text x="525" y="198" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">Node 3</text>
  <rect x="470" y="207" width="110" height="22" rx="4" fill="#ffb74d"/>
  <text x="525" y="223" text-anchor="middle" font-size="8" fill="#fff">users O–T</text>

  <rect x="610" y="180" width="130" height="55" rx="6" fill="white" stroke="#999" stroke-width="1.5"/>
  <text x="675" y="198" text-anchor="middle" font-size="9" font-weight="bold" fill="#333">Node 4</text>
  <rect x="620" y="207" width="110" height="22" rx="4" fill="#ffa726"/>
  <text x="675" y="223" text-anchor="middle" font-size="8" fill="#fff">users U–Z</text>

  <text x="610" y="256" text-anchor="middle" font-size="9" fill="#555">Each node: independent CPU, RAM, disk</text>
  <text x="610" y="272" text-anchor="middle" font-size="9" fill="#555">No single point of failure (with replicas)</text>
  <text x="610" y="288" text-anchor="middle" font-size="9" fill="#555">Cross-shard queries require network hops</text>
</svg>

### Sharding Strategies

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Range-based** | `user_id 1–1M → Shard 1`, `1M–2M → Shard 2` | Range scans efficient | Hot-spot on latest shard |
| **Hash-based** | `hash(user_id) % N → Shard N` | Even distribution | No range scan support; resharding is expensive |
| **Directory-based** | Lookup table maps key → shard | Flexible, can rebalance | Lookup service is a bottleneck / SPOF |
| **Geo-based** | Route by region / geography | Data locality, lower latency | Uneven distribution across regions |
| **Consistent hashing** | Hash ring with virtual nodes | Adding/removing nodes moves minimal data | More complex to implement |

### Shard Key Selection — Critical Decision

The **shard key** determines which node holds each row. A bad shard key leads to hot spots, cross-shard queries, and data skew.

| Good Shard Key | Bad Shard Key | Why |
|---|---|---|
| `user_id` | `created_at` | Timestamp creates a hot shard for current time |
| `tenant_id` (multi-tenant) | `status` | Low cardinality — most rows share same status |
| `order_id` (hash) | `country` | Uneven distribution (most users in a few countries) |
| Compound: `(tenant_id, user_id)` | Auto-increment PK alone | Sequential IDs hit one shard |

### Cross-Shard Challenges

| Challenge | Description | Mitigation |
|---|---|---|
| **Cross-shard joins** | JOINing data on different nodes requires scatter-gather | Denormalise; co-locate related data on same shard |
| **Cross-shard transactions** | 2PC across shards is slow and complex | Saga pattern; design for shard-local transactions |
| **Global unique IDs** | Auto-increment doesn't work across nodes | UUIDs, Snowflake IDs, Twitter Snowflake algorithm |
| **Rebalancing** | Adding a shard requires moving data | Consistent hashing; virtual shards |
| **Schema changes** | DDL must be applied to every shard | Rolling migrations, schema version table |
| **Aggregations** | `COUNT(*)`, `AVG()` require results from all shards | Application-level merge; pre-aggregated tables |

### Application-Level Sharding Example

```python
import hashlib

SHARD_COUNT = 4
SHARD_CONNECTIONS = {
    0: "postgresql://db-shard-0:5432/app",
    1: "postgresql://db-shard-1:5432/app",
    2: "postgresql://db-shard-2:5432/app",
    3: "postgresql://db-shard-3:5432/app",
}

def get_shard(user_id: int) -> int:
    h = hashlib.md5(str(user_id).encode()).hexdigest()
    return int(h, 16) % SHARD_COUNT

def get_connection(user_id: int):
    shard_id = get_shard(user_id)
    return connect(SHARD_CONNECTIONS[shard_id])

# Usage
conn = get_connection(user_id=42)
conn.execute("SELECT * FROM orders WHERE user_id = %s", (42,))
```

### Database-Native Sharding

| System | Sharding Approach |
|---|---|
| **Citus (PostgreSQL)** | Distributed tables; coordinator node routes queries |
| **Vitess (MySQL)** | Middleware proxy; used by YouTube, Slack |
| **CockroachDB** | Automatic range-based sharding with Raft consensus |
| **MongoDB** | Built-in sharding with config servers and mongos router |
| **Cassandra** | Consistent hashing on partition key; no coordinator |
| **TiDB (MySQL)** | Automatic splitting of TiKV regions |

---

## 11. Sharding vs Partitioning

| Aspect | Partitioning | Sharding |
|---|---|---|
| **Scope** | Single database instance | Multiple database instances |
| **Transparency** | Fully transparent to application | Application may need awareness |
| **Cross-partition/shard queries** | Local, fast | Network I/O, slower |
| **Transactions** | Standard ACID across partitions | Distributed transactions (complex) |
| **Scaling** | Vertical (bigger machine) | Horizontal (more machines) |
| **Failure blast radius** | Single instance — all partitions down | One shard down — others survive |
| **Complexity** | Low — built-in RDBMS feature | High — routing, rebalancing, distributed TX |
| **When to use** | Table is large but fits on one machine | Data or write load exceeds single machine capacity |
| **Migration path** | Start here first | Shard when partitioning isn't enough |

### Decision Flowchart

<svg viewBox="0 0 700 440" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <text x="350" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Do You Need Partitioning or Sharding?</text>

  <!-- Q1 -->
  <rect x="200" y="42" width="300" height="40" rx="20" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="350" y="67" text-anchor="middle" font-size="10" fill="#333">Does your table have > 100M rows or > 100 GB?</text>

  <line x1="250" y1="82" x2="150" y2="110" stroke="#666" stroke-width="1.5" marker-end="url(#fArr)"/>
  <text x="180" y="95" font-size="9" fill="#c62828">No</text>
  <line x1="450" y1="82" x2="530" y2="110" stroke="#666" stroke-width="1.5" marker-end="url(#fArr)"/>
  <text x="500" y="95" font-size="9" fill="#2e7d32">Yes</text>
  <defs><marker id="fArr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#666"/></marker></defs>

  <!-- No branch -->
  <rect x="50" y="110" width="200" height="40" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="150" y="135" text-anchor="middle" font-size="10" fill="#333">No partitioning needed. Use indexes.</text>

  <!-- Yes → Q2 -->
  <rect x="420" y="110" width="250" height="40" rx="20" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="545" y="135" text-anchor="middle" font-size="10" fill="#333">Does it fit on one machine (disk + RAM)?</text>

  <line x1="480" y1="150" x2="400" y2="180" stroke="#666" stroke-width="1.5" marker-end="url(#fArr)"/>
  <text x="420" y="168" font-size="9" fill="#c62828">No</text>
  <line x1="610" y1="150" x2="610" y2="185" stroke="#666" stroke-width="1.5" marker-end="url(#fArr)"/>
  <text x="625" y="172" font-size="9" fill="#2e7d32">Yes</text>

  <!-- Yes to single machine → Partition -->
  <rect x="490" y="185" width="230" height="50" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="605" y="207" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">Use PARTITIONING</text>
  <text x="605" y="225" text-anchor="middle" font-size="9" fill="#555">Range, List, or Hash within one DB</text>

  <!-- No → Q3 -->
  <rect x="230" y="180" width="260" height="40" rx="20" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="360" y="205" text-anchor="middle" font-size="10" fill="#333">Is write throughput the bottleneck?</text>

  <line x1="310" y1="220" x2="230" y2="255" stroke="#666" stroke-width="1.5" marker-end="url(#fArr)"/>
  <text x="250" y="240" font-size="9" fill="#c62828">No (reads)</text>
  <line x1="430" y1="220" x2="500" y2="255" stroke="#666" stroke-width="1.5" marker-end="url(#fArr)"/>
  <text x="470" y="240" font-size="9" fill="#2e7d32">Yes</text>

  <!-- Read bottleneck -->
  <rect x="90" y="255" width="260" height="50" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="220" y="277" text-anchor="middle" font-size="10" font-weight="bold" fill="#6a1b9a">Read Replicas + Partitioning</text>
  <text x="220" y="295" text-anchor="middle" font-size="9" fill="#555">Scale reads with replicas; partition for pruning</text>

  <!-- Write bottleneck → Shard -->
  <rect x="400" y="255" width="230" height="50" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="515" y="277" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">Use SHARDING</text>
  <text x="515" y="295" text-anchor="middle" font-size="9" fill="#555">Distribute writes across nodes</text>

  <!-- Bottom note -->
  <rect x="130" y="340" width="440" height="85" rx="10" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="350" y="362" text-anchor="middle" font-size="11" font-weight="bold" fill="#333">Rule of Thumb</text>
  <text x="350" y="382" text-anchor="middle" font-size="10" fill="#555">1. Start with good indexes</text>
  <text x="350" y="398" text-anchor="middle" font-size="10" fill="#555">2. Add partitioning when table size causes maintenance pain</text>
  <text x="350" y="414" text-anchor="middle" font-size="10" fill="#555">3. Shard only when a single machine can't handle the load</text>
</svg>

---

## 12. Worked Examples

### Example 1 — Beginner: Range Partitioning an Event Log

**Scenario**: A web application stores page views. The table has 500M rows and most queries filter by date.

```sql
-- PostgreSQL
CREATE TABLE page_views (
    view_id     BIGSERIAL,
    user_id     INT         NOT NULL,
    page_url    TEXT        NOT NULL,
    viewed_at   TIMESTAMP   NOT NULL,
    device      VARCHAR(20)
) PARTITION BY RANGE (viewed_at);

-- Monthly partitions
CREATE TABLE page_views_2025_01 PARTITION OF page_views
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE page_views_2025_02 PARTITION OF page_views
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
CREATE TABLE page_views_2025_03 PARTITION OF page_views
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
-- ... more months

CREATE TABLE page_views_default PARTITION OF page_views DEFAULT;

-- Index on user_id (created on parent → propagates to all partitions)
CREATE INDEX idx_pv_user ON page_views (user_id);

-- Query with pruning
EXPLAIN (COSTS OFF)
SELECT COUNT(*), page_url
FROM page_views
WHERE viewed_at >= '2025-02-01' AND viewed_at < '2025-03-01'
GROUP BY page_url
ORDER BY COUNT(*) DESC
LIMIT 10;
-- Scans ONLY page_views_2025_02

-- Archiving: drop old data instantly
ALTER TABLE page_views DETACH PARTITION page_views_2024_01;
DROP TABLE page_views_2024_01;  -- instant, no vacuum needed
```

---

### Example 2 — Beginner: List Partitioning by Region

**Scenario**: A global SaaS application needs to isolate data by region for compliance (GDPR requires EU data stays in EU).

```sql
CREATE TABLE customer_data (
    customer_id BIGSERIAL,
    region      VARCHAR(10) NOT NULL,
    name        VARCHAR(150),
    email       VARCHAR(200),
    data        JSONB,
    created_at  TIMESTAMP DEFAULT NOW()
) PARTITION BY LIST (region);

CREATE TABLE customer_data_eu PARTITION OF customer_data
    FOR VALUES IN ('EU')
    TABLESPACE ts_eu_datacenter;  -- physically store in EU

CREATE TABLE customer_data_na PARTITION OF customer_data
    FOR VALUES IN ('NA')
    TABLESPACE ts_na_datacenter;

CREATE TABLE customer_data_apac PARTITION OF customer_data
    FOR VALUES IN ('APAC')
    TABLESPACE ts_apac_datacenter;

CREATE TABLE customer_data_other PARTITION OF customer_data DEFAULT;

-- Query: only EU data scanned
SELECT name, email
FROM customer_data
WHERE region = 'EU' AND created_at > '2025-01-01';
```

---

### Example 3 — Intermediate: Composite Partitioning for IoT

**Scenario**: An IoT platform collects 10 billion sensor readings per year. Queries filter by time range and often by device_id.

```sql
-- Level 1: Range by timestamp (monthly)
CREATE TABLE sensor_readings (
    reading_id  BIGSERIAL,
    device_id   INT          NOT NULL,
    reading_ts  TIMESTAMPTZ  NOT NULL,
    temperature NUMERIC(5,2),
    humidity    NUMERIC(5,2),
    pressure    NUMERIC(7,2)
) PARTITION BY RANGE (reading_ts);

-- Level 2: Hash by device_id within each month
CREATE TABLE sensor_readings_2025_01 PARTITION OF sensor_readings
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01')
    PARTITION BY HASH (device_id);

-- 16 hash sub-partitions per month
DO $$
BEGIN
    FOR i IN 0..15 LOOP
        EXECUTE format(
            'CREATE TABLE sensor_readings_2025_01_h%s '
            'PARTITION OF sensor_readings_2025_01 '
            'FOR VALUES WITH (MODULUS 16, REMAINDER %s)',
            i, i
        );
    END LOOP;
END $$;

-- Repeat for each month...

-- Query: range prunes to January, hash prunes to device's bucket
SELECT reading_ts, temperature, humidity
FROM sensor_readings
WHERE reading_ts >= '2025-01-15' AND reading_ts < '2025-01-16'
  AND device_id = 4207;
-- Scans only 1 of 16 sub-partitions within January

-- Maintenance: VACUUM one sub-partition at a time
VACUUM (VERBOSE) sensor_readings_2025_01_h3;
```

---

### Example 4 — Intermediate: Partition Pruning Benchmark

**Scenario**: Demonstrating pruning impact on a 200M-row orders table.

```sql
-- Setup: 200M rows, quarterly partitions, 4 years = 16 partitions
-- Each partition: ~12.5M rows

-- Query A: WITH pruning (1 quarter)
EXPLAIN ANALYZE
SELECT customer_id, SUM(total) AS spent
FROM orders
WHERE order_date BETWEEN '2025-04-01' AND '2025-06-30'
GROUP BY customer_id
ORDER BY spent DESC
LIMIT 100;
-- Scans 12.5M rows in orders_2025_q2
-- Execution time: ~1.2 seconds

-- Query B: WITHOUT pruning (function wraps key)
EXPLAIN ANALYZE
SELECT customer_id, SUM(total) AS spent
FROM orders
WHERE EXTRACT(QUARTER FROM order_date) = 2
  AND EXTRACT(YEAR FROM order_date) = 2025
GROUP BY customer_id
ORDER BY spent DESC
LIMIT 100;
-- Scans ALL 200M rows across all 16 partitions
-- Execution time: ~18.5 seconds

-- Fix: Rewrite to enable pruning
-- ALWAYS filter directly on the partition key column
-- without wrapping it in functions.
```

---

### Example 5 — Advanced: Application-Level Sharding with Consistent Hashing

**Scenario**: A social media platform with 500M users needs to shard the user activity table across 8 database nodes. Uses consistent hashing with virtual nodes to minimise data movement when adding shards.

```python
import hashlib
from bisect import bisect_right
from dataclasses import dataclass

@dataclass
class ShardNode:
    node_id: int
    dsn: str

class ConsistentHashRing:
    def __init__(self, nodes: list[ShardNode], virtual_nodes: int = 150):
        self.ring: list[tuple[int, ShardNode]] = []
        self.virtual_nodes = virtual_nodes
        for node in nodes:
            self._add_node(node)
        self.ring.sort(key=lambda x: x[0])

    def _hash(self, key: str) -> int:
        return int(hashlib.sha256(key.encode()).hexdigest(), 16)

    def _add_node(self, node: ShardNode):
        for i in range(self.virtual_nodes):
            vnode_key = f"{node.node_id}:vn{i}"
            h = self._hash(vnode_key)
            self.ring.append((h, node))

    def get_node(self, key: str) -> ShardNode:
        h = self._hash(str(key))
        hashes = [r[0] for r in self.ring]
        idx = bisect_right(hashes, h) % len(self.ring)
        return self.ring[idx][1]

    def add_node(self, node: ShardNode):
        self._add_node(node)
        self.ring.sort(key=lambda x: x[0])


# Setup: 8 shard nodes
nodes = [
    ShardNode(i, f"postgresql://shard-{i}:5432/social")
    for i in range(8)
]
ring = ConsistentHashRing(nodes, virtual_nodes=150)

# Route a user to their shard
user_id = 12345678
shard = ring.get_node(str(user_id))
print(f"User {user_id} → Shard {shard.node_id}")
# Output: User 12345678 → Shard 3

# Adding a 9th node: only ~1/9 of keys need to move
new_node = ShardNode(8, "postgresql://shard-8:5432/social")
ring.add_node(new_node)
```

```sql
-- On each shard: local table (no cross-shard awareness)
CREATE TABLE user_activity (
    activity_id  BIGSERIAL PRIMARY KEY,
    user_id      BIGINT    NOT NULL,
    action       VARCHAR(50),
    target_id    BIGINT,
    created_at   TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Each shard also partitions locally by time for pruning
CREATE TABLE user_activity_2025_05 PARTITION OF user_activity
    FOR VALUES FROM ('2025-05-01') TO ('2025-06-01');
```

---

### Example 6 — Advanced: Citus Distributed Partitioning (PostgreSQL)

**Scenario**: Using Citus to turn PostgreSQL into a distributed database with automatic sharding and partition pruning.

```sql
-- On the Citus coordinator node:

-- Create a distributed table (sharded by tenant_id)
CREATE TABLE events (
    tenant_id   INT          NOT NULL,
    event_id    BIGSERIAL,
    event_type  VARCHAR(50),
    payload     JSONB,
    created_at  TIMESTAMPTZ  NOT NULL,
    PRIMARY KEY (tenant_id, event_id, created_at)
) PARTITION BY RANGE (created_at);

-- Monthly partitions (standard PostgreSQL partitioning)
CREATE TABLE events_2025_05 PARTITION OF events
    FOR VALUES FROM ('2025-05-01') TO ('2025-06-01');
CREATE TABLE events_2025_06 PARTITION OF events
    FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');

-- Distribute the parent across worker nodes
SELECT create_distributed_table('events', 'tenant_id');

-- Citus creates 32 shards per partition (default)
-- Each shard lives on a worker node
-- Result: sharding (across nodes) + partitioning (within each node)

-- Co-locate related tables on the same shard
CREATE TABLE tenants (
    tenant_id  INT PRIMARY KEY,
    name       VARCHAR(200),
    plan       VARCHAR(20)
);
SELECT create_distributed_table('tenants', 'tenant_id');
SELECT create_reference_table('plans');  -- replicated to all nodes

-- Query: Citus routes to the correct shard AND prunes partitions
SELECT event_type, COUNT(*)
FROM events
WHERE tenant_id = 42
  AND created_at >= '2025-05-01'
  AND created_at <  '2025-06-01'
GROUP BY event_type;
-- Hits: 1 shard on 1 worker, 1 partition (May 2025)

-- Cross-tenant aggregation: scatter-gather
SELECT tenant_id, COUNT(*) AS event_count
FROM events
WHERE created_at >= '2025-05-01'
GROUP BY tenant_id
ORDER BY event_count DESC
LIMIT 20;
-- Citus parallelises across all workers, merges on coordinator
```

---

## 13. Common Interview Questions

### Q1: What is table partitioning and why is it useful?

**A:** Table partitioning splits a large logical table into smaller physical sub-tables based on a partition key. Benefits: (1) **partition pruning** — queries scan only relevant partitions, not the whole table; (2) **maintenance** — VACUUM, REINDEX operate per-partition; (3) **archiving** — dropping a partition is O(1) vs expensive DELETE; (4) **parallel execution** — partitions can be scanned concurrently. The application sees one table; the DB manages the split transparently.

---

### Q2: Compare range, list, and hash partitioning. When would you use each?

**A:** **Range** partitions on continuous values (dates, IDs) — best for time-series data where queries filter by date range. **List** partitions on discrete values (region, status, department) — best for categorical data or compliance isolation. **Hash** distributes rows evenly using a hash function — best when there's no natural range/list key and even distribution is the priority. Range supports range-scan pruning; hash only supports exact-match pruning; list supports exact-match on the listed values.

---

### Q3: What is partition pruning and how can you verify it?

**A:** Partition pruning is the optimiser's ability to skip partitions that provably cannot contain matching rows. It activates when the `WHERE` clause filters directly on the partition key with constants. Verify with `EXPLAIN` — only the relevant partitions appear in the plan. Common mistakes that prevent pruning: wrapping the partition key in functions (`YEAR(date_col)`), using non-partition columns in the filter, or applying an expression to the key.

---

### Q4: What is the difference between horizontal and vertical partitioning?

**A:** **Horizontal** partitioning splits by rows — each partition has all columns but a subset of rows (the standard meaning of "partitioning"). **Vertical** partitioning splits by columns — frequently accessed "hot" columns are separated from rarely accessed "cold" columns (like BLOBs), joined by PK when the full row is needed. Vertical partitioning improves cache efficiency and I/O for narrow queries.

---

### Q5: How is sharding different from partitioning?

**A:** Partitioning splits a table within a **single database instance** — all partitions share one CPU/RAM/disk. Sharding distributes data across **multiple independent database instances** (nodes). Partitioning is a DB-native feature with transparent routing; sharding involves application-level or middleware routing, distributed transactions, and cross-shard query complexity. Use partitioning first; shard when a single machine can't handle the load.

---

### Q6: What makes a good shard key? What makes a bad one?

**A:** A good shard key has: (1) **high cardinality** — many distinct values for even distribution, (2) **alignment with query patterns** — most queries include the shard key, avoiding scatter-gather, (3) **stability** — the key doesn't change (would require re-sharding). Bad shard keys: timestamps (hot shard on current period), low-cardinality columns (status, boolean), or columns not commonly in WHERE clauses.

---

### Q7: How do you handle cross-shard joins and transactions?

**A:** Cross-shard joins require scatter-gather (query all shards, merge results) — mitigate by co-locating related data on the same shard (e.g., shard users and their orders by `user_id`). Cross-shard transactions need distributed protocols (2PC) which are slow — mitigate with the Saga pattern (compensating transactions), or design for shard-local transactions. Reference tables (small lookup tables) can be replicated to every shard.

---

### Q8: Explain composite (sub-)partitioning with an example.

**A:** Composite partitioning applies two strategies in sequence. Example: a billion-row IoT table partitioned first by `RANGE(timestamp)` (monthly) then by `HASH(device_id)` (16 buckets per month). A query for `WHERE timestamp = '2025-03-15' AND device_id = 42` prunes to one month's partition, then to one of 16 hash buckets — scanning only ~1/192nd of the data. It combines the range-scan benefit of range partitioning with the even-distribution benefit of hash partitioning.

---

### Q9: What is consistent hashing and why is it used for sharding?

**A:** Consistent hashing places both shard nodes and data keys on a hash ring. Each key maps to the next node clockwise on the ring. When a node is added or removed, only the keys between adjacent nodes are redistributed — not all keys. With virtual nodes (each physical node has ~100+ positions on the ring), the distribution is smooth. This minimises data movement during rebalancing compared to `hash % N` which reshuffles nearly everything when N changes.

---

### Q10: When should you NOT partition a table?

**A:** Don't partition when: (1) the table is small (under ~10M rows / a few GB) — partitioning adds overhead without benefit; (2) queries rarely filter on the partition key — no pruning benefit; (3) the application does mostly random-access lookups by PK — a good index is sufficient; (4) cross-partition queries are the norm — overhead from scanning all partitions with the append node may be worse than scanning one table; (5) the additional DDL/management complexity isn't justified.

---

### Q11: How do you archive old data from a partitioned table?

**A:** In PostgreSQL: `ALTER TABLE parent DETACH PARTITION old_partition;` removes it from the partition tree instantly (no data scan). Then either `DROP TABLE old_partition;` or move it to cold storage. In MySQL: `ALTER TABLE t DROP PARTITION p_old;` is instant. In SQL Server: merge the range boundary after switching the partition out. This is O(1) metadata operation — vastly faster than `DELETE WHERE date < cutoff` which generates WAL, leaves dead tuples, and requires vacuum.

---

### Q12: Explain partition-wise joins and partition-wise aggregation.

**A:** **Partition-wise join**: when two tables are partitioned on the same key with matching boundaries, the optimiser joins them partition-by-partition instead of joining the full tables — reducing memory and enabling parallel execution. **Partition-wise aggregation**: the optimiser aggregates within each partition independently, then merges results — each partition's aggregation can run in parallel. Both are PostgreSQL 12+ features, enabled with `enable_partitionwise_join` and `enable_partitionwise_aggregate` settings.

---

## 14. Key Takeaways

<svg viewBox="0 0 800 520" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="28" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Partitioning & Sharding — Key Takeaways</text>

  <rect x="20" y="48" width="370" height="62" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="35" y="70" font-size="11" font-weight="bold" fill="#1565c0">1. Partition for Pruning</text>
  <text x="35" y="86" font-size="9" fill="#555">The #1 benefit is partition pruning — queries skip</text>
  <text x="35" y="100" font-size="9" fill="#555">irrelevant partitions. Always filter on the partition key.</text>

  <rect x="410" y="48" width="370" height="62" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="425" y="70" font-size="11" font-weight="bold" fill="#2e7d32">2. Choose the Right Strategy</text>
  <text x="425" y="86" font-size="9" fill="#555">Range for time-series, list for categories, hash for</text>
  <text x="425" y="100" font-size="9" fill="#555">even distribution. Composite when one level isn't enough.</text>

  <rect x="20" y="120" width="370" height="62" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="35" y="142" font-size="11" font-weight="bold" fill="#e65100">3. Don't Wrap the Partition Key</text>
  <text x="35" y="158" font-size="9" fill="#555">WHERE YEAR(date_col)=2025 prevents pruning. Write</text>
  <text x="35" y="172" font-size="9" fill="#555">WHERE date_col >= '2025-01-01' AND date_col < '2026-01-01'.</text>

  <rect x="410" y="120" width="370" height="62" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="425" y="142" font-size="11" font-weight="bold" fill="#c62828">4. Drop Partitions, Don't DELETE</text>
  <text x="425" y="158" font-size="9" fill="#555">Archiving via DROP PARTITION is O(1). DELETE generates</text>
  <text x="425" y="172" font-size="9" fill="#555">WAL, dead tuples, and needs VACUUM. Always partition.</text>

  <rect x="20" y="192" width="370" height="62" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="35" y="214" font-size="11" font-weight="bold" fill="#6a1b9a">5. Partition First, Shard Later</text>
  <text x="35" y="230" font-size="9" fill="#555">Start with partitioning (single instance). Only shard when</text>
  <text x="35" y="244" font-size="9" fill="#555">write throughput or data volume exceeds one machine.</text>

  <rect x="410" y="192" width="370" height="62" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="425" y="214" font-size="11" font-weight="bold" fill="#00695c">6. Shard Key = Make or Break</text>
  <text x="425" y="230" font-size="9" fill="#555">High cardinality, query-aligned, stable. Bad shard key</text>
  <text x="425" y="244" font-size="9" fill="#555">→ hot spots, cross-shard queries, uneven distribution.</text>

  <rect x="20" y="264" width="370" height="62" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="35" y="286" font-size="11" font-weight="bold" fill="#f57f17">7. Co-locate Related Data</text>
  <text x="35" y="302" font-size="9" fill="#555">Shard related tables by the same key (user_id, tenant_id)</text>
  <text x="35" y="316" font-size="9" fill="#555">so joins stay local to one shard. Avoid scatter-gather.</text>

  <rect x="410" y="264" width="370" height="62" rx="10" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="425" y="286" font-size="11" font-weight="bold" fill="#3949ab">8. Consistent Hashing for Rebalancing</text>
  <text x="425" y="302" font-size="9" fill="#555">When sharding, use consistent hashing + virtual nodes</text>
  <text x="425" y="316" font-size="9" fill="#555">so adding a shard moves only ~1/N of the data.</text>

  <rect x="20" y="336" width="370" height="62" rx="10" fill="#fbe9e7" stroke="#bf360c" stroke-width="2"/>
  <text x="35" y="358" font-size="11" font-weight="bold" fill="#bf360c">9. Vertical Partitioning for Wide Tables</text>
  <text x="35" y="374" font-size="9" fill="#555">Split hot columns from cold/large columns (BLOBs).</text>
  <text x="35" y="388" font-size="9" fill="#555">Hot-path queries hit narrow, cacheable rows.</text>

  <rect x="410" y="336" width="370" height="62" rx="10" fill="#e0f7fa" stroke="#006064" stroke-width="2"/>
  <text x="425" y="358" font-size="11" font-weight="bold" fill="#006064">10. Verify with EXPLAIN</text>
  <text x="425" y="374" font-size="9" fill="#555">Always check EXPLAIN to confirm pruning is happening.</text>
  <text x="425" y="388" font-size="9" fill="#555">If all partitions appear in the plan, fix your WHERE clause.</text>

  <!-- Scaling ladder -->
  <rect x="100" y="420" width="600" height="85" rx="12" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="400" y="442" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Scaling Ladder</text>
  <text x="120" y="462" font-size="10" fill="#555">1. Indexes → 2. Read replicas → 3. Table partitioning → 4. Sharding</text>
  <text x="120" y="480" font-size="10" fill="#555">Each step adds complexity. Don't jump to sharding before exhausting earlier steps.</text>
  <text x="120" y="496" font-size="10" fill="#555">Most applications never need to go past step 3.</text>
</svg>

---

## 15. Download

<a href="21_partitioning.md" download="21_partitioning.md" style="display:inline-block;padding:14px 28px;background:#1565c0;color:white;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;cursor:pointer;">Download 21_partitioning.md</a>
