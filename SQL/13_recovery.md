# Phase 13 — Recovery & Crash Management

> **Curriculum:** DBMS A-to-Z | **File:** `13_recovery.md`  
> **Prerequisites:** Phases 1–12 (Foundations → Concurrency Control)

---

## Table of Contents

1. [Why Recovery Matters](#1-why-recovery-matters)
2. [Types of Failures](#2-types-of-failures)
3. [Log-Based Recovery: Undo & Redo Logs](#3-log-based-recovery-undo--redo-logs)
4. [Write-Ahead Logging (WAL)](#4-write-ahead-logging-wal)
5. [Checkpointing](#5-checkpointing)
6. [The ARIES Recovery Algorithm](#6-the-aries-recovery-algorithm)
7. [Shadow Paging](#7-shadow-paging)
8. [Recovery in PostgreSQL](#8-recovery-in-postgresql)
9. [Recovery in MySQL InnoDB](#9-recovery-in-mysql-innodb)
10. [Backup & Point-in-Time Recovery (PITR)](#10-backup--point-in-time-recovery-pitr)
11. [Worked Examples](#11-worked-examples)
12. [Common Interview Questions](#12-common-interview-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Why Recovery Matters

Databases must survive the unexpected: power failures, OS crashes, disk corruption, killed processes, and hardware faults. The **recovery subsystem** ensures that after any crash, the database can be restored to a **consistent state** — all committed transactions present, all uncommitted transactions undone — without data loss.

### The Two Guarantees

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="120" viewBox="0 0 560 120">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <rect x="10" y="15" width="250" height="80" fill="#27AE60" rx="8"/>
  <text x="135" y="42" text-anchor="middle" fill="white" font-weight="bold" font-size="14">REDO</text>
  <text x="135" y="62" text-anchor="middle" fill="white">Committed transactions</text>
  <text x="135" y="80" text-anchor="middle" fill="white">MUST be present after crash</text>

  <rect x="300" y="15" width="250" height="80" fill="#E74C3C" rx="8"/>
  <text x="425" y="42" text-anchor="middle" fill="white" font-weight="bold" font-size="14">UNDO</text>
  <text x="425" y="62" text-anchor="middle" fill="white">Uncommitted transactions</text>
  <text x="425" y="80" text-anchor="middle" fill="white">MUST be removed after crash</text>
</svg>
```

Without recovery:
- **Durability** is violated — committed data disappears.
- **Atomicity** is violated — partial transactions remain.
- **Consistency** is violated — the database may contain contradictory data.

---

## 2. Types of Failures

| Failure Type | Cause | Data at risk | Recovery method |
|--------------|-------|-------------|----------------|
| **Transaction failure** | Application error, constraint violation, deadlock abort | Only the failed transaction | ROLLBACK (undo log) |
| **System failure (soft crash)** | OS crash, power loss, process killed | Buffered data not yet flushed to disk | WAL replay (redo + undo) |
| **Media failure (hard crash)** | Disk corruption, drive failure | Entire data files | Restore from backup + WAL replay |
| **Network failure** | Partition, timeout | Distributed transactions | 2PC protocol, failover |

### Volatile vs Non-Volatile Storage

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="600" height="170" viewBox="0 0 600 170">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- RAM (volatile) -->
  <rect x="10" y="20" width="240" height="120" fill="#FDEDEC" stroke="#C0392B" rx="8"/>
  <text x="130" y="44" text-anchor="middle" font-weight="bold" fill="#7B241C" font-size="12">Volatile (RAM)</text>
  <line x1="20" y1="52" x2="240" y2="52" stroke="#F5B7B1"/>
  <text x="130" y="72" text-anchor="middle" fill="#555">Buffer Pool (dirty pages)</text>
  <text x="130" y="90" text-anchor="middle" fill="#555">Transaction state</text>
  <text x="130" y="108" text-anchor="middle" fill="#555">Lock tables</text>
  <text x="130" y="130" text-anchor="middle" fill="#E74C3C" font-weight="bold">Lost on crash ⚡</text>

  <!-- Arrow -->
  <text x="280" y="80" text-anchor="middle" fill="#555" font-size="20">→</text>
  <text x="280" y="100" text-anchor="middle" fill="#555" font-size="10">flush</text>

  <!-- Disk (non-volatile) -->
  <rect x="310" y="20" width="270" height="120" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="445" y="44" text-anchor="middle" font-weight="bold" fill="#145A32" font-size="12">Non-Volatile (Disk / SSD)</text>
  <line x1="320" y1="52" x2="570" y2="52" stroke="#A9DFBF"/>
  <text x="445" y="72" text-anchor="middle" fill="#555">Data files (heap, indexes)</text>
  <text x="445" y="90" text-anchor="middle" fill="#555">WAL / Redo log files</text>
  <text x="445" y="108" text-anchor="middle" fill="#555">Undo log (or undo tablespace)</text>
  <text x="445" y="130" text-anchor="middle" fill="#27AE60" font-weight="bold">Survives crash ✓</text>
</svg>
```

The gap between what's in RAM and what's on disk is the core problem. **Write-Ahead Logging** bridges it.

---

## 3. Log-Based Recovery: Undo & Redo Logs

### What Is a Log?

The recovery log (also called the **transaction log** or **write-ahead log**) is an **append-only** sequential file on durable storage. Every data modification generates a **log record** before the actual data page is written.

### Log Record Structure

```
<LSN, TxnID, PageID, Offset, Length, OldValue, NewValue, PrevLSN>
```

| Field | Meaning |
|-------|---------|
| **LSN** | Log Sequence Number — monotonically increasing unique identifier |
| **TxnID** | Transaction that produced this record |
| **PageID** | Data page being modified |
| **OldValue** | Value before the change (for UNDO) |
| **NewValue** | Value after the change (for REDO) |
| **PrevLSN** | Previous log record for the same transaction (forms a chain) |

### Undo Log

Contains the **old value** — used to reverse uncommitted changes.

```
Log: <LSN=101, T1, accounts.101, balance, 1000, -> >
     T1 changed balance from 1000 to something.
     If T1 didn't commit → UNDO: restore balance to 1000
```

### Redo Log

Contains the **new value** — used to reapply committed changes that may not have reached disk.

```
Log: <LSN=101, T1, accounts.101, balance, -> , 500>
     T1 set balance to 500.
     If T1 committed but the data page wasn't flushed → REDO: set balance to 500
```

### Undo/Redo Log (most common)

Stores **both** old and new values in the same record — supports both operations.

```
Log: <LSN=101, T1, accounts.101, balance, 1000, 500>
     T1 changed balance from 1000 to 500.
     UNDO: set to 1000 (if uncommitted)
     REDO: set to 500 (if committed but not flushed)
```

---

## 4. Write-Ahead Logging (WAL)

### The WAL Protocol

**Rule:** Before a modified data page is written to disk, all log records describing that modification MUST be flushed to durable storage first.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="250" viewBox="0 0 620 250">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs><marker id="aw" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <text x="310" y="18" text-anchor="middle" font-weight="bold" fill="#333" font-size="13">Write-Ahead Logging — Execution Flow</text>

  <!-- Step 1 -->
  <rect x="10" y="35" width="150" height="50" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="85" y="56" text-anchor="middle" fill="#1A5276" font-weight="bold">① Modify Page</text>
  <text x="85" y="74" text-anchor="middle" fill="#1A5276">in Buffer Pool (RAM)</text>

  <line x1="160" y1="60" x2="200" y2="60" stroke="#555" stroke-width="1.5" marker-end="url(#aw)"/>

  <!-- Step 2 -->
  <rect x="200" y="35" width="170" height="50" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="285" y="56" text-anchor="middle" fill="#784212" font-weight="bold">② Write Log Record</text>
  <text x="285" y="74" text-anchor="middle" fill="#784212">to WAL buffer (RAM)</text>

  <line x1="370" y1="60" x2="410" y2="60" stroke="#555" stroke-width="1.5" marker-end="url(#aw)"/>

  <!-- Step 3 -->
  <rect x="410" y="35" width="190" height="50" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="505" y="56" text-anchor="middle" fill="#2D6A2D" font-weight="bold">③ Flush WAL to Disk</text>
  <text x="505" y="74" text-anchor="middle" fill="#2D6A2D">BEFORE data page flush</text>

  <!-- COMMIT path -->
  <line x1="505" y1="85" x2="505" y2="115" stroke="#27AE60" stroke-width="1.5" marker-end="url(#aw)"/>
  <rect x="410" y="115" width="190" height="40" fill="#27AE60" rx="6"/>
  <text x="505" y="133" text-anchor="middle" fill="white" font-weight="bold">④ COMMIT Record</text>
  <text x="505" y="148" text-anchor="middle" fill="white">flushed to WAL on disk</text>

  <!-- Data page flush (async, later) -->
  <line x1="85" y1="85" x2="85" y2="175" stroke="#8E44AD" stroke-width="1.5" stroke-dasharray="5"/>
  <rect x="10" y="175" width="200" height="50" fill="#D7BDE2" stroke="#8E44AD" rx="6"/>
  <text x="110" y="196" text-anchor="middle" fill="#4A235A" font-weight="bold">⑤ Data Page Flush</text>
  <text x="110" y="214" text-anchor="middle" fill="#4A235A">(async, lazy — can happen later)</text>

  <text x="310" y="242" fill="#7D3C98" font-size="11" font-style="italic" text-anchor="middle">WAL on disk is the source of truth — data pages can always be reconstructed from it</text>
</svg>
```

### Why WAL Works

1. **Sequential writes are fast:** The log is append-only (sequential I/O), much faster than random data page writes.
2. **Crash safety:** If the system crashes after step ③ but before step ⑤, the committed log records survive on disk and can be replayed to reconstruct the data pages.
3. **Data page writes can be deferred:** The buffer pool can batch and coalesce dirty page writes, improving I/O efficiency.

### WAL Record Types

| Record | When written | Purpose |
|--------|-------------|---------|
| `UPDATE` | When a row is modified | Contains before/after image for undo/redo |
| `INSERT` | When a row is inserted | Contains the inserted row (for redo) |
| `DELETE` | When a row is deleted | Contains the deleted row (for undo) |
| `COMMIT` | When transaction commits | Marks the transaction as durable |
| `ABORT / ROLLBACK` | When transaction rolls back | Marks the transaction as cancelled |
| `CHECKPOINT` | Periodically | Marks a recovery starting point (see Section 5) |
| `COMPENSATION (CLR)` | During rollback | Records that an undo has been performed |

---

## 5. Checkpointing

### The Problem Without Checkpoints

On crash recovery, the database would have to scan the **entire** WAL from the beginning of time to determine which transactions to redo and undo. For a database that has been running for years, this is impractical.

### What a Checkpoint Does

A **checkpoint** is a marker in the WAL that says: "All data pages for committed transactions up to this point have been flushed to disk." On recovery, the database only needs to replay the WAL from the **last checkpoint** forward — dramatically reducing recovery time.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="200" viewBox="0 0 620 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs><marker id="acp" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- WAL timeline -->
  <line x1="40" y1="80" x2="590" y2="80" stroke="#555" stroke-width="2" marker-end="url(#acp)"/>
  <text x="590" y="100" text-anchor="end" fill="#555">WAL →</text>

  <!-- Old records (before checkpoint) -->
  <rect x="50"  y="60" width="50" height="40" fill="#FADBD8" stroke="#C0392B" rx="3"/>
  <rect x="110" y="60" width="50" height="40" fill="#FADBD8" stroke="#C0392B" rx="3"/>
  <rect x="170" y="60" width="50" height="40" fill="#FADBD8" stroke="#C0392B" rx="3"/>
  <text x="135" y="130" text-anchor="middle" fill="#C0392B" font-size="10">Already on disk — skip during recovery</text>

  <!-- Checkpoint marker -->
  <line x1="250" y1="30" x2="250" y2="110" stroke="#E74C3C" stroke-width="3"/>
  <text x="250" y="22" text-anchor="middle" fill="#E74C3C" font-weight="bold" font-size="12">CHECKPOINT</text>

  <!-- Records after checkpoint -->
  <rect x="280" y="60" width="50" height="40" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <rect x="340" y="60" width="50" height="40" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <rect x="400" y="60" width="50" height="40" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <rect x="460" y="60" width="50" height="40" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <text x="390" y="130" text-anchor="middle" fill="#1E8449" font-size="10">Need to redo/undo during recovery</text>

  <!-- CRASH -->
  <line x1="530" y1="40" x2="530" y2="110" stroke="#E74C3C" stroke-width="2" stroke-dasharray="6"/>
  <text x="530" y="32" text-anchor="middle" fill="#E74C3C" font-weight="bold">⚡ CRASH</text>

  <text x="310" y="170" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">Recovery starts from the last checkpoint — not from the beginning of the WAL</text>
</svg>
```

### Checkpoint Process

1. **Write a BEGIN_CHECKPOINT record** to the WAL.
2. **Flush all dirty pages** from the buffer pool to disk (for committed transactions).
3. **Write an END_CHECKPOINT record** to the WAL (includes the list of active transactions at checkpoint time).
4. **Update the master record** (a fixed location on disk) to point to this checkpoint.

### Checkpoint Types

| Type | Behaviour | Impact |
|------|-----------|--------|
| **Sharp (quiescent) checkpoint** | Pauses all transactions; flushes all dirty pages | ❌ Blocks all activity |
| **Fuzzy checkpoint** | Flushes dirty pages in the background; transactions continue | ✅ No blocking — used in practice |
| **Incremental checkpoint** | Only flushes pages dirty since the last checkpoint | ✅ Faster, less I/O |

```sql
-- PostgreSQL: manual checkpoint
CHECKPOINT;

-- PostgreSQL: configure checkpoint frequency
SHOW checkpoint_timeout;        -- default: 5 minutes
SHOW max_wal_size;              -- WAL size that triggers a checkpoint

-- MySQL InnoDB: configure checkpoint
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_max_dirty_pages_pct';
```

---

## 6. The ARIES Recovery Algorithm

**ARIES** (Algorithm for Recovery and Isolation Exploiting Semantics) is the industry-standard recovery algorithm used (in spirit or directly) by PostgreSQL, MySQL InnoDB, SQL Server, and DB2.

### Three Phases

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="660" height="260" viewBox="0 0 660 260">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.h{font-size:12px;font-weight:bold}</style>
  <defs><marker id="aar" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- WAL timeline at top -->
  <line x1="40" y1="30" x2="620" y2="30" stroke="#555" stroke-width="2" marker-end="url(#aar)"/>
  <line x1="120" y1="20" x2="120" y2="40" stroke="#E74C3C" stroke-width="2"/>
  <text x="120" y="15" text-anchor="middle" fill="#E74C3C" font-size="10">Checkpoint</text>
  <line x1="560" y1="20" x2="560" y2="40" stroke="#E74C3C" stroke-width="2"/>
  <text x="560" y="15" text-anchor="middle" fill="#E74C3C" font-size="10">⚡ Crash</text>

  <!-- Phase 1: Analysis -->
  <rect x="10" y="60" width="200" height="170" fill="#DAE8FC" stroke="#2980B9" rx="8"/>
  <text x="110" y="84" text-anchor="middle" class="h" fill="#1A5276">① ANALYSIS</text>
  <line x1="20" y1="92" x2="200" y2="92" stroke="#AED6F1"/>
  <text x="110" y="112" text-anchor="middle" fill="#333">Scan WAL from checkpoint</text>
  <text x="110" y="130" text-anchor="middle" fill="#333">→ forward to crash point</text>
  <text x="110" y="156" text-anchor="middle" fill="#333">Build two lists:</text>
  <text x="110" y="176" text-anchor="middle" fill="#27AE60" font-weight="bold">Winners: committed txns</text>
  <text x="110" y="196" text-anchor="middle" fill="#E74C3C" font-weight="bold">Losers: active (uncommitted)</text>
  <text x="110" y="218" text-anchor="middle" fill="#555" font-size="10">+ Dirty Page Table</text>

  <line x1="210" y1="145" x2="240" y2="145" stroke="#555" stroke-width="1.5" marker-end="url(#aar)"/>

  <!-- Phase 2: Redo -->
  <rect x="240" y="60" width="190" height="170" fill="#D5E8D4" stroke="#82B366" rx="8"/>
  <text x="335" y="84" text-anchor="middle" class="h" fill="#2D6A2D">② REDO</text>
  <line x1="250" y1="92" x2="420" y2="92" stroke="#A9DFBF"/>
  <text x="335" y="112" text-anchor="middle" fill="#333">Scan WAL from checkpoint</text>
  <text x="335" y="130" text-anchor="middle" fill="#333">→ forward to crash point</text>
  <text x="335" y="156" text-anchor="middle" fill="#333">Reapply ALL logged</text>
  <text x="335" y="174" text-anchor="middle" fill="#333">operations (winners</text>
  <text x="335" y="192" text-anchor="middle" fill="#333">AND losers) to restore</text>
  <text x="335" y="210" text-anchor="middle" fill="#333">exact pre-crash state</text>

  <line x1="430" y1="145" x2="460" y2="145" stroke="#555" stroke-width="1.5" marker-end="url(#aar)"/>

  <!-- Phase 3: Undo -->
  <rect x="460" y="60" width="190" height="170" fill="#FDEDEC" stroke="#C0392B" rx="8"/>
  <text x="555" y="84" text-anchor="middle" class="h" fill="#7B241C">③ UNDO</text>
  <line x1="470" y1="92" x2="640" y2="92" stroke="#F5B7B1"/>
  <text x="555" y="112" text-anchor="middle" fill="#333">Scan WAL BACKWARD</text>
  <text x="555" y="130" text-anchor="middle" fill="#333">from crash point</text>
  <text x="555" y="156" text-anchor="middle" fill="#333">Reverse all operations</text>
  <text x="555" y="174" text-anchor="middle" fill="#333">by LOSER transactions</text>
  <text x="555" y="192" text-anchor="middle" fill="#333">(write CLR records to</text>
  <text x="555" y="210" text-anchor="middle" fill="#333">prevent re-undo on crash)</text>
</svg>
```

### Phase Details

**① Analysis Phase**
- Scan WAL from the last checkpoint to the end (crash point).
- Reconstruct the **Transaction Table** (which transactions were active at crash).
- Reconstruct the **Dirty Page Table** (which pages had unflushed changes).
- Classify transactions as **winners** (committed before crash) or **losers** (active at crash).

**② Redo Phase (repeating history)**
- Scan WAL forward from checkpoint.
- Reapply every logged operation — both committed and uncommitted — to bring the database to the exact state it was in just before the crash.
- Why redo losers too? Because their changes may have been flushed to disk before the crash; we need the state to be consistent before undoing them.

**③ Undo Phase**
- Scan WAL backward.
- Reverse every operation by loser (uncommitted) transactions.
- Write **Compensation Log Records (CLR)** for each undo action. CLRs ensure that if the system crashes during recovery, the undo work is not repeated.

### ARIES Key Principles

| Principle | Meaning |
|-----------|---------|
| **Write-Ahead Logging** | Log before data page |
| **Repeating history during redo** | Redo ALL operations (committed + uncommitted) to restore exact crash state |
| **Logging changes during undo** | Write CLRs so undo is idempotent across crashes during recovery |
| **Steal / No-force** | Dirty pages CAN be flushed before commit (steal); data pages DON'T have to be flushed at commit (no-force) |

### Steal vs No-Steal, Force vs No-Force

| Policy | Description | Used by |
|--------|-------------|---------|
| **Steal** | Dirty (uncommitted) pages CAN be flushed to disk | ARIES (requires undo log) |
| **No-Steal** | Dirty pages are NEVER flushed until commit | Simple but restricts buffer pool |
| **Force** | All dirty pages MUST be flushed at commit | Simplifies recovery (no redo needed) |
| **No-Force** | Dirty pages flushed lazily after commit | ARIES (requires redo log) |

ARIES uses **Steal + No-Force** — the most flexible policy, but requires both undo and redo during recovery.

---

## 7. Shadow Paging

### Concept

An older, simpler alternative to WAL-based recovery. The database maintains two copies of each page:
- **Current page:** receives modifications.
- **Shadow page:** the last committed version.

On commit, the page table pointer is atomically swapped from shadow to current. On crash, the shadow pages represent the last consistent state.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="480" height="200" viewBox="0 0 480 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs><marker id="asp" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- Page table -->
  <rect x="10" y="40" width="120" height="100" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="70" y="64" text-anchor="middle" fill="#1A5276" font-weight="bold">Page Table</text>
  <text x="70" y="84" text-anchor="middle" fill="#1A5276">Page 1 →</text>
  <text x="70" y="104" text-anchor="middle" fill="#1A5276">Page 2 →</text>
  <text x="70" y="124" text-anchor="middle" fill="#1A5276">Page 3 →</text>

  <!-- Current pages -->
  <rect x="180" y="20" width="100" height="40" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="230" y="45" text-anchor="middle" fill="#784212">Current P1</text>
  <rect x="180" y="70" width="100" height="40" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="230" y="95" text-anchor="middle" fill="#784212">Current P2</text>
  <rect x="180" y="120" width="100" height="40" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="230" y="145" text-anchor="middle" fill="#784212">Current P3</text>

  <line x1="130" y1="84"  x2="178" y2="40"  stroke="#555" stroke-width="1" marker-end="url(#asp)"/>
  <line x1="130" y1="104" x2="178" y2="90"  stroke="#555" stroke-width="1" marker-end="url(#asp)"/>
  <line x1="130" y1="124" x2="178" y2="140" stroke="#555" stroke-width="1" marker-end="url(#asp)"/>

  <!-- Shadow pages -->
  <rect x="340" y="20" width="100" height="40" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="390" y="45" text-anchor="middle" fill="#2D6A2D">Shadow P1</text>
  <rect x="340" y="70" width="100" height="40" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="390" y="95" text-anchor="middle" fill="#2D6A2D">Shadow P2</text>
  <rect x="340" y="120" width="100" height="40" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="390" y="145" text-anchor="middle" fill="#2D6A2D">Shadow P3</text>

  <text x="390" y="185" text-anchor="middle" fill="#27AE60" font-size="10" font-weight="bold">Last committed version</text>
  <text x="230" y="185" text-anchor="middle" fill="#E67E22" font-size="10" font-weight="bold">In-progress changes</text>
</svg>
```

### Shadow Paging vs WAL

| Aspect | Shadow Paging | WAL (ARIES) |
|--------|--------------|-------------|
| Complexity | Simple | Complex |
| Storage | 2× (shadow copy) | Log files + data |
| Random I/O | High (page writes scattered) | Low (sequential log writes) |
| Large transactions | Expensive (many shadow pages) | Efficient (just log records) |
| Production use | Rarely (SQLite in journal mode) | All major RDBMS |

Shadow paging is largely **historical / academic**. All production databases use WAL.

---

## 8. Recovery in PostgreSQL

### PostgreSQL WAL Architecture

PostgreSQL stores WAL files as 16 MB segments in the `pg_wal/` directory.

```sql
-- View current WAL position
SELECT pg_current_wal_lsn();

-- View WAL settings
SHOW wal_level;            -- minimal, replica, logical
SHOW max_wal_size;         -- triggers checkpoint when exceeded
SHOW checkpoint_timeout;   -- max time between checkpoints
SHOW wal_buffers;          -- WAL buffer size in shared memory
```

### Crash Recovery Process

1. PostgreSQL starts and detects unclean shutdown (missing `postmaster.pid` or `pg_control` flag).
2. **Reads the last checkpoint location** from `pg_control`.
3. **Replays WAL from checkpoint** forward — redo all records.
4. **Rolls back uncommitted transactions** (undo using the visibility rules of MVCC — dead tuple versions are simply invisible).
5. Database is consistent and ready for connections.

### PostgreSQL's "Redo-Only" Recovery

PostgreSQL's MVCC means **explicit undo is not needed** at the WAL level. Uncommitted changes are already invisible to new transactions via `xmin`/`xmax` visibility. WAL recovery only needs to redo — the dead versions are cleaned up by VACUUM later.

```sql
-- Monitor recovery progress (during startup after crash)
SELECT pg_is_in_recovery();    -- TRUE if still recovering
SELECT pg_last_wal_replay_lsn();   -- how far recovery has progressed
```

---

## 9. Recovery in MySQL InnoDB

### InnoDB's Redo + Undo Architecture

InnoDB maintains two separate log systems:

| Log | Location | Purpose |
|-----|----------|---------|
| **Redo log** | `ib_logfile0`, `ib_logfile1` (circular) | Replay committed changes after crash |
| **Undo log** | `ibdata1` or undo tablespace | Reverse uncommitted changes + MVCC snapshots |

### InnoDB Crash Recovery

1. **Redo phase:** Read redo logs from the last checkpoint; replay all changes (committed + uncommitted).
2. **Undo phase:** Read undo logs; roll back any transactions that were active at crash time.
3. **Purge:** Background thread later removes old undo log versions no longer needed for MVCC.

```sql
-- Monitor InnoDB recovery
SHOW ENGINE INNODB STATUS\G
-- Look for "Log sequence number" and "Last checkpoint at"

-- Configuration
SHOW VARIABLES LIKE 'innodb_log_file_size';      -- redo log file size
SHOW VARIABLES LIKE 'innodb_log_files_in_group';  -- number of redo log files
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
-- 1 = flush to disk on every commit (safest, default)
-- 2 = flush to OS cache on commit (faster, slight durability risk)
-- 0 = flush once per second (fastest, most risk)
```

### The Doublewrite Buffer

InnoDB pages are 16 KB but disk sectors are typically 512 bytes. A crash mid-write can corrupt a page (partial or "torn" write). InnoDB's **doublewrite buffer** writes pages to a sequential area first, then to their final location — ensuring a clean copy always exists.

---

## 10. Backup & Point-in-Time Recovery (PITR)

### Point-in-Time Recovery

PITR lets you restore the database to **any specific moment** — not just the last backup. It works by restoring a base backup and then replaying WAL up to the desired timestamp.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="160" viewBox="0 0 620 160">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>
  <defs><marker id="api" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- Timeline -->
  <line x1="40" y1="80" x2="590" y2="80" stroke="#555" stroke-width="2" marker-end="url(#api)"/>

  <!-- Base backup -->
  <line x1="100" y1="50" x2="100" y2="110" stroke="#2980B9" stroke-width="3"/>
  <text x="100" y="40" text-anchor="middle" fill="#2980B9" font-weight="bold">Base Backup</text>
  <text x="100" y="126" text-anchor="middle" fill="#555" font-size="10">Mon 2:00 AM</text>

  <!-- WAL segments -->
  <rect x="120" y="68" width="80" height="24" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <text x="160" y="84" text-anchor="middle" fill="#145A32" font-size="10">WAL 001</text>
  <rect x="210" y="68" width="80" height="24" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <text x="250" y="84" text-anchor="middle" fill="#145A32" font-size="10">WAL 002</text>
  <rect x="300" y="68" width="80" height="24" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <text x="340" y="84" text-anchor="middle" fill="#145A32" font-size="10">WAL 003</text>
  <rect x="390" y="68" width="80" height="24" fill="#A9DFBF" stroke="#1E8449" rx="3"/>
  <text x="430" y="84" text-anchor="middle" fill="#145A32" font-size="10">WAL 004</text>

  <!-- Target point -->
  <line x1="450" y1="50" x2="450" y2="110" stroke="#E74C3C" stroke-width="2"/>
  <text x="450" y="40" text-anchor="middle" fill="#E74C3C" font-weight="bold">Target: Wed 3:15 PM</text>

  <!-- Disaster -->
  <line x1="530" y1="50" x2="530" y2="110" stroke="#C0392B" stroke-width="2" stroke-dasharray="6"/>
  <text x="530" y="40" text-anchor="middle" fill="#C0392B" font-weight="bold">⚡ Disaster Thu</text>

  <text x="310" y="150" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">Restore base backup + replay WAL 001–004 stopping at Wed 3:15 PM</text>
</svg>
```

### PostgreSQL PITR

```sql
-- Step 1: Take a base backup
pg_basebackup -D /backup/base -Ft -z -Xs -P

-- Step 2: Archive WAL segments (configured in postgresql.conf)
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'

-- Step 3: Restore (after disaster)
-- Copy base backup to data directory
-- Create recovery.signal (PostgreSQL 12+)
-- Set target in postgresql.conf:
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2024-06-19 15:15:00'
recovery_target_action = 'promote'

-- Start PostgreSQL — it replays WAL up to the target time
```

### MySQL PITR

```bash
# Step 1: Full backup with mysqldump or xtrabackup
mysqldump --all-databases --single-transaction --flush-logs > backup.sql

# Step 2: Enable binary logging (already default in MySQL 8.0)
# log_bin = ON in my.cnf

# Step 3: Restore and replay binary logs
mysql < backup.sql
mysqlbinlog --stop-datetime="2024-06-19 15:15:00" binlog.000042 binlog.000043 | mysql
```

### Backup Strategies

| Strategy | Method | RPO | RTO |
|----------|--------|-----|-----|
| **Full backup + PITR** | pg_basebackup + WAL archiving | Seconds (WAL lag) | Minutes–hours |
| **Continuous archiving** | WAL streaming to standby | ~0 (synchronous) | Seconds (failover) |
| **Logical backup** | pg_dump / mysqldump | Hours (dump frequency) | Hours (restore time) |
| **Snapshot** | Filesystem / LVM / cloud snapshot | Depends on frequency | Minutes |

> **RPO** = Recovery Point Objective (how much data can you lose)  
> **RTO** = Recovery Time Objective (how fast can you recover)

---

## 11. Worked Examples

### Example 1 — WAL Recovery After Crash (Conceptual Walkthrough)

```
Timeline:
  LSN 100: T1 BEGIN
  LSN 101: T1 UPDATE accounts SET balance=500 WHERE id=1 (was 1000)
  LSN 102: T2 BEGIN
  LSN 103: T2 UPDATE accounts SET balance=800 WHERE id=2 (was 1000)
  LSN 104: T1 COMMIT          ← T1 committed, WAL flushed
  LSN 105: T2 UPDATE accounts SET balance=600 WHERE id=3 (was 1000)
  ⚡ CRASH — T2 never committed, data pages for T1 may not be flushed

Recovery:
  ANALYSIS: T1 = winner (committed), T2 = loser (active at crash)
  REDO:     Replay LSN 101–105 forward → restore exact crash state
  UNDO:     Reverse T2's changes (LSN 105, 103) → id=2 back to 1000, id=3 back to 1000
  Result:   T1's changes persist (id=1 = 500), T2's changes gone
```

### Example 2 — Checkpoint Reduces Recovery Time

```
Without checkpoint:
  WAL has 10 million records over 3 days
  Recovery scans all 10M records → 20 minutes

With checkpoint every 5 minutes:
  Last checkpoint was 5 minutes before crash
  Recovery scans only ~50,000 records → 3 seconds
```

### Example 3 — PostgreSQL Point-in-Time Recovery

```bash
# Scenario: accidental DROP TABLE at 15:00, need to restore to 14:59

# 1. Stop PostgreSQL
pg_ctl stop -D /var/lib/postgresql/data

# 2. Restore base backup
rm -rf /var/lib/postgresql/data/*
tar xzf /backup/base/base.tar.gz -C /var/lib/postgresql/data/

# 3. Configure recovery target
cat >> /var/lib/postgresql/data/postgresql.conf << 'EOF'
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2024-06-19 14:59:00'
recovery_target_action = 'promote'
EOF

# 4. Create recovery signal
touch /var/lib/postgresql/data/recovery.signal

# 5. Start PostgreSQL — replays WAL up to 14:59
pg_ctl start -D /var/lib/postgresql/data

# 6. Verify: the dropped table should exist with data as of 14:59
psql -c "SELECT count(*) FROM the_recovered_table;"
```

---

## 12. Common Interview Questions

**Q1. What is Write-Ahead Logging (WAL)?**  
WAL requires that every data modification be written to a sequential log on durable storage BEFORE the modified data page is written to disk. This ensures that after a crash, all committed transactions can be redone from the log, and all uncommitted transactions can be undone.

**Q2. What is a checkpoint and why is it important?**  
A checkpoint is a WAL marker indicating that all dirty pages for committed transactions up to that point have been flushed to disk. It limits how far back recovery must scan — without checkpoints, the entire WAL from the beginning of time would need scanning.

**Q3. Explain the three phases of ARIES recovery.**  
1. **Analysis:** Scan WAL from the last checkpoint forward to identify winners (committed) and losers (uncommitted), and build a dirty page table.  
2. **Redo:** Replay all logged operations (both winners and losers) to restore the exact pre-crash state.  
3. **Undo:** Reverse all operations by loser transactions, writing CLRs to prevent re-undoing on subsequent crashes.

**Q4. Why does ARIES redo uncommitted transactions in the redo phase?**  
Because uncommitted ("loser") transactions may have had their dirty pages flushed to disk before the crash (the "steal" policy). The data pages on disk contain a mix of committed and uncommitted changes. Redoing everything restores the exact pre-crash state, after which undo can cleanly reverse the losers.

**Q5. What is a Compensation Log Record (CLR)?**  
A log record written during the undo phase to record that a specific undo action has been performed. If the system crashes during recovery, CLRs prevent the same undo from being applied twice (idempotent recovery).

**Q6. What is the difference between undo and redo logs?**  
An undo log stores the old (before) value — used to reverse uncommitted changes. A redo log stores the new (after) value — used to reapply committed changes that weren't flushed to disk. ARIES-style systems store both in the same record (undo/redo log).

**Q7. What does steal/no-force mean in the context of buffer management?**  
**Steal:** The buffer pool can flush uncommitted (dirty) pages to disk (requires undo on crash). **No-force:** The buffer pool does NOT have to flush dirty pages at commit time (requires redo on crash). ARIES uses steal + no-force for maximum flexibility and performance.

**Q8. What is Point-in-Time Recovery (PITR)?**  
PITR restores a database to any specific moment by restoring a base backup and replaying WAL/binary logs up to the desired timestamp. Useful for recovering from accidental data loss (e.g., a dropped table).

**Q9. How does PostgreSQL handle crash recovery differently from InnoDB?**  
PostgreSQL's MVCC means recovery is "redo-only" at the WAL level — uncommitted row versions are invisible via xmin/xmax visibility, so explicit undo isn't needed in the WAL. InnoDB uses both redo logs and separate undo logs, requiring a distinct undo phase during recovery.

**Q10. What is shadow paging and why isn't it used in production?**  
Shadow paging keeps two copies of each page (current and shadow). On commit, pointers are atomically swapped. It's simple but causes excessive random I/O (scattered page writes), doesn't support efficient large transactions, and offers no incremental recovery. WAL-based recovery is superior in all practical dimensions.

**Q11. What is `innodb_flush_log_at_trx_commit` and what are the trade-offs?**  
This MySQL setting controls when the redo log is flushed:  
- `1` (default): flush to disk on every commit — fully durable, slowest.  
- `2`: flush to OS cache on commit — a crash loses at most ~1 second of transactions.  
- `0`: flush once per second regardless of commits — fastest, up to 1 second of data loss on crash.

**Q12. How does WAL archiving support high availability?**  
WAL segments can be continuously streamed to a standby server (streaming replication). The standby replays WAL in real time, maintaining an almost-identical copy. On primary failure, the standby is promoted — RPO near zero, RTO in seconds.

---

## 13. Key Takeaways

| Concept | One-line rule |
|---------|--------------|
| **WAL** | Log before data page — sequential writes are fast and crash-safe |
| **Undo log** | Old value — reverses uncommitted changes |
| **Redo log** | New value — reapplies committed changes not yet on disk |
| **Checkpoint** | Limits recovery scan — "everything before here is on disk" |
| **ARIES** | Analysis → Redo (all) → Undo (losers); uses CLRs for idempotent recovery |
| **Steal + No-Force** | Max flexibility — dirty pages can flush early; commit doesn't force flush |
| **CLR** | Compensation log record — prevents double-undo during recovery |
| **Shadow paging** | Academic curiosity — WAL is superior in all practical ways |
| **PostgreSQL recovery** | Redo-only at WAL level; MVCC visibility handles "undo" naturally |
| **InnoDB recovery** | Redo log + separate undo log; doublewrite buffer prevents torn pages |
| **PITR** | Base backup + WAL replay to any timestamp — essential disaster recovery |
| **Checkpoint frequency** | Shorter = faster recovery; longer = less I/O overhead during normal ops |

### Recovery Decision Guide

```
System crash (power loss, OS crash)?
  → Automatic WAL recovery on restart (seconds to minutes)

Media failure (disk corruption)?
  → Restore base backup + replay WAL (PITR)

Accidental DROP TABLE / DELETE?
  → PITR to just before the mistake

Need near-zero downtime?
  → Streaming replication + automatic failover

How often to checkpoint?
  → Balance recovery time (shorter = faster) vs normal I/O (longer = less overhead)
  → Default: every 5 min or when max_wal_size is reached
```

---

*Phase 13 complete. Phase 14 covers Stored Procedures, Functions, Triggers & Cursors (`14_programmability.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="13_recovery.md" download="13_recovery.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 13_recovery.md
  </button>
</a>
