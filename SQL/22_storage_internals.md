# Phase 22 — File Organization & Storage Internals

> Beneath every SQL query lies a physical reality: bytes on disk, pages in memory, and algorithms deciding which blocks to keep cached. Understanding **storage internals** transforms you from someone who writes queries to someone who understands *why* a query is fast or slow at the hardware level.

---

## Table of Contents

1. [The Storage Hierarchy](#1-storage-hierarchy)
2. [Disk I/O Fundamentals](#2-disk-io)
3. [Pages, Extents & Tablespaces](#3-pages-extents-tablespaces)
4. [Slotted Page Format](#4-slotted-page)
5. [Heap Files (Unordered)](#5-heap-files)
6. [Sorted Files (Ordered)](#6-sorted-files)
7. [Hashed Files](#7-hashed-files)
8. [File Organization Comparison](#8-file-org-comparison)
9. [Buffer Pool Management](#9-buffer-pool)
10. [LRU & Clock Replacement Policies](#10-replacement-policies)
11. [Write-Ahead Logging & Dirty Pages](#11-wal-dirty-pages)
12. [TOAST & Large Objects](#12-toast)
13. [Worked Examples](#13-worked-examples)
14. [Common Interview Questions](#14-interview-questions)
15. [Key Takeaways](#15-key-takeaways)
16. [Download](#16-download)

---

## 1. The Storage Hierarchy

Every database operates across a memory/storage hierarchy where each level trades **capacity** for **speed**.

### Access Latencies

| Level | Typical Size | Latency | DB Analogy |
|---|---|---|---|
| **CPU L1 cache** | 64 KB | ~1 ns | — |
| **CPU L2 cache** | 256 KB | ~4 ns | — |
| **CPU L3 cache** | 8–32 MB | ~12 ns | — |
| **RAM (DRAM)** | 64–512 GB | ~100 ns | Buffer pool / shared buffers |
| **NVMe SSD** | 1–16 TB | ~20 µs (20,000 ns) | Data files, WAL |
| **SATA SSD** | 1–8 TB | ~100 µs | Data files |
| **HDD (spinning)** | 1–20 TB | ~5–10 ms (5,000,000 ns) | Cold storage, archives |
| **Network storage** | Unlimited | ~1–10 ms | Cloud EBS, S3 |

### Real-World Analogy

Think of the storage hierarchy as a **desk, filing cabinet, and warehouse**:
- **CPU cache / RAM** = your desk — tiny, but everything on it is instantly accessible
- **SSD** = a filing cabinet in the same room — fast to reach, holds a lot more
- **HDD / network** = a warehouse across town — huge capacity, but every trip takes time

The database's job is to keep the **working set** (the data you're actively querying) on the desk (RAM) and avoid trips to the warehouse (disk) as much as possible.

<svg viewBox="0 0 780 340" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Storage Hierarchy — Speed vs Capacity</text>

  <!-- Pyramid -->
  <polygon points="390,50 180,300 600,300" fill="none" stroke="#666" stroke-width="2"/>

  <!-- L1/L2/L3 -->
  <line x1="310" y1="100" x2="470" y2="100" stroke="#ddd" stroke-width="1"/>
  <rect x="320" y="60" width="140" height="38" rx="4" fill="#c62828" opacity="0.85"/>
  <text x="390" y="84" text-anchor="middle" font-size="10" font-weight="bold" fill="white">CPU Cache (L1–L3)</text>
  <text x="620" y="80" font-size="9" fill="#c62828">~1–12 ns | KB–MB</text>
  <text x="620" y="95" font-size="9" fill="#888">Fastest, smallest</text>

  <!-- RAM -->
  <line x1="270" y1="150" x2="510" y2="150" stroke="#ddd" stroke-width="1"/>
  <rect x="280" y="108" width="220" height="40" rx="4" fill="#e65100" opacity="0.85"/>
  <text x="390" y="130" text-anchor="middle" font-size="11" font-weight="bold" fill="white">RAM (Buffer Pool)</text>
  <text x="390" y="143" text-anchor="middle" font-size="9" fill="#ffe0b2">shared_buffers / innodb_buffer_pool</text>
  <text x="620" y="130" font-size="9" fill="#e65100">~100 ns | 64–512 GB</text>

  <!-- SSD -->
  <line x1="235" y1="205" x2="545" y2="205" stroke="#ddd" stroke-width="1"/>
  <rect x="245" y="160" width="290" height="42" rx="4" fill="#f57f17" opacity="0.85"/>
  <text x="390" y="182" text-anchor="middle" font-size="11" font-weight="bold" fill="white">SSD (NVMe / SATA)</text>
  <text x="390" y="196" text-anchor="middle" font-size="9" fill="#fff9c4">Data files, WAL, indexes</text>
  <text x="620" y="180" font-size="9" fill="#f57f17">~20–100 µs | 1–16 TB</text>

  <!-- HDD -->
  <line x1="205" y1="255" x2="575" y2="255" stroke="#ddd" stroke-width="1"/>
  <rect x="215" y="212" width="350" height="40" rx="4" fill="#2e7d32" opacity="0.85"/>
  <text x="390" y="234" text-anchor="middle" font-size="11" font-weight="bold" fill="white">HDD / Network Storage</text>
  <text x="390" y="248" text-anchor="middle" font-size="9" fill="#c8e6c9">Archives, backups, cold data</text>
  <text x="620" y="234" font-size="9" fill="#2e7d32">~5–10 ms | TB–PB</text>

  <!-- Bottom label -->
  <rect x="185" y="262" width="410" height="35" rx="4" fill="#1565c0" opacity="0.85"/>
  <text x="390" y="284" text-anchor="middle" font-size="11" font-weight="bold" fill="white">Cloud Object Storage (S3, GCS, Azure Blob)</text>
  <text x="620" y="280" font-size="9" fill="#1565c0">~50–200 ms | Unlimited</text>

  <!-- Arrows -->
  <text x="130" y="180" font-size="10" fill="#666" transform="rotate(-90,130,180)">← Faster        Larger →</text>
</svg>

---

## 2. Disk I/O Fundamentals

### Why the Database Thinks in Pages, Not Rows

Disk I/O doesn't operate on individual rows — it operates on **fixed-size blocks** (pages). Reading 1 byte from disk costs almost the same as reading 8 KB, because the minimum transfer unit is a page. This is why databases:

1. **Read and write in pages** (typically 4 KB or 8 KB)
2. **Cache pages in RAM** (buffer pool) to avoid repeated disk reads
3. **Organise data to minimise pages read** per query (indexes, clustering)

### Random I/O vs Sequential I/O

| Type | What Happens | Speed (HDD) | Speed (SSD) |
|---|---|---|---|
| **Sequential read** | Read pages in order (contiguous) | ~200 MB/s | ~3,000 MB/s (NVMe) |
| **Random read** | Read pages scattered across disk | ~1 MB/s (200 IOPS × 4KB) | ~500 MB/s (100K IOPS × 4KB) |
| **Sequential write** | Append pages in order | ~150 MB/s | ~2,000 MB/s |
| **Random write** | Write to scattered locations | ~1 MB/s | ~400 MB/s |

> **Key Insight**: On HDD, sequential I/O is **200x** faster than random I/O. On SSD, the gap narrows to ~6x, but sequential is still faster. This is why full table scans (sequential) can beat index lookups (random) when returning a large fraction of the table.

### I/O Cost Model

Database query optimisers estimate cost in terms of page I/O:

```
Cost ≈ (pages_read_sequentially × seq_page_cost)
      + (pages_read_randomly × random_page_cost)
      + (CPU_tuples_processed × cpu_tuple_cost)
```

PostgreSQL defaults:
- `seq_page_cost = 1.0`
- `random_page_cost = 4.0` (random is 4x more expensive)
- `cpu_tuple_cost = 0.01`

---

## 3. Pages, Extents & Tablespaces

### Pages (Blocks)

A **page** (or block) is the fundamental unit of I/O and storage. Everything — table rows, index entries, WAL records — is stored in fixed-size pages.

| RDBMS | Default Page Size | Configurable? |
|---|---|---|
| **PostgreSQL** | 8 KB | Compile-time only (`--with-blocksize`) |
| **MySQL (InnoDB)** | 16 KB | `innodb_page_size` (set at init) |
| **SQL Server** | 8 KB | Not configurable |
| **Oracle** | 8 KB | `DB_BLOCK_SIZE` (set at DB creation) |

### Extents

An **extent** is a group of contiguous pages allocated together. Allocating in bulk (extents) rather than one page at a time reduces fragmentation and improves sequential read performance.

| RDBMS | Extent Size | Details |
|---|---|---|
| **PostgreSQL** | N/A | Allocates pages individually (no extent concept) |
| **MySQL (InnoDB)** | 1 MB (64 pages × 16 KB) | Small tables start with 16 KB extents |
| **SQL Server** | 64 KB (8 pages × 8 KB) | Uniform (single object) or mixed (shared) |
| **Oracle** | Configurable | Typically 64 KB – 8 MB, auto-managed |

### Tablespaces

A **tablespace** maps logical storage to physical directories/devices. You can place performance-critical tables on fast SSDs and archives on cheaper HDDs.

```sql
-- PostgreSQL: Create tablespace on fast SSD
CREATE TABLESPACE fast_ssd LOCATION '/mnt/nvme/pg_data';

-- Place a table on that tablespace
CREATE TABLE hot_orders (
    order_id BIGSERIAL PRIMARY KEY,
    total    NUMERIC(12,2)
) TABLESPACE fast_ssd;

-- Move an existing table
ALTER TABLE cold_archives SET TABLESPACE slow_hdd;

-- Place an index on a separate tablespace
CREATE INDEX idx_orders_date ON hot_orders (order_date)
    TABLESPACE fast_ssd;
```

<svg viewBox="0 0 780 300" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Pages, Extents & Tablespaces</text>

  <!-- Tablespace -->
  <rect x="30" y="48" width="720" height="240" rx="14" fill="#f5f5f5" stroke="#666" stroke-width="2"/>
  <text x="50" y="72" font-size="12" font-weight="bold" fill="#333">Tablespace: fast_ssd (/mnt/nvme/pg_data)</text>

  <!-- Data file 1 -->
  <rect x="55" y="88" width="330" height="180" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="70" y="108" font-size="10" font-weight="bold" fill="#1565c0">Data File: orders (base/16384/16385)</text>

  <!-- Extent 1 -->
  <rect x="70" y="118" width="290" height="55" rx="4" fill="white" stroke="#999" stroke-width="1"/>
  <text x="80" y="133" font-size="8" fill="#888">Extent 1 (8 pages)</text>
  <rect x="80" y="138" width="30" height="28" rx="2" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="95" y="156" text-anchor="middle" font-size="7" fill="#333">P0</text>
  <rect x="115" y="138" width="30" height="28" rx="2" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="130" y="156" text-anchor="middle" font-size="7" fill="#333">P1</text>
  <rect x="150" y="138" width="30" height="28" rx="2" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="165" y="156" text-anchor="middle" font-size="7" fill="#333">P2</text>
  <rect x="185" y="138" width="30" height="28" rx="2" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="200" y="156" text-anchor="middle" font-size="7" fill="#333">P3</text>
  <rect x="220" y="138" width="30" height="28" rx="2" fill="#90caf9" stroke="#1565c0" stroke-width="1"/>
  <text x="235" y="156" text-anchor="middle" font-size="7" fill="#333">P4</text>
  <rect x="255" y="138" width="30" height="28" rx="2" fill="#90caf9" stroke="#1565c0" stroke-width="1"/>
  <text x="270" y="156" text-anchor="middle" font-size="7" fill="#333">P5</text>
  <rect x="290" y="138" width="30" height="28" rx="2" fill="#90caf9" stroke="#1565c0" stroke-width="1"/>
  <text x="305" y="156" text-anchor="middle" font-size="7" fill="#333">P6</text>
  <rect x="325" y="138" width="30" height="28" rx="2" fill="#90caf9" stroke="#1565c0" stroke-width="1"/>
  <text x="340" y="156" text-anchor="middle" font-size="7" fill="#333">P7</text>

  <!-- Extent 2 -->
  <rect x="70" y="180" width="290" height="55" rx="4" fill="white" stroke="#999" stroke-width="1"/>
  <text x="80" y="195" font-size="8" fill="#888">Extent 2 (8 pages)</text>
  <rect x="80" y="200" width="30" height="28" rx="2" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1"/>
  <text x="95" y="218" text-anchor="middle" font-size="7" fill="#333">P8</text>
  <rect x="115" y="200" width="30" height="28" rx="2" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1"/>
  <text x="130" y="218" text-anchor="middle" font-size="7" fill="#333">P9</text>
  <rect x="150" y="200" width="30" height="28" rx="2" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1"/>
  <text x="165" y="218" text-anchor="middle" font-size="7" fill="#333">P10</text>
  <rect x="185" y="200" width="30" height="28" rx="2" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="200" y="218" text-anchor="middle" font-size="7" fill="#aaa">free</text>
  <rect x="220" y="200" width="30" height="28" rx="2" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="235" y="218" text-anchor="middle" font-size="7" fill="#aaa">free</text>
  <rect x="255" y="200" width="30" height="28" rx="2" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="270" y="218" text-anchor="middle" font-size="7" fill="#aaa">free</text>
  <rect x="290" y="200" width="30" height="28" rx="2" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="305" y="218" text-anchor="middle" font-size="7" fill="#aaa">free</text>
  <rect x="325" y="200" width="30" height="28" rx="2" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="340" y="218" text-anchor="middle" font-size="7" fill="#aaa">free</text>

  <!-- Page detail callout -->
  <rect x="420" y="88" width="310" height="180" rx="8" fill="#fff9c4" stroke="#f57f17" stroke-width="1.5"/>
  <text x="435" y="108" font-size="10" font-weight="bold" fill="#f57f17">Inside a Page (8 KB)</text>
  <rect x="435" y="118" width="280" height="22" rx="3" fill="#fff176"/>
  <text x="575" y="133" text-anchor="middle" font-size="9" fill="#333">Page Header (24 bytes)</text>
  <rect x="435" y="143" width="280" height="18" rx="3" fill="#ffe082"/>
  <text x="575" y="156" text-anchor="middle" font-size="9" fill="#333">Item pointers (line pointer array)</text>
  <rect x="435" y="164" width="280" height="60" rx="3" fill="#ffecb3"/>
  <text x="575" y="185" text-anchor="middle" font-size="9" fill="#555">Free space</text>
  <text x="575" y="200" text-anchor="middle" font-size="8" fill="#888">(grows toward each other)</text>
  <rect x="435" y="227" width="280" height="30" rx="3" fill="#ffe082"/>
  <text x="575" y="242" text-anchor="middle" font-size="9" fill="#333">Tuple data (rows, packed from end)</text>
  <text x="575" y="254" text-anchor="middle" font-size="8" fill="#888">↑ tuples grow backward</text>

  <!-- Arrow between line ptrs and tuples -->
  <text x="440" y="220" font-size="8" fill="#f57f17">Item ptrs →     ← Tuples</text>
</svg>

---

## 4. Slotted Page Format

The **slotted page** is the standard page layout used by virtually all modern databases (PostgreSQL, MySQL/InnoDB, SQL Server, Oracle). It allows variable-length rows and efficient deletion/compaction.

### Structure

<svg viewBox="0 0 760 420" xmlns="http://www.w3.org/2000/svg" style="max-width:760px; font-family:Arial,sans-serif;">
  <text x="380" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Slotted Page Layout (8 KB Page)</text>

  <!-- Page boundary -->
  <rect x="100" y="45" width="400" height="350" rx="4" fill="white" stroke="#333" stroke-width="2"/>

  <!-- Header -->
  <rect x="100" y="45" width="400" height="35" fill="#c62828" rx="4"/>
  <text x="300" y="67" text-anchor="middle" font-size="11" font-weight="bold" fill="white">Page Header (24 bytes)</text>

  <!-- Line pointers -->
  <rect x="100" y="80" width="400" height="55" fill="#e3f2fd"/>
  <text x="110" y="97" font-size="10" font-weight="bold" fill="#1565c0">Line Pointer Array (Item IDs)</text>
  <rect x="110" y="103" width="60" height="24" rx="3" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="140" y="120" text-anchor="middle" font-size="8" fill="#333">LP 1</text>
  <rect x="175" y="103" width="60" height="24" rx="3" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="205" y="120" text-anchor="middle" font-size="8" fill="#333">LP 2</text>
  <rect x="240" y="103" width="60" height="24" rx="3" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="270" y="120" text-anchor="middle" font-size="8" fill="#333">LP 3</text>
  <rect x="305" y="103" width="60" height="24" rx="3" fill="#bbdefb" stroke="#1565c0" stroke-width="1"/>
  <text x="335" y="120" text-anchor="middle" font-size="8" fill="#333">LP 4</text>
  <rect x="370" y="103" width="60" height="24" rx="3" fill="#e0e0e0" stroke="#999" stroke-width="1"/>
  <text x="400" y="120" text-anchor="middle" font-size="8" fill="#888">LP 5</text>
  <text x="440" y="120" font-size="8" fill="#888">→</text>

  <!-- Free space -->
  <rect x="100" y="135" width="400" height="115" fill="#f5f5f5"/>
  <text x="300" y="195" text-anchor="middle" font-size="14" fill="#bbb">Free Space</text>
  <text x="300" y="215" text-anchor="middle" font-size="9" fill="#999">Line pointers grow ↓   Tuples grow ↑</text>

  <!-- Tuples (packed from bottom) -->
  <rect x="100" y="250" width="400" height="40" fill="#c8e6c9"/>
  <text x="300" y="275" text-anchor="middle" font-size="10" fill="#333">Tuple 4 (120 bytes) — newest</text>

  <rect x="100" y="290" width="400" height="35" fill="#a5d6a7"/>
  <text x="300" y="312" text-anchor="middle" font-size="10" fill="#333">Tuple 3 (95 bytes)</text>

  <rect x="100" y="325" width="400" height="35" fill="#81c784"/>
  <text x="300" y="347" text-anchor="middle" font-size="10" fill="white">Tuple 2 (140 bytes)</text>

  <rect x="100" y="360" width="400" height="35" fill="#66bb6a" rx="4"/>
  <text x="300" y="382" text-anchor="middle" font-size="10" fill="white">Tuple 1 (200 bytes) — oldest</text>

  <!-- Arrows from LPs to tuples -->
  <line x1="140" y1="127" x2="140" y2="360" stroke="#1565c0" stroke-width="1" stroke-dasharray="4,3"/>
  <line x1="205" y1="127" x2="205" y2="325" stroke="#1565c0" stroke-width="1" stroke-dasharray="4,3"/>
  <line x1="270" y1="127" x2="270" y2="290" stroke="#1565c0" stroke-width="1" stroke-dasharray="4,3"/>
  <line x1="335" y1="127" x2="335" y2="250" stroke="#1565c0" stroke-width="1" stroke-dasharray="4,3"/>

  <!-- Annotations -->
  <text x="530" y="62" font-size="9" fill="#c62828">LSN, checksum, flags</text>
  <text x="530" y="78" font-size="9" fill="#c62828">pd_lower, pd_upper offsets</text>

  <text x="530" y="108" font-size="9" fill="#1565c0">Each LP: 4 bytes</text>
  <text x="530" y="122" font-size="9" fill="#1565c0">(offset, length, flags)</text>

  <text x="530" y="190" font-size="9" fill="#999">pd_lower → end of LPs</text>
  <text x="530" y="206" font-size="9" fill="#999">pd_upper → start of tuples</text>
  <text x="530" y="222" font-size="9" fill="#999">Free = pd_upper − pd_lower</text>

  <text x="530" y="300" font-size="9" fill="#2e7d32">Variable-length rows</text>
  <text x="530" y="316" font-size="9" fill="#2e7d32">packed from page end</text>
  <text x="530" y="336" font-size="9" fill="#2e7d32">Deletion: set LP to "dead"</text>
  <text x="530" y="352" font-size="9" fill="#2e7d32">VACUUM compacts free space</text>
</svg>

### How It Works

| Component | Purpose |
|---|---|
| **Page header** | LSN (for crash recovery), checksum, `pd_lower` (end of LP array), `pd_upper` (start of tuples), free space info |
| **Line pointer array** | Fixed-size slots (4 bytes each) that store `(offset, length)` pointing to each tuple. Grows forward from header. |
| **Free space** | Gap between line pointers and tuples. Shrinks as page fills. |
| **Tuple data** | Actual row data. Grows backward from the end of the page. |

### Why Slotted Pages?

1. **Variable-length rows**: Tuples can be any size — the line pointer maps to the exact offset
2. **Stable addresses**: External references (indexes, CTID) point to `(page_number, line_pointer_slot)` — the tuple can be moved within the page (compacted) without invalidating references
3. **Efficient deletion**: Mark the line pointer as "dead" — no data movement needed until VACUUM
4. **Compaction**: VACUUM or page defragmentation slides tuples together, reclaiming free space

### PostgreSQL CTID

```sql
-- Every PostgreSQL row has a hidden CTID (page, slot)
SELECT ctid, order_id, total FROM orders LIMIT 5;
```

| ctid | order_id | total |
|---|---|---|
| (0,1) | 1001 | 250.00 |
| (0,2) | 1002 | 89.50 |
| (0,3) | 1003 | 430.00 |
| (1,1) | 1004 | 175.00 |
| (1,2) | 1005 | 60.00 |

`(0,1)` means page 0, line pointer slot 1. `(1,1)` means page 1, slot 1.

---

## 5. Heap Files (Unordered)

A **heap file** stores rows in **insertion order** with no particular sort. This is the default storage organisation for most RDBMS table data.

### How Heap Works

- New rows are appended to the first page with enough free space
- No ordering guarantee — rows from different time periods can be on the same page
- PostgreSQL maintains a **Free Space Map (FSM)** to quickly find pages with room
- MySQL/InnoDB stores rows ordered by primary key (clustered index) — not a pure heap

### PostgreSQL Heap Internals

```sql
-- Check how many pages a table uses
SELECT
    pg_relation_size('orders') AS bytes,
    pg_relation_size('orders') / 8192 AS pages,
    pg_size_pretty(pg_relation_size('orders')) AS human_readable
FROM pg_class WHERE relname = 'orders';
```

| bytes | pages | human_readable |
|---|---|---|
| 819200000 | 100000 | 781 MB |

```sql
-- Check page-level stats with pageinspect extension
CREATE EXTENSION pageinspect;

SELECT
    lp,
    lp_off,
    lp_len,
    t_xmin,
    t_xmax,
    t_ctid
FROM heap_page_items(get_raw_page('orders', 0))
LIMIT 5;
```

### Heap File Operations Cost

| Operation | Cost | Why |
|---|---|---|
| **INSERT** | O(1) — append | Find a page with space (FSM), write tuple |
| **SELECT by PK** (no index) | O(N) — full scan | Must check every page |
| **SELECT by PK** (with index) | O(log N) + 1 random I/O | Index lookup → CTID → fetch page |
| **SELECT range** (no index) | O(N) — full scan | No ordering, must scan all |
| **DELETE** | O(1) per row (mark dead) | Set xmax, LP becomes dead; VACUUM later |
| **UPDATE** | Insert new version + mark old dead | PostgreSQL MVCC: new tuple, new CTID |

### Free Space Map (FSM)

PostgreSQL tracks free space per page in a companion file (`relname_fsm`):

```sql
-- Install the extension
CREATE EXTENSION pg_freespacemap;

-- Check free space per page
SELECT
    blkno AS page,
    avail AS free_bytes
FROM pg_freespace('orders')
WHERE avail > 0
ORDER BY blkno
LIMIT 10;
```

---

## 6. Sorted Files (Ordered)

A **sorted file** (or **clustered file**) stores rows in a defined sort order. This makes range scans extremely fast because consecutive rows are on consecutive pages — pure sequential I/O.

### Clustered Indexes

| RDBMS | Clustered Index Behaviour |
|---|---|
| **MySQL/InnoDB** | The **primary key IS the clustered index** — leaf pages contain the full row. Every InnoDB table is a B+Tree organised by PK. |
| **SQL Server** | One clustered index per table (default: PK). Leaf level contains data rows in sort order. |
| **PostgreSQL** | No automatic clustering. `CLUSTER` command sorts the table once, but new inserts are unordered. |
| **Oracle** | **Index-Organised Table (IOT)** — stores rows in PK order in the B-Tree itself. |

### PostgreSQL CLUSTER

```sql
-- One-time physical reorder by an index
CLUSTER orders USING idx_orders_date;

-- After CLUSTER: rows with similar dates are on adjacent pages
-- Range scans on order_date become sequential I/O
-- But: CLUSTER is a full table lock, and new inserts go to heap positions

-- pg_repack: online alternative (no lock)
-- Install extension, then:
-- $ pg_repack --table orders --order-by order_date dbname
```

### InnoDB Clustered Index (MySQL)

```sql
-- InnoDB automatically clusters by PRIMARY KEY
CREATE TABLE orders (
    order_id    BIGINT AUTO_INCREMENT PRIMARY KEY,  -- clustered index
    customer_id INT,
    order_date  DATE,
    total       DECIMAL(12,2)
) ENGINE=InnoDB;

-- The leaf pages of the PK B+Tree contain the full row data.
-- Secondary indexes store (secondary_key → primary_key),
-- requiring a "bookmark lookup" to fetch the row.
```

<svg viewBox="0 0 780 350" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Heap File vs Clustered (Sorted) File</text>

  <!-- Heap (left) -->
  <rect x="30" y="50" width="330" height="280" rx="12" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="195" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">Heap File (Unordered)</text>

  <!-- Pages with scattered dates -->
  <rect x="50" y="90" width="130" height="70" rx="4" fill="white" stroke="#999" stroke-width="1"/>
  <text x="115" y="105" text-anchor="middle" font-size="8" fill="#888">Page 0</text>
  <text x="60" y="120" font-size="8" fill="#c62828">Jan 3</text>
  <text x="100" y="120" font-size="8" fill="#1565c0">Mar 15</text>
  <text x="60" y="135" font-size="8" fill="#2e7d32">Jul 22</text>
  <text x="100" y="135" font-size="8" fill="#e65100">Feb 8</text>
  <text x="60" y="150" font-size="8" fill="#6a1b9a">Nov 1</text>

  <rect x="200" y="90" width="130" height="70" rx="4" fill="white" stroke="#999" stroke-width="1"/>
  <text x="265" y="105" text-anchor="middle" font-size="8" fill="#888">Page 1</text>
  <text x="210" y="120" font-size="8" fill="#e65100">Feb 12</text>
  <text x="260" y="120" font-size="8" fill="#c62828">Jan 28</text>
  <text x="210" y="135" font-size="8" fill="#2e7d32">Aug 5</text>
  <text x="260" y="135" font-size="8" fill="#1565c0">Apr 19</text>
  <text x="210" y="150" font-size="8" fill="#6a1b9a">Dec 3</text>

  <text x="195" y="185" text-anchor="middle" font-size="9" fill="#555">Dates are scattered randomly across pages</text>

  <rect x="50" y="200" width="280" height="50" rx="6" fill="#ffcdd2"/>
  <text x="60" y="218" font-size="9" fill="#333">Range scan WHERE date BETWEEN Jan AND Mar:</text>
  <text x="60" y="234" font-size="9" fill="#c62828">Must read ALL pages (full table scan)</text>
  <text x="60" y="246" font-size="9" fill="#c62828">because matching rows are on every page</text>

  <text x="195" y="280" text-anchor="middle" font-size="10" fill="#c62828">INSERT: O(1) — fast</text>
  <text x="195" y="296" text-anchor="middle" font-size="10" fill="#c62828">Range scan: O(N) — slow</text>
  <text x="195" y="312" text-anchor="middle" font-size="10" fill="#c62828">Random I/O pattern</text>

  <!-- Sorted (right) -->
  <rect x="400" y="50" width="350" height="280" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="575" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">Clustered / Sorted File</text>

  <rect x="420" y="90" width="130" height="70" rx="4" fill="white" stroke="#999" stroke-width="1"/>
  <text x="485" y="105" text-anchor="middle" font-size="8" fill="#888">Page 0</text>
  <text x="430" y="120" font-size="8" fill="#c62828">Jan 3</text>
  <text x="480" y="120" font-size="8" fill="#c62828">Jan 15</text>
  <text x="430" y="135" font-size="8" fill="#c62828">Jan 28</text>
  <text x="480" y="135" font-size="8" fill="#e65100">Feb 8</text>
  <text x="430" y="150" font-size="8" fill="#e65100">Feb 12</text>

  <rect x="570" y="90" width="130" height="70" rx="4" fill="white" stroke="#999" stroke-width="1"/>
  <text x="635" y="105" text-anchor="middle" font-size="8" fill="#888">Page 1</text>
  <text x="580" y="120" font-size="8" fill="#1565c0">Mar 1</text>
  <text x="630" y="120" font-size="8" fill="#1565c0">Mar 15</text>
  <text x="580" y="135" font-size="8" fill="#1565c0">Apr 2</text>
  <text x="630" y="135" font-size="8" fill="#1565c0">Apr 19</text>
  <text x="580" y="150" font-size="8" fill="#1565c0">May 7</text>

  <text x="575" y="185" text-anchor="middle" font-size="9" fill="#555">Dates are sorted — adjacent dates on same pages</text>

  <rect x="420" y="200" width="300" height="50" rx="6" fill="#c8e6c9"/>
  <text x="430" y="218" font-size="9" fill="#333">Range scan WHERE date BETWEEN Jan AND Mar:</text>
  <text x="430" y="234" font-size="9" fill="#2e7d32">Read pages 0–1 sequentially — only 2 pages!</text>
  <text x="430" y="246" font-size="9" fill="#2e7d32">Matching rows are contiguous</text>

  <text x="575" y="280" text-anchor="middle" font-size="10" fill="#2e7d32">INSERT: O(N) — must maintain order</text>
  <text x="575" y="296" text-anchor="middle" font-size="10" fill="#2e7d32">Range scan: O(log N + K) — fast</text>
  <text x="575" y="312" text-anchor="middle" font-size="10" fill="#2e7d32">Sequential I/O pattern</text>
</svg>

---

## 7. Hashed Files

A **hashed file** uses a hash function to map a search key directly to a **bucket** (page or set of pages). This gives O(1) lookups for exact-match queries but does not support range scans.

### Static Hashing

```
bucket_number = hash(key) % N_buckets
```

Each bucket is one or more pages. A key maps to exactly one bucket.

**Problem**: As the table grows, buckets fill up and overflow chains form (linked pages), degrading performance.

### Dynamic Hashing Schemes

| Scheme | How It Works | Advantage |
|---|---|---|
| **Extendible hashing** | Uses a directory of pointers; doubles directory and splits one bucket at a time | No overflow chains; graceful growth |
| **Linear hashing** | Splits buckets in round-robin order triggered by load factor | No directory needed; incremental growth |

### PostgreSQL Hash Index

```sql
-- PostgreSQL hash index (useful for exact-match only)
CREATE INDEX idx_sessions_token ON sessions USING hash (session_token);

-- Good for: WHERE session_token = 'abc123'
-- Bad for:  WHERE session_token > 'abc' (range scan not supported)
-- Bad for:  ORDER BY session_token (no ordering)
```

### Hash File Operations Cost

| Operation | Cost |
|---|---|
| **Exact match** (`WHERE key = X`) | O(1) — hash → bucket → scan bucket |
| **Range scan** (`WHERE key BETWEEN X AND Y`) | O(N) — must scan all buckets |
| **INSERT** | O(1) — hash → bucket → append |
| **DELETE** | O(1) — hash → bucket → mark deleted |

<svg viewBox="0 0 780 300" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="22" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Static Hashing — Bucket Structure</text>

  <!-- Hash function -->
  <rect x="40" y="50" width="140" height="40" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="110" y="75" text-anchor="middle" font-size="11" font-weight="bold" fill="#1565c0">hash(key) % 4</text>

  <!-- Arrows -->
  <line x1="185" y1="70" x2="265" y2="70" stroke="#666" stroke-width="1.5" marker-end="url(#hArr)"/>
  <line x1="185" y1="70" x2="265" y2="130" stroke="#666" stroke-width="1.5" marker-end="url(#hArr)"/>
  <line x1="185" y1="70" x2="265" y2="190" stroke="#666" stroke-width="1.5" marker-end="url(#hArr)"/>
  <line x1="185" y1="70" x2="265" y2="250" stroke="#666" stroke-width="1.5" marker-end="url(#hArr)"/>
  <defs><marker id="hArr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#666"/></marker></defs>

  <!-- Bucket 0 -->
  <rect x="270" y="50" width="110" height="40" rx="6" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="325" y="75" text-anchor="middle" font-size="10" fill="#333">Bucket 0</text>
  <rect x="390" y="50" width="80" height="40" rx="4" fill="#ffe0b2" stroke="#e65100" stroke-width="1"/>
  <text x="430" y="68" text-anchor="middle" font-size="8" fill="#333">Alice</text>
  <text x="430" y="82" text-anchor="middle" font-size="8" fill="#333">Eve</text>

  <!-- Bucket 1 -->
  <rect x="270" y="110" width="110" height="40" rx="6" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="325" y="135" text-anchor="middle" font-size="10" fill="#333">Bucket 1</text>
  <rect x="390" y="110" width="80" height="40" rx="4" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1"/>
  <text x="430" y="128" text-anchor="middle" font-size="8" fill="#333">Bob</text>
  <text x="430" y="142" text-anchor="middle" font-size="8" fill="#333">Frank</text>
  <!-- Overflow -->
  <line x1="475" y1="130" x2="510" y2="130" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#hArr)"/>
  <rect x="515" y="110" width="80" height="40" rx="4" fill="#a5d6a7" stroke="#2e7d32" stroke-width="1"/>
  <text x="555" y="128" text-anchor="middle" font-size="8" fill="#333">Grace</text>
  <text x="555" y="142" text-anchor="middle" font-size="8" fill="#888">overflow</text>

  <!-- Bucket 2 -->
  <rect x="270" y="170" width="110" height="40" rx="6" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="325" y="195" text-anchor="middle" font-size="10" fill="#333">Bucket 2</text>
  <rect x="390" y="170" width="80" height="40" rx="4" fill="#e1bee7" stroke="#6a1b9a" stroke-width="1"/>
  <text x="430" y="188" text-anchor="middle" font-size="8" fill="#333">Carol</text>
  <text x="430" y="202" text-anchor="middle" font-size="8" fill="#333">Henry</text>

  <!-- Bucket 3 -->
  <rect x="270" y="230" width="110" height="40" rx="6" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="325" y="255" text-anchor="middle" font-size="10" fill="#333">Bucket 3</text>
  <rect x="390" y="230" width="80" height="40" rx="4" fill="#f8bbd0" stroke="#c62828" stroke-width="1"/>
  <text x="430" y="248" text-anchor="middle" font-size="8" fill="#333">Dave</text>
  <text x="430" y="262" text-anchor="middle" font-size="8" fill="#888">(1 slot free)</text>

  <!-- Note -->
  <rect x="520" y="175" width="230" height="100" rx="8" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="535" y="195" font-size="9" font-weight="bold" fill="#333">Hashing Trade-offs</text>
  <text x="535" y="213" font-size="9" fill="#2e7d32">+ O(1) exact-match lookup</text>
  <text x="535" y="229" font-size="9" fill="#2e7d32">+ Fast INSERT and DELETE</text>
  <text x="535" y="245" font-size="9" fill="#c62828">− No range scan support</text>
  <text x="535" y="261" font-size="9" fill="#c62828">− Overflow chains degrade perf</text>
</svg>

---

## 8. File Organization Comparison

| Aspect | Heap (Unordered) | Sorted (Clustered) | Hashed |
|---|---|---|---|
| **Row order** | Insertion order | Sorted by key | Hash bucket |
| **Point query (PK)** | O(N) without index, O(log N) with | O(log N) binary search | **O(1)** |
| **Range scan** | O(N) — full scan | **O(log N + K)** — sequential | O(N) — must scan all buckets |
| **INSERT** | **O(1)** — append anywhere | O(N) — maintain sort order | O(1) — hash to bucket |
| **DELETE** | O(1) — mark dead | O(N) — may leave gaps | O(1) — mark dead in bucket |
| **UPDATE** | O(1) — new version (MVCC) | O(N) — may need reposition | O(1) — within bucket |
| **Sequential scan** | Reads all pages | Reads all pages (but sorted!) | Reads all buckets |
| **Best for** | General OLTP, mixed workloads | Range-heavy analytics | Exact-match lookups |
| **RDBMS default** | PostgreSQL, Oracle | MySQL/InnoDB (by PK), SQL Server clustered | Rarely the default |

---

## 9. Buffer Pool Management

The **buffer pool** (called **shared_buffers** in PostgreSQL, **innodb_buffer_pool** in MySQL, **buffer cache** in SQL Server) is the in-memory cache of disk pages. It's the single most important performance component — every page read from disk first goes through the buffer pool.

### How Buffer Pool Works

<svg viewBox="0 0 800 380" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Buffer Pool — Page Lifecycle</text>

  <!-- Application query -->
  <rect x="20" y="55" width="160" height="40" rx="8" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="100" y="80" text-anchor="middle" font-size="10" font-weight="bold" fill="#3949ab">SQL Query</text>

  <line x1="185" y1="75" x2="250" y2="75" stroke="#666" stroke-width="1.5" marker-end="url(#bArr)"/>
  <defs><marker id="bArr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#666"/></marker></defs>
  <text x="215" y="68" font-size="8" fill="#888">request</text>
  <text x="215" y="90" font-size="8" fill="#888">page</text>

  <!-- Buffer pool -->
  <rect x="255" y="40" width="340" height="130" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2.5"/>
  <text x="425" y="62" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Buffer Pool (RAM)</text>

  <!-- Page slots -->
  <rect x="275" y="72" width="55" height="40" rx="4" fill="#2e7d32" stroke="#1b5e20" stroke-width="1.5"/>
  <text x="302" y="89" text-anchor="middle" font-size="8" fill="white">Page 5</text>
  <text x="302" y="104" text-anchor="middle" font-size="7" fill="#c8e6c9">clean</text>

  <rect x="340" y="72" width="55" height="40" rx="4" fill="#c62828" stroke="#b71c1c" stroke-width="1.5"/>
  <text x="367" y="89" text-anchor="middle" font-size="8" fill="white">Page 12</text>
  <text x="367" y="104" text-anchor="middle" font-size="7" fill="#ffcdd2">dirty</text>

  <rect x="405" y="72" width="55" height="40" rx="4" fill="#2e7d32" stroke="#1b5e20" stroke-width="1.5"/>
  <text x="432" y="89" text-anchor="middle" font-size="8" fill="white">Page 3</text>
  <text x="432" y="104" text-anchor="middle" font-size="7" fill="#c8e6c9">clean</text>

  <rect x="470" y="72" width="55" height="40" rx="4" fill="#c62828" stroke="#b71c1c" stroke-width="1.5"/>
  <text x="497" y="89" text-anchor="middle" font-size="8" fill="white">Page 8</text>
  <text x="497" y="104" text-anchor="middle" font-size="7" fill="#ffcdd2">dirty</text>

  <rect x="535" y="72" width="55" height="40" rx="4" fill="#e0e0e0" stroke="#999" stroke-width="1.5"/>
  <text x="562" y="89" text-anchor="middle" font-size="8" fill="#666">free</text>
  <text x="562" y="104" text-anchor="middle" font-size="7" fill="#999">slot</text>

  <!-- Hash table -->
  <text x="425" y="132" text-anchor="middle" font-size="9" fill="#1565c0">Page table (hash map): page_id → buffer frame</text>
  <text x="425" y="148" text-anchor="middle" font-size="9" fill="#1565c0">Tracks: pin_count, dirty_flag, last_used</text>

  <!-- Hit path -->
  <text x="425" y="180" text-anchor="middle" font-size="10" font-weight="bold" fill="#2e7d32">Cache HIT</text>
  <text x="425" y="196" text-anchor="middle" font-size="9" fill="#555">Page already in buffer → return immediately (~100 ns)</text>

  <!-- Disk -->
  <rect x="260" y="230" width="340" height="130" rx="12" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="430" y="255" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">Disk (SSD / HDD)</text>

  <rect x="280" y="268" width="50" height="30" rx="3" fill="#ffe0b2" stroke="#e65100" stroke-width="1"/>
  <text x="305" y="288" text-anchor="middle" font-size="7" fill="#333">P0</text>
  <rect x="335" y="268" width="50" height="30" rx="3" fill="#ffe0b2" stroke="#e65100" stroke-width="1"/>
  <text x="360" y="288" text-anchor="middle" font-size="7" fill="#333">P1</text>
  <rect x="390" y="268" width="50" height="30" rx="3" fill="#ffe0b2" stroke="#e65100" stroke-width="1"/>
  <text x="415" y="288" text-anchor="middle" font-size="7" fill="#333">P2</text>
  <rect x="445" y="268" width="50" height="30" rx="3" fill="#ffe0b2" stroke="#e65100" stroke-width="1"/>
  <text x="470" y="288" text-anchor="middle" font-size="7" fill="#333">...</text>
  <rect x="500" y="268" width="50" height="30" rx="3" fill="#ffe0b2" stroke="#e65100" stroke-width="1"/>
  <text x="525" y="288" text-anchor="middle" font-size="7" fill="#333">Pn</text>

  <text x="430" y="322" text-anchor="middle" font-size="9" fill="#555">Data files on filesystem</text>
  <text x="430" y="338" text-anchor="middle" font-size="9" fill="#555">base/&lt;db_oid&gt;/&lt;relfilenode&gt;</text>

  <!-- Miss path -->
  <line x1="560" y1="170" x2="560" y2="230" stroke="#c62828" stroke-width="1.5" marker-end="url(#bArr)" stroke-dasharray="5,3"/>
  <text x="640" y="196" font-size="9" font-weight="bold" fill="#c62828">Cache MISS</text>
  <text x="640" y="212" font-size="8" fill="#555">Read from disk (~20 µs)</text>

  <!-- Evict + write back -->
  <line x1="260" y1="230" x2="260" y2="170" stroke="#6a1b9a" stroke-width="1.5" marker-end="url(#bArr)" stroke-dasharray="5,3"/>
  <text x="125" y="196" font-size="9" font-weight="bold" fill="#6a1b9a">Eviction</text>
  <text x="125" y="212" font-size="8" fill="#555">If dirty → write back first</text>
  <text x="125" y="226" font-size="8" fill="#555">Then load new page</text>
</svg>

### Buffer Pool Lookup Flow

1. **Query requests page** (e.g., page 42 of `orders` table)
2. **Check page table** (hash map: `page_id → buffer_frame`)
3. **HIT**: page is in buffer → increment pin count, return pointer (microseconds)
4. **MISS**: page not in buffer →
   a. Find a free frame or **evict** a page using replacement policy
   b. If evicted page is **dirty**, write it to disk first
   c. Read requested page from disk into the frame
   d. Update page table, return pointer

### Configuration

```sql
-- PostgreSQL: shared_buffers (typically 25% of system RAM)
SHOW shared_buffers;          -- e.g., '8GB'
-- effective_cache_size (hint for planner, includes OS cache)
SHOW effective_cache_size;    -- e.g., '24GB'

-- Monitor hit ratio
SELECT
    sum(blks_hit) AS buffer_hits,
    sum(blks_read) AS disk_reads,
    ROUND(sum(blks_hit) * 100.0 / 
          NULLIF(sum(blks_hit) + sum(blks_read), 0), 2) AS hit_ratio_pct
FROM pg_stat_database;
-- Target: > 99% hit ratio
```

```sql
-- MySQL: innodb_buffer_pool_size (typically 70-80% of system RAM)
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

---

## 10. LRU & Clock Replacement Policies

When the buffer pool is full and a new page needs to be loaded, the DBMS must decide which existing page to **evict**. The replacement policy makes this decision.

### LRU (Least Recently Used)

Evict the page that hasn't been accessed for the longest time.

**Implementation**: Maintain a doubly-linked list. On access, move the page to the head (MRU end). On eviction, remove from the tail (LRU end).

```
Most Recently Used                              Least Recently Used
  HEAD ←→ Page 12 ←→ Page 5 ←→ Page 3 ←→ Page 8 ←→ TAIL
         ↑ accessed last                        ↑ evict this one
```

**LRU Problem — Sequential Flooding**: A full table scan reads every page once, flooding the buffer pool and evicting frequently-used pages. After the scan, the buffer pool is full of pages that won't be used again.

### Clock (Second-Chance Algorithm)

Used by PostgreSQL. An approximation of LRU that avoids the overhead of maintaining a linked list.

<svg viewBox="0 0 780 380" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Clock Replacement Algorithm</text>

  <!-- Clock circle -->
  <circle cx="250" cy="210" r="140" fill="none" stroke="#1565c0" stroke-width="2.5"/>

  <!-- Clock hand -->
  <line x1="250" y1="210" x2="250" y2="85" stroke="#c62828" stroke-width="3" marker-end="url(#clockHand)"/>
  <defs><marker id="clockHand" viewBox="0 0 10 10" refX="5" refY="0" markerWidth="8" markerHeight="8" orient="auto"><path d="M 0 10 L 5 0 L 10 10 z" fill="#c62828"/></marker></defs>

  <!-- Page slots around the clock -->
  <!-- 12 o'clock -->
  <rect x="225" y="55" width="50" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="250" y="71" text-anchor="middle" font-size="8" fill="#333">P5</text>
  <text x="250" y="80" text-anchor="middle" font-size="7" fill="#2e7d32">ref=1</text>

  <!-- 2 o'clock -->
  <rect x="345" y="95" width="50" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="370" y="111" text-anchor="middle" font-size="8" fill="#333">P12</text>
  <text x="370" y="120" text-anchor="middle" font-size="7" fill="#2e7d32">ref=1</text>

  <!-- 4 o'clock -->
  <rect x="365" y="210" width="50" height="30" rx="4" fill="#ffcdd2" stroke="#c62828" stroke-width="1.5"/>
  <text x="390" y="226" text-anchor="middle" font-size="8" fill="#333">P3</text>
  <text x="390" y="235" text-anchor="middle" font-size="7" fill="#c62828">ref=0</text>

  <!-- 6 o'clock -->
  <rect x="225" y="330" width="50" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="250" y="346" text-anchor="middle" font-size="8" fill="#333">P8</text>
  <text x="250" y="355" text-anchor="middle" font-size="7" fill="#2e7d32">ref=1</text>

  <!-- 8 o'clock -->
  <rect x="95" y="270" width="50" height="30" rx="4" fill="#ffcdd2" stroke="#c62828" stroke-width="1.5"/>
  <text x="120" y="286" text-anchor="middle" font-size="8" fill="#333">P1</text>
  <text x="120" y="295" text-anchor="middle" font-size="7" fill="#c62828">ref=0</text>

  <!-- 10 o'clock -->
  <rect x="90" y="130" width="50" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="115" y="146" text-anchor="middle" font-size="8" fill="#333">P22</text>
  <text x="115" y="155" text-anchor="middle" font-size="7" fill="#2e7d32">ref=1</text>

  <!-- Algorithm description -->
  <rect x="440" y="55" width="320" height="300" rx="10" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="600" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Clock Algorithm Steps</text>

  <text x="460" y="105" font-size="10" fill="#1565c0">1. Each frame has a reference bit (ref)</text>
  <text x="460" y="125" font-size="10" fill="#1565c0">2. When a page is accessed: set ref = 1</text>
  <text x="460" y="150" font-size="10" fill="#333">On eviction — clock hand sweeps:</text>
  <text x="460" y="172" font-size="10" fill="#2e7d32">3. If ref = 1: set ref = 0 (second chance)</text>
  <text x="475" y="190" font-size="10" fill="#2e7d32">   → move hand clockwise, skip</text>
  <text x="460" y="215" font-size="10" fill="#c62828">4. If ref = 0: EVICT this page</text>
  <text x="475" y="233" font-size="10" fill="#c62828">   → if dirty, flush to disk first</text>
  <text x="475" y="251" font-size="10" fill="#c62828">   → load new page into this frame</text>

  <rect x="460" y="270" width="280" height="70" rx="6" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="470" y="290" font-size="9" fill="#333">PostgreSQL uses this with usage_count</text>
  <text x="470" y="306" font-size="9" fill="#333">(0–5). Each access increments count.</text>
  <text x="470" y="322" font-size="9" fill="#333">Sweep decrements; evict when count = 0.</text>
</svg>

### Replacement Policy Comparison

| Policy | Description | Pro | Con |
|---|---|---|---|
| **LRU** | Evict least recently accessed | Simple, intuitive | Sequential flooding; O(1) with linked list but constant factor overhead |
| **Clock** | Approximate LRU with ref bits | Low overhead, good enough | Slightly less accurate than true LRU |
| **LRU-K** | Track last K accesses; evict by Kth | Resists sequential flooding | More memory per frame |
| **2Q** | Hot queue + cold queue | Scan-resistant | Two queues to manage |
| **ARC** | Adaptive Replacement Cache | Self-tuning; balances recency + frequency | More complex; patented (IBM) |

### PostgreSQL's Clock-Sweep

PostgreSQL uses a **clock-sweep with usage counts** (0–5):
- On each access: `usage_count = min(usage_count + 1, 5)`
- On eviction sweep: if `usage_count > 0`, decrement and move on; if `usage_count = 0`, evict
- Pages with high usage counts (frequently accessed) survive many sweeps → "stickier"

### Ring Buffer for Sequential Scans

PostgreSQL protects the buffer pool from sequential flooding using a **ring buffer**:

```sql
-- Large sequential scans get a small private buffer ring (256 KB)
-- instead of using the full shared buffer pool.
-- This prevents a full table scan from evicting hot cached pages.

-- Configure: not directly configurable, but automatic for
-- sequential scans larger than 1/4 of shared_buffers
```

---

## 11. Write-Ahead Logging & Dirty Pages

### The Dirty Page Problem

When a query modifies data, the change is made to the page **in memory** (buffer pool). The page is now "dirty" — its in-memory version differs from the on-disk version. The DBMS must eventually write dirty pages to disk, but doing so after every modification would be extremely slow.

### Write-Ahead Logging (WAL)

**Rule**: Before a dirty page is written to disk, the corresponding WAL record must be flushed to the WAL file first. This guarantees crash recovery — if the system crashes before the dirty page is written, the WAL records can reconstruct the changes.

```
1. Transaction modifies Page 42 in buffer pool → page becomes dirty
2. WAL record describing the change is written to WAL buffer
3. On COMMIT: WAL buffer is flushed to WAL file on disk (fsync)
4. At some later time: background writer flushes dirty Page 42 to data file
5. If crash between steps 3 and 4: recovery replays WAL to reconstruct Page 42
```

### Checkpoint

A **checkpoint** forces all dirty pages to disk and writes a checkpoint record to WAL. After a checkpoint, recovery only needs to replay WAL records since the last checkpoint.

```sql
-- PostgreSQL: force a checkpoint
CHECKPOINT;

-- View checkpoint activity
SELECT
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend
FROM pg_stat_bgwriter;
```

### Background Writer

PostgreSQL has a **background writer** (bgwriter) that continuously flushes dirty pages in small batches, so checkpoints don't need to write everything at once:

```sql
-- Configuration
SHOW bgwriter_delay;           -- 200ms (interval between rounds)
SHOW bgwriter_lru_maxpages;    -- 100 (max pages per round)
SHOW bgwriter_lru_multiplier;  -- 2.0 (scale factor based on recent demand)
```

---

## 12. TOAST & Large Objects

### The TOAST Mechanism (PostgreSQL)

**TOAST** (The Oversized-Attribute Storage Technique) handles values that don't fit in a single page. When a row exceeds ~2 KB, PostgreSQL automatically:

1. **Compresses** the large value (using pglz or lz4)
2. If still too large, **slices** it into chunks stored in a separate TOAST table
3. The main table row stores a small pointer to the TOAST chunks

```sql
-- Check TOAST table for a relation
SELECT
    c.relname AS table,
    t.relname AS toast_table
FROM pg_class c
JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relname = 'documents';
```

### TOAST Strategies

| Strategy | Behaviour |
|---|---|
| **PLAIN** | No TOAST — value must fit in a page (used for fixed-length types like INT) |
| **EXTENDED** | Compress first, then store out-of-line if still too large (default for TEXT, JSONB) |
| **EXTERNAL** | Store out-of-line without compression (good for pre-compressed data) |
| **MAIN** | Compress but try to keep in-line; only TOAST if compression isn't enough |

```sql
-- Change TOAST strategy for a column
ALTER TABLE documents ALTER COLUMN body SET STORAGE EXTERNAL;
```

### InnoDB Overflow Pages (MySQL)

MySQL/InnoDB has a similar mechanism:
- **COMPACT / DYNAMIC** row format: large columns stored in overflow pages with a 20-byte pointer in the main page
- **COMPRESSED** row format: pages compressed with zlib

---

## 13. Worked Examples

### Example 1 — Beginner: Counting Pages and Estimating I/O

**Scenario**: Estimate how many pages a table uses and predict sequential scan time.

```sql
-- Table: 10 million rows, average row size 200 bytes
-- Page size: 8 KB (8192 bytes)
-- Usable space per page: ~8000 bytes (after header + line pointers)
-- Rows per page: 8000 / 200 = 40

-- Total pages: 10,000,000 / 40 = 250,000 pages
-- Total size: 250,000 × 8 KB = ~1.9 GB

-- PostgreSQL confirms:
SELECT
    relname,
    relpages,
    reltuples::BIGINT AS row_estimate,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relname = 'orders';

-- Sequential scan time estimate (NVMe SSD @ 3 GB/s):
-- 1.9 GB / 3 GB/s ≈ 0.63 seconds
-- (Plus CPU time for processing 10M tuples)

-- Random I/O (index lookup of 1000 rows on different pages):
-- 1000 pages × 20 µs per random read = 20 ms
```

---

### Example 2 — Beginner: Buffer Pool Hit Ratio

**Scenario**: Monitor the buffer pool to verify it's properly sized.

```sql
-- PostgreSQL: overall hit ratio
SELECT
    datname,
    blks_hit,
    blks_read,
    ROUND(blks_hit * 100.0 / NULLIF(blks_hit + blks_read, 0), 2)
        AS hit_ratio_pct
FROM pg_stat_database
WHERE datname = current_database();
```

| datname | blks_hit | blks_read | hit_ratio_pct |
|---|---|---|---|
| myapp | 482,910,325 | 1,523,418 | 99.69 |

```sql
-- Per-table hit ratio
SELECT
    schemaname,
    relname,
    heap_blks_hit,
    heap_blks_read,
    ROUND(heap_blks_hit * 100.0 /
          NULLIF(heap_blks_hit + heap_blks_read, 0), 2)
        AS table_hit_pct,
    idx_blks_hit,
    idx_blks_read,
    ROUND(idx_blks_hit * 100.0 /
          NULLIF(idx_blks_hit + idx_blks_read, 0), 2)
        AS index_hit_pct
FROM pg_statio_user_tables
WHERE relname = 'orders';
```

**Rule of thumb**: hit ratio < 95% suggests `shared_buffers` is too small or queries have poor access patterns.

---

### Example 3 — Intermediate: Observing the Slotted Page with pageinspect

**Scenario**: Inspect the physical layout of a page to understand how rows are stored.

```sql
-- Install extension
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Create test table and insert data
CREATE TABLE demo (id SERIAL, name VARCHAR(50), amount NUMERIC(10,2));
INSERT INTO demo (name, amount)
SELECT 'User_' || g, (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100) g;

-- Examine page header
SELECT * FROM page_header(get_raw_page('demo', 0));
```

| lsn | checksum | flags | lower | upper | special | pagesize | version | prune_xid |
|---|---|---|---|---|---|---|---|---|
| 0/1A3B5C0 | 0 | 0 | 424 | 3344 | 8192 | 8192 | 4 | 0 |

```sql
-- lower (424): end of line pointer array
-- upper (3344): start of tuple data
-- Free space: 3344 - 424 = 2920 bytes

-- Examine individual tuples
SELECT
    lp       AS slot,
    lp_off   AS offset,
    lp_len   AS length,
    lp_flags AS flags,
    t_xmin   AS inserted_by_txn,
    t_ctid   AS physical_location
FROM heap_page_items(get_raw_page('demo', 0))
LIMIT 10;
```

| slot | offset | length | flags | inserted_by_txn | physical_location |
|---|---|---|---|---|---|
| 1 | 8152 | 42 | 1 | 1001 | (0,1) |
| 2 | 8104 | 44 | 1 | 1001 | (0,2) |
| 3 | 8056 | 45 | 1 | 1001 | (0,3) |
| 4 | 8008 | 43 | 1 | 1001 | (0,4) |
| 5 | 7960 | 44 | 1 | 1001 | (0,5) |

Tuples are packed from the end of the page (offset 8152, 8104, etc.), growing toward the line pointers.

---

### Example 4 — Intermediate: Clustering vs Non-Clustered Scan

**Scenario**: Demonstrate the performance difference between scanning a clustered table and a non-clustered heap.

```sql
-- Create table with 5M rows
CREATE TABLE events_heap (
    event_id   SERIAL PRIMARY KEY,
    user_id    INT,
    event_date DATE,
    payload    TEXT DEFAULT repeat('x', 100)
);

INSERT INTO events_heap (user_id, event_date)
SELECT
    (random() * 100000)::INT,
    '2024-01-01'::DATE + (random() * 365)::INT
FROM generate_series(1, 5000000);

CREATE INDEX idx_events_date ON events_heap (event_date);

-- Measure: range scan on non-clustered heap
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM events_heap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-30';
-- Reads ~14K rows scattered across thousands of pages
-- Buffers: shared hit=12458 read=3210  (many random reads)
-- Time: ~120 ms

-- Now cluster by event_date
CLUSTER events_heap USING idx_events_date;

-- Measure again: same query on clustered table
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM events_heap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-30';
-- Same ~14K rows, but now on contiguous pages
-- Buffers: shared hit=340 read=12  (sequential reads)
-- Time: ~8 ms  (15x faster)
```

---

### Example 5 — Advanced: Buffer Pool Internals with pg_buffercache

**Scenario**: Inspect exactly which pages are in the buffer pool and their state.

```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- What's in the buffer pool right now?
SELECT
    c.relname,
    COUNT(*)                     AS buffers,
    pg_size_pretty(COUNT(*) * 8192) AS cached_size,
    ROUND(COUNT(*) * 100.0 / (
        SELECT setting::INT FROM pg_settings
        WHERE name = 'shared_buffers'
    ), 2) AS pct_of_buffer_pool,
    SUM(CASE WHEN b.isdirty THEN 1 ELSE 0 END) AS dirty_buffers,
    ROUND(AVG(b.usagecount), 2) AS avg_usage_count
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 15;
```

| relname | buffers | cached_size | pct_of_buffer_pool | dirty_buffers | avg_usage_count |
|---|---|---|---|---|---|
| orders | 125,000 | 977 MB | 12.21 | 3,420 | 3.85 |
| orders_pkey | 45,000 | 352 MB | 4.39 | 102 | 4.12 |
| line_items | 82,000 | 641 MB | 8.01 | 5,813 | 2.97 |
| idx_li_order | 28,000 | 219 MB | 2.73 | 45 | 3.55 |
| customers | 15,000 | 117 MB | 1.46 | 234 | 4.40 |
| ... | ... | ... | ... | ... | ... |

```sql
-- Identify "cold" pages that could be evicted
SELECT
    c.relname,
    COUNT(*) AS cold_buffers
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE b.usagecount = 0
  AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY cold_buffers DESC
LIMIT 10;
```

---

### Example 6 — Advanced: InnoDB Clustered Index Internals

**Scenario**: Understanding how InnoDB stores data in a B+Tree clustered by primary key, and how secondary indexes differ.

```sql
-- MySQL/InnoDB: The table IS the clustered index (B+Tree by PK)

CREATE TABLE products (
    product_id   INT AUTO_INCREMENT PRIMARY KEY,
    name         VARCHAR(200),
    category     VARCHAR(50),
    price        DECIMAL(10,2),
    description  TEXT
) ENGINE=InnoDB;

-- Insert 1M rows
INSERT INTO products (name, category, price, description)
SELECT
    CONCAT('Product_', seq),
    ELT(1 + FLOOR(RAND() * 5), 'Electronics','Books','Clothing','Food','Tools'),
    ROUND(RAND() * 500, 2),
    REPEAT('Description text. ', 10)
FROM (
    SELECT @row := @row + 1 AS seq
    FROM information_schema.columns a, information_schema.columns b,
         (SELECT @row := 0) r
    LIMIT 1000000
) t;

-- Secondary index
CREATE INDEX idx_products_category ON products (category);

-- How InnoDB handles queries:

-- 1. Clustered index lookup (by PK):
--    B+Tree root → internal pages → leaf page containing full row
--    ONE tree traversal, data is in the leaf
EXPLAIN SELECT * FROM products WHERE product_id = 42;
-- type: const (single row from clustered index)

-- 2. Secondary index lookup:
--    Secondary B+Tree: leaf stores (category, product_id)
--    Then "bookmark lookup": use product_id to traverse clustered index
EXPLAIN SELECT * FROM products WHERE category = 'Electronics' LIMIT 10;
-- type: ref (secondary index) → then implicit clustered index lookup

-- 3. Covering index (no bookmark lookup needed):
CREATE INDEX idx_cat_price ON products (category, price);
EXPLAIN SELECT category, price FROM products WHERE category = 'Books';
-- Extra: Using index (covering — all columns are in the index)

-- Check InnoDB buffer pool contents
SELECT
    TABLE_NAME,
    INDEX_NAME,
    NUMBER_RECORDS,
    DATA_SIZE,
    IS_STALE
FROM information_schema.INNODB_BUFFER_PAGE_LRU
WHERE TABLE_NAME LIKE '%products%'
LIMIT 10;
```

---

## 14. Common Interview Questions

### Q1: What is a page (block) in a database? Why do databases use fixed-size pages?

**A:** A page is the minimum unit of I/O — typically 8 KB (PostgreSQL, SQL Server) or 16 KB (InnoDB). Databases use fixed-size pages because: (1) disk I/O reads in blocks anyway — might as well align; (2) buffer pool management is simpler with uniform-size slots; (3) pages can be directly mapped to memory frames; (4) checksumming and recovery operate at the page level.

---

### Q2: Explain the slotted page format. Why is it used?

**A:** A slotted page has: a header (LSN, free-space offsets), a line pointer array growing forward, free space in the middle, and tuple data growing backward from the page end. Line pointers are fixed-size slots that store (offset, length) pointing to each tuple. Benefits: (1) supports variable-length rows; (2) tuples can be compacted/moved within the page without invalidating external references (which point to the slot number, not the byte offset); (3) deletion is O(1) — just invalidate the slot.

---

### Q3: What is the buffer pool and why is it important?

**A:** The buffer pool is an in-memory cache of disk pages (shared_buffers in PostgreSQL, innodb_buffer_pool in MySQL). It's the most critical performance component because RAM access is ~1000x faster than SSD. Every page read goes through the buffer pool: on a hit, the page is served from memory; on a miss, it's read from disk and cached. A properly sized buffer pool (typically 25% of RAM for PostgreSQL, 70-80% for MySQL) can achieve >99% hit ratios.

---

### Q4: Compare heap files, sorted files, and hashed files.

**A:** **Heap** files store rows in insertion order — fast INSERT (O(1)) but slow range scans (O(N)). **Sorted** files maintain rows in key order — excellent for range scans (sequential I/O) but expensive INSERTs (must maintain order). **Hashed** files map keys to buckets — O(1) exact-match lookups but no range scan support. Most RDBMS use heap + B-Tree indexes as the default, with InnoDB being the exception (clustered B+Tree by PK).

---

### Q5: What is the difference between sequential and random I/O? Why does it matter?

**A:** Sequential I/O reads contiguous pages in order; random I/O reads pages scattered across disk. On HDD, sequential is ~200x faster (no seek/rotation latency). On SSD, sequential is still ~6x faster (no seek but better pipeline utilisation). This is why the query planner assigns `random_page_cost = 4.0` vs `seq_page_cost = 1.0`, and why full table scans (sequential) can beat index lookups (random) for queries returning >5-10% of the table.

---

### Q6: Explain the LRU and Clock replacement policies.

**A:** **LRU** evicts the least recently accessed page using a linked list. Simple but vulnerable to sequential flooding (a full scan evicts all hot pages). **Clock** (used by PostgreSQL) approximates LRU with reference bits/usage counts. A "clock hand" sweeps around buffer frames: if usage_count > 0, decrement and skip; if 0, evict. Pages accessed frequently get higher counts and survive more sweeps. PostgreSQL also uses a ring buffer for large sequential scans to avoid polluting the main buffer pool.

---

### Q7: What is Write-Ahead Logging (WAL) and how does it relate to dirty pages?

**A:** A dirty page is a buffer pool page modified in memory but not yet written to the data file on disk. WAL requires that the log record for any change is flushed to disk **before** the dirty page is written. This ensures crash recovery: if the system crashes, dirty pages are lost, but the WAL records survive and can be replayed. Checkpoints periodically flush all dirty pages to disk, establishing a recovery baseline.

---

### Q8: What is TOAST in PostgreSQL?

**A:** TOAST (The Oversized-Attribute Storage Technique) handles values that exceed ~2 KB. PostgreSQL first tries to compress the value; if still too large, it splits the data into chunks stored in a separate TOAST table, leaving a small pointer in the main row. This keeps the main table's pages efficient for scanning. TOAST is automatic and transparent. Strategies: EXTENDED (compress + out-of-line), EXTERNAL (out-of-line without compression), MAIN (compress, avoid out-of-line), PLAIN (no TOAST).

---

### Q9: How does InnoDB's clustered index differ from PostgreSQL's heap storage?

**A:** In InnoDB, the primary key B+Tree IS the table — leaf pages contain the full row data, so rows are physically sorted by PK. Secondary indexes store `(secondary_key → PK)` and require a "bookmark lookup" to fetch the row. In PostgreSQL, the table is a heap (unordered), and all indexes (including the PK index) store `(key → CTID)` pointing to the heap. PostgreSQL has no automatic clustering; the `CLUSTER` command sorts once but doesn't maintain order.

---

### Q10: How do you choose an appropriate page size?

**A:** Smaller pages (4 KB): less wasted space for small rows, less I/O amplification for point queries, but more pages to manage and more I/O operations for scans. Larger pages (16–32 KB): fewer I/O operations for scans, can fit wider rows without TOAST/overflow, but more wasted space and read amplification for point queries. Most systems use 8 KB (good balance). InnoDB uses 16 KB to fit wider rows and reduce B+Tree depth. OLAP systems sometimes use 32–64 KB for scan efficiency.

---

### Q11: What is the Free Space Map (FSM) in PostgreSQL?

**A:** The FSM is a per-table companion file that tracks how much free space each page has. When a new row needs to be inserted, PostgreSQL consults the FSM to find a page with enough room, avoiding a linear search through all pages. VACUUM updates the FSM as it reclaims space from dead tuples. Without the FSM, inserts would always append to the end, causing table bloat even when earlier pages have space.

---

### Q12: Why can a sequential scan be faster than an index scan?

**A:** An index scan reads pages in random order (each CTID points to a different page), while a sequential scan reads pages in order. When a query returns a large fraction of the table (typically >5-10%), the cost of random I/O from the index exceeds the cost of reading the entire table sequentially. The query planner compares `N × seq_page_cost` (full scan) vs `M × random_page_cost` (index) and chooses the cheaper path. On HDD, the crossover is ~5%; on SSD, ~10-20%.

---

## 15. Key Takeaways

<svg viewBox="0 0 800 520" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="28" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Storage Internals — Key Takeaways</text>

  <rect x="20" y="48" width="370" height="62" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="35" y="70" font-size="11" font-weight="bold" fill="#1565c0">1. Everything Is Pages</text>
  <text x="35" y="86" font-size="9" fill="#555">Databases read/write in fixed-size pages (8–16 KB).</text>
  <text x="35" y="100" font-size="9" fill="#555">Optimise for fewer page reads, not fewer row reads.</text>

  <rect x="410" y="48" width="370" height="62" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="425" y="70" font-size="11" font-weight="bold" fill="#2e7d32">2. Buffer Pool Is Everything</text>
  <text x="425" y="86" font-size="9" fill="#555">RAM is 1000x faster than SSD. A 99%+ hit ratio means</text>
  <text x="425" y="100" font-size="9" fill="#555">almost no disk reads. Size your buffer pool correctly.</text>

  <rect x="20" y="120" width="370" height="62" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="35" y="142" font-size="11" font-weight="bold" fill="#e65100">3. Sequential > Random I/O</text>
  <text x="35" y="158" font-size="9" fill="#555">Sequential reads are 6–200x faster than random.</text>
  <text x="35" y="172" font-size="9" fill="#555">Cluster data to make range scans sequential.</text>

  <rect x="410" y="120" width="370" height="62" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="425" y="142" font-size="11" font-weight="bold" fill="#c62828">4. Heap Is Default, Not Best</text>
  <text x="425" y="158" font-size="9" fill="#555">Heap files are unordered — great for writes, bad for</text>
  <text x="425" y="172" font-size="9" fill="#555">range scans. CLUSTER or use InnoDB clustered indexes.</text>

  <rect x="20" y="192" width="370" height="62" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="35" y="214" font-size="11" font-weight="bold" fill="#6a1b9a">5. Slotted Pages Enable Flexibility</text>
  <text x="35" y="230" font-size="9" fill="#555">Line pointers decouple row identity from physical</text>
  <text x="35" y="244" font-size="9" fill="#555">position. Rows can move within a page safely.</text>

  <rect x="410" y="192" width="370" height="62" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="425" y="214" font-size="11" font-weight="bold" fill="#00695c">6. WAL Before Data</text>
  <text x="425" y="230" font-size="9" fill="#555">Write-ahead logging ensures crash safety. Log records</text>
  <text x="425" y="244" font-size="9" fill="#555">hit disk before dirty pages do. Checkpoints flush.</text>

  <rect x="20" y="264" width="370" height="62" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="35" y="286" font-size="11" font-weight="bold" fill="#f57f17">7. Clock-Sweep > Naive LRU</text>
  <text x="35" y="302" font-size="9" fill="#555">PostgreSQL's clock with usage counts is scan-resistant.</text>
  <text x="35" y="316" font-size="9" fill="#555">Large scans get ring buffers to avoid cache pollution.</text>

  <rect x="410" y="264" width="370" height="62" rx="10" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="425" y="286" font-size="11" font-weight="bold" fill="#3949ab">8. TOAST Handles Big Values</text>
  <text x="425" y="302" font-size="9" fill="#555">Values > 2 KB are compressed and/or chunked into a</text>
  <text x="425" y="316" font-size="9" fill="#555">separate table. Main table stays scannable and compact.</text>

  <rect x="20" y="336" width="370" height="62" rx="10" fill="#fbe9e7" stroke="#bf360c" stroke-width="2"/>
  <text x="35" y="358" font-size="11" font-weight="bold" fill="#bf360c">9. Know Your Working Set</text>
  <text x="35" y="374" font-size="9" fill="#555">If your hot data fits in buffer pool → RAM speed.</text>
  <text x="35" y="388" font-size="9" fill="#555">If not → disk I/O dominates. Partition, archive, index.</text>

  <rect x="410" y="336" width="370" height="62" rx="10" fill="#e0f7fa" stroke="#006064" stroke-width="2"/>
  <text x="425" y="358" font-size="11" font-weight="bold" fill="#006064">10. Monitor with Extensions</text>
  <text x="425" y="374" font-size="9" fill="#555">pageinspect, pg_buffercache, pg_freespacemap reveal</text>
  <text x="425" y="388" font-size="9" fill="#555">what's happening at the physical level. Use them.</text>

  <!-- Bottom summary -->
  <rect x="100" y="420" width="600" height="85" rx="12" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="400" y="442" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Performance Hierarchy</text>
  <text x="120" y="462" font-size="10" fill="#555">Buffer pool hit (100 ns) > SSD read (20 µs) > HDD read (5 ms)</text>
  <text x="120" y="478" font-size="10" fill="#555">Sequential scan > Random I/O    |    Clustered > Heap for range scans</text>
  <text x="120" y="494" font-size="10" fill="#555">Design your schema and queries to stay in the buffer pool.</text>
</svg>

---

## 16. Download

<a href="22_storage_internals.md" download="22_storage_internals.md" style="display:inline-block;padding:14px 28px;background:#2e7d32;color:white;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;cursor:pointer;">Download 22_storage_internals.md</a>
