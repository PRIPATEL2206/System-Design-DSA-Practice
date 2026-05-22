# Phase 12 — Concurrency Control

> **Curriculum:** DBMS A-to-Z | **File:** `12_concurrency_control.md`  
> **Prerequisites:** Phases 1–11 (Foundations → Transactions & ACID)

---

## Table of Contents

1. [Why Concurrency Control?](#1-why-concurrency-control)
2. [Lock-Based Protocols](#2-lock-based-protocols)
3. [Two-Phase Locking (2PL)](#3-two-phase-locking-2pl)
4. [Lock Granularity](#4-lock-granularity)
5. [Timestamp Ordering Protocol](#5-timestamp-ordering-protocol)
6. [MVCC — Multi-Version Concurrency Control](#6-mvcc--multi-version-concurrency-control)
7. [MVCC in PostgreSQL — A Deep Dive](#7-mvcc-in-postgresql--a-deep-dive)
8. [Optimistic vs Pessimistic Concurrency](#8-optimistic-vs-pessimistic-concurrency)
9. [Deadlocks](#9-deadlocks)
10. [Deadlock Detection, Prevention & Recovery](#10-deadlock-detection-prevention--recovery)
11. [Worked Examples](#11-worked-examples)
12. [Common Interview Questions](#12-common-interview-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Why Concurrency Control?

Modern databases serve hundreds or thousands of concurrent transactions. Without coordination, these transactions would read and write the same data simultaneously, producing all the anomalies covered in Phase 11: dirty reads, lost updates, phantom rows, and inconsistent states.

**Concurrency control** is the set of mechanisms that ensures transactions execute **correctly in parallel** — as if they were running one at a time (serializability) — while maximizing throughput.

### The Spectrum

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="600" height="100" viewBox="0 0 600 100">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <line x1="40" y1="50" x2="560" y2="50" stroke="#555" stroke-width="2"/>
  <!-- Serial end -->
  <circle cx="40" cy="50" r="6" fill="#C0392B"/>
  <text x="40" y="30" text-anchor="middle" fill="#C0392B" font-weight="bold">Serial</text>
  <text x="40" y="80" text-anchor="middle" fill="#555" font-size="10">One txn at a time</text>
  <text x="40" y="94" text-anchor="middle" fill="#555" font-size="10">100% correct, 0% concurrency</text>
  <!-- Concurrent end -->
  <circle cx="560" cy="50" r="6" fill="#27AE60"/>
  <text x="560" y="30" text-anchor="middle" fill="#27AE60" font-weight="bold">Free-for-all</text>
  <text x="560" y="80" text-anchor="middle" fill="#555" font-size="10">No control</text>
  <text x="560" y="94" text-anchor="middle" fill="#555" font-size="10">100% concurrency, 0% correct</text>
  <!-- Middle -->
  <circle cx="300" cy="50" r="8" fill="#2980B9"/>
  <text x="300" y="30" text-anchor="middle" fill="#2980B9" font-weight="bold">Concurrency Control</text>
  <text x="300" y="80" text-anchor="middle" fill="#555" font-size="10">Balance correctness &amp; throughput</text>
</svg>
```

### Main Approaches

| Approach | How it works | Used by |
|----------|-------------|---------|
| **Lock-based** | Acquire locks before accessing data; hold/release by protocol | Classic 2PL |
| **Timestamp-based** | Order transactions by timestamp; abort on conflict | Academic, some embedded DBs |
| **MVCC** | Each transaction sees a snapshot; writers don't block readers | PostgreSQL, Oracle, MySQL InnoDB |
| **Optimistic** | Execute freely, validate at commit, abort on conflict | Low-contention workloads |

---

## 2. Lock-Based Protocols

### Lock Types

A lock controls which operations other transactions may perform on a data item.

| Lock | Symbol | Allows concurrent... | Blocks concurrent... |
|------|--------|---------------------|---------------------|
| **Shared (S)** | S-lock / Read lock | Other S-locks (reads) | X-locks (writes) |
| **Exclusive (X)** | X-lock / Write lock | Nothing | S-locks and X-locks |

### Lock Compatibility Matrix

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="340" height="140" viewBox="0 0 340 140">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>
  <!-- Header -->
  <rect x="0" y="0" width="340" height="35" fill="#2C3E50" rx="4"/>
  <text x="70"  y="22" text-anchor="middle" fill="white" font-weight="bold">Request ↓ / Held →</text>
  <text x="200" y="22" text-anchor="middle" fill="white" font-weight="bold">S (Shared)</text>
  <text x="290" y="22" text-anchor="middle" fill="white" font-weight="bold">X (Exclusive)</text>
  <!-- S row -->
  <rect x="0" y="35" width="340" height="35" fill="#EBF5FB"/>
  <text x="70"  y="58" text-anchor="middle" fill="#1A5276" font-weight="bold">S (Shared)</text>
  <text x="200" y="58" text-anchor="middle" fill="#27AE60" font-weight="bold">✅ Compatible</text>
  <text x="290" y="58" text-anchor="middle" fill="#E74C3C" font-weight="bold">❌ Conflict</text>
  <!-- X row -->
  <rect x="0" y="70" width="340" height="35" fill="#FDFEFE"/>
  <text x="70"  y="93" text-anchor="middle" fill="#1A5276" font-weight="bold">X (Exclusive)</text>
  <text x="200" y="93" text-anchor="middle" fill="#E74C3C" font-weight="bold">❌ Conflict</text>
  <text x="290" y="93" text-anchor="middle" fill="#E74C3C" font-weight="bold">❌ Conflict</text>

  <text x="170" y="128" text-anchor="middle" fill="#555" font-size="10" font-style="italic">Multiple readers allowed; any writer blocks all others</text>
</svg>
```

### SQL-Level Locking

```sql
-- Shared lock (read lock): prevents writes by other transactions
SELECT * FROM accounts WHERE account_id = 101 FOR SHARE;       -- PostgreSQL
SELECT * FROM accounts WHERE account_id = 101 LOCK IN SHARE MODE; -- MySQL

-- Exclusive lock (write lock): prevents all access by other transactions
SELECT * FROM accounts WHERE account_id = 101 FOR UPDATE;       -- All RDBMS
-- Row is locked until COMMIT or ROLLBACK

-- NOWAIT: fail immediately instead of waiting for the lock
SELECT * FROM accounts WHERE account_id = 101 FOR UPDATE NOWAIT;

-- SKIP LOCKED: skip locked rows (useful for job queues)
SELECT * FROM job_queue WHERE status = 'pending'
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

---

## 3. Two-Phase Locking (2PL)

### Concept

**Two-Phase Locking** is a protocol that guarantees **serializability** by dividing each transaction's life into two phases:

1. **Growing phase:** The transaction may acquire locks but must not release any.
2. **Shrinking phase:** The transaction may release locks but must not acquire any.

Once a transaction releases its first lock, it can never acquire another.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="240" viewBox="0 0 560 240">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Axes -->
  <line x1="60" y1="200" x2="520" y2="200" stroke="#555" stroke-width="1.5" marker-end="url(#a2pl)"/>
  <line x1="60" y1="200" x2="60"  y2="20"  stroke="#555" stroke-width="1.5" marker-end="url(#a2pl)"/>
  <defs><marker id="a2pl" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>
  <text x="290" y="220" text-anchor="middle" fill="#555">Time →</text>
  <text x="30" y="110" fill="#555" transform="rotate(-90,30,110)">Locks Held</text>

  <!-- Growing phase -->
  <line x1="80"  y1="190" x2="270" y2="50"  stroke="#2980B9" stroke-width="3"/>
  <text x="175" y="105" text-anchor="middle" fill="#2980B9" font-weight="bold" font-size="12">GROWING</text>
  <text x="175" y="120" text-anchor="middle" fill="#2980B9" font-size="10">Acquiring locks</text>

  <!-- Lock point -->
  <circle cx="270" cy="50" r="5" fill="#E74C3C"/>
  <text x="270" y="38" text-anchor="middle" fill="#E74C3C" font-weight="bold" font-size="10">LOCK POINT</text>

  <!-- Shrinking phase -->
  <line x1="270" y1="50"  x2="480" y2="190" stroke="#27AE60" stroke-width="3"/>
  <text x="375" y="105" text-anchor="middle" fill="#27AE60" font-weight="bold" font-size="12">SHRINKING</text>
  <text x="375" y="120" text-anchor="middle" fill="#27AE60" font-size="10">Releasing locks</text>

  <!-- COMMIT label -->
  <line x1="480" y1="190" x2="480" y2="200" stroke="#555" stroke-dasharray="4"/>
  <text x="480" y="215" text-anchor="middle" fill="#555" font-size="10">COMMIT</text>
</svg>
```

### Variants of 2PL

| Variant | Lock Release | Prevents deadlock? | Used in practice? |
|---------|-------------|-------------------|-------------------|
| **Basic 2PL** | Release after lock point (growing → shrinking) | ❌ No | Rarely alone |
| **Strict 2PL** | Release X-locks only at COMMIT/ROLLBACK | ❌ No | ✅ Most RDBMS |
| **Rigorous 2PL** | Release ALL locks (S + X) only at COMMIT/ROLLBACK | ❌ No | ✅ SQL Server |

**Strict 2PL** is the standard in production databases. It holds write locks until commit, preventing cascading aborts (where one rollback forces other transactions to roll back too because they read uncommitted data).

### Why 2PL Guarantees Serializability

The serialization order of transactions is determined by their **lock points** (the moment a transaction has acquired all its locks). Transactions are effectively serialized in the order of their lock points — even though they physically overlap in time.

---

## 4. Lock Granularity

The DBMS can lock at different levels of the data hierarchy. Finer granularity = more concurrency but more lock overhead. Coarser granularity = less overhead but more blocking.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="500" height="260" viewBox="0 0 500 260">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs><marker id="agr" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- Database level -->
  <rect x="180" y="10" width="140" height="35" fill="#2C3E50" rx="6"/>
  <text x="250" y="32" text-anchor="middle" fill="white" font-weight="bold">Database</text>
  <line x1="250" y1="45" x2="250" y2="60" stroke="#555" stroke-width="1.5" marker-end="url(#agr)"/>

  <!-- Table level -->
  <rect x="180" y="60" width="140" height="35" fill="#8E44AD" rx="6"/>
  <text x="250" y="82" text-anchor="middle" fill="white" font-weight="bold">Table</text>
  <line x1="250" y1="95" x2="250" y2="110" stroke="#555" stroke-width="1.5" marker-end="url(#agr)"/>

  <!-- Page level -->
  <rect x="180" y="110" width="140" height="35" fill="#2980B9" rx="6"/>
  <text x="250" y="132" text-anchor="middle" fill="white" font-weight="bold">Page (Block)</text>
  <line x1="250" y1="145" x2="250" y2="160" stroke="#555" stroke-width="1.5" marker-end="url(#agr)"/>

  <!-- Row level -->
  <rect x="180" y="160" width="140" height="35" fill="#27AE60" rx="6"/>
  <text x="250" y="182" text-anchor="middle" fill="white" font-weight="bold">Row (Tuple)</text>

  <!-- Labels -->
  <text x="80"  y="25" text-anchor="middle" fill="#555">Coarsest</text>
  <text x="80"  y="40" text-anchor="middle" fill="#555">Low concurrency</text>
  <text x="80"  y="55" text-anchor="middle" fill="#555">Low overhead</text>
  <line x1="140" y1="30" x2="175" y2="30" stroke="#555" stroke-dasharray="4"/>

  <text x="420" y="175" text-anchor="middle" fill="#555">Finest</text>
  <text x="420" y="190" text-anchor="middle" fill="#555">High concurrency</text>
  <text x="420" y="205" text-anchor="middle" fill="#555">High overhead</text>
  <line x1="325" y1="178" x2="380" y2="178" stroke="#555" stroke-dasharray="4"/>

  <text x="250" y="245" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">Most OLTP databases lock at the ROW level for maximum concurrency</text>
</svg>
```

### Intention Locks (Multi-Granularity Locking)

When a transaction wants a row lock, it first acquires an **intention lock** on the containing table — signaling that it holds (or intends to hold) a finer-grained lock inside.

| Lock | Meaning |
|------|---------|
| **IS** (Intention Shared) | "I hold or plan to hold S-locks on rows inside this table" |
| **IX** (Intention Exclusive) | "I hold or plan to hold X-locks on rows inside this table" |
| **SIX** (Shared + Intention Exclusive) | "I hold an S-lock on the whole table and will X-lock individual rows" |

```sql
-- Implicit intention locks (transparent to the user):
-- When T1 does SELECT * FROM employees WHERE emp_id = 5 FOR UPDATE:
-- 1. IS or IX lock acquired on the employees TABLE
-- 2. X-lock acquired on the ROW with emp_id = 5
-- Other transactions can still read/write OTHER rows in the table.
```

### Explicit Table-Level Locks

```sql
-- PostgreSQL: explicit table lock
LOCK TABLE employees IN ACCESS EXCLUSIVE MODE;  -- blocks everything
LOCK TABLE employees IN SHARE MODE;              -- blocks writes, allows reads

-- MySQL: explicit table lock
LOCK TABLES employees WRITE;
UNLOCK TABLES;
```

---

## 5. Timestamp Ordering Protocol

### Concept

Instead of locks, each transaction is assigned a **unique timestamp** at birth (e.g., system clock or counter). The protocol enforces that the result of concurrent execution is equivalent to the serial order of timestamps.

Each data item X maintains:
- **W-timestamp(X):** timestamp of the last transaction that wrote X.
- **R-timestamp(X):** timestamp of the last transaction that read X.

### Rules

| Transaction T wants to: | Rule | Violation action |
|------------------------|------|-----------------|
| **Read X** | If `TS(T) < W-timestamp(X)` → T tries to read a value written by a younger transaction | ABORT T, restart with new TS |
| **Write X** | If `TS(T) < R-timestamp(X)` → a younger transaction already read the old value | ABORT T, restart with new TS |
| **Write X** | If `TS(T) < W-timestamp(X)` → a younger transaction already overwrote X | Skip write (Thomas Write Rule) |

### Thomas Write Rule

If T's write would be immediately overwritten by a younger transaction's write, the write is simply **ignored** (skipped) instead of aborting T. This reduces unnecessary restarts.

### Practical Use

Pure timestamp ordering is **rarely used in production RDBMS** — it causes too many aborts under high contention. However, it is foundational theory and influences modern techniques like MVCC and Serializable Snapshot Isolation (SSI).

---

## 6. MVCC — Multi-Version Concurrency Control

### Concept

**MVCC** is the dominant concurrency control mechanism in modern databases. Instead of blocking readers with locks, the database maintains **multiple versions** of each row. Each transaction sees a consistent **snapshot** of the database as of its start time. Writers create new versions; readers see old versions — **readers never block writers, and writers never block readers**.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="250" viewBox="0 0 620 250">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <text x="310" y="18" text-anchor="middle" font-weight="bold" fill="#333" font-size="13">MVCC: Multiple Row Versions</text>

  <!-- Time axis -->
  <line x1="50" y1="230" x2="580" y2="230" stroke="#555" stroke-width="1.5" marker-end="url(#amv2)"/>
  <defs><marker id="amv2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>
  <text x="310" y="245" text-anchor="middle" fill="#555">Time →</text>

  <!-- Version 1 -->
  <rect x="60" y="50" width="130" height="55" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="125" y="72" text-anchor="middle" fill="#2D6A2D" font-weight="bold">Version 1</text>
  <text x="125" y="92" text-anchor="middle" fill="#2D6A2D">balance = 1000</text>
  <text x="125" y="120" text-anchor="middle" fill="#555" font-size="10">Created by T100</text>

  <!-- Arrow v1 → v2 -->
  <line x1="190" y1="78" x2="240" y2="78" stroke="#555" stroke-width="1.5" marker-end="url(#amv2)"/>
  <text x="215" y="70" text-anchor="middle" fill="#E74C3C" font-size="10">T200 UPDATEs</text>

  <!-- Version 2 -->
  <rect x="240" y="50" width="130" height="55" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="305" y="72" text-anchor="middle" fill="#1A5276" font-weight="bold">Version 2</text>
  <text x="305" y="92" text-anchor="middle" fill="#1A5276">balance = 1500</text>
  <text x="305" y="120" text-anchor="middle" fill="#555" font-size="10">Created by T200</text>

  <!-- Arrow v2 → v3 -->
  <line x1="370" y1="78" x2="420" y2="78" stroke="#555" stroke-width="1.5" marker-end="url(#amv2)"/>
  <text x="395" y="70" text-anchor="middle" fill="#E74C3C" font-size="10">T300 UPDATEs</text>

  <!-- Version 3 -->
  <rect x="420" y="50" width="130" height="55" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="485" y="72" text-anchor="middle" fill="#784212" font-weight="bold">Version 3</text>
  <text x="485" y="92" text-anchor="middle" fill="#784212">balance = 800</text>
  <text x="485" y="120" text-anchor="middle" fill="#555" font-size="10">Created by T300</text>

  <!-- Reader sees version based on snapshot -->
  <rect x="100" y="155" width="200" height="40" fill="#EAFAF1" stroke="#1E8449" rx="6"/>
  <text x="200" y="173" text-anchor="middle" fill="#145A32" font-weight="bold">T150 (started before T200)</text>
  <text x="200" y="189" text-anchor="middle" fill="#145A32">Sees Version 1: balance = 1000</text>

  <rect x="330" y="155" width="200" height="40" fill="#FEF9E7" stroke="#F39C12" rx="6"/>
  <text x="430" y="173" text-anchor="middle" fill="#7D6608" font-weight="bold">T250 (started after T200)</text>
  <text x="430" y="189" text-anchor="middle" fill="#7D6608">Sees Version 2: balance = 1500</text>
</svg>
```

### Key Properties of MVCC

| Property | Behaviour |
|----------|-----------|
| **Readers never block writers** | A reading transaction never waits for a writing transaction |
| **Writers never block readers** | A writing transaction never waits for a reading transaction |
| **Writers may block writers** | Two transactions writing the same row → one waits or aborts |
| **Snapshot isolation** | Each transaction sees a consistent snapshot from its start time |
| **Garbage collection needed** | Old versions must be reclaimed (VACUUM in PostgreSQL, purge in MySQL) |

### MVCC vs Locking

| | Lock-based (2PL) | MVCC |
|---|-----------------|------|
| Readers block writers | ✅ Yes | ❌ No |
| Writers block readers | ✅ Yes | ❌ No |
| Deadlock risk | High | Low (only writer-writer) |
| Storage | No extra versions | Multiple row versions |
| Garbage collection | Not needed | Required (VACUUM) |
| Throughput | Lower (more blocking) | Higher (more concurrency) |

---

## 7. MVCC in PostgreSQL — A Deep Dive

PostgreSQL implements MVCC using hidden **system columns** on every row.

### Hidden Columns

| Column | Meaning |
|--------|---------|
| `xmin` | Transaction ID that **created** (inserted) this row version |
| `xmax` | Transaction ID that **deleted or updated** this row version (0 if still live) |
| `ctid` | Physical location of the row on disk (page, offset) |

```sql
-- View hidden columns (PostgreSQL)
SELECT xmin, xmax, ctid, emp_id, name, salary
FROM   employees
LIMIT  5;

-- Sample output:
-- xmin | xmax | ctid  | emp_id | name  | salary
-- 100  | 0    | (0,1) | 1      | Alice | 95000    ← live row, created by txn 100
-- 100  | 205  | (0,2) | 2      | Bob   | 78000    ← marked for deletion by txn 205
```

### How UPDATE Works in PostgreSQL

PostgreSQL's UPDATE does **not modify the row in-place**. Instead:
1. The old row is marked as dead: `xmax = current_txn_id`.
2. A new row version is inserted with `xmin = current_txn_id`, `xmax = 0`.
3. The index is updated to point to the new version (or uses HOT — Heap Only Tuple — optimization when possible).

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="180" viewBox="0 0 560 180">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Old version -->
  <rect x="30" y="30" width="200" height="60" fill="#FDEDEC" stroke="#C0392B" rx="6"/>
  <text x="130" y="52" text-anchor="middle" fill="#7B241C" font-weight="bold">Old Version (Dead)</text>
  <text x="130" y="72" text-anchor="middle" fill="#7B241C">salary=78000, xmax=205</text>
  <text x="130" y="105" text-anchor="middle" fill="#C0392B" font-size="10">Invisible to txns started after 205 commits</text>

  <line x1="230" y1="60" x2="310" y2="60" stroke="#555" stroke-width="1.5" marker-end="url(#apg)"/>
  <defs><marker id="apg" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>
  <text x="270" y="52" text-anchor="middle" fill="#E74C3C" font-size="10">UPDATE</text>

  <!-- New version -->
  <rect x="310" y="30" width="220" height="60" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="420" y="52" text-anchor="middle" fill="#2D6A2D" font-weight="bold">New Version (Live)</text>
  <text x="420" y="72" text-anchor="middle" fill="#2D6A2D">salary=85000, xmin=205, xmax=0</text>
  <text x="420" y="105" text-anchor="middle" fill="#27AE60" font-size="10">Visible to txns started after 205 commits</text>

  <text x="280" y="155" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">VACUUM later removes dead rows whose xmax is visible to all active transactions</text>
</svg>
```

### VACUUM — Garbage Collection

Dead row versions accumulate after updates and deletes. **VACUUM** removes them and reclaims space.

```sql
-- Manual vacuum
VACUUM employees;                     -- basic: marks dead rows as reusable
VACUUM FULL employees;                -- rewrites table, reclaims disk space (locks table)
VACUUM (VERBOSE, ANALYZE) employees;  -- verbose output + update statistics

-- PostgreSQL runs AUTOVACUUM automatically in the background
-- Monitor it:
SELECT relname, n_dead_tup, last_autovacuum
FROM   pg_stat_user_tables
WHERE  n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### Transaction ID Wraparound

PostgreSQL's transaction IDs are 32-bit integers (~4 billion). If the counter wraps around, old transactions could appear to be "in the future" — breaking visibility. VACUUM also prevents this by **freezing** old transaction IDs on rows.

```sql
-- Check how close the database is to wraparound
SELECT datname,
       age(datfrozenxid) AS txns_until_wraparound,
       ROUND(100.0 * age(datfrozenxid) / 2147483647, 1) AS pct
FROM   pg_database;
-- If pct exceeds ~80%, autovacuum should be checked — manual VACUUM FREEZE may be needed
```

---

## 8. Optimistic vs Pessimistic Concurrency

### Pessimistic Concurrency (Locking)

Assume conflicts **will happen**. Lock rows before reading/writing them. Other transactions must wait.

```sql
-- Pessimistic: lock the row immediately
BEGIN;
SELECT balance FROM accounts WHERE account_id = 101 FOR UPDATE;   -- X-lock acquired
-- ... compute new balance ...
UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
COMMIT;   -- lock released
```

**Pros:** Guarantees no conflict at commit; simple to reason about.  
**Cons:** Reduces concurrency; risk of deadlocks; held locks block other transactions.

### Optimistic Concurrency (Validate at Commit)

Assume conflicts are **rare**. Execute freely without locks, then validate at commit time. If a conflict is detected, abort and retry.

```sql
-- Optimistic: use a version column
-- Step 1: read current state
SELECT balance, version FROM accounts WHERE account_id = 101;
-- Returns: balance=1000, version=7

-- Step 2: compute new value in application
new_balance = 1000 - 500   -- = 500

-- Step 3: conditional update (check version)
BEGIN;
UPDATE accounts
SET    balance = 500, version = 8
WHERE  account_id = 101 AND version = 7;   -- CAS (compare-and-swap)
-- If rows_updated = 0 → another transaction modified the row → retry from step 1
COMMIT;
```

**Pros:** No blocking; maximum concurrency; great for read-heavy workloads.  
**Cons:** Wasted work on conflicts (must retry); not suitable for high-contention scenarios.

### Comparison

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="170" viewBox="0 0 620 170">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <rect x="0" y="0" width="620" height="30" fill="#2C3E50" rx="4"/>
  <text x="80"  y="20" text-anchor="middle" fill="white" font-weight="bold">Aspect</text>
  <text x="260" y="20" text-anchor="middle" fill="white" font-weight="bold">Pessimistic (Locking)</text>
  <text x="490" y="20" text-anchor="middle" fill="white" font-weight="bold">Optimistic (Version)</text>

  <rect x="0" y="30" width="620" height="32" fill="#EBF5FB"/>
  <text x="80"  y="50" text-anchor="middle" fill="#333">Conflict assumption</text>
  <text x="260" y="50" text-anchor="middle" fill="#555">Conflicts will happen</text>
  <text x="490" y="50" text-anchor="middle" fill="#555">Conflicts are rare</text>

  <rect x="0" y="62" width="620" height="32" fill="#FDFEFE"/>
  <text x="80"  y="82" text-anchor="middle" fill="#333">Blocking</text>
  <text x="260" y="82" text-anchor="middle" fill="#E74C3C">Blocks other txns</text>
  <text x="490" y="82" text-anchor="middle" fill="#27AE60">No blocking</text>

  <rect x="0" y="94" width="620" height="32" fill="#EBF5FB"/>
  <text x="80"  y="114" text-anchor="middle" fill="#333">On conflict</text>
  <text x="260" y="114" text-anchor="middle" fill="#555">Wait for lock release</text>
  <text x="490" y="114" text-anchor="middle" fill="#555">Abort + retry</text>

  <rect x="0" y="126" width="620" height="32" fill="#FDFEFE"/>
  <text x="80"  y="146" text-anchor="middle" fill="#333">Best for</text>
  <text x="260" y="146" text-anchor="middle" fill="#555">High-contention writes</text>
  <text x="490" y="146" text-anchor="middle" fill="#555">Read-heavy, low contention</text>
</svg>
```

---

## 9. Deadlocks

### Concept

A **deadlock** occurs when two or more transactions are each waiting for a lock held by another — and no transaction can proceed.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="400" height="240" viewBox="0 0 400 240">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs>
    <marker id="adl" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/>
    </marker>
  </defs>

  <text x="200" y="18" text-anchor="middle" font-weight="bold" fill="#C0392B" font-size="13">Deadlock Cycle (Wait-For Graph)</text>

  <!-- T1 -->
  <circle cx="120" cy="80" r="40" fill="#2980B9"/>
  <text x="120" y="85" text-anchor="middle" fill="white" font-weight="bold" font-size="14">T1</text>

  <!-- T2 -->
  <circle cx="280" cy="80" r="40" fill="#8E44AD"/>
  <text x="280" y="85" text-anchor="middle" fill="white" font-weight="bold" font-size="14">T2</text>

  <!-- Resource A -->
  <rect x="80"  y="160" width="80" height="40" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="120" y="185" text-anchor="middle" fill="#2D6A2D" font-weight="bold">Row A</text>

  <!-- Resource B -->
  <rect x="240" y="160" width="80" height="40" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="280" y="185" text-anchor="middle" fill="#784212" font-weight="bold">Row B</text>

  <!-- T1 holds A -->
  <line x1="120" y1="120" x2="120" y2="157" stroke="#2980B9" stroke-width="2"/>
  <text x="85" y="142" fill="#2980B9" font-size="10">holds</text>

  <!-- T2 holds B -->
  <line x1="280" y1="120" x2="280" y2="157" stroke="#8E44AD" stroke-width="2"/>
  <text x="310" y="142" fill="#8E44AD" font-size="10">holds</text>

  <!-- T1 waits for B -->
  <line x1="160" y1="80" x2="238" y2="80" stroke="#E74C3C" stroke-width="2" stroke-dasharray="6" marker-end="url(#adl)"/>
  <text x="200" y="72" text-anchor="middle" fill="#E74C3C" font-size="10">waits for B</text>

  <!-- T2 waits for A -->
  <path d="M 250,110 Q 200,150 160,110" fill="none" stroke="#E74C3C" stroke-width="2" stroke-dasharray="6" marker-end="url(#adl)"/>
  <text x="200" y="140" text-anchor="middle" fill="#E74C3C" font-size="10">waits for A</text>

  <text x="200" y="228" text-anchor="middle" fill="#C0392B" font-size="11" font-style="italic">Circular wait → DEADLOCK — neither can proceed</text>
</svg>
```

### Classic Scenario

```
T1: BEGIN; UPDATE accounts SET ... WHERE id = 101;  -- X-lock on row 101
T2: BEGIN; UPDATE accounts SET ... WHERE id = 202;  -- X-lock on row 202
T1:        UPDATE accounts SET ... WHERE id = 202;  -- WAITS for T2's lock on 202
T2:        UPDATE accounts SET ... WHERE id = 101;  -- WAITS for T1's lock on 101
-- DEADLOCK: T1 waits for T2, T2 waits for T1
```

---

## 10. Deadlock Detection, Prevention & Recovery

### Detection: Wait-For Graph

The DBMS periodically checks for **cycles** in the wait-for graph. If a cycle is found → deadlock. One transaction is chosen as the **victim**, aborted, and rolled back to break the cycle.

```sql
-- PostgreSQL: deadlock detection runs automatically (default: every 1 second)
SHOW deadlock_timeout;    -- default: 1s

-- When a deadlock is detected:
-- ERROR:  deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 678;
--         blocked by process 67890.
-- HINT:  See server log for query details.
```

### Prevention Strategies

| Strategy | How it works |
|----------|-------------|
| **Consistent lock ordering** | All transactions lock resources in the same order (e.g., by ascending PK). No cycle can form. |
| **Lock timeouts** | If a lock can't be acquired within N seconds, abort the requestor |
| **Wait-Die** | Older transaction waits; younger transaction dies (rolls back). Prevents circular wait. |
| **Wound-Wait** | Older transaction "wounds" (aborts) younger transaction holding the needed lock. Younger waits for older. |

### Wait-Die vs Wound-Wait

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="600" height="160" viewBox="0 0 600 160">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Wait-Die -->
  <rect x="10" y="10" width="280" height="130" fill="#EBF5FB" stroke="#2980B9" rx="8"/>
  <text x="150" y="32" text-anchor="middle" font-weight="bold" fill="#1A5276" font-size="12">Wait-Die</text>
  <line x1="20" y1="40" x2="280" y2="40" stroke="#AED6F1"/>
  <text x="150" y="60" text-anchor="middle" fill="#333">If requesting txn is OLDER:</text>
  <text x="150" y="76" text-anchor="middle" fill="#27AE60">→ WAIT for the younger txn</text>
  <text x="150" y="98" text-anchor="middle" fill="#333">If requesting txn is YOUNGER:</text>
  <text x="150" y="114" text-anchor="middle" fill="#E74C3C">→ DIE (abort and restart)</text>
  <text x="150" y="134" text-anchor="middle" fill="#555" font-size="10">Younger txns may starve (repeated aborts)</text>

  <!-- Wound-Wait -->
  <rect x="310" y="10" width="280" height="130" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="450" y="32" text-anchor="middle" font-weight="bold" fill="#145A32" font-size="12">Wound-Wait</text>
  <line x1="320" y1="40" x2="580" y2="40" stroke="#A9DFBF"/>
  <text x="450" y="60" text-anchor="middle" fill="#333">If requesting txn is OLDER:</text>
  <text x="450" y="76" text-anchor="middle" fill="#E74C3C">→ WOUND (abort the younger holder)</text>
  <text x="450" y="98" text-anchor="middle" fill="#333">If requesting txn is YOUNGER:</text>
  <text x="450" y="114" text-anchor="middle" fill="#27AE60">→ WAIT for the older txn</text>
  <text x="450" y="134" text-anchor="middle" fill="#555" font-size="10">Older txns never wait → no starvation</text>
</svg>
```

### Recovery from Deadlock

When the DBMS detects a deadlock:
1. **Choose a victim:** Usually the transaction with the least work done, or the youngest transaction.
2. **Abort the victim:** Roll back its changes.
3. **Release locks:** Other transactions can proceed.
4. **Retry (optional):** Application retries the aborted transaction.

```sql
-- Application-level deadlock retry pattern (pseudocode):
max_retries = 3
for attempt in range(max_retries):
    try:
        BEGIN;
        -- ... operations ...
        COMMIT;
        break;  -- success
    except DeadlockDetected:
        ROLLBACK;
        if attempt == max_retries - 1:
            raise;  -- give up after max retries
        sleep(random(0.01, 0.1));  -- jitter before retry
```

### Reducing Deadlocks in Practice

| Technique | How it helps |
|-----------|-------------|
| Keep transactions short | Fewer locks held for less time = fewer chances for cycles |
| Lock in consistent order | Always lock table/row with lower PK first → breaks circular wait |
| Use `FOR UPDATE NOWAIT` | Fail fast instead of waiting → detect contention early |
| Use `FOR UPDATE SKIP LOCKED` | Skip locked rows entirely (great for job queues) |
| Reduce isolation level | READ COMMITTED holds fewer locks than SERIALIZABLE |
| Use optimistic locking | Avoid locks altogether for low-contention workloads |

---

## 11. Worked Examples

### Example 1 — Pessimistic Locking: Bank Transfer

```sql
BEGIN;

-- Lock both accounts in consistent order (lower ID first) to prevent deadlock
SELECT balance FROM accounts WHERE account_id = 101 FOR UPDATE;
SELECT balance FROM accounts WHERE account_id = 202 FOR UPDATE;

-- Validate
-- (application checks balance of 101 >= 500)

-- Execute
UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 202;

COMMIT;
```

### Example 2 — Optimistic Locking: Update Product Price

```sql
-- Read current state
SELECT product_id, price, version FROM products WHERE product_id = 10;
-- Returns: 10, 29.99, 5

-- Application computes new price
-- new_price = 34.99

-- Conditional update
BEGIN;
UPDATE products
SET    price   = 34.99,
       version = 6
WHERE  product_id = 10
  AND  version    = 5;

-- Check result
-- IF row_count = 0 → conflict detected → retry from read
-- IF row_count = 1 → success
COMMIT;
```

### Example 3 — Job Queue with SKIP LOCKED

```sql
-- Worker process picks up the next available job, skipping locked ones
BEGIN;

SELECT job_id, payload
FROM   job_queue
WHERE  status = 'pending'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;

-- Process the job...
UPDATE job_queue SET status = 'processing' WHERE job_id = :job_id;

COMMIT;
-- Other workers simultaneously pick up different jobs — no blocking
```

### Example 4 — Demonstrating MVCC Visibility (PostgreSQL)

```sql
-- Session 1:
BEGIN;
SELECT xmin, xmax, name, salary FROM employees WHERE emp_id = 1;
-- Returns: xmin=100, xmax=0, Alice, 95000

-- Session 2 (concurrent):
BEGIN;
UPDATE employees SET salary = 100000 WHERE emp_id = 1;
COMMIT;  -- new version created with xmin=200

-- Session 1 (still in same transaction):
SELECT xmin, xmax, name, salary FROM employees WHERE emp_id = 1;
-- Still returns: xmin=100, xmax=0, Alice, 95000
-- MVCC snapshot isolation: Session 1 sees the version from before Session 2's commit
COMMIT;

-- Session 1 (new transaction):
SELECT salary FROM employees WHERE emp_id = 1;
-- Now returns: 100000 — sees Session 2's committed version
```

### Example 5 — Deadlock Scenario and Resolution

```sql
-- Session 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- holds X-lock on account 1

-- Session 2:
BEGIN;
UPDATE accounts SET balance = balance - 200 WHERE account_id = 2;
-- holds X-lock on account 2

-- Session 1:
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
-- WAITS for Session 2's lock on account 2

-- Session 2:
UPDATE accounts SET balance = balance + 200 WHERE account_id = 1;
-- WAITS for Session 1's lock on account 1
-- DEADLOCK DETECTED!
-- PostgreSQL aborts one session:
-- ERROR: deadlock detected

-- Resolution: the aborted session rolls back; the other proceeds
-- Prevention: both sessions should lock account 1 first, then account 2
```

---

## 12. Common Interview Questions

**Q1. What is the difference between shared and exclusive locks?**  
A shared (S) lock allows concurrent reads but blocks writes. An exclusive (X) lock blocks all other locks (both shared and exclusive). Multiple S-locks can coexist; an X-lock is mutually exclusive with everything.

**Q2. What is Two-Phase Locking (2PL) and why is it important?**  
2PL divides a transaction into a growing phase (acquiring locks) and a shrinking phase (releasing locks). Once a lock is released, no new locks can be acquired. This guarantees serializability — the result is equivalent to some serial execution order.

**Q3. What is MVCC and which databases use it?**  
MVCC (Multi-Version Concurrency Control) maintains multiple versions of each row so readers see a consistent snapshot without blocking writers. Used by PostgreSQL, Oracle, MySQL InnoDB, and SQL Server (Snapshot Isolation). The key benefit: readers never block writers and vice versa.

**Q4. How does PostgreSQL implement MVCC?**  
Each row has hidden `xmin` (creating transaction) and `xmax` (deleting transaction) columns. An UPDATE creates a new row version (new `xmin`) and marks the old one dead (`xmax`). A transaction's snapshot determines which versions it can see based on the committed state of `xmin`/`xmax` at snapshot time. VACUUM garbage-collects dead versions.

**Q5. What is a deadlock and how does PostgreSQL handle it?**  
A deadlock occurs when two or more transactions each hold a lock the other needs, forming a circular wait. PostgreSQL detects deadlocks by checking for cycles in the wait-for graph (every `deadlock_timeout` seconds, default 1s). It aborts the least-costly transaction to break the cycle.

**Q6. How do you prevent deadlocks?**  
Lock resources in a consistent global order (e.g., ascending PK), keep transactions short, use `NOWAIT` or `SKIP LOCKED` to avoid waiting, and consider optimistic locking for low-contention scenarios.

**Q7. What is the difference between optimistic and pessimistic concurrency control?**  
Pessimistic control locks data before accessing it — assumes conflicts will happen. Optimistic control proceeds without locks and validates at commit — assumes conflicts are rare. Pessimistic is safer for high contention; optimistic gives better throughput for read-heavy, low-contention workloads.

**Q8. What is the Wait-Die deadlock prevention scheme?**  
If a transaction requesting a lock is older than the holder, it waits. If it is younger, it "dies" (aborts and restarts). This prevents cycles because waiting always goes from older to younger — no circular dependency can form.

**Q9. What is lock granularity and why does it matter?**  
Lock granularity determines the size of the data unit being locked: database, table, page, or row. Finer granularity (row-level) enables more concurrency but requires more lock management overhead. Coarser granularity (table-level) has less overhead but blocks more transactions.

**Q10. What is an intention lock?**  
An intention lock on a table signals that a transaction holds or intends to hold a finer-grained lock (row-level) inside that table. It prevents another transaction from acquiring a conflicting coarse-grained lock (e.g., table-level exclusive lock) without checking every row. Types: IS (Intention Shared), IX (Intention Exclusive), SIX (Shared + IX).

**Q11. What is transaction ID wraparound in PostgreSQL?**  
PostgreSQL uses 32-bit transaction IDs. After ~4 billion transactions, the counter wraps around. Without VACUUM freezing old rows, the database could misinterpret old rows as "in the future" and lose visibility. VACUUM FREEZE marks old rows with a special frozen flag to prevent this. If autovacuum falls behind, PostgreSQL will shut down to prevent data corruption.

**Q12. Explain the difference between `FOR UPDATE`, `FOR SHARE`, `FOR UPDATE NOWAIT`, and `FOR UPDATE SKIP LOCKED`.**  
- `FOR UPDATE`: Acquire exclusive row lock; wait if locked by another transaction.
- `FOR SHARE`: Acquire shared row lock; allows concurrent reads but blocks writes.
- `FOR UPDATE NOWAIT`: Acquire exclusive lock; fail immediately with an error if already locked.
- `FOR UPDATE SKIP LOCKED`: Acquire exclusive lock; silently skip rows that are already locked — perfect for work-queue patterns.

---

## 13. Key Takeaways

| Concept | One-line rule |
|---------|--------------|
| **Shared lock (S)** | Multiple readers allowed; blocks writers |
| **Exclusive lock (X)** | Blocks everything — one writer at a time |
| **Two-Phase Locking** | Grow then shrink; guarantees serializability |
| **Strict 2PL** | Hold X-locks until commit — standard in production |
| **Lock granularity** | Row-level for OLTP, table-level for bulk operations |
| **Intention locks** | Signal finer-grained locks inside; prevent coarse-lock conflicts |
| **Timestamp ordering** | Order by birth timestamp; abort on conflict — mostly academic |
| **MVCC** | Multiple row versions; readers never block writers |
| **PostgreSQL MVCC** | xmin/xmax per row; VACUUM reclaims dead versions |
| **Pessimistic locking** | Lock before read/write; safe for high contention |
| **Optimistic locking** | Version column; validate at commit; retry on conflict |
| **Deadlock** | Circular wait; detected via wait-for graph; victim aborted |
| **Prevent deadlocks** | Consistent lock order, short transactions, SKIP LOCKED |

### Concurrency Control Decision Guide

```
High contention (bank transfers, inventory)?
  → Pessimistic: SELECT ... FOR UPDATE
  → Lock in consistent order to avoid deadlocks

Low contention (user profiles, content edits)?
  → Optimistic: version column + conditional UPDATE
  → Retry on conflict (rare)

Read-heavy workload?
  → MVCC handles it naturally (readers never blocked)
  → Consider READ COMMITTED isolation

Job/task queue?
  → FOR UPDATE SKIP LOCKED — workers never block each other

Need serializability?
  → PostgreSQL SSI (SERIALIZABLE) — MVCC + dependency tracking
  → Or Strict 2PL at the cost of more blocking
```

---

*Phase 12 complete. Phase 13 covers Recovery & Crash Management (`13_recovery.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="12_concurrency_control.md" download="12_concurrency_control.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 12_concurrency_control.md
  </button>
</a>
