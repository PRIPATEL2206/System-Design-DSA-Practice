# Phase 18 — BASE vs ACID & Eventual Consistency

> **ACID** guarantees correctness at the cost of availability and latency. **BASE** trades strict consistency for availability and performance. Understanding when each applies — and the mechanics of eventual consistency — is essential for modern system design.

---

## Table of Contents

1. [ACID Recap — The Gold Standard](#1-acid-recap)
2. [BASE — The Distributed Alternative](#2-base)
3. [ACID vs BASE — Side-by-Side](#3-acid-vs-base)
4. [Eventual Consistency — Deep Dive](#4-eventual-consistency)
5. [Consistency Models Spectrum](#5-consistency-models-spectrum)
6. [Conflict Detection — Vector Clocks](#6-vector-clocks)
7. [Conflict Resolution Strategies](#7-conflict-resolution)
8. [CRDTs — Conflict-Free Replicated Data Types](#8-crdts)
9. [Read-Your-Writes & Session Guarantees](#9-session-guarantees)
10. [Real-World Systems & Their Consistency Models](#10-real-world-systems)
11. [Worked Examples](#11-worked-examples)
12. [Common Interview Questions](#12-interview-questions)
13. [Key Takeaways](#13-key-takeaways)
14. [Download](#14-download)

---

## 1. ACID Recap — The Gold Standard

<svg viewBox="0 0 750 220" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">ACID Properties</text>

  <!-- Atomicity -->
  <rect x="20" y="50" width="165" height="150" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="102" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#1565c0">Atomicity</text>
  <text x="102" y="100" text-anchor="middle" font-size="10" fill="#333">All or nothing.</text>
  <text x="102" y="118" text-anchor="middle" font-size="10" fill="#333">If any part fails,</text>
  <text x="102" y="134" text-anchor="middle" font-size="10" fill="#333">everything rolls back.</text>
  <text x="102" y="160" text-anchor="middle" font-size="22" fill="#1565c0">⚛</text>
  <text x="102" y="185" text-anchor="middle" font-size="9" fill="#888">Bank transfer: debit +</text>
  <text x="102" y="196" text-anchor="middle" font-size="9" fill="#888">credit both or neither</text>

  <!-- Consistency -->
  <rect x="200" y="50" width="165" height="150" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="282" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#2e7d32">Consistency</text>
  <text x="282" y="100" text-anchor="middle" font-size="10" fill="#333">DB moves from one</text>
  <text x="282" y="118" text-anchor="middle" font-size="10" fill="#333">valid state to another.</text>
  <text x="282" y="134" text-anchor="middle" font-size="10" fill="#333">Constraints enforced.</text>
  <text x="282" y="160" text-anchor="middle" font-size="22" fill="#2e7d32">✓</text>
  <text x="282" y="185" text-anchor="middle" font-size="9" fill="#888">FK, CHECK, UNIQUE</text>
  <text x="282" y="196" text-anchor="middle" font-size="9" fill="#888">all satisfied after txn</text>

  <!-- Isolation -->
  <rect x="380" y="50" width="165" height="150" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="462" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#e65100">Isolation</text>
  <text x="462" y="100" text-anchor="middle" font-size="10" fill="#333">Concurrent transactions</text>
  <text x="462" y="118" text-anchor="middle" font-size="10" fill="#333">don't interfere.</text>
  <text x="462" y="134" text-anchor="middle" font-size="10" fill="#333">Each sees a consistent view.</text>
  <text x="462" y="160" text-anchor="middle" font-size="22" fill="#e65100">⧫</text>
  <text x="462" y="185" text-anchor="middle" font-size="9" fill="#888">SERIALIZABLE or</text>
  <text x="462" y="196" text-anchor="middle" font-size="9" fill="#888">SNAPSHOT isolation</text>

  <!-- Durability -->
  <rect x="560" y="50" width="165" height="150" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="642" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#6a1b9a">Durability</text>
  <text x="642" y="100" text-anchor="middle" font-size="10" fill="#333">Once committed,</text>
  <text x="642" y="118" text-anchor="middle" font-size="10" fill="#333">data survives crashes.</text>
  <text x="642" y="134" text-anchor="middle" font-size="10" fill="#333">Written to stable storage.</text>
  <text x="642" y="160" text-anchor="middle" font-size="22" fill="#6a1b9a">💾</text>
  <text x="642" y="185" text-anchor="middle" font-size="9" fill="#888">WAL, fsync, replication</text>
  <text x="642" y="196" text-anchor="middle" font-size="9" fill="#888">to standby</text>
</svg>

### ACID Systems

| System | Notes |
|---|---|
| **PostgreSQL** | Full ACID, MVCC, SERIALIZABLE isolation |
| **MySQL InnoDB** | Full ACID with redo + undo logs |
| **SQL Server** | Full ACID with snapshot isolation |
| **Oracle** | Full ACID, read-consistency via undo tablespaces |
| **Google Spanner** | Distributed ACID with external consistency |
| **CockroachDB** | Distributed ACID with SERIALIZABLE default |

### The Cost of ACID

- **Latency**: synchronous writes, lock waits, consensus rounds
- **Throughput**: lock contention limits concurrency
- **Availability**: strong consistency requires quorum → minority nodes can't serve
- **Scalability**: distributed transactions (2PC) are expensive and fragile

---

## 2. BASE — The Distributed Alternative

### What Does BASE Stand For?

| Letter | Meaning | Description |
|---|---|---|
| **BA** | **B**asically **A**vailable | The system guarantees availability — it will respond (maybe with stale data) |
| **S** | **S**oft state | The system's state may change over time, even without new input (due to replication propagation) |
| **E** | **E**ventual consistency | If no new updates are made, all replicas will eventually converge to the same state |

### Real-World Analogy

**ACID** is like a bank: every transaction is recorded, balanced, and auditable in real-time. You never see an inconsistent balance.

**BASE** is like a social media post: when you publish a photo, your friend in another country might not see it for a few seconds. It's **eventually** there, but not **instantly** everywhere. That brief delay is acceptable because availability and speed matter more than perfect synchrony.

<svg viewBox="0 0 750 280" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">BASE Properties — Visual</text>

  <!-- Basically Available -->
  <rect x="20" y="50" width="220" height="200" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="130" y="78" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">Basically Available</text>
  <line x1="40" y1="88" x2="220" y2="88" stroke="#c8e6c9"/>

  <text x="130" y="110" text-anchor="middle" font-size="11" fill="#333">System always responds</text>
  <text x="130" y="130" text-anchor="middle" font-size="11" fill="#333">No "service unavailable"</text>

  <rect x="45" y="145" width="170" height="40" rx="6" fill="#c8e6c9"/>
  <text x="130" y="163" text-anchor="middle" font-size="10" fill="#333">Node A: balance = $500</text>
  <text x="130" y="178" text-anchor="middle" font-size="9" fill="#2e7d32">(latest)</text>
  <rect x="45" y="190" width="170" height="40" rx="6" fill="#fff9c4" stroke="#f9a825"/>
  <text x="130" y="208" text-anchor="middle" font-size="10" fill="#333">Node B: balance = $1000</text>
  <text x="130" y="223" text-anchor="middle" font-size="9" fill="#e65100">(stale but available)</text>

  <!-- Soft State -->
  <rect x="265" y="50" width="220" height="200" rx="12" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="375" y="78" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Soft State</text>
  <line x1="285" y1="88" x2="465" y2="88" stroke="#ffe0b2"/>

  <text x="375" y="110" text-anchor="middle" font-size="11" fill="#333">State changes over time</text>
  <text x="375" y="130" text-anchor="middle" font-size="11" fill="#333">without new input</text>

  <text x="375" y="158" text-anchor="middle" font-size="10" fill="#555">t=0  Node B: $1000</text>
  <text x="375" y="178" text-anchor="middle" font-size="14" fill="#e65100">↓ replication</text>
  <text x="375" y="198" text-anchor="middle" font-size="10" fill="#555">t=2s Node B: $500</text>
  <text x="375" y="222" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">State "softened" by async replication</text>

  <!-- Eventual Consistency -->
  <rect x="510" y="50" width="220" height="200" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="620" y="78" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Eventually Consistent</text>
  <line x1="530" y1="88" x2="710" y2="88" stroke="#bbdefb"/>

  <text x="620" y="110" text-anchor="middle" font-size="11" fill="#333">Given enough time,</text>
  <text x="620" y="130" text-anchor="middle" font-size="11" fill="#333">all replicas converge</text>

  <text x="620" y="158" text-anchor="middle" font-size="10" fill="#555">Node A: $500</text>
  <text x="620" y="178" text-anchor="middle" font-size="10" fill="#555">Node B: $500</text>
  <text x="620" y="198" text-anchor="middle" font-size="10" fill="#555">Node C: $500</text>
  <text x="620" y="222" text-anchor="middle" font-size="10" fill="#1565c0" font-weight="bold">All agree (eventually)</text>
</svg>

---

## 3. ACID vs BASE — Side-by-Side

<svg viewBox="0 0 750 440" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">ACID vs BASE — Comprehensive Comparison</text>

  <!-- Headers -->
  <rect x="20" y="48" width="180" height="32" rx="4" fill="#37474f"/>
  <text x="110" y="69" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">Dimension</text>
  <rect x="205" y="48" width="255" height="32" rx="4" fill="#1565c0"/>
  <text x="332" y="69" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">ACID</text>
  <rect x="465" y="48" width="265" height="32" rx="4" fill="#2e7d32"/>
  <text x="597" y="69" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">BASE</text>

  <!-- Row 1 -->
  <rect x="20" y="85" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="107" text-anchor="middle" font-size="10" fill="#333">Consistency model</text>
  <rect x="205" y="85" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="107" text-anchor="middle" font-size="10" fill="#333">Strong (linearisable)</text>
  <rect x="465" y="85" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="107" text-anchor="middle" font-size="10" fill="#333">Eventual</text>

  <!-- Row 2 -->
  <rect x="20" y="120" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="142" text-anchor="middle" font-size="10" fill="#333">Availability during partition</text>
  <rect x="205" y="120" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="142" text-anchor="middle" font-size="10" fill="#c62828">May reject requests (CP)</text>
  <rect x="465" y="120" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="142" text-anchor="middle" font-size="10" fill="#2e7d32">Always responds (AP)</text>

  <!-- Row 3 -->
  <rect x="20" y="155" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="177" text-anchor="middle" font-size="10" fill="#333">Latency</text>
  <rect x="205" y="155" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="177" text-anchor="middle" font-size="10" fill="#333">Higher (sync replication, locks)</text>
  <rect x="465" y="155" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="177" text-anchor="middle" font-size="10" fill="#333">Lower (local reads, async writes)</text>

  <!-- Row 4 -->
  <rect x="20" y="190" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="212" text-anchor="middle" font-size="10" fill="#333">Scalability</text>
  <rect x="205" y="190" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="212" text-anchor="middle" font-size="10" fill="#333">Harder (distributed txns, 2PC)</text>
  <rect x="465" y="190" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="212" text-anchor="middle" font-size="10" fill="#333">Easier (no coordination needed)</text>

  <!-- Row 5 -->
  <rect x="20" y="225" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="247" text-anchor="middle" font-size="10" fill="#333">State management</text>
  <rect x="205" y="225" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="247" text-anchor="middle" font-size="10" fill="#333">Hard state (persisted immediately)</text>
  <rect x="465" y="225" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="247" text-anchor="middle" font-size="10" fill="#333">Soft state (may change over time)</text>

  <!-- Row 6 -->
  <rect x="20" y="260" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="282" text-anchor="middle" font-size="10" fill="#333">Conflict handling</text>
  <rect x="205" y="260" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="282" text-anchor="middle" font-size="10" fill="#333">Prevented (locks, MVCC)</text>
  <rect x="465" y="260" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="282" text-anchor="middle" font-size="10" fill="#333">Detected + resolved after the fact</text>

  <!-- Row 7 -->
  <rect x="20" y="295" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="317" text-anchor="middle" font-size="10" fill="#333">Programming model</text>
  <rect x="205" y="295" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="317" text-anchor="middle" font-size="10" fill="#333">Simple (DB handles correctness)</text>
  <rect x="465" y="295" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="317" text-anchor="middle" font-size="10" fill="#333">Complex (app handles conflicts)</text>

  <!-- Row 8 -->
  <rect x="20" y="330" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="352" text-anchor="middle" font-size="10" fill="#333">Typical systems</text>
  <rect x="205" y="330" width="255" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="332" y="352" text-anchor="middle" font-size="10" fill="#333">PostgreSQL, MySQL, Spanner</text>
  <rect x="465" y="330" width="265" height="35" fill="#e8f5e9" stroke="#ddd"/>
  <text x="597" y="352" text-anchor="middle" font-size="10" fill="#333">Cassandra, DynamoDB, CouchDB</text>

  <!-- Decision bar -->
  <rect x="20" y="380" width="710" height="45" rx="8" fill="#fff8e1" stroke="#f9a825" stroke-width="1.5"/>
  <text x="375" y="400" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">When to Choose</text>
  <text x="375" y="418" text-anchor="middle" font-size="10" fill="#333">ACID: money, inventory, medical records — correctness is non-negotiable  |  BASE: feeds, counters, caches, IoT — speed > precision</text>
</svg>

### It's Not Always One or the Other

Many modern systems offer **tunable consistency**:

- **Cassandra**: `CONSISTENCY ONE` (BASE-like) through `CONSISTENCY ALL` (ACID-like) per query
- **DynamoDB**: Eventually consistent reads (default) or strongly consistent reads (optional)
- **Cosmos DB**: Five levels from Strong to Eventual
- **CockroachDB**: SERIALIZABLE by default, can relax per-read

---

## 4. Eventual Consistency — Deep Dive

### Formal Definition

> A system is **eventually consistent** if, given no further updates, all replicas will eventually return the same value for any given data item.

### The Consistency Window

The time between a write and when all replicas reflect that write is the **consistency window** (or **replication lag**).

<svg viewBox="0 0 750 250" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowEC" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
  </defs>

  <text x="375" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Eventual Consistency — Timeline</text>

  <!-- Timeline axis -->
  <line x1="50" y1="180" x2="700" y2="180" stroke="#666" stroke-width="2" marker-end="url(#arrowEC)"/>
  <text x="710" y="185" font-size="10" fill="#666">time</text>

  <!-- Write event -->
  <line x1="130" y1="180" x2="130" y2="50" stroke="#c62828" stroke-width="2" stroke-dasharray="4,3"/>
  <rect x="90" y="40" width="80" height="25" rx="4" fill="#c62828"/>
  <text x="130" y="57" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">WRITE v2</text>

  <!-- Node A -->
  <rect x="50" y="85" width="60" height="22" rx="3" fill="#ffcdd2"/>
  <text x="80" y="101" text-anchor="middle" font-size="9" fill="#333">A: v1</text>
  <rect x="145" y="85" width="60" height="22" rx="3" fill="#c8e6c9"/>
  <text x="175" y="101" text-anchor="middle" font-size="9" fill="#333">A: v2</text>
  <text x="80" y="78" font-size="8" fill="#888">Node A</text>

  <!-- Node B -->
  <rect x="50" y="118" width="150" height="22" rx="3" fill="#ffcdd2"/>
  <text x="125" y="134" text-anchor="middle" font-size="9" fill="#333">B: v1 (stale)</text>
  <rect x="260" y="118" width="60" height="22" rx="3" fill="#c8e6c9"/>
  <text x="290" y="134" text-anchor="middle" font-size="9" fill="#333">B: v2</text>
  <text x="80" y="113" font-size="8" fill="#888">Node B</text>

  <!-- Node C -->
  <rect x="50" y="150" width="280" height="22" rx="3" fill="#ffcdd2"/>
  <text x="190" y="166" text-anchor="middle" font-size="9" fill="#333">C: v1 (stale)</text>
  <rect x="390" y="150" width="60" height="22" rx="3" fill="#c8e6c9"/>
  <text x="420" y="166" text-anchor="middle" font-size="9" fill="#333">C: v2</text>
  <text x="80" y="145" font-size="8" fill="#888">Node C</text>

  <!-- Consistency window -->
  <rect x="130" y="192" width="260" height="22" rx="4" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="260" y="207" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">Consistency Window</text>

  <!-- Converged -->
  <rect x="395" y="192" width="120" height="22" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="455" y="207" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">All consistent</text>

  <!-- Time markers -->
  <text x="130" y="230" text-anchor="middle" font-size="9" fill="#666">t=0</text>
  <text x="260" y="230" text-anchor="middle" font-size="9" fill="#666">t=200ms</text>
  <text x="395" y="230" text-anchor="middle" font-size="9" fill="#666">t=500ms</text>
</svg>

### How Eventually Consistent Systems Converge

| Mechanism | Description |
|---|---|
| **Anti-entropy (read repair)** | On read, if replicas disagree, update the stale ones |
| **Hinted handoff** | If a node is down during write, another stores a "hint" and forwards it when the node recovers |
| **Merkle trees** | Hash trees that quickly compare data between replicas and identify differences |
| **Gossip protocol** | Nodes periodically exchange state with random peers (epidemic propagation) |
| **Background replication** | Async replication streams push updates to followers |

### Anti-Entropy — Read Repair Example

```
Client reads key K from 3 replicas:
  Node A → v3 (latest)
  Node B → v2 (stale)
  Node C → v3 (latest)

Read repair:
  Return v3 to client
  Asynchronously update Node B: v2 → v3
```

---

## 5. Consistency Models Spectrum

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Consistency Models — From Strongest to Weakest</text>

  <!-- Spectrum bar -->
  <defs>
    <linearGradient id="spectrumGrad" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" style="stop-color:#1565c0"/>
      <stop offset="50%" style="stop-color:#f9a825"/>
      <stop offset="100%" style="stop-color:#c62828"/>
    </linearGradient>
  </defs>
  <rect x="50" y="48" width="650" height="18" rx="9" fill="url(#spectrumGrad)"/>
  <text x="50" y="82" font-size="10" fill="#1565c0" font-weight="bold">Strongest</text>
  <text x="650" y="82" text-anchor="end" font-size="10" fill="#c62828" font-weight="bold">Weakest</text>

  <!-- Linearisable -->
  <line x1="90" y1="66" x2="90" y2="100" stroke="#1565c0" stroke-width="2"/>
  <rect x="20" y="100" width="140" height="70" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="90" y="120" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">Linearisable</text>
  <text x="90" y="136" text-anchor="middle" font-size="9" fill="#333">Reads see latest write.</text>
  <text x="90" y="150" text-anchor="middle" font-size="9" fill="#333">Real-time ordering.</text>
  <text x="90" y="164" text-anchor="middle" font-size="8" fill="#888">Spanner, etcd</text>

  <!-- Sequential -->
  <line x1="230" y1="66" x2="230" y2="100" stroke="#2e7d32" stroke-width="2"/>
  <rect x="165" y="100" width="130" height="70" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="230" y="120" text-anchor="middle" font-size="10" font-weight="bold" fill="#2e7d32">Sequential</text>
  <text x="230" y="136" text-anchor="middle" font-size="9" fill="#333">All nodes see same</text>
  <text x="230" y="150" text-anchor="middle" font-size="9" fill="#333">order of operations.</text>
  <text x="230" y="164" text-anchor="middle" font-size="8" fill="#888">ZooKeeper</text>

  <!-- Causal -->
  <line x1="370" y1="66" x2="370" y2="100" stroke="#f9a825" stroke-width="2"/>
  <rect x="305" y="100" width="130" height="70" rx="8" fill="#fff9c4" stroke="#f9a825" stroke-width="1.5"/>
  <text x="370" y="120" text-anchor="middle" font-size="10" font-weight="bold" fill="#f9a825">Causal</text>
  <text x="370" y="136" text-anchor="middle" font-size="9" fill="#333">Causally related ops</text>
  <text x="370" y="150" text-anchor="middle" font-size="9" fill="#333">seen in order.</text>
  <text x="370" y="164" text-anchor="middle" font-size="8" fill="#888">MongoDB (causal sessions)</text>

  <!-- Session -->
  <line x1="510" y1="66" x2="510" y2="100" stroke="#e65100" stroke-width="2"/>
  <rect x="440" y="100" width="140" height="70" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="510" y="120" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">Session / Read-</text>
  <text x="510" y="134" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">Your-Writes</text>
  <text x="510" y="150" text-anchor="middle" font-size="9" fill="#333">You see your own writes.</text>
  <text x="510" y="164" text-anchor="middle" font-size="8" fill="#888">DynamoDB, Cosmos DB</text>

  <!-- Eventual -->
  <line x1="650" y1="66" x2="650" y2="100" stroke="#c62828" stroke-width="2"/>
  <rect x="585" y="100" width="130" height="70" rx="8" fill="#ffebee" stroke="#c62828" stroke-width="1.5"/>
  <text x="650" y="120" text-anchor="middle" font-size="10" font-weight="bold" fill="#c62828">Eventual</text>
  <text x="650" y="136" text-anchor="middle" font-size="9" fill="#333">Replicas converge</text>
  <text x="650" y="150" text-anchor="middle" font-size="9" fill="#333">"eventually."</text>
  <text x="650" y="164" text-anchor="middle" font-size="8" fill="#888">Cassandra (CL=ONE), DNS</text>

  <!-- Properties bar -->
  <rect x="50" y="195" width="650" height="120" rx="8" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="218" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Definitions</text>
  <text x="70" y="240" font-size="10" fill="#1565c0" font-weight="bold">Linearisable:</text>
  <text x="170" y="240" font-size="10" fill="#333">Every read returns the value of the most recent write. Acts like a single copy.</text>
  <text x="70" y="260" font-size="10" fill="#2e7d32" font-weight="bold">Sequential:</text>
  <text x="155" y="260" font-size="10" fill="#333">All processes see operations in the same total order (but it may not match real-time).</text>
  <text x="70" y="280" font-size="10" fill="#f9a825" font-weight="bold">Causal:</text>
  <text x="125" y="280" font-size="10" fill="#333">If operation A causally precedes B, every process sees A before B. Concurrent ops may differ.</text>
  <text x="70" y="300" font-size="10" fill="#c62828" font-weight="bold">Eventual:</text>
  <text x="140" y="300" font-size="10" fill="#333">No ordering guarantee during updates. Given time, all replicas converge.</text>
</svg>

### Cosmos DB — Five Consistency Levels

Azure Cosmos DB offers the most granular set of consistency levels:

| Level | Guarantee | Latency | Use Case |
|---|---|---|---|
| **Strong** | Linearisable reads | Highest | Financial transactions |
| **Bounded Staleness** | Reads lag by at most K versions or T seconds | High | Leaderboards, stock tickers |
| **Session** | Read-your-writes within a session | Medium | User profile, shopping cart |
| **Consistent Prefix** | Reads never see out-of-order writes | Low | Social feeds, status updates |
| **Eventual** | No ordering guarantee | Lowest | View counters, analytics |

---

## 6. Conflict Detection — Vector Clocks

### The Problem

In a multi-leader or leaderless system, two nodes can independently write to the same key simultaneously. When they sync, which version wins?

### What Is a Vector Clock?

A **vector clock** is a list of `(node, counter)` pairs that tracks the causal history of each version. It lets you determine whether two versions are:
- **Causally ordered** (one happened before the other)
- **Concurrent** (neither caused the other — conflict!)

### How Vector Clocks Work

<svg viewBox="0 0 750 360" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowVC" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#333"/>
    </marker>
  </defs>

  <text x="375" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Vector Clocks — Detecting Concurrent Writes</text>

  <!-- Timeline -->
  <line x1="50" y1="60" x2="700" y2="60" stroke="#666" stroke-width="1.5" marker-end="url(#arrowVC)"/>
  <line x1="50" y1="140" x2="700" y2="140" stroke="#666" stroke-width="1.5" marker-end="url(#arrowVC)"/>
  <line x1="50" y1="220" x2="700" y2="220" stroke="#666" stroke-width="1.5" marker-end="url(#arrowVC)"/>
  <text x="30" y="65" text-anchor="middle" font-size="10" fill="#1565c0" font-weight="bold">A</text>
  <text x="30" y="145" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">B</text>
  <text x="30" y="225" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">C</text>
  <text x="710" y="60" font-size="9" fill="#666">t</text>

  <!-- Event 1: A writes -->
  <circle cx="120" cy="60" r="18" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="120" y="55" text-anchor="middle" font-size="8" fill="#1565c0" font-weight="bold">W1</text>
  <text x="120" y="67" text-anchor="middle" font-size="7" fill="#333">[A:1]</text>

  <!-- Replicate A→B -->
  <line x1="130" y1="78" x2="195" y2="130" stroke="#1565c0" stroke-width="1" stroke-dasharray="4,3" marker-end="url(#arrowVC)"/>

  <!-- Event 2: B writes (based on W1) -->
  <circle cx="210" cy="140" r="18" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="210" y="135" text-anchor="middle" font-size="8" fill="#2e7d32" font-weight="bold">W2</text>
  <text x="210" y="147" text-anchor="middle" font-size="7" fill="#333">[A:1,B:1]</text>

  <!-- Event 3: A writes independently -->
  <circle cx="320" cy="60" r="18" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="320" y="55" text-anchor="middle" font-size="8" fill="#1565c0" font-weight="bold">W3</text>
  <text x="320" y="67" text-anchor="middle" font-size="7" fill="#333">[A:2]</text>

  <!-- Event 4: C writes independently -->
  <circle cx="330" cy="220" r="18" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="330" y="215" text-anchor="middle" font-size="8" fill="#e65100" font-weight="bold">W4</text>
  <text x="330" y="227" text-anchor="middle" font-size="7" fill="#333">[A:1,C:1]</text>

  <!-- Conflict zone -->
  <rect x="290" y="35" width="80" height="210" rx="6" fill="none" stroke="#c62828" stroke-width="2" stroke-dasharray="6,3"/>
  <text x="330" y="268" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">CONCURRENT!</text>
  <text x="330" y="282" text-anchor="middle" font-size="9" fill="#c62828">W3 vs W4: neither</text>
  <text x="330" y="294" text-anchor="middle" font-size="9" fill="#c62828">dominates the other</text>

  <!-- Resolution -->
  <circle cx="510" cy="140" r="22" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="510" y="135" text-anchor="middle" font-size="8" fill="#f9a825" font-weight="bold">MERGE</text>
  <text x="510" y="148" text-anchor="middle" font-size="7" fill="#333">[A:2,B:1,C:1]</text>

  <line x1="338" y1="60" x2="492" y2="130" stroke="#c62828" stroke-width="1" stroke-dasharray="3,3" marker-end="url(#arrowVC)"/>
  <line x1="348" y1="220" x2="495" y2="155" stroke="#c62828" stroke-width="1" stroke-dasharray="3,3" marker-end="url(#arrowVC)"/>
  <line x1="228" y1="140" x2="488" y2="140" stroke="#2e7d32" stroke-width="1" stroke-dasharray="3,3" marker-end="url(#arrowVC)"/>

  <!-- Comparison rules -->
  <rect x="100" y="315" width="550" height="35" rx="6" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="337" text-anchor="middle" font-size="10" fill="#333">V1 ≤ V2 (every entry in V1 ≤ corresponding entry in V2) → V1 happened before V2.  Otherwise → concurrent.</text>
</svg>

### Comparing Vector Clocks

```
V1 = [A:2, B:1]
V2 = [A:1, B:1, C:1]

Compare element by element:
  A: 2 > 1  (V1 > V2 on this component)
  B: 1 = 1  (equal)
  C: 0 < 1  (V1 < V2 on this component)

V1 is NOT ≤ V2 and V2 is NOT ≤ V1 → CONCURRENT (conflict!)
```

### Lamport Timestamps vs Vector Clocks

| | Lamport Timestamp | Vector Clock |
|---|---|---|
| **Size** | Single integer | N integers (one per node) |
| **Can detect causality** | Only ordering, not concurrency | Full causal history |
| **Can detect conflicts** | No | Yes |
| **Scalability** | O(1) space | O(N) space — grows with nodes |
| **Used by** | Google Spanner (TrueTime) | Amazon Dynamo, Riak |

---

## 7. Conflict Resolution Strategies

When concurrent writes are detected (via vector clocks or timestamps), the system must resolve the conflict.

### Strategies

<svg viewBox="0 0 750 330" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Conflict Resolution Strategies</text>

  <!-- LWW -->
  <rect x="20" y="50" width="220" height="130" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="130" y="75" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Last Write Wins (LWW)</text>
  <text x="130" y="95" text-anchor="middle" font-size="10" fill="#333">Highest timestamp wins.</text>
  <text x="130" y="112" text-anchor="middle" font-size="10" fill="#333">Simple but lossy — silent</text>
  <text x="130" y="126" text-anchor="middle" font-size="10" fill="#333">data loss on conflict.</text>
  <text x="130" y="150" text-anchor="middle" font-size="9" fill="#2e7d32" font-weight="bold">Used by: Cassandra</text>
  <text x="130" y="168" text-anchor="middle" font-size="9" fill="#c62828">Risk: clock skew → wrong winner</text>

  <!-- App-level -->
  <rect x="265" y="50" width="220" height="130" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="375" y="75" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Application Resolution</text>
  <text x="375" y="95" text-anchor="middle" font-size="10" fill="#333">Return all versions to app.</text>
  <text x="375" y="112" text-anchor="middle" font-size="10" fill="#333">App (or user) picks winner</text>
  <text x="375" y="126" text-anchor="middle" font-size="10" fill="#333">or merges them.</text>
  <text x="375" y="150" text-anchor="middle" font-size="9" fill="#2e7d32" font-weight="bold">Used by: Riak (siblings)</text>
  <text x="375" y="168" text-anchor="middle" font-size="9" fill="#e65100">Complex but most correct</text>

  <!-- CRDTs -->
  <rect x="510" y="50" width="220" height="130" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="620" y="75" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">CRDTs</text>
  <text x="620" y="95" text-anchor="middle" font-size="10" fill="#333">Data structure that auto-</text>
  <text x="620" y="112" text-anchor="middle" font-size="10" fill="#333">merges without conflicts.</text>
  <text x="620" y="126" text-anchor="middle" font-size="10" fill="#333">Mathematically guaranteed.</text>
  <text x="620" y="150" text-anchor="middle" font-size="9" fill="#2e7d32" font-weight="bold">Used by: Riak, Redis CRDB</text>
  <text x="620" y="168" text-anchor="middle" font-size="9" fill="#e65100">Limited data types</text>

  <!-- Additional strategies -->
  <rect x="20" y="200" width="710" height="110" rx="10" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="225" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Other Strategies</text>

  <text x="50" y="250" font-size="10" fill="#1565c0" font-weight="bold">Read Repair:</text>
  <text x="135" y="250" font-size="10" fill="#333">On read, detect stale replicas and update them. (Cassandra, DynamoDB)</text>

  <text x="50" y="272" font-size="10" fill="#2e7d32" font-weight="bold">Merge Functions:</text>
  <text x="165" y="272" font-size="10" fill="#333">Custom function that combines conflicting values. (CouchDB reduce functions)</text>

  <text x="50" y="294" font-size="10" fill="#e65100" font-weight="bold">Operational Transform:</text>
  <text x="195" y="294" font-size="10" fill="#333">Transform concurrent ops to maintain consistency. (Google Docs, collaborative editing)</text>
</svg>

---

## 8. CRDTs — Conflict-Free Replicated Data Types

### What Are CRDTs?

CRDTs are data structures designed so that **concurrent updates always merge deterministically** — no conflicts, no coordination needed. They achieve **strong eventual consistency**: once all updates propagate, all replicas converge to the **same** state, guaranteed by the mathematical properties of the data structure.

### Types of CRDTs

| CRDT | Type | Operation | Merge Rule | Use Case |
|---|---|---|---|---|
| **G-Counter** | Grow-only counter | Increment | Sum of per-node maxes | View counts, likes |
| **PN-Counter** | Positive-Negative counter | Increment / Decrement | G-Counter pair (P - N) | Inventory adjustments |
| **G-Set** | Grow-only set | Add | Union | Tag collections |
| **OR-Set** | Observed-Remove set | Add / Remove | Track add with unique IDs | Shopping cart items |
| **LWW-Register** | Last-Writer-Wins register | Write | Highest timestamp wins | User profile fields |
| **MV-Register** | Multi-Value register | Write | Keep all concurrent values | Conflict-aware storage |
| **LWW-Element-Set** | Set with timestamps | Add / Remove | Per-element LWW | Feature flags |

### G-Counter Example

<svg viewBox="0 0 700 260" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <text x="350" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">G-Counter CRDT — Grow-Only Counter</text>

  <!-- Node A -->
  <rect x="30" y="50" width="180" height="80" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="120" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#1565c0">Node A</text>
  <text x="120" y="92" text-anchor="middle" font-size="10" fill="#333">Local: {A:3, B:0, C:0}</text>
  <text x="120" y="108" text-anchor="middle" font-size="10" fill="#333">Value = 3 + 0 + 0 = 3</text>
  <text x="120" y="125" text-anchor="middle" font-size="9" fill="#1565c0">Incremented 3 times locally</text>

  <!-- Node B -->
  <rect x="260" y="50" width="180" height="80" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="350" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">Node B</text>
  <text x="350" y="92" text-anchor="middle" font-size="10" fill="#333">Local: {A:0, B:5, C:0}</text>
  <text x="350" y="108" text-anchor="middle" font-size="10" fill="#333">Value = 0 + 5 + 0 = 5</text>
  <text x="350" y="125" text-anchor="middle" font-size="9" fill="#2e7d32">Incremented 5 times locally</text>

  <!-- Node C -->
  <rect x="490" y="50" width="180" height="80" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="580" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#e65100">Node C</text>
  <text x="580" y="92" text-anchor="middle" font-size="10" fill="#333">Local: {A:0, B:0, C:2}</text>
  <text x="580" y="108" text-anchor="middle" font-size="10" fill="#333">Value = 0 + 0 + 2 = 2</text>
  <text x="580" y="125" text-anchor="middle" font-size="9" fill="#e65100">Incremented 2 times locally</text>

  <!-- Merge -->
  <text x="350" y="160" text-anchor="middle" font-size="14" fill="#666">↓ MERGE (max of each entry) ↓</text>

  <rect x="180" y="175" width="340" height="65" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="350" y="198" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">Merged State</text>
  <text x="350" y="218" text-anchor="middle" font-size="11" fill="#333">{A: max(3,0,0), B: max(0,5,0), C: max(0,0,2)} = {A:3, B:5, C:2}</text>
  <text x="350" y="234" text-anchor="middle" font-size="11" fill="#6a1b9a" font-weight="bold">Total = 3 + 5 + 2 = 10</text>
</svg>

### OR-Set Example (Shopping Cart)

```
Node A: add("milk"),  add("eggs")     → {milk@a1, eggs@a2}
Node B: add("milk"),  remove("milk")  → {milk@b1} → remove(milk@b1) → {}

Merge:
  milk@a1 was added on A, never removed → KEEP
  milk@b1 was added on B, then removed → REMOVE
  eggs@a2 was added on A → KEEP

Result: {milk@a1, eggs@a2} = {milk, eggs}
```

The OR-Set tracks unique add IDs, so a remove only removes the specific version it has seen — not versions added concurrently on other nodes.

### Why CRDTs Matter

| Property | Benefit |
|---|---|
| **No coordination** | Nodes can operate independently (offline, partitioned) |
| **Deterministic merge** | Same inputs always produce the same merged state |
| **Strong eventual consistency** | All replicas converge once they've seen the same updates |
| **Conflict-free** | No resolution logic needed — the math handles it |

### Limitations

- Only a limited set of data types (counters, sets, registers, maps)
- Can grow in memory (tombstones for deletes)
- Not suitable for arbitrary business logic (e.g., bank balance can't be a G-Counter because it needs decrements with invariant checking)

---

## 9. Read-Your-Writes & Session Guarantees

### The Problem

In an eventually consistent system, you write to Node A, but your next read hits Node B (which hasn't received the update yet). You see **stale data from your own write** — a terrible user experience.

### Session Consistency Guarantees

| Guarantee | Definition |
|---|---|
| **Read Your Writes** | If you write a value, subsequent reads by you will see that value (or a later one) |
| **Monotonic Reads** | Once you've seen a version, you'll never see an older one |
| **Monotonic Writes** | Your writes are applied in the order you issued them |
| **Writes Follow Reads** | If you read a value then write, your write is based on at least that version |

### Implementation Patterns

```
1. STICKY SESSIONS (route to same node)
   Client → Load Balancer → always Node A
   Pro: simple
   Con: uneven load, failover breaks it

2. VERSION TRACKING (client sends last-seen version)
   Client: "I last saw version 42"
   Node B: "My latest is version 40 — wait or redirect"
   Pro: flexible routing
   Con: client must track state

3. READ FROM LEADER (after writes)
   After write → read from leader for N seconds
   After N seconds → read from any replica
   Pro: strong guarantee
   Con: leader becomes bottleneck for reads

4. CAUSAL TOKENS
   Write returns a token encoding the causal timestamp
   Read sends the token — node waits until it's caught up
   Pro: precise, minimal waiting
   Con: adds complexity to API
```

### PostgreSQL — Synchronous Read-After-Write

```sql
-- Session 1: Write (goes to primary)
INSERT INTO posts(user_id, content) VALUES (42, 'Hello world!');

-- Session 1: Subsequent read (also from primary to guarantee freshness)
-- With synchronous_commit = on (default), the write is durable
SELECT * FROM posts WHERE user_id = 42 ORDER BY created_at DESC LIMIT 1;

-- For reads that DON'T need freshness (e.g., timeline feed):
-- Route to read replica (async, might lag by milliseconds)
```

---

## 10. Real-World Systems & Their Consistency Models

| System | Default Consistency | Tunable? | Conflict Resolution |
|---|---|---|---|
| **PostgreSQL** | Strong (SERIALIZABLE) | Isolation levels | Locks / MVCC — no conflicts |
| **MySQL** | Strong (REPEATABLE READ) | Isolation levels | Locks / MVCC — no conflicts |
| **Google Spanner** | Linearisable | No — always strong | TrueTime + Paxos |
| **CockroachDB** | Serialisable | Can weaken for reads | Raft — no conflicts |
| **MongoDB** | Causal (sessions) | Read/write concern | Leader election, retryable writes |
| **Cassandra** | Eventual (ONE) | Per-query (ONE→ALL) | LWW timestamps |
| **DynamoDB** | Eventual | Strong reads optional | LWW or app-level |
| **CouchDB** | Eventual | No | Revision tree, app resolves |
| **Riak** | Eventual | Quorum tunable | Vector clocks, CRDTs, siblings |
| **Redis** | Eventual (async replication) | WAIT command | LWW |
| **Cosmos DB** | Session (default) | 5 levels | Depends on level |

### Consistency vs Latency Trade-Off (Measured)

| System | Consistency | Write Latency (p99) | Read Latency (p99) |
|---|---|---|---|
| PostgreSQL (local) | Strong | 2 ms | 1 ms |
| Spanner (multi-region) | Linearisable | 15 ms | 5 ms |
| CockroachDB (multi-region) | Serialisable | 20 ms | 8 ms |
| Cassandra (QUORUM) | Strong | 5 ms | 3 ms |
| Cassandra (ONE) | Eventual | 1 ms | 0.5 ms |
| DynamoDB (eventual) | Eventual | 5 ms | 1 ms |
| DynamoDB (strong) | Strong | 5 ms | 3 ms |

---

## 11. Worked Examples

### Example 1 — Beginner: ACID vs BASE Decision for an Online Store

```
Scenario: Building an e-commerce platform.

ACID needed for:
├── Order placement (inventory decremented atomically with order creation)
├── Payment processing (charge + record must be atomic)
└── Inventory count (must be accurate to prevent overselling)

BASE acceptable for:
├── Product catalog browsing (1-second stale data is fine)
├── Product reviews & ratings (eventual consistency OK)
├── User activity feed (delays acceptable)
├── Search index (seconds behind is fine)
└── Recommendation engine (uses historical data anyway)

Architecture:
┌─────────────────┐      ┌─────────────────┐
│   PostgreSQL     │      │   Cassandra      │
│   (ACID)         │      │   (BASE)         │
│                  │      │                  │
│ • orders         │      │ • product_views  │
│ • payments       │ ───► │ • reviews        │
│ • inventory      │ CDC  │ • search_index   │
│ • user_accounts  │      │ • recommendations│
└─────────────────┘      └─────────────────┘

CDC (Change Data Capture) streams updates
from ACID store to BASE store asynchronously.
```

### Example 2 — Beginner: Observing Eventual Consistency

```sql
-- Cassandra: demonstrate consistency levels

-- Write with QUORUM (strong write)
CONSISTENCY QUORUM;
INSERT INTO user_profiles (user_id, name, bio)
VALUES (42, 'Alice', 'Software engineer')
USING TIMESTAMP 1714000000000;

-- Immediate read with ONE (may be stale)
CONSISTENCY ONE;
SELECT * FROM user_profiles WHERE user_id = 42;
-- Might return old bio if this node hasn't received the update yet

-- Read with QUORUM (guaranteed to see latest)
CONSISTENCY QUORUM;
SELECT * FROM user_profiles WHERE user_id = 42;
-- Always returns 'Software engineer' (overlap with write quorum)

-- Read with ALL (wait for every replica)
CONSISTENCY ALL;
SELECT * FROM user_profiles WHERE user_id = 42;
-- Slowest but reads from all replicas
```

### Example 3 — Intermediate: Implementing a G-Counter

```python
class GCounter:
    """Grow-only Counter CRDT."""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.counts: dict[str, int] = {}

    def increment(self):
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + 1

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: 'GCounter'):
        """Merge by taking the max of each node's count."""
        all_nodes = set(self.counts) | set(other.counts)
        for node in all_nodes:
            self.counts[node] = max(
                self.counts.get(node, 0),
                other.counts.get(node, 0)
            )

# Simulation
counter_a = GCounter("A")
counter_b = GCounter("B")

counter_a.increment()  # A:1
counter_a.increment()  # A:2
counter_a.increment()  # A:3

counter_b.increment()  # B:1
counter_b.increment()  # B:2

print(counter_a.value())  # 3
print(counter_b.value())  # 2

# Merge
counter_a.merge(counter_b)
print(counter_a.value())  # 5  (A:3 + B:2)

counter_b.merge(counter_a)
print(counter_b.value())  # 5  (A:3 + B:2) — same result regardless of order
```

### Example 4 — Intermediate: PN-Counter (Increment and Decrement)

```python
class PNCounter:
    """Positive-Negative Counter CRDT."""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.p = {}  # positive counts (increments)
        self.n = {}  # negative counts (decrements)

    def increment(self):
        self.p[self.node_id] = self.p.get(self.node_id, 0) + 1

    def decrement(self):
        self.n[self.node_id] = self.n.get(self.node_id, 0) + 1

    def value(self) -> int:
        return sum(self.p.values()) - sum(self.n.values())

    def merge(self, other: 'PNCounter'):
        for node in set(self.p) | set(other.p):
            self.p[node] = max(self.p.get(node, 0), other.p.get(node, 0))
        for node in set(self.n) | set(other.n):
            self.n[node] = max(self.n.get(node, 0), other.n.get(node, 0))

# Stock counter example
stock_a = PNCounter("warehouse_a")
stock_b = PNCounter("warehouse_b")

stock_a.increment()    # +1 (received item)
stock_a.increment()    # +1
stock_b.decrement()    # -1 (sold item)

stock_a.merge(stock_b)
print(stock_a.value())  # 2 - 1 = 1
```

### Example 5 — Intermediate: Vector Clock Comparison

```python
class VectorClock:
    def __init__(self):
        self.clock: dict[str, int] = {}

    def increment(self, node_id: str):
        self.clock[node_id] = self.clock.get(node_id, 0) + 1

    def merge(self, other: 'VectorClock') -> 'VectorClock':
        merged = VectorClock()
        all_nodes = set(self.clock) | set(other.clock)
        for node in all_nodes:
            merged.clock[node] = max(
                self.clock.get(node, 0),
                other.clock.get(node, 0)
            )
        return merged

    def __le__(self, other: 'VectorClock') -> bool:
        """True if self happened-before or equals other."""
        for node, count in self.clock.items():
            if count > other.clock.get(node, 0):
                return False
        return True

    def is_concurrent(self, other: 'VectorClock') -> bool:
        return not (self <= other) and not (other <= self)

# Example: two writers update concurrently
vc_a = VectorClock()
vc_a.increment("A")   # [A:1]
vc_a.increment("A")   # [A:2]

vc_b = VectorClock()
vc_b.increment("A")   # [A:1] (started from same state as vc_a at [A:1])
vc_b.increment("B")   # [A:1, B:1]

print(vc_a <= vc_b)              # False (A:2 > A:1)
print(vc_b <= vc_a)              # False (B:1 > B:0)
print(vc_a.is_concurrent(vc_b))  # True — CONFLICT!

# Resolve by merging
merged = vc_a.merge(vc_b)
print(merged.clock)              # {A: 2, B: 1}
```

### Example 6 — Advanced: Read-Your-Writes with Version Tokens

```python
# Application-level read-your-writes guarantee

class VersionedStore:
    """Simulates a multi-node store with version tracking."""

    def __init__(self):
        self.nodes = {
            "node_a": {"data": {}, "version": 0},
            "node_b": {"data": {}, "version": 0},
        }

    def write(self, key: str, value: str, node: str = "node_a") -> int:
        """Write to a specific node, return version token."""
        self.nodes[node]["version"] += 1
        version = self.nodes[node]["version"]
        self.nodes[node]["data"][key] = (value, version)
        return version  # client stores this token

    def read(self, key: str, min_version: int = 0) -> tuple:
        """Read from any node that has at least min_version."""
        for node_name, node in self.nodes.items():
            if node["version"] >= min_version and key in node["data"]:
                return node["data"][key]

        # No node is caught up — either wait or read from primary
        primary = self.nodes["node_a"]
        return primary["data"].get(key, (None, 0))

# Usage
store = VersionedStore()

# Client writes and gets a version token
write_version = store.write("profile:42", "Alice - Engineer")  # v=1

# Client includes the version token on next read
# This ensures they read from a node that has at least version 1
result = store.read("profile:42", min_version=write_version)
print(result)  # ('Alice - Engineer', 1) — guaranteed fresh
```

---

## 12. Common Interview Questions

### Q1: What does BASE stand for and how does it differ from ACID?

**Answer:** BASE = **B**asically **A**vailable, **S**oft state, **E**ventually consistent. Unlike ACID (which guarantees strong consistency, atomicity, and isolation), BASE prioritises **availability** — the system always responds, even if data may be temporarily stale. In BASE, state can change without new input (soft state due to replication), and all replicas converge given time (eventual consistency). ACID uses locks/MVCC to prevent conflicts; BASE detects and resolves them after the fact.

### Q2: What is eventual consistency?

**Answer:** A consistency model where, if no new updates are made to a data item, all replicas will **eventually** return the same value. The "consistency window" — the time between a write and full propagation — varies from milliseconds to seconds. Convergence happens via mechanisms like anti-entropy (read repair), hinted handoff, gossip protocol, and background replication. It's the consistency model used by Cassandra (at CL=ONE), DynamoDB (default reads), and DNS.

### Q3: What is a vector clock and how does it detect conflicts?

**Answer:** A vector clock is a list of `(node, counter)` pairs that tracks the causal history of a data version. Each node increments its own counter on every write. To compare two versions V1 and V2: if every entry in V1 ≤ the corresponding entry in V2, then V1 **happened before** V2 (no conflict). If neither dominates the other, the writes are **concurrent** — a conflict that needs resolution. Unlike simple timestamps, vector clocks detect true concurrency without relying on synchronised clocks.

### Q4: What are CRDTs?

**Answer:** **Conflict-Free Replicated Data Types** are data structures that can be independently updated on different nodes and always merge deterministically without conflicts. They achieve **strong eventual consistency** — once all updates propagate, all replicas converge to the same state, guaranteed by mathematical properties (commutativity, associativity, idempotence of the merge operation). Examples: G-Counter (grow-only counter), PN-Counter, OR-Set (add/remove set), LWW-Register. Used by Riak, Redis Enterprise, and collaborative editing tools.

### Q5: When would you choose BASE over ACID?

**Answer:** Choose BASE when:
- **Availability** is more important than instant correctness (social feeds, recommendation engines)
- **Latency** requirements are strict (sub-millisecond reads from local replica)
- **Scale** requires many write nodes without coordination (IoT sensor ingestion)
- Brief **staleness** doesn't cause business harm (view counters, activity logs)

Stick with ACID when correctness is non-negotiable (financial transactions, inventory, medical records).

### Q6: What is the difference between strong consistency and linearisability?

**Answer:** **Strong consistency** is an informal term meaning "reads always see the latest write." **Linearisability** is the formal, strongest version: every operation appears to take effect at a single point in time between its start and completion, and this ordering respects real-time order. Linearisability implies: if write W completes before read R starts (wall-clock time), R must see W's value. Spanner achieves this with TrueTime; most "strongly consistent" systems actually provide serializability or sequential consistency, which are slightly weaker.

### Q7: What is Last-Write-Wins (LWW) and what are its problems?

**Answer:** LWW resolves conflicts by keeping the write with the highest timestamp. Problems:
1. **Clock skew**: If Node A's clock is 5 seconds ahead, A's writes always win
2. **Silent data loss**: Concurrent writes on Node B are silently discarded
3. **No merge**: If two users add different items to a cart simultaneously, one is lost

LWW is used by Cassandra because it's simple and fast. For data where losing concurrent writes is unacceptable, use CRDTs (OR-Set for carts) or application-level conflict resolution.

### Q8: Explain read-your-writes consistency.

**Answer:** A session guarantee where: if a client writes a value, subsequent reads by **that same client** will always see that value (or a newer one). Without this, a user could post a comment then refresh and not see it — confusing UX. Implementation: sticky sessions (always read from the same node), version tokens (client sends last-write-version, node waits until caught up), or read-from-leader after writes.

### Q9: What is anti-entropy and how does read repair work?

**Answer:** Anti-entropy is the process of reconciling data across replicas to ensure convergence. **Read repair** is one mechanism: when a coordinator reads from multiple replicas and detects a stale version (via timestamps or vector clocks), it asynchronously updates the stale replica with the latest version. Cassandra and DynamoDB use read repair. **Merkle trees** are another mechanism: hash trees that efficiently compare large datasets between replicas to find differences, used by Cassandra's anti-entropy repair process.

### Q10: Can you have ACID in a distributed system?

**Answer:** Yes, but with trade-offs. Google Spanner and CockroachDB provide distributed ACID transactions with external consistency / serializability. The cost is **latency** (synchronous replication, consensus rounds, TrueTime wait-out in Spanner) and **availability** (minority partitions can't serve writes). Spanner's innovation was showing that with GPS+atomic clocks, the consistency cost can be bounded to ~7 ms. For many applications, this is acceptable — "distributed ACID" is no longer theoretical.

### Q11: What is the difference between causal consistency and eventual consistency?

**Answer:** **Eventual consistency** makes no ordering guarantees — replicas converge eventually but may see operations in different orders during the consistency window. **Causal consistency** preserves cause-and-effect relationships: if operation A causally affects operation B (e.g., A is a write, B reads A's result), then every replica sees A before B. Concurrent (unrelated) operations may still be seen in different orders. Causal consistency is stronger than eventual but weaker than linearisable, and it's achievable without sacrificing availability (unlike linearisability). MongoDB offers causal sessions.

### Q12: How do you test for eventual consistency issues in production?

**Answer:**
1. **Read-after-write tests**: Write a unique value, immediately read from a different node, measure the lag
2. **Consistency checker**: Background job that compares a sample of keys across replicas
3. **Stale-read monitoring**: Log the version of data served; alert if it's older than threshold
4. **Chaos testing**: Inject network partitions (e.g., with Toxiproxy) and verify convergence after healing
5. **Vector clock analysis**: Track sibling counts in Riak — high sibling counts indicate unresolved conflicts
6. **Client-side assertion**: Application checks that read version ≥ expected version; logs violations

---

## 13. Key Takeaways

<svg viewBox="0 0 750 520" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">Phase 18 — Key Takeaways</text>

  <!-- Card 1 -->
  <rect x="20" y="50" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="40" y="75" font-size="13" font-weight="bold" fill="#1565c0">ACID = Correctness First</text>
  <text x="40" y="95" font-size="11" fill="#333">Strong consistency, atomicity, isolation.</text>
  <text x="40" y="112" font-size="11" fill="#333">Costs: latency, availability, scalability.</text>

  <!-- Card 2 -->
  <rect x="390" y="50" width="340" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="410" y="75" font-size="13" font-weight="bold" fill="#2e7d32">BASE = Availability First</text>
  <text x="410" y="95" font-size="11" fill="#333">Eventually consistent, soft state.</text>
  <text x="410" y="112" font-size="11" fill="#333">Benefits: speed, scale, fault tolerance.</text>

  <!-- Card 3 -->
  <rect x="20" y="150" width="340" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="40" y="175" font-size="13" font-weight="bold" fill="#e65100">Eventual ≠ Never</text>
  <text x="40" y="195" font-size="11" fill="#333">Replicas converge in ms to seconds.</text>
  <text x="40" y="212" font-size="11" fill="#333">Anti-entropy, read repair, gossip.</text>

  <!-- Card 4 -->
  <rect x="390" y="150" width="340" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="410" y="175" font-size="13" font-weight="bold" fill="#6a1b9a">Vector Clocks Detect Conflicts</text>
  <text x="410" y="195" font-size="11" fill="#333">Compare causal history per-node.</text>
  <text x="410" y="212" font-size="11" fill="#333">Concurrent = conflict. Ordered = safe.</text>

  <!-- Card 5 -->
  <rect x="20" y="250" width="340" height="80" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="40" y="275" font-size="13" font-weight="bold" fill="#c62828">LWW Is Simple but Lossy</text>
  <text x="40" y="295" font-size="11" fill="#333">Last timestamp wins; concurrent</text>
  <text x="40" y="312" font-size="11" fill="#333">writes silently lost. Clock skew risk.</text>

  <!-- Card 6 -->
  <rect x="390" y="250" width="340" height="80" rx="10" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="410" y="275" font-size="13" font-weight="bold" fill="#f9a825">CRDTs Auto-Merge</text>
  <text x="410" y="295" font-size="11" fill="#333">Math guarantees convergence.</text>
  <text x="410" y="312" font-size="11" fill="#333">Counters, sets, registers — no conflicts.</text>

  <!-- Card 7 -->
  <rect x="20" y="350" width="340" height="80" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="40" y="375" font-size="13" font-weight="bold" fill="#00695c">Read-Your-Writes Matters</text>
  <text x="40" y="395" font-size="11" fill="#333">Users must see their own updates.</text>
  <text x="40" y="412" font-size="11" fill="#333">Sticky sessions or version tokens.</text>

  <!-- Card 8 -->
  <rect x="390" y="350" width="340" height="80" rx="10" fill="#ffccbc" stroke="#bf360c" stroke-width="2"/>
  <text x="410" y="375" font-size="13" font-weight="bold" fill="#bf360c">Most Systems Are Hybrid</text>
  <text x="410" y="395" font-size="11" fill="#333">ACID for money + BASE for feeds.</text>
  <text x="410" y="412" font-size="11" fill="#333">CDC bridges the two worlds.</text>

  <!-- Decision guide -->
  <rect x="80" y="455" width="590" height="50" rx="10" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="375" y="478" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Decision Guide</text>
  <text x="375" y="496" text-anchor="middle" font-size="10" fill="#555">Wrong data = real harm? → ACID  |  Stale data = minor annoyance? → BASE  |  Both domains? → Hybrid</text>
</svg>

---

## 14. Download

<a href="18_base_vs_acid.md" download="18_base_vs_acid.md" style="display:inline-block;padding:14px 28px;background:linear-gradient(135deg,#2e7d32,#1b5e20);color:#fff;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;box-shadow:0 4px 15px rgba(46,125,50,0.3);margin:10px 0;">Download 18_base_vs_acid.md</a>
