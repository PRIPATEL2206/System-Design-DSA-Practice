# Phase 11 — Transactions & ACID

> **Curriculum:** DBMS A-to-Z | **File:** `11_transactions_acid.md`  
> **Prerequisites:** Phases 1–10 (Foundations → Indexes & Optimization)

---

## Table of Contents

1. [What Is a Transaction?](#1-what-is-a-transaction)
2. [Transaction States](#2-transaction-states)
3. [ACID Properties — Deep Dive](#3-acid-properties--deep-dive)
4. [COMMIT, ROLLBACK & SAVEPOINT](#4-commit-rollback--savepoint)
5. [Isolation Levels](#5-isolation-levels)
6. [Concurrency Anomalies: Dirty, Non-Repeatable & Phantom Reads](#6-concurrency-anomalies)
7. [Isolation Levels vs Anomalies — The Matrix](#7-isolation-levels-vs-anomalies--the-matrix)
8. [Practical Isolation Level Examples](#8-practical-isolation-level-examples)
9. [Autocommit & Explicit Transactions](#9-autocommit--explicit-transactions)
10. [Transaction Design Patterns](#10-transaction-design-patterns)
11. [Worked Examples (Banking, E-Commerce, Inventory)](#11-worked-examples)
12. [Common Interview Questions](#12-common-interview-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. What Is a Transaction?

A **transaction** is a unit of work that groups one or more SQL statements so they execute as a single, indivisible operation. Either **all** statements succeed and are permanently saved, or **none** of them take effect — the database returns to its state before the transaction began.

### Real-World Analogy

A bank transfer: moving $500 from Account A to Account B involves two operations:
1. Debit Account A by $500.
2. Credit Account B by $500.

If the system crashes after step 1 but before step 2, $500 has vanished. A transaction groups both steps so they either both complete or both are cancelled — money is never lost.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="580" height="170" viewBox="0 0 580 170">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <!-- BEGIN -->
  <rect x="10" y="60" width="90" height="40" fill="#2C3E50" rx="6"/>
  <text x="55" y="85" text-anchor="middle" fill="white" font-weight="bold">BEGIN</text>

  <line x1="100" y1="80" x2="135" y2="80" stroke="#555" stroke-width="1.5" marker-end="url(#at)"/>
  <defs><marker id="at" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- Operations -->
  <rect x="135" y="40" width="130" height="35" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="200" y="63" text-anchor="middle" fill="#1A5276">Debit Acct A −$500</text>
  <rect x="135" y="85" width="130" height="35" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="200" y="108" text-anchor="middle" fill="#1A5276">Credit Acct B +$500</text>

  <!-- Success path -->
  <line x1="265" y1="57" x2="310" y2="57" stroke="#27AE60" stroke-width="1.5" marker-end="url(#atg)"/>
  <line x1="265" y1="102" x2="310" y2="102" stroke="#27AE60" stroke-width="1.5" marker-end="url(#atg)"/>
  <defs><marker id="atg" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#27AE60"/>
  </marker></defs>

  <rect x="310" y="60" width="90" height="40" fill="#27AE60" rx="6"/>
  <text x="355" y="85" text-anchor="middle" fill="white" font-weight="bold">COMMIT</text>
  <text x="355" y="140" text-anchor="middle" fill="#27AE60" font-size="11">Both changes permanent</text>

  <!-- Failure path -->
  <line x1="200" y1="120" x2="200" y2="150" stroke="#E74C3C" stroke-width="1.5"/>
  <line x1="200" y1="150" x2="460" y2="150" stroke="#E74C3C" stroke-width="1.5"/>
  <line x1="460" y1="150" x2="460" y2="100" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#atr)"/>
  <defs><marker id="atr" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/>
  </marker></defs>

  <rect x="420" y="60" width="100" height="40" fill="#E74C3C" rx="6"/>
  <text x="470" y="85" text-anchor="middle" fill="white" font-weight="bold">ROLLBACK</text>
  <text x="470" y="140" text-anchor="middle" fill="#E74C3C" font-size="11">All changes undone</text>
</svg>
```

### Why Transactions Matter

Without transactions, concurrent users and system failures would leave databases in **inconsistent intermediate states** — partial updates, phantom balances, duplicate orders. Transactions are the fundamental guarantee that data remains trustworthy.

---

## 2. Transaction States

A transaction passes through well-defined states from birth to completion (or failure).

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="280" viewBox="0 0 680 280">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.lbl{font-size:10px;fill:#555;font-style:italic}</style>
  <defs><marker id="ats" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- ACTIVE -->
  <rect x="270" y="10" width="140" height="50" fill="#2980B9" rx="8"/>
  <text x="340" y="32" text-anchor="middle" fill="white" font-weight="bold" font-size="12">ACTIVE</text>
  <text x="340" y="50" text-anchor="middle" fill="#AED6F1" font-size="10">Executing statements</text>

  <!-- ACTIVE → PARTIALLY COMMITTED -->
  <line x1="340" y1="60" x2="340" y2="100" stroke="#555" stroke-width="1.5" marker-end="url(#ats)"/>
  <text x="355" y="85" class="lbl">Last stmt executes</text>

  <!-- PARTIALLY COMMITTED -->
  <rect x="230" y="100" width="220" height="50" fill="#8E44AD" rx="8"/>
  <text x="340" y="122" text-anchor="middle" fill="white" font-weight="bold" font-size="12">PARTIALLY COMMITTED</text>
  <text x="340" y="140" text-anchor="middle" fill="#D7BDE2" font-size="10">All stmts done, not yet written to disk</text>

  <!-- PARTIALLY COMMITTED → COMMITTED -->
  <line x1="280" y1="150" x2="160" y2="200" stroke="#27AE60" stroke-width="1.5" marker-end="url(#ats)"/>
  <text x="185" y="182" class="lbl">Write to disk succeeds</text>

  <!-- PARTIALLY COMMITTED → FAILED -->
  <line x1="400" y1="150" x2="520" y2="200" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#ats)"/>
  <text x="490" y="182" class="lbl">Write fails / error</text>

  <!-- ACTIVE → FAILED -->
  <line x1="410" y1="35" x2="530" y2="90" stroke="#E74C3C" stroke-width="1.5" marker-end="url(#ats)"/>
  <text x="495" y="60" class="lbl">Error / abort</text>

  <!-- COMMITTED -->
  <rect x="60" y="200" width="160" height="50" fill="#27AE60" rx="8"/>
  <text x="140" y="222" text-anchor="middle" fill="white" font-weight="bold" font-size="12">COMMITTED</text>
  <text x="140" y="240" text-anchor="middle" fill="#A9DFBF" font-size="10">Changes permanent &amp; visible</text>

  <!-- FAILED -->
  <rect x="450" y="200" width="160" height="50" fill="#E74C3C" rx="8"/>
  <text x="530" y="222" text-anchor="middle" fill="white" font-weight="bold" font-size="12">FAILED</text>
  <text x="530" y="240" text-anchor="middle" fill="#FADBD8" font-size="10">Cannot proceed</text>

  <!-- FAILED → ABORTED -->
  <line x1="530" y1="250" x2="530" y2="265" stroke="#555" stroke-width="1.5" marker-end="url(#ats)"/>

  <!-- ABORTED -->
  <rect x="450" y="255" width="160" height="20" fill="#922B21" rx="4"/>
  <text x="530" y="268" text-anchor="middle" fill="white" font-size="10">ABORTED (rolled back)</text>

  <!-- ABORTED → restart or terminate -->
  <text x="340" y="275" text-anchor="middle" fill="#7D3C98" font-size="10" font-style="italic">From ABORTED: transaction can be restarted (new txn) or terminated</text>
</svg>
```

### State Descriptions

| State | Description |
|-------|-------------|
| **Active** | Transaction is executing statements; changes made to buffer pool but not yet on disk |
| **Partially Committed** | Last statement has executed; output in memory, waiting for write-ahead log flush |
| **Committed** | Log flushed to disk; changes are permanent and visible to others |
| **Failed** | An error occurred or the transaction was explicitly aborted |
| **Aborted** | Rollback complete; database restored to pre-transaction state |

---

## 3. ACID Properties — Deep Dive

**ACID** is the set of four properties that guarantee reliable transaction processing. Every production RDBMS (PostgreSQL, MySQL InnoDB, Oracle, SQL Server) guarantees all four.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="120" viewBox="0 0 680 120">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <rect x="10"  y="10" width="150" height="100" fill="#2980B9" rx="8"/>
  <text x="85"  y="40" text-anchor="middle" fill="white" font-size="22" font-weight="bold">A</text>
  <text x="85"  y="65" text-anchor="middle" fill="white" font-weight="bold">Atomicity</text>
  <text x="85"  y="85" text-anchor="middle" fill="#AED6F1" font-size="10">All or nothing</text>

  <rect x="180" y="10" width="150" height="100" fill="#8E44AD" rx="8"/>
  <text x="255" y="40" text-anchor="middle" fill="white" font-size="22" font-weight="bold">C</text>
  <text x="255" y="65" text-anchor="middle" fill="white" font-weight="bold">Consistency</text>
  <text x="255" y="85" text-anchor="middle" fill="#D7BDE2" font-size="10">Valid state → valid state</text>

  <rect x="350" y="10" width="150" height="100" fill="#27AE60" rx="8"/>
  <text x="425" y="40" text-anchor="middle" fill="white" font-size="22" font-weight="bold">I</text>
  <text x="425" y="65" text-anchor="middle" fill="white" font-weight="bold">Isolation</text>
  <text x="425" y="85" text-anchor="middle" fill="#A9DFBF" font-size="10">Concurrent txns invisible</text>

  <rect x="520" y="10" width="150" height="100" fill="#E67E22" rx="8"/>
  <text x="595" y="40" text-anchor="middle" fill="white" font-size="22" font-weight="bold">D</text>
  <text x="595" y="65" text-anchor="middle" fill="white" font-weight="bold">Durability</text>
  <text x="595" y="85" text-anchor="middle" fill="#FAD7A0" font-size="10">Committed = permanent</text>
</svg>
```

---

### A — Atomicity

**Definition:** A transaction is an atomic (indivisible) unit. Either ALL of its operations complete successfully and are committed, or NONE of them take effect and the database is rolled back.

**Mechanism:** The DBMS maintains an **undo log** (rollback log). If anything fails, it replays the undo log to reverse every completed operation within the transaction.

**Banking example:**

```sql
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;  -- Debit Alice
UPDATE accounts SET balance = balance + 500 WHERE account_id = 202;  -- Credit Bob

-- System crashes here? Or: second UPDATE fails due to constraint?
-- ROLLBACK undoes the first UPDATE too — Alice's balance is restored.
-- Money is never in limbo.

COMMIT;
```

**What atomicity prevents:** Partial updates — a debit without a corresponding credit, half an order placed, one inventory record updated but not its counterpart.

---

### C — Consistency

**Definition:** A transaction brings the database from one **valid state** to another valid state. Every constraint, rule, and integrity requirement must hold **before and after** the transaction.

**Mechanism:** Constraints (PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK, NOT NULL) are enforced at commit time. Application logic must also preserve business rules.

**Example — CHECK constraint upholds consistency:**

```sql
ALTER TABLE accounts ADD CONSTRAINT chk_positive_balance
    CHECK (balance >= 0);

BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 101;  -- balance was 800
-- Constraint violation: balance would become -200
-- Transaction ROLLS BACK — constraint was never broken permanently
COMMIT;
```

**Important nuance:** Consistency is partly the DBMS's job (constraints) and partly the **application's** job (business rules). If your application logic has a bug, the DBMS cannot save you from writing logically inconsistent data.

---

### I — Isolation

**Definition:** Concurrent transactions execute as if they were the **only** transaction running. The intermediate states of a transaction are invisible to other transactions.

**Mechanism:** Locks, MVCC (Multi-Version Concurrency Control), or a combination. The degree of isolation is configurable via **isolation levels** (covered in depth in Section 5).

**Example — without isolation:**

```
T1: reads account balance = 1000
T2: reads account balance = 1000 (concurrently)
T1: writes balance = 1000 - 500 = 500
T2: writes balance = 1000 - 300 = 700   ← WRONG! Overwrites T1's debit
Final balance = 700 instead of 200
```

**With isolation (SERIALIZABLE):**  
T2 waits for T1 to commit, then reads the updated balance (500) and writes 500 − 300 = 200. Correct.

---

### D — Durability

**Definition:** Once a transaction is committed, its changes are **permanent** — they survive system crashes, power failures, and restarts.

**Mechanism:** **Write-Ahead Logging (WAL)**. Before any data page is written to disk, the log record describing the change is written to the WAL file (which is flushed to durable storage). On recovery after a crash, the DBMS replays the WAL to restore all committed transactions.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="160" viewBox="0 0 560 160">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs><marker id="adur" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <rect x="10" y="50" width="110" height="50" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="65" y="72" text-anchor="middle" fill="#1A5276" font-weight="bold">Transaction</text>
  <text x="65" y="90" text-anchor="middle" fill="#1A5276">COMMIT;</text>

  <line x1="120" y1="75" x2="160" y2="75" stroke="#555" stroke-width="1.5" marker-end="url(#adur)"/>

  <rect x="160" y="40" width="130" height="70" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="225" y="66" text-anchor="middle" fill="#784212" font-weight="bold">WAL Log</text>
  <text x="225" y="84" text-anchor="middle" fill="#784212">Log record flushed</text>
  <text x="225" y="100" text-anchor="middle" fill="#784212">to durable disk FIRST</text>

  <line x1="290" y1="75" x2="330" y2="75" stroke="#555" stroke-width="1.5" marker-end="url(#adur)"/>

  <rect x="330" y="40" width="110" height="70" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="385" y="66" text-anchor="middle" fill="#2D6A2D" font-weight="bold">Data Pages</text>
  <text x="385" y="84" text-anchor="middle" fill="#2D6A2D">Written to disk</text>
  <text x="385" y="100" text-anchor="middle" fill="#2D6A2D">(async, later)</text>

  <line x1="440" y1="75" x2="480" y2="75" stroke="#555" stroke-width="1.5" marker-end="url(#adur)"/>

  <rect x="480" y="50" width="70" height="50" fill="#27AE60" rx="6"/>
  <text x="515" y="72" text-anchor="middle" fill="white" font-weight="bold">Durable</text>
  <text x="515" y="90" text-anchor="middle" fill="white">✓</text>

  <text x="280" y="145" fill="#7D3C98" font-size="11" font-style="italic" text-anchor="middle">WAL ensures committed data survives crashes — even before data pages hit disk</text>
</svg>
```

---

## 4. COMMIT, ROLLBACK & SAVEPOINT

### COMMIT

Permanently writes all changes made in the transaction to the database and releases all locks.

```sql
BEGIN;
    UPDATE inventory SET qty = qty - 1 WHERE product_id = 55;
    INSERT INTO order_items (order_id, product_id, qty, unit_price)
    VALUES (1001, 55, 1, 29.99);
COMMIT;   -- both changes are now permanent
```

### ROLLBACK

Undoes all changes made since the transaction began (or since the last SAVEPOINT) and releases all locks.

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
    UPDATE accounts SET balance = balance + 500 WHERE account_id = 999;
    -- Oops — account 999 doesn't exist, or some validation failed
ROLLBACK;  -- debit is undone, balance of 101 restored
```

### SAVEPOINT — Partial Rollback

A **SAVEPOINT** marks a point within a transaction. You can roll back to the savepoint without abandoning the entire transaction — partial work is preserved.

```sql
BEGIN;
    -- Step 1: Reserve hotel
    INSERT INTO reservations (booking_id, hotel_id, check_in, check_out)
    VALUES (2001, 5, '2024-07-01', '2024-07-05');

    SAVEPOINT after_hotel;

    -- Step 2: Reserve flight (might fail)
    INSERT INTO flight_bookings (booking_id, flight_id, seat)
    VALUES (2001, 'AA456', '14A');

    -- Step 3: Reserve rental car (fails — no cars available)
    INSERT INTO car_rentals (booking_id, car_id)
    VALUES (2001, 99);
    -- ERROR: car_id 99 not available

    -- Roll back only to after_hotel — keep the hotel reservation
    ROLLBACK TO SAVEPOINT after_hotel;

    -- Continue: try a different car
    INSERT INTO car_rentals (booking_id, car_id)
    VALUES (2001, 12);

COMMIT;
-- Hotel + car 12 committed; failed car 99 attempt is gone
```

### RELEASE SAVEPOINT

Removes a named savepoint (frees the log resources without rolling back):

```sql
SAVEPOINT sp1;
-- ... work ...
RELEASE SAVEPOINT sp1;   -- savepoint consumed; can't roll back to sp1 anymore
```

### Syntax Across Databases

| Operation | PostgreSQL | MySQL | SQL Server | Oracle |
|-----------|-----------|-------|-----------|--------|
| Begin | `BEGIN` / `START TRANSACTION` | `START TRANSACTION` | `BEGIN TRANSACTION` | Implicit |
| Commit | `COMMIT` | `COMMIT` | `COMMIT` | `COMMIT` |
| Rollback | `ROLLBACK` | `ROLLBACK` | `ROLLBACK` | `ROLLBACK` |
| Savepoint | `SAVEPOINT name` | `SAVEPOINT name` | `SAVE TRANSACTION name` | `SAVEPOINT name` |
| Rollback to sp | `ROLLBACK TO name` | `ROLLBACK TO name` | `ROLLBACK TRANSACTION name` | `ROLLBACK TO name` |

---

## 5. Isolation Levels

The SQL standard defines four isolation levels, ordered from least to most isolated. Higher isolation = fewer anomalies, but more locking/blocking and lower concurrency.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="200" viewBox="0 0 620 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Concurrency axis -->
  <line x1="30" y1="170" x2="590" y2="170" stroke="#2980B9" stroke-width="2" marker-end="url(#aiso)"/>
  <defs><marker id="aiso" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#2980B9"/>
  </marker></defs>
  <text x="310" y="190" text-anchor="middle" fill="#2980B9">Higher Concurrency →</text>

  <!-- Isolation axis -->
  <line x1="30" y1="170" x2="30" y2="20" stroke="#E74C3C" stroke-width="2" marker-end="url(#aiso2)"/>
  <defs><marker id="aiso2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/>
  </marker></defs>
  <text x="15" y="100" fill="#E74C3C" transform="rotate(-90,15,100)">Higher Isolation →</text>

  <!-- Levels -->
  <rect x="50"  y="130" width="130" height="35" fill="#27AE60" rx="6"/>
  <text x="115" y="153" text-anchor="middle" fill="white" font-weight="bold">READ UNCOMMITTED</text>

  <rect x="200" y="95" width="130" height="35" fill="#2980B9" rx="6"/>
  <text x="265" y="118" text-anchor="middle" fill="white" font-weight="bold">READ COMMITTED</text>

  <rect x="350" y="60" width="130" height="35" fill="#8E44AD" rx="6"/>
  <text x="415" y="83" text-anchor="middle" fill="white" font-weight="bold">REPEATABLE READ</text>

  <rect x="460" y="25" width="130" height="35" fill="#C0392B" rx="6"/>
  <text x="525" y="48" text-anchor="middle" fill="white" font-weight="bold">SERIALIZABLE</text>

  <text x="50"  y="125" fill="#27AE60"  font-size="10">Most concurrent</text>
  <text x="460" y="20"  fill="#C0392B"  font-size="10">Most isolated</text>
</svg>
```

### Setting the Isolation Level

```sql
-- PostgreSQL / MySQL
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
-- ... statements ...
COMMIT;

-- Or for the whole session:
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- PostgreSQL
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;                      -- MySQL

-- SQL Server
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
-- ...
COMMIT;

-- Oracle (supports only READ COMMITTED and SERIALIZABLE)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## 6. Concurrency Anomalies

When transactions run concurrently without sufficient isolation, three classic **anomalies** can occur.

### Anomaly 1 — Dirty Read

**T1 reads data that T2 has modified but not yet committed.** If T2 rolls back, T1 has read data that never actually existed.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="200" viewBox="0 0 560 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <!-- Timeline -->
  <line x1="80" y1="30" x2="80" y2="180" stroke="#2980B9" stroke-width="2"/>
  <line x1="400" y1="30" x2="400" y2="180" stroke="#E74C3C" stroke-width="2"/>
  <text x="80"  y="22" text-anchor="middle" fill="#2980B9" font-weight="bold">T1</text>
  <text x="400" y="22" text-anchor="middle" fill="#E74C3C" font-weight="bold">T2</text>
  <!-- T2 writes dirty value -->
  <line x1="400" y1="60" x2="200" y2="60" stroke="#E67E22" stroke-width="1.5" stroke-dasharray="5"/>
  <rect x="280" y="45" width="200" height="25" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="380" y="62" text-anchor="middle" fill="#784212">T2: UPDATE balance=1500 (not committed)</text>
  <!-- T1 reads dirty value -->
  <line x1="80" y1="100" x2="280" y2="100" stroke="#2980B9" stroke-width="1.5"/>
  <rect x="90" y="85" width="190" height="25" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="185" y="102" text-anchor="middle" fill="#1A5276">T1: READ balance → 1500 ← DIRTY!</text>
  <!-- T2 rolls back -->
  <rect x="310" y="125" width="160" height="25" fill="#FDEDEC" stroke="#C0392B" rx="4"/>
  <text x="390" y="142" text-anchor="middle" fill="#C0392B">T2: ROLLBACK</text>
  <!-- Result -->
  <text x="280" y="182" text-anchor="middle" fill="#C0392B" font-size="11" font-style="italic">T1 acted on balance=1500 — value that never existed!</text>
</svg>
```

**Prevented by:** READ COMMITTED and above.

---

### Anomaly 2 — Non-Repeatable Read

**T1 reads a row twice within the same transaction, but T2 updates the row between the two reads.** T1 gets different values for the same row.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="220" viewBox="0 0 560 220">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <line x1="80"  y1="30" x2="80"  y2="200" stroke="#2980B9" stroke-width="2"/>
  <line x1="400" y1="30" x2="400" y2="200" stroke="#E74C3C" stroke-width="2"/>
  <text x="80"  y="22" text-anchor="middle" fill="#2980B9" font-weight="bold">T1</text>
  <text x="400" y="22" text-anchor="middle" fill="#E74C3C" font-weight="bold">T2</text>

  <rect x="90" y="45" width="190" height="25" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="185" y="62" text-anchor="middle" fill="#1A5276">T1: READ balance → 1000</text>

  <rect x="310" y="85" width="180" height="25" fill="#FDEDEC" stroke="#C0392B" rx="4"/>
  <text x="400" y="102" text-anchor="middle" fill="#C0392B">T2: UPDATE balance=1500; COMMIT</text>

  <rect x="90" y="125" width="220" height="25" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="200" y="142" text-anchor="middle" fill="#1A5276">T1: READ balance again → 1500 ← CHANGED!</text>

  <text x="280" y="200" text-anchor="middle" fill="#C0392B" font-size="11" font-style="italic">Same query, same transaction, different results. T1 cannot rely on its first read.</text>
</svg>
```

**Prevented by:** REPEATABLE READ and above.

---

### Anomaly 3 — Phantom Read

**T1 runs a range query twice. Between the two reads, T2 inserts (or deletes) rows that match T1's query.** The set of rows changes — "phantom" rows appear or disappear.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="230" viewBox="0 0 560 230">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <line x1="80"  y1="30" x2="80"  y2="210" stroke="#2980B9" stroke-width="2"/>
  <line x1="400" y1="30" x2="400" y2="210" stroke="#E74C3C" stroke-width="2"/>
  <text x="80"  y="22" text-anchor="middle" fill="#2980B9" font-weight="bold">T1</text>
  <text x="400" y="22" text-anchor="middle" fill="#E74C3C" font-weight="bold">T2</text>

  <rect x="30" y="45" width="280" height="25" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="170" y="62" text-anchor="middle" fill="#1A5276">T1: SELECT COUNT(*) WHERE salary&gt;80k → 3 rows</text>

  <rect x="300" y="85" width="240" height="25" fill="#FDEDEC" stroke="#C0392B" rx="4"/>
  <text x="420" y="102" text-anchor="middle" fill="#C0392B">T2: INSERT employee (salary=95000); COMMIT</text>

  <rect x="30" y="130" width="290" height="25" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="175" y="147" text-anchor="middle" fill="#1A5276">T1: SELECT COUNT(*) WHERE salary&gt;80k → 4 rows ← PHANTOM!</text>

  <text x="280" y="208" text-anchor="middle" fill="#C0392B" font-size="11" font-style="italic">New row appeared mid-transaction. T1's count is inconsistent.</text>
</svg>
```

**Prevented by:** SERIALIZABLE only (and REPEATABLE READ in MySQL InnoDB via gap locks).

---

### Anomaly 4 — Lost Update (bonus)

Two transactions read a value, modify it, and write back — the second write overwrites the first without seeing it.

```sql
-- T1 and T2 both read balance = 1000
T1: balance = 1000 - 200 = 800  → WRITE 800
T2: balance = 1000 - 300 = 700  → WRITE 700  (overwrites T1's 800)
-- Final: 700, should be 500
```

**Prevented by:** Proper locking, REPEATABLE READ (which detects the conflict), or optimistic locking with a version column.

---

## 7. Isolation Levels vs Anomalies — The Matrix

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="660" height="190" viewBox="0 0 660 190">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Header -->
  <rect x="0" y="0" width="660" height="35" fill="#2C3E50" rx="4"/>
  <text x="130" y="22" text-anchor="middle" fill="white" font-weight="bold">Isolation Level</text>
  <text x="310" y="22" text-anchor="middle" fill="white" font-weight="bold">Dirty Read</text>
  <text x="440" y="22" text-anchor="middle" fill="white" font-weight="bold">Non-Repeatable Read</text>
  <text x="590" y="22" text-anchor="middle" fill="white" font-weight="bold">Phantom Read</text>

  <!-- READ UNCOMMITTED -->
  <rect x="0" y="35" width="660" height="38" fill="#FDEDEC"/>
  <text x="130" y="58" text-anchor="middle" fill="#922B21" font-weight="bold">READ UNCOMMITTED</text>
  <text x="310" y="58" text-anchor="middle" fill="#E74C3C">❌ Possible</text>
  <text x="440" y="58" text-anchor="middle" fill="#E74C3C">❌ Possible</text>
  <text x="590" y="58" text-anchor="middle" fill="#E74C3C">❌ Possible</text>

  <!-- READ COMMITTED -->
  <rect x="0" y="73" width="660" height="38" fill="#EBF5FB"/>
  <text x="130" y="96" text-anchor="middle" fill="#1A5276" font-weight="bold">READ COMMITTED</text>
  <text x="310" y="96" text-anchor="middle" fill="#27AE60">✅ Prevented</text>
  <text x="440" y="96" text-anchor="middle" fill="#E74C3C">❌ Possible</text>
  <text x="590" y="96" text-anchor="middle" fill="#E74C3C">❌ Possible</text>

  <!-- REPEATABLE READ -->
  <rect x="0" y="111" width="660" height="38" fill="#F0FFF4"/>
  <text x="130" y="134" text-anchor="middle" fill="#145A32" font-weight="bold">REPEATABLE READ</text>
  <text x="310" y="134" text-anchor="middle" fill="#27AE60">✅ Prevented</text>
  <text x="440" y="134" text-anchor="middle" fill="#27AE60">✅ Prevented</text>
  <text x="590" y="134" text-anchor="middle" fill="#E74C3C">❌ Possible*</text>

  <!-- SERIALIZABLE -->
  <rect x="0" y="149" width="660" height="38" fill="#F9FBE7"/>
  <text x="130" y="172" text-anchor="middle" fill="#7D6608" font-weight="bold">SERIALIZABLE</text>
  <text x="310" y="172" text-anchor="middle" fill="#27AE60">✅ Prevented</text>
  <text x="440" y="172" text-anchor="middle" fill="#27AE60">✅ Prevented</text>
  <text x="590" y="172" text-anchor="middle" fill="#27AE60">✅ Prevented</text>
</svg>
```

> \* MySQL InnoDB's REPEATABLE READ also prevents phantom reads via gap locks — an extension beyond the SQL standard.

### Default Isolation Levels by RDBMS

| Database | Default Isolation Level |
|----------|------------------------|
| PostgreSQL | READ COMMITTED |
| MySQL InnoDB | REPEATABLE READ |
| Oracle | READ COMMITTED |
| SQL Server | READ COMMITTED |
| DB2 | CURSOR STABILITY (≈ READ COMMITTED) |

---

## 8. Practical Isolation Level Examples

### READ UNCOMMITTED — Dirty read demonstration

```sql
-- Session 1 (T1): start a transaction, update but don't commit
BEGIN;
UPDATE accounts SET balance = 9999 WHERE account_id = 1;
-- DO NOT COMMIT YET

-- Session 2 (T2): set to READ UNCOMMITTED, read the dirty value
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE account_id = 1;
-- Returns 9999 — dirty read of T1's uncommitted change
COMMIT;

-- Session 1: rollback — 9999 never existed
ROLLBACK;
```

**Use case (rare):** Approximate aggregate counts on very large tables where perfect accuracy isn't required and you want maximum concurrency.

---

### READ COMMITTED — Most common default

```sql
-- Session 1 (T1): read an account balance
BEGIN;
SELECT balance FROM accounts WHERE account_id = 1;
-- Returns 1000

-- Session 2 (T2): update and commit
BEGIN;
UPDATE accounts SET balance = 1500 WHERE account_id = 1;
COMMIT;

-- Session 1 (T1): read the same account again
SELECT balance FROM accounts WHERE account_id = 1;
-- Returns 1500 — non-repeatable read! (T2's committed change is now visible)
COMMIT;
```

**Use case:** OLTP applications (e-commerce, CRM) where you want to see the latest committed data. The default for PostgreSQL and Oracle.

---

### REPEATABLE READ — Stable snapshot

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Session 1 (T1): read salary at the start of the transaction
BEGIN;
SELECT salary FROM employees WHERE emp_id = 5;
-- Returns 70000

-- Session 2 (T2): update and commit
UPDATE employees SET salary = 80000 WHERE emp_id = 5;
COMMIT;

-- Session 1 (T1): read salary again — still sees 70000 (snapshot at txn start)
SELECT salary FROM employees WHERE emp_id = 5;
-- Returns 70000 — repeatable read guaranteed
COMMIT;
```

**Use case:** Financial reports that must be self-consistent — e.g., computing totals and line items from the same snapshot.

---

### SERIALIZABLE — Full isolation

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- T1: count employees with salary > 80000 in dept 10
BEGIN;
SELECT COUNT(*) FROM employees WHERE dept_id = 10 AND salary > 80000;
-- Returns 2

-- T2: insert a new high-salary employee in dept 10
BEGIN;
INSERT INTO employees (emp_id, name, dept_id, salary)
VALUES (99, 'New Hire', 10, 95000);
COMMIT;
-- BLOCKED or CONFLICT DETECTED in T1's range — depending on implementation

-- T1: count again
SELECT COUNT(*) FROM employees WHERE dept_id = 10 AND salary > 80000;
-- Returns 2 (T2's insert is not visible — phantoms prevented)
COMMIT;
```

**Use case:** Bank transfers, seat reservations, stock trading — anywhere concurrent anomalies could cause financial or logical errors.

---

## 9. Autocommit & Explicit Transactions

### Autocommit Mode

Most databases run in **autocommit** mode by default: every individual SQL statement is automatically wrapped in its own transaction and committed immediately.

```sql
-- In autocommit mode (default for psql, MySQL CLI):
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
-- This is automatically: BEGIN; UPDATE ...; COMMIT;

-- Danger: no way to rollback after autocommit
```

### Disabling Autocommit

```sql
-- PostgreSQL: autocommit is always ON at the psql level
-- Use BEGIN; to start an explicit transaction block (disables autocommit for that block)
BEGIN;
UPDATE ...;
UPDATE ...;
COMMIT;          -- or ROLLBACK

-- MySQL: disable session autocommit
SET autocommit = 0;
UPDATE ...;
COMMIT;          -- must commit explicitly
SET autocommit = 1;

-- Python (psycopg2 / SQLAlchemy): connection.autocommit = False
-- Java (JDBC): connection.setAutoCommit(false);
```

### Application-Level Transaction Management

```python
# Python + psycopg2
import psycopg2

conn = psycopg2.connect("dbname=bank")
conn.autocommit = False          # disable autocommit

try:
    cur = conn.cursor()
    cur.execute("UPDATE accounts SET balance = balance - 500 WHERE id = 1")
    cur.execute("UPDATE accounts SET balance = balance + 500 WHERE id = 2")
    conn.commit()                # persist both changes
except Exception as e:
    conn.rollback()              # undo everything
    raise e
finally:
    cur.close()
    conn.close()
```

---

## 10. Transaction Design Patterns

### Pattern 1 — Keep Transactions Short

Long transactions hold locks, block other users, and increase rollback cost on failure. Golden rule: **do all computation outside the transaction, then wrap only the writes**.

```sql
-- BAD: long transaction holds locks while doing slow external work
BEGIN;
SELECT product_id, price FROM products WHERE product_id = 55;
-- ... application calls external pricing API (takes 2 seconds) ...
UPDATE order_items SET unit_price = 29.99 WHERE order_id = 1001 AND product_id = 55;
COMMIT;

-- GOOD: fetch data, compute outside the transaction, then open a short write transaction
-- (App code: price = fetch_from_api(); )
BEGIN;
UPDATE order_items SET unit_price = 29.99 WHERE order_id = 1001 AND product_id = 55;
COMMIT;  -- transaction held for microseconds
```

### Pattern 2 — Consistent Lock Ordering to Prevent Deadlocks

Always acquire locks in the **same order** across all transactions:

```sql
-- BAD: T1 locks A then B; T2 locks B then A → deadlock
T1: LOCK TABLE accounts; UPDATE account A; UPDATE account B;
T2: LOCK TABLE accounts; UPDATE account B; UPDATE account A;  -- deadlock!

-- GOOD: always lock the lower account_id first
BEGIN;
SELECT * FROM accounts WHERE account_id = LEAST(101, 202)    FOR UPDATE;
SELECT * FROM accounts WHERE account_id = GREATEST(101, 202) FOR UPDATE;
UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 202;
COMMIT;
```

### Pattern 3 — Idempotent Transactions

Design transactions so they can be safely retried on failure without duplicate effects:

```sql
-- Idempotent: use INSERT ... ON CONFLICT DO NOTHING / DO UPDATE
INSERT INTO order_status (order_id, status, updated_at)
VALUES (1001, 'Shipped', NOW())
ON CONFLICT (order_id) DO UPDATE
    SET status = EXCLUDED.status,
        updated_at = EXCLUDED.updated_at;
```

### Pattern 4 — Optimistic Locking with a Version Column

For low-contention scenarios where you want to detect, not prevent, concurrent modification:

```sql
-- Table has a version column
ALTER TABLE products ADD COLUMN version INT DEFAULT 0;

-- Read the version
SELECT product_id, price, version FROM products WHERE product_id = 55;
-- Returns: 55, 29.99, 7

-- Update only if version hasn't changed
UPDATE products
SET    price   = 31.99,
       version = version + 1
WHERE  product_id = 55
  AND  version    = 7;   -- conditional on the version we read

-- If rows_affected = 0: someone else modified it first → retry
```

---

## 11. Worked Examples

### Example 1 — Banking Transfer (Atomicity + Isolation)

```sql
BEGIN;

-- Lock both rows to prevent concurrent modification
SELECT balance FROM accounts WHERE account_id IN (101, 202) FOR UPDATE;

-- Validate sufficient funds
DO $$
DECLARE v_balance NUMERIC;
BEGIN
    SELECT balance INTO v_balance FROM accounts WHERE account_id = 101;
    IF v_balance < 500 THEN
        RAISE EXCEPTION 'Insufficient funds: balance = %', v_balance;
    END IF;
END $$;

-- Execute transfer
UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 202;

-- Record the transaction
INSERT INTO transfer_log (from_account, to_account, amount, txn_time)
VALUES (101, 202, 500, NOW());

COMMIT;
```

### Example 2 — E-Commerce Order Placement (SAVEPOINT)

```sql
BEGIN;

-- Insert the order header
INSERT INTO orders (order_id, customer_id, order_date, status)
VALUES (5001, 42, NOW(), 'Pending');

SAVEPOINT order_header_saved;

-- Try to insert each order item
INSERT INTO order_items (order_id, product_id, qty, unit_price)
VALUES (5001, 10, 2, 49.99);

INSERT INTO order_items (order_id, product_id, qty, unit_price)
VALUES (5001, 99, 1, 199.99);
-- product_id 99 doesn't exist → FK violation!

ROLLBACK TO SAVEPOINT order_header_saved;
-- Order header preserved, item inserts undone

-- Log the issue
INSERT INTO order_errors (order_id, error_msg)
VALUES (5001, 'Product 99 not found');

COMMIT;
-- Order header + error log committed; item failures cleanly handled
```

### Example 3 — Inventory Update with Optimistic Locking (E-Commerce)

```sql
-- Step 1: Read current stock and version
SELECT product_id, stock_qty, version
FROM   products
WHERE  product_id = 101;
-- Returns: 101, 50, 12

-- Step 2: Attempt update with version check
BEGIN;
UPDATE products
SET    stock_qty = stock_qty - 3,
       version   = version + 1
WHERE  product_id = 101
  AND  version    = 12          -- optimistic lock check
  AND  stock_qty  >= 3;         -- business rule: must have enough stock

-- Check if update succeeded
GET DIAGNOSTICS rows_updated = ROW_COUNT;
IF rows_updated = 0 THEN
    ROLLBACK;
    -- Retry: re-read stock and version, try again
ELSE
    COMMIT;
END IF;
```

### Example 4 — Read-Consistent Report (REPEATABLE READ)

```sql
-- Accounts receivable report must be internally consistent
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;

-- All three queries see the exact same snapshot of the database
SELECT SUM(invoice_amount) AS total_invoiced FROM invoices WHERE status = 'Open';
SELECT SUM(payment_amount) AS total_paid     FROM payments  WHERE payment_date <= CURRENT_DATE;
SELECT SUM(balance)        AS total_balance  FROM accounts  WHERE account_type = 'AR';
-- These three numbers will be consistent even if other transactions commit between them

COMMIT;
```

---

## 12. Common Interview Questions

**Q1. What does ACID stand for and what does each property guarantee?**  
- **Atomicity:** All operations in a transaction succeed or all are rolled back — no partial updates.  
- **Consistency:** A transaction takes the database from one valid state to another; all constraints hold.  
- **Isolation:** Concurrent transactions do not interfere with each other.  
- **Durability:** Committed transactions survive system crashes via WAL/redo logs.

**Q2. What is a dirty read?**  
Reading data written by a transaction that hasn't committed yet. If the writer rolls back, the reader has seen data that never officially existed. Prevented by READ COMMITTED and above.

**Q3. What is the difference between a non-repeatable read and a phantom read?**  
A non-repeatable read is when a row you read earlier now returns different values (another transaction updated it). A phantom read is when a range query returns different rows (another transaction inserted or deleted rows matching your WHERE clause). Non-repeatable reads are about changed values; phantom reads are about a changed set of rows.

**Q4. What isolation level is the default in PostgreSQL vs MySQL?**  
PostgreSQL defaults to READ COMMITTED. MySQL InnoDB defaults to REPEATABLE READ.

**Q5. How does SERIALIZABLE isolation work without locking every row?**  
Modern databases use **Serializable Snapshot Isolation (SSI)** (PostgreSQL 9.1+). Transactions run concurrently on snapshots, but the engine tracks read/write dependencies and aborts any transaction whose execution would not be equivalent to some serial order. This provides full isolation with better concurrency than traditional lock-based serializable.

**Q6. What is a SAVEPOINT used for?**  
To create a rollback point within a transaction, allowing partial rollback without abandoning the entire transaction. Useful in complex operations where one step may fail (e.g., a multi-resource booking) and you want to retry or log the failure while preserving successful earlier steps.

**Q7. What is the difference between ROLLBACK and ROLLBACK TO SAVEPOINT?**  
`ROLLBACK` undoes the entire transaction back to its start. `ROLLBACK TO SAVEPOINT name` undoes only to the named checkpoint; the transaction remains open and can continue.

**Q8. What is the lost update problem and how do you prevent it?**  
Two transactions read the same value, each modifies it based on the read, and the second write overwrites the first. Prevention: use `SELECT ... FOR UPDATE` (pessimistic locking), optimistic locking (version column), or SERIALIZABLE isolation.

**Q9. What is Write-Ahead Logging (WAL) and why is it critical for durability?**  
WAL requires that every change be written to a sequential log on durable storage before the data page is written. On a crash, the database replays the WAL to restore all committed transactions and undo any uncommitted ones — guaranteeing durability and enabling fast, safe recovery.

**Q10. Can a transaction span multiple databases?**  
Yes, using a **distributed transaction** with a two-phase commit (2PC) protocol. A coordinator sends a "prepare" phase to all participants; if all agree, it sends "commit"; otherwise "rollback". This is covered in Phase 17 (Distributed Databases).

**Q11. What does `SELECT ... FOR UPDATE` do?**  
It acquires an exclusive row-level lock on the selected rows, preventing other transactions from updating or locking the same rows until the current transaction commits or rolls back. Used for pessimistic concurrency control.

**Q12. Why should transactions be kept as short as possible?**  
Long transactions hold locks that block other users, increase the risk of deadlocks, consume WAL/undo log space, and significantly increase rollback time if they fail. Short transactions maximise concurrency and reduce the blast radius of failures.

---

## 13. Key Takeaways

| Concept | One-line rule |
|---------|--------------|
| **Atomicity** | All or nothing — undo log rolls back partial transactions |
| **Consistency** | Constraints must hold before and after; logic is app's responsibility |
| **Isolation** | Concurrent transactions are invisible to each other — configurable via level |
| **Durability** | WAL ensures committed changes survive crashes |
| **READ UNCOMMITTED** | Dirty reads possible — almost never use |
| **READ COMMITTED** | No dirty reads; default for PostgreSQL, Oracle, SQL Server |
| **REPEATABLE READ** | No dirty/non-repeatable reads; default for MySQL InnoDB |
| **SERIALIZABLE** | Full isolation — no anomalies; use for financial/critical ops |
| **SAVEPOINT** | Partial rollback within a transaction |
| **Keep txns short** | Short transactions = fewer locks, higher concurrency, faster recovery |
| **WAL** | Write-ahead log is the engine of durability |
| **FOR UPDATE** | Pessimistic lock — prevents lost updates |

### Quick Decision Guide

```
What isolation level should I use?
  ├─ Approximate reads, maximum throughput?   → READ UNCOMMITTED (rare)
  ├─ General OLTP (e-commerce, CRM)?          → READ COMMITTED (default)
  ├─ Self-consistent financial reports?       → REPEATABLE READ
  └─ Bank transfers, seat reservations?       → SERIALIZABLE

How to prevent concurrent modification?
  ├─ High contention, must not retry?         → SELECT ... FOR UPDATE (pessimistic)
  └─ Low contention, retry is acceptable?     → Version column (optimistic)
```

---

*Phase 11 complete. Phase 12 covers Concurrency Control (`12_concurrency_control.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="11_transactions_acid.md" download="11_transactions_acid.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 11_transactions_acid.md
  </button>
</a>
