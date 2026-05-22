# Phase 17 — CAP Theorem & Distributed Databases

> **When a database spans multiple machines**, new trade-offs emerge that don't exist on a single server. The CAP theorem formalises the fundamental impossibility, and distributed systems engineering is the art of working within that constraint.

---

## Table of Contents

1. [Why Distribute a Database?](#1-why-distribute-a-database)
2. [CAP Theorem — The Fundamental Trade-Off](#2-cap-theorem)
3. [CAP Triangle — Visual Breakdown](#3-cap-triangle)
4. [CP, AP, and CA Systems — Real Examples](#4-cp-ap-ca-systems)
5. [PACELC Theorem — CAP Extended](#5-pacelc-theorem)
6. [Replication — Keeping Copies in Sync](#6-replication)
7. [Sharding — Splitting Data Across Nodes](#7-sharding)
8. [Consensus Algorithms — Paxos & Raft](#8-consensus-algorithms)
9. [Distributed Transactions — 2PC & 3PC](#9-distributed-transactions)
10. [Real-World Architectures](#10-real-world-architectures)
11. [Worked Examples](#11-worked-examples)
12. [Common Interview Questions](#12-interview-questions)
13. [Key Takeaways](#13-key-takeaways)
14. [Download](#14-download)

---

## 1. Why Distribute a Database?

### Single-Node Limits

| Limit | Problem |
|---|---|
| **Vertical scaling ceiling** | You can only buy so much RAM / CPU for one machine |
| **Single point of failure** | One server crash = total downtime |
| **Geographic latency** | Users in Tokyo hitting a server in Virginia = 200 ms RTT |
| **Read throughput** | One machine can serve only so many concurrent reads |
| **Storage capacity** | A single disk or SAN has finite space |

### What Distribution Gives You

| Benefit | Mechanism |
|---|---|
| **Horizontal scalability** | Add machines instead of buying bigger ones |
| **High availability** | If node A fails, node B takes over |
| **Low latency** | Place replicas near users (edge / multi-region) |
| **Read scalability** | Spread reads across replicas |
| **Storage scalability** | Shard data across many machines |

### Real-World Analogy

A single-node database is like a single-branch bank. It works fine for a small town, but if you're a national bank you need branches everywhere (replicas for reads), regional vaults (shards for storage), and a headquarters that keeps the books consistent (consensus). The CAP theorem says: during a network partition between branches, you can either **lock the branch** (consistency) or **keep serving customers with possibly stale info** (availability) — but not both.

---

## 2. CAP Theorem — The Fundamental Trade-Off

### Statement (Brewer's Theorem, 2000)

> In a distributed data store, it is impossible to simultaneously guarantee all three of:
> - **C**onsistency — every read receives the most recent write
> - **A**vailability — every request receives a (non-error) response
> - **P**artition tolerance — the system continues to operate despite network partitions
>
> During a network partition, you must choose between **C** and **A**.

### Formal Definitions

| Property | Definition | Analogy |
|---|---|---|
| **Consistency** | All nodes see the same data at the same time. A read after a write returns the written value, regardless of which node you ask. | Every bank branch shows the same account balance |
| **Availability** | Every request to a non-failed node returns a response (not an error), though the data may not be the latest. | Every branch answers the phone, even if their balance sheet is slightly out of date |
| **Partition Tolerance** | The system continues to operate even when network messages between nodes are lost or delayed. | Branches keep working even when the phone lines between them go down |

### Why You Can't Have All Three

During a **network partition** (P), nodes on different sides of the partition cannot communicate:

- If you choose **Consistency**: nodes on the minority side must stop serving requests (they can't confirm with the majority), sacrificing **Availability**.
- If you choose **Availability**: all nodes keep serving requests, but they might return stale data, sacrificing **Consistency**.
- You **cannot choose to avoid Partitions** in a distributed system — networks fail. P is not optional.

<svg viewBox="0 0 750 400" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowCAP" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#c62828"/>
    </marker>
  </defs>

  <text x="375" y="28" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Why You Must Choose — Partition Scenario</text>

  <!-- Node A -->
  <rect x="50" y="70" width="180" height="100" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="140" y="95" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Node A</text>
  <text x="140" y="115" text-anchor="middle" font-size="11" fill="#333">balance = $1000</text>
  <text x="140" y="135" text-anchor="middle" font-size="10" fill="#555">Client writes: balance = $500</text>
  <text x="140" y="155" text-anchor="middle" font-size="10" fill="#1565c0" font-weight="bold">Updated to $500 ✓</text>

  <!-- Network partition -->
  <line x1="260" y1="120" x2="440" y2="120" stroke="#c62828" stroke-width="3" stroke-dasharray="10,6"/>
  <text x="350" y="110" text-anchor="middle" font-size="12" font-weight="bold" fill="#c62828">NETWORK PARTITION</text>
  <text x="350" y="140" text-anchor="middle" font-size="20" fill="#c62828">✕</text>

  <!-- Node B -->
  <rect x="470" y="70" width="180" height="100" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="560" y="95" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">Node B</text>
  <text x="560" y="115" text-anchor="middle" font-size="11" fill="#333">balance = $1000</text>
  <text x="560" y="135" text-anchor="middle" font-size="10" fill="#555">Client reads: balance = ?</text>

  <!-- Choice CP -->
  <rect x="50" y="210" width="280" height="100" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="190" y="235" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Choice CP — Consistency</text>
  <text x="190" y="258" text-anchor="middle" font-size="11" fill="#333">Node B: "I can't reach Node A.</text>
  <text x="190" y="275" text-anchor="middle" font-size="11" fill="#333">I might have stale data."</text>
  <text x="190" y="295" text-anchor="middle" font-size="11" fill="#c62828" font-weight="bold">→ Returns ERROR (unavailable)</text>

  <!-- Choice AP -->
  <rect x="400" y="210" width="300" height="100" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="550" y="235" text-anchor="middle" font-size="13" font-weight="bold" fill="#6a1b9a">Choice AP — Availability</text>
  <text x="550" y="258" text-anchor="middle" font-size="11" fill="#333">Node B: "I'll respond with what</text>
  <text x="550" y="275" text-anchor="middle" font-size="11" fill="#333">I have — might be stale."</text>
  <text x="550" y="295" text-anchor="middle" font-size="11" fill="#6a1b9a" font-weight="bold">→ Returns $1000 (wrong but available)</text>

  <!-- Bottom -->
  <rect x="100" y="340" width="500" height="40" rx="8" fill="#ffebee" stroke="#c62828" stroke-width="1.5"/>
  <text x="350" y="365" text-anchor="middle" font-size="12" fill="#c62828" font-weight="bold">During a partition, you MUST choose one. Both correct + available is impossible.</text>
</svg>

---

## 3. CAP Triangle — Visual Breakdown

<svg viewBox="0 0 750 480" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">CAP Theorem Triangle</text>

  <!-- Triangle -->
  <polygon points="375,70 120,370 630,370" fill="none" stroke="#37474f" stroke-width="3"/>

  <!-- C vertex -->
  <circle cx="375" cy="70" r="35" fill="#e3f2fd" stroke="#1565c0" stroke-width="3"/>
  <text x="375" y="65" text-anchor="middle" font-size="20" font-weight="bold" fill="#1565c0">C</text>
  <text x="375" y="82" text-anchor="middle" font-size="9" fill="#1565c0">Consistency</text>

  <!-- A vertex -->
  <circle cx="120" cy="370" r="35" fill="#e8f5e9" stroke="#2e7d32" stroke-width="3"/>
  <text x="120" y="365" text-anchor="middle" font-size="20" font-weight="bold" fill="#2e7d32">A</text>
  <text x="120" y="382" text-anchor="middle" font-size="9" fill="#2e7d32">Availability</text>

  <!-- P vertex -->
  <circle cx="630" cy="370" r="35" fill="#fff3e0" stroke="#e65100" stroke-width="3"/>
  <text x="630" y="365" text-anchor="middle" font-size="20" font-weight="bold" fill="#e65100">P</text>
  <text x="630" y="382" text-anchor="middle" font-size="9" fill="#e65100">Partition Tol.</text>

  <!-- CA edge -->
  <text x="210" y="200" text-anchor="middle" font-size="13" font-weight="bold" fill="#6a1b9a">CA</text>
  <text x="210" y="218" text-anchor="middle" font-size="10" fill="#555">Single-node or same-</text>
  <text x="210" y="232" text-anchor="middle" font-size="10" fill="#555">LAN systems only</text>
  <text x="210" y="250" text-anchor="middle" font-size="9" fill="#888">PostgreSQL, MySQL (single)</text>
  <text x="210" y="264" text-anchor="middle" font-size="9" fill="#888">Oracle RAC</text>

  <!-- CP edge -->
  <text x="540" y="200" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">CP</text>
  <text x="540" y="218" text-anchor="middle" font-size="10" fill="#555">Consistent but may</text>
  <text x="540" y="232" text-anchor="middle" font-size="10" fill="#555">reject requests</text>
  <text x="540" y="250" text-anchor="middle" font-size="9" fill="#888">HBase, MongoDB (default),</text>
  <text x="540" y="264" text-anchor="middle" font-size="9" fill="#888">Spanner, CockroachDB, etcd</text>

  <!-- AP edge -->
  <text x="375" y="400" text-anchor="middle" font-size="13" font-weight="bold" fill="#00695c">AP</text>
  <text x="375" y="418" text-anchor="middle" font-size="10" fill="#555">Available but may return stale data</text>
  <text x="375" y="436" text-anchor="middle" font-size="9" fill="#888">Cassandra, DynamoDB, CouchDB, Riak</text>

  <!-- Note -->
  <rect x="170" y="455" width="410" height="22" rx="4" fill="#fff8e1" stroke="#f9a825" stroke-width="1"/>
  <text x="375" y="471" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">P is mandatory in distributed systems → real choice is between CP and AP</text>
</svg>

---

## 4. CP, AP, and CA Systems — Real Examples

### CP Systems (Consistency + Partition Tolerance)

During a partition, some nodes become **unavailable** to maintain consistency.

| System | How It Achieves CP |
|---|---|
| **Google Spanner** | TrueTime + Paxos consensus; linearizable reads; nodes stall if they lose quorum |
| **CockroachDB** | Raft consensus per range; minority partitions stop serving writes |
| **HBase** | Single RegionServer per region; if it fails, region is unavailable until failover |
| **etcd / ZooKeeper** | Raft/ZAB consensus; minority nodes reject writes |
| **MongoDB** (default) | Writes go to primary; if primary is partitioned, no writes until new election |

**Best for:** Financial transactions, inventory counts, anything where stale data = incorrect behaviour.

### AP Systems (Availability + Partition Tolerance)

During a partition, all nodes keep serving requests, but may return **stale or conflicting** data.

| System | How It Achieves AP |
|---|---|
| **Cassandra** | Tunable consistency (ONE/QUORUM/ALL); at ONE, always available |
| **Amazon DynamoDB** | Eventually consistent reads by default; every node accepts writes |
| **CouchDB** | Multi-master replication; conflicts detected and resolved later |
| **Riak** | Vector clocks for conflict detection; read-repair for convergence |

**Best for:** Social media feeds, shopping carts, analytics counters — where a brief stale read is acceptable.

### CA Systems (Consistency + Availability)

Only possible when there are **no network partitions** — essentially single-node or tightly-coupled clusters on the same LAN.

| System | Why CA |
|---|---|
| **PostgreSQL (single node)** | No network partition possible on one machine |
| **MySQL (single node)** | Same — one machine, full ACID |
| **Oracle RAC** | Shared-disk architecture over InfiniBand — treats partition as node failure |

> **Important nuance:** In a truly distributed system (nodes connected by network), partitions *will* happen. CA is a theoretical category that only applies to systems that don't face partitions.

### Comparison Table

<svg viewBox="0 0 750 310" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">CP vs AP — Behaviour During Partition</text>

  <!-- Header -->
  <rect x="30" y="48" width="160" height="32" rx="4" fill="#37474f"/>
  <text x="110" y="69" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">Scenario</text>
  <rect x="195" y="48" width="250" height="32" rx="4" fill="#1565c0"/>
  <text x="320" y="69" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">CP System (e.g., Spanner)</text>
  <rect x="450" y="48" width="270" height="32" rx="4" fill="#2e7d32"/>
  <text x="585" y="69" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">AP System (e.g., Cassandra)</text>

  <!-- Row 1 -->
  <rect x="30" y="85" width="160" height="40" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="110" text-anchor="middle" font-size="10" fill="#333">Write during partition</text>
  <rect x="195" y="85" width="250" height="40" fill="#e3f2fd" stroke="#ddd"/>
  <text x="320" y="105" text-anchor="middle" font-size="10" fill="#333">Rejected on minority side</text>
  <text x="320" y="118" text-anchor="middle" font-size="9" fill="#c62828">(unavailable)</text>
  <rect x="450" y="85" width="270" height="40" fill="#e8f5e9" stroke="#ddd"/>
  <text x="585" y="105" text-anchor="middle" font-size="10" fill="#333">Accepted on both sides</text>
  <text x="585" y="118" text-anchor="middle" font-size="9" fill="#e65100">(may conflict)</text>

  <!-- Row 2 -->
  <rect x="30" y="125" width="160" height="40" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="150" text-anchor="middle" font-size="10" fill="#333">Read during partition</text>
  <rect x="195" y="125" width="250" height="40" fill="#e3f2fd" stroke="#ddd"/>
  <text x="320" y="145" text-anchor="middle" font-size="10" fill="#333">Returns latest or error</text>
  <text x="320" y="158" text-anchor="middle" font-size="9" fill="#2e7d32">(always correct if returned)</text>
  <rect x="450" y="125" width="270" height="40" fill="#e8f5e9" stroke="#ddd"/>
  <text x="585" y="145" text-anchor="middle" font-size="10" fill="#333">Always returns data</text>
  <text x="585" y="158" text-anchor="middle" font-size="9" fill="#e65100">(may be stale)</text>

  <!-- Row 3 -->
  <rect x="30" y="165" width="160" height="40" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="190" text-anchor="middle" font-size="10" fill="#333">After partition heals</text>
  <rect x="195" y="165" width="250" height="40" fill="#e3f2fd" stroke="#ddd"/>
  <text x="320" y="185" text-anchor="middle" font-size="10" fill="#333">Nodes resync; no conflicts</text>
  <text x="320" y="198" text-anchor="middle" font-size="9" fill="#2e7d32">(data was always consistent)</text>
  <rect x="450" y="165" width="270" height="40" fill="#e8f5e9" stroke="#ddd"/>
  <text x="585" y="185" text-anchor="middle" font-size="10" fill="#333">Conflict resolution needed</text>
  <text x="585" y="198" text-anchor="middle" font-size="9" fill="#e65100">(LWW, vector clocks, merge)</text>

  <!-- Row 4 -->
  <rect x="30" y="205" width="160" height="40" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="230" text-anchor="middle" font-size="10" fill="#333">Latency (normal ops)</text>
  <rect x="195" y="205" width="250" height="40" fill="#e3f2fd" stroke="#ddd"/>
  <text x="320" y="225" text-anchor="middle" font-size="10" fill="#333">Higher — must wait for quorum</text>
  <text x="320" y="238" text-anchor="middle" font-size="9" fill="#888">Synchronous replication</text>
  <rect x="450" y="205" width="270" height="40" fill="#e8f5e9" stroke="#ddd"/>
  <text x="585" y="225" text-anchor="middle" font-size="10" fill="#333">Lower — respond from local node</text>
  <text x="585" y="238" text-anchor="middle" font-size="9" fill="#888">Asynchronous replication</text>

  <!-- Row 5 -->
  <rect x="30" y="245" width="160" height="40" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="270" text-anchor="middle" font-size="10" fill="#333">Best for</text>
  <rect x="195" y="245" width="250" height="40" fill="#e3f2fd" stroke="#ddd"/>
  <text x="320" y="265" text-anchor="middle" font-size="10" fill="#333">Banking, inventory, bookings</text>
  <text x="320" y="278" text-anchor="middle" font-size="9" fill="#888">Correctness > availability</text>
  <rect x="450" y="245" width="270" height="40" fill="#e8f5e9" stroke="#ddd"/>
  <text x="585" y="265" text-anchor="middle" font-size="10" fill="#333">Social feeds, analytics, IoT</text>
  <text x="585" y="278" text-anchor="middle" font-size="9" fill="#888">Availability > strict correctness</text>
</svg>

---

## 5. PACELC Theorem — CAP Extended

### The Problem with CAP

CAP only describes behaviour **during a partition**. But most of the time there is no partition — what trade-offs matter then?

### PACELC Statement (Abadi, 2012)

> If there is a **P**artition, choose between **A**vailability and **C**onsistency.
> **E**lse (normal operation), choose between **L**atency and **C**onsistency.

### The Full Framework

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">PACELC Theorem</text>

  <!-- P branch -->
  <rect x="250" y="50" width="250" height="40" rx="8" fill="#ffebee" stroke="#c62828" stroke-width="2"/>
  <text x="375" y="76" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">IF Partition…</text>

  <!-- PA -->
  <line x1="315" y1="90" x2="200" y2="130" stroke="#c62828" stroke-width="2"/>
  <rect x="80" y="130" width="180" height="50" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="170" y="155" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">PA — Availability</text>
  <text x="170" y="172" text-anchor="middle" font-size="10" fill="#555">Keep serving, stale OK</text>

  <!-- PC -->
  <line x1="435" y1="90" x2="550" y2="130" stroke="#c62828" stroke-width="2"/>
  <rect x="450" y="130" width="180" height="50" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="540" y="155" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">PC — Consistency</text>
  <text x="540" y="172" text-anchor="middle" font-size="10" fill="#555">Block until quorum</text>

  <!-- E branch -->
  <rect x="250" y="210" width="250" height="40" rx="8" fill="#fff8e1" stroke="#f9a825" stroke-width="2"/>
  <text x="375" y="236" text-anchor="middle" font-size="13" font-weight="bold" fill="#f9a825">ELSE (no partition)…</text>

  <!-- EL -->
  <line x1="315" y1="250" x2="200" y2="280" stroke="#f9a825" stroke-width="2"/>
  <rect x="80" y="280" width="180" height="50" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="170" y="305" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">EL — Low Latency</text>
  <text x="170" y="322" text-anchor="middle" font-size="10" fill="#555">Respond from local replica</text>

  <!-- EC -->
  <line x1="435" y1="250" x2="550" y2="280" stroke="#f9a825" stroke-width="2"/>
  <rect x="450" y="280" width="180" height="50" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="540" y="305" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">EC — Consistency</text>
  <text x="540" y="322" text-anchor="middle" font-size="10" fill="#555">Wait for replication</text>
</svg>

### PACELC Classification of Real Systems

| System | During Partition | Else (Normal) | PACELC Class |
|---|---|---|---|
| **Cassandra** | Availability | Low Latency | PA/EL |
| **DynamoDB** | Availability | Low Latency | PA/EL |
| **Riak** | Availability | Low Latency | PA/EL |
| **Google Spanner** | Consistency | Consistency | PC/EC |
| **CockroachDB** | Consistency | Consistency | PC/EC |
| **MongoDB** (default) | Consistency | Consistency | PC/EC |
| **PNUTS (Yahoo)** | Consistency | Low Latency | PC/EL |
| **Cosmos DB** (session) | Availability | Low Latency | PA/EL |
| **PostgreSQL (sync rep)** | Consistency | Consistency | PC/EC |

> **Key insight:** PA/EL systems (Cassandra, DynamoDB) sacrifice consistency for both availability and low latency — they're optimised for speed. PC/EC systems (Spanner, CockroachDB) sacrifice latency for correctness everywhere.

---

## 6. Replication — Keeping Copies in Sync

### Replication Topologies

<svg viewBox="0 0 750 430" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowRep" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
    <marker id="arrowBi" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#e65100"/>
    </marker>
  </defs>

  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Replication Topologies</text>

  <!-- Leader-Follower -->
  <rect x="15" y="45" width="220" height="170" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="125" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Leader–Follower</text>
  <text x="125" y="82" text-anchor="middle" font-size="9" fill="#555">(Master–Slave)</text>

  <rect x="80" y="95" width="90" height="30" rx="6" fill="#1565c0"/>
  <text x="125" y="115" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Leader (RW)</text>

  <line x1="80" y1="130" x2="50" y2="155" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRep)"/>
  <line x1="125" y1="130" x2="125" y2="155" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRep)"/>
  <line x1="170" y1="130" x2="200" y2="155" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRep)"/>

  <rect x="25" y="158" width="60" height="25" rx="4" fill="#bbdefb"/>
  <text x="55" y="175" text-anchor="middle" font-size="9" fill="#333">F1 (R)</text>
  <rect x="95" y="158" width="60" height="25" rx="4" fill="#bbdefb"/>
  <text x="125" y="175" text-anchor="middle" font-size="9" fill="#333">F2 (R)</text>
  <rect x="165" y="158" width="60" height="25" rx="4" fill="#bbdefb"/>
  <text x="195" y="175" text-anchor="middle" font-size="9" fill="#333">F3 (R)</text>

  <text x="125" y="205" text-anchor="middle" font-size="9" fill="#1565c0">Writes → Leader only</text>

  <!-- Multi-Leader -->
  <rect x="260" y="45" width="220" height="170" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="370" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">Multi-Leader</text>
  <text x="370" y="82" text-anchor="middle" font-size="9" fill="#555">(Master–Master)</text>

  <rect x="290" y="100" width="70" height="30" rx="6" fill="#e65100"/>
  <text x="325" y="120" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">L1 (RW)</text>
  <rect x="395" y="100" width="70" height="30" rx="6" fill="#e65100"/>
  <text x="430" y="120" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">L2 (RW)</text>

  <line x1="360" y1="115" x2="395" y2="115" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowBi)"/>
  <line x1="395" y1="118" x2="360" y2="118" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowBi)"/>

  <line x1="325" y1="130" x2="325" y2="155" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowBi)"/>
  <line x1="430" y1="130" x2="430" y2="155" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowBi)"/>

  <rect x="300" y="158" width="50" height="25" rx="4" fill="#ffe0b2"/>
  <text x="325" y="175" text-anchor="middle" font-size="9" fill="#333">F1</text>
  <rect x="405" y="158" width="50" height="25" rx="4" fill="#ffe0b2"/>
  <text x="430" y="175" text-anchor="middle" font-size="9" fill="#333">F2</text>

  <text x="370" y="205" text-anchor="middle" font-size="9" fill="#e65100">Writes → any leader (conflicts!)</text>

  <!-- Leaderless -->
  <rect x="505" y="45" width="230" height="170" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="620" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Leaderless</text>
  <text x="620" y="82" text-anchor="middle" font-size="9" fill="#555">(Dynamo-style)</text>

  <rect x="535" y="100" width="55" height="30" rx="6" fill="#2e7d32"/>
  <text x="562" y="120" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">N1</text>
  <rect x="605" y="100" width="55" height="30" rx="6" fill="#2e7d32"/>
  <text x="632" y="120" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">N2</text>
  <rect x="675" y="100" width="55" height="30" rx="6" fill="#2e7d32"/>
  <text x="702" y="120" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">N3</text>

  <!-- Interconnect lines -->
  <line x1="575" y1="130" x2="620" y2="135" stroke="#2e7d32" stroke-width="1"/>
  <line x1="645" y1="130" x2="690" y2="135" stroke="#2e7d32" stroke-width="1"/>
  <line x1="575" y1="135" x2="690" y2="140" stroke="#2e7d32" stroke-width="1"/>

  <text x="620" y="160" text-anchor="middle" font-size="9" fill="#333">Client writes to W nodes,</text>
  <text x="620" y="172" text-anchor="middle" font-size="9" fill="#333">reads from R nodes</text>
  <text x="620" y="188" text-anchor="middle" font-size="9" fill="#2e7d32" font-weight="bold">W + R > N → consistent</text>
  <text x="620" y="205" text-anchor="middle" font-size="9" fill="#2e7d32">No single leader</text>

  <!-- Comparison table below -->
  <rect x="15" y="240" width="720" height="30" rx="4" fill="#37474f"/>
  <text x="100" y="260" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Topology</text>
  <text x="260" y="260" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Write Scalability</text>
  <text x="410" y="260" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Conflict Risk</text>
  <text x="560" y="260" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Failover</text>
  <text x="680" y="260" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Example</text>

  <rect x="15" y="275" width="720" height="25" fill="#f5f5f5" stroke="#ddd"/>
  <text x="100" y="292" text-anchor="middle" font-size="10" fill="#333">Leader–Follower</text>
  <text x="260" y="292" text-anchor="middle" font-size="10" fill="#333">Single leader bottleneck</text>
  <text x="410" y="292" text-anchor="middle" font-size="10" fill="#2e7d32">None</text>
  <text x="560" y="292" text-anchor="middle" font-size="10" fill="#333">Promote follower</text>
  <text x="680" y="292" text-anchor="middle" font-size="10" fill="#888">PostgreSQL, MySQL</text>

  <rect x="15" y="300" width="720" height="25" fill="#fafafa" stroke="#ddd"/>
  <text x="100" y="317" text-anchor="middle" font-size="10" fill="#333">Multi-Leader</text>
  <text x="260" y="317" text-anchor="middle" font-size="10" fill="#333">High (multiple writers)</text>
  <text x="410" y="317" text-anchor="middle" font-size="10" fill="#c62828">High — need resolution</text>
  <text x="560" y="317" text-anchor="middle" font-size="10" fill="#333">Automatic</text>
  <text x="680" y="317" text-anchor="middle" font-size="10" fill="#888">CouchDB, Galera</text>

  <rect x="15" y="325" width="720" height="25" fill="#f5f5f5" stroke="#ddd"/>
  <text x="100" y="342" text-anchor="middle" font-size="10" fill="#333">Leaderless</text>
  <text x="260" y="342" text-anchor="middle" font-size="10" fill="#333">High (any node writes)</text>
  <text x="410" y="342" text-anchor="middle" font-size="10" fill="#e65100">Managed via quorum</text>
  <text x="560" y="342" text-anchor="middle" font-size="10" fill="#333">No leader to fail</text>
  <text x="680" y="342" text-anchor="middle" font-size="10" fill="#888">Cassandra, DynamoDB</text>

  <!-- Quorum formula -->
  <rect x="15" y="365" width="720" height="50" rx="8" fill="#fff8e1" stroke="#f9a825" stroke-width="1.5"/>
  <text x="375" y="388" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Quorum Formula: W + R > N</text>
  <text x="375" y="407" text-anchor="middle" font-size="11" fill="#333">N=3 replicas, W=2 (write to 2), R=2 (read from 2) → overlap guarantees seeing latest write</text>
</svg>

### Synchronous vs Asynchronous Replication

| Type | Behaviour | Consistency | Latency | Data Loss Risk |
|---|---|---|---|---|
| **Synchronous** | Leader waits for follower ACK before committing | Strong | Higher | Zero (if follower confirms) |
| **Semi-synchronous** | Wait for at least 1 follower ACK | Strong (for that replica) | Medium | Very low |
| **Asynchronous** | Leader commits immediately; follower catches up | Eventual | Lowest | Possible (replication lag) |

### PostgreSQL Replication Setup

```sql
-- Primary (postgresql.conf)
-- wal_level = replica
-- max_wal_senders = 5
-- synchronous_standby_names = 'standby1'

-- Create replication user
CREATE ROLE replicator REPLICATION LOGIN PASSWORD 'rep_password';

-- Standby (recovery.conf or postgresql.auto.conf)
-- primary_conninfo = 'host=primary_ip user=replicator password=rep_password'
-- primary_slot_name = 'standby1_slot'

-- Monitor replication lag
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
  FROM pg_stat_replication;
```

---

## 7. Sharding — Splitting Data Across Nodes

### Concept

**Sharding** (horizontal partitioning across machines) splits a table's rows across multiple database instances. Each shard holds a subset of the data.

### Sharding Strategies

<svg viewBox="0 0 750 370" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Sharding Strategies</text>

  <!-- Range Sharding -->
  <rect x="15" y="50" width="230" height="300" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="130" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Range Sharding</text>

  <rect x="35" y="90" width="190" height="30" rx="4" fill="#bbdefb"/>
  <text x="130" y="110" text-anchor="middle" font-size="10" fill="#333">Shard 1: user_id 1–1M</text>
  <rect x="35" y="125" width="190" height="30" rx="4" fill="#90caf9"/>
  <text x="130" y="145" text-anchor="middle" font-size="10" fill="#333">Shard 2: user_id 1M–2M</text>
  <rect x="35" y="160" width="190" height="30" rx="4" fill="#64b5f6"/>
  <text x="130" y="180" text-anchor="middle" font-size="10" fill="#fff">Shard 3: user_id 2M–3M</text>

  <text x="130" y="215" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">✓ Range queries efficient</text>
  <text x="130" y="232" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">✓ Simple to implement</text>
  <text x="130" y="255" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">✗ Hot spots (recent data)</text>
  <text x="130" y="272" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">✗ Uneven shard sizes</text>

  <text x="130" y="305" text-anchor="middle" font-size="9" fill="#888">Used by: HBase, Spanner,</text>
  <text x="130" y="318" text-anchor="middle" font-size="9" fill="#888">CockroachDB</text>

  <!-- Hash Sharding -->
  <rect x="260" y="50" width="230" height="300" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="375" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">Hash Sharding</text>

  <rect x="280" y="90" width="190" height="30" rx="4" fill="#c8e6c9"/>
  <text x="375" y="110" text-anchor="middle" font-size="10" fill="#333">hash(key) % 3 = 0 → Shard 1</text>
  <rect x="280" y="125" width="190" height="30" rx="4" fill="#a5d6a7"/>
  <text x="375" y="145" text-anchor="middle" font-size="10" fill="#333">hash(key) % 3 = 1 → Shard 2</text>
  <rect x="280" y="160" width="190" height="30" rx="4" fill="#81c784"/>
  <text x="375" y="180" text-anchor="middle" font-size="10" fill="#fff">hash(key) % 3 = 2 → Shard 3</text>

  <text x="375" y="215" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">✓ Even distribution</text>
  <text x="375" y="232" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">✓ No hot spots</text>
  <text x="375" y="255" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">✗ Range queries need all shards</text>
  <text x="375" y="272" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">✗ Resharding is expensive</text>

  <text x="375" y="305" text-anchor="middle" font-size="9" fill="#888">Used by: Cassandra,</text>
  <text x="375" y="318" text-anchor="middle" font-size="9" fill="#888">DynamoDB, Redis Cluster</text>

  <!-- Directory Sharding -->
  <rect x="505" y="50" width="230" height="300" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="620" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Directory Sharding</text>

  <rect x="555" y="95" width="130" height="40" rx="4" fill="#ffe0b2" stroke="#e65100"/>
  <text x="620" y="113" text-anchor="middle" font-size="10" fill="#333">Lookup Table</text>
  <text x="620" y="128" text-anchor="middle" font-size="9" fill="#555">key → shard mapping</text>

  <rect x="525" y="150" width="60" height="25" rx="4" fill="#ffcc80"/>
  <text x="555" y="167" text-anchor="middle" font-size="9" fill="#333">Shard 1</text>
  <rect x="590" y="150" width="60" height="25" rx="4" fill="#ffcc80"/>
  <text x="620" y="167" text-anchor="middle" font-size="9" fill="#333">Shard 2</text>
  <rect x="655" y="150" width="60" height="25" rx="4" fill="#ffcc80"/>
  <text x="685" y="167" text-anchor="middle" font-size="9" fill="#333">Shard 3</text>

  <text x="620" y="210" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">✓ Maximum flexibility</text>
  <text x="620" y="227" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">✓ Easy to rebalance</text>
  <text x="620" y="250" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">✗ Lookup = extra hop</text>
  <text x="620" y="267" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">✗ Directory is SPOF</text>

  <text x="620" y="305" text-anchor="middle" font-size="9" fill="#888">Used by: custom solutions,</text>
  <text x="620" y="318" text-anchor="middle" font-size="9" fill="#888">Vitess, some middleware</text>
</svg>

### Consistent Hashing

Standard hash sharding (`hash % N`) is fragile — adding a shard remaps almost all keys. **Consistent hashing** maps both keys and nodes onto a ring:

- Add a node → only neighbouring keys migrate
- Remove a node → only its keys redistribute
- **Virtual nodes** ensure even distribution

```
Ring:  0 ──── NodeA ──── NodeB ──── NodeC ──── 0 (wrap)
Key k → hash(k) → clockwise to nearest node
```

### Cross-Shard Queries (The Hard Problem)

| Pattern | Description | Difficulty |
|---|---|---|
| **Single-shard query** | Query includes the shard key in WHERE | Easy — route to one shard |
| **Scatter-gather** | Query needs all shards, combine results | Medium — parallel fan-out |
| **Cross-shard join** | JOIN between tables on different shards | Hard — data shuffle required |
| **Cross-shard transaction** | ACID across shards | Hardest — needs 2PC |

---

## 8. Consensus Algorithms — Paxos & Raft

### Why Consensus?

In a distributed system, nodes must agree on a single value (leader, committed transaction, configuration) even when some nodes fail or messages are delayed. Consensus algorithms solve this.

### Raft (Simplified Consensus)

Raft divides consensus into three sub-problems: **leader election**, **log replication**, and **safety**.

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowRaft" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
  </defs>

  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Raft Consensus — Leader Election + Log Replication</text>

  <!-- Phase 1: Election -->
  <rect x="20" y="50" width="220" height="130" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="130" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">1. Leader Election</text>

  <circle cx="80" cy="110" r="20" fill="#1565c0"/>
  <text x="80" y="115" text-anchor="middle" font-size="9" fill="#fff" font-weight="bold">Cand.</text>
  <circle cx="130" cy="110" r="20" fill="#bbdefb" stroke="#1565c0"/>
  <text x="130" y="115" text-anchor="middle" font-size="9" fill="#333">Vote</text>
  <circle cx="180" cy="110" r="20" fill="#bbdefb" stroke="#1565c0"/>
  <text x="180" y="115" text-anchor="middle" font-size="9" fill="#333">Vote</text>

  <text x="130" y="155" text-anchor="middle" font-size="9" fill="#555">Timeout → candidate</text>
  <text x="130" y="168" text-anchor="middle" font-size="9" fill="#555">Gets majority → leader</text>

  <!-- Arrow -->
  <line x1="240" y1="115" x2="275" y2="115" stroke="#1565c0" stroke-width="2" marker-end="url(#arrowRaft)"/>

  <!-- Phase 2: Replication -->
  <rect x="280" y="50" width="210" height="130" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="385" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">2. Log Replication</text>

  <rect x="310" y="88" width="150" height="20" rx="3" fill="#c8e6c9"/>
  <text x="385" y="103" text-anchor="middle" font-size="9" fill="#333">Leader: [A][B][C][D]</text>
  <rect x="310" y="112" width="120" height="20" rx="3" fill="#c8e6c9"/>
  <text x="370" y="127" text-anchor="middle" font-size="9" fill="#333">F1: [A][B][C]</text>
  <rect x="310" y="136" width="120" height="20" rx="3" fill="#c8e6c9"/>
  <text x="370" y="151" text-anchor="middle" font-size="9" fill="#333">F2: [A][B][C]</text>

  <text x="385" y="175" text-anchor="middle" font-size="9" fill="#555">AppendEntries RPC</text>

  <!-- Arrow -->
  <line x1="490" y1="115" x2="525" y2="115" stroke="#1565c0" stroke-width="2" marker-end="url(#arrowRaft)"/>

  <!-- Phase 3: Commit -->
  <rect x="530" y="50" width="200" height="130" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="630" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">3. Commit</text>

  <text x="630" y="100" text-anchor="middle" font-size="10" fill="#333">Majority replicated?</text>
  <text x="630" y="120" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">Yes → committed</text>
  <text x="630" y="140" text-anchor="middle" font-size="10" fill="#555">Leader advances</text>
  <text x="630" y="155" text-anchor="middle" font-size="10" fill="#555">commitIndex</text>
  <text x="630" y="175" text-anchor="middle" font-size="10" fill="#555">Notify followers</text>

  <!-- Terms -->
  <rect x="20" y="200" width="710" height="120" rx="10" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="225" text-anchor="middle" font-size="13" font-weight="bold" fill="#333">Raft Key Concepts</text>

  <text x="50" y="250" font-size="10" fill="#1565c0" font-weight="bold">Term:</text>
  <text x="90" y="250" font-size="10" fill="#333">Logical clock. Each term has at most one leader. New election = new term.</text>

  <text x="50" y="272" font-size="10" fill="#2e7d32" font-weight="bold">Heartbeat:</text>
  <text x="115" y="272" font-size="10" fill="#333">Leader sends periodic AppendEntries (even empty) to prevent new elections.</text>

  <text x="50" y="294" font-size="10" fill="#e65100" font-weight="bold">Safety:</text>
  <text x="95" y="294" font-size="10" fill="#333">A candidate can only win if its log is at least as up-to-date as the majority. Committed entries are never lost.</text>

  <text x="50" y="316" font-size="10" fill="#6a1b9a" font-weight="bold">Quorum:</text>
  <text x="105" y="316" font-size="10" fill="#333">Majority of nodes (3 of 5, 2 of 3). Ensures overlap between any two majorities.</text>
</svg>

### Paxos vs Raft

| Aspect | Paxos | Raft |
|---|---|---|
| **Inventor** | Leslie Lamport (1989) | Ongaro & Ousterhout (2014) |
| **Understandability** | Notoriously hard to understand | Designed for clarity |
| **Leader** | Proposer (not always a stable leader) | Stable leader per term |
| **Phases** | Prepare → Promise → Accept → Learn | RequestVote → AppendEntries |
| **Used by** | Google Chubby, Azure | etcd, CockroachDB, TiKV |
| **Multi-Paxos** | Extension for multiple values (complex) | Built-in log replication |

---

## 9. Distributed Transactions — 2PC & 3PC

### Two-Phase Commit (2PC)

<svg viewBox="0 0 750 320" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrow2PC" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
    <marker id="arrow2PCr" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#2e7d32"/>
    </marker>
  </defs>

  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Two-Phase Commit (2PC)</text>

  <!-- Coordinator -->
  <rect x="280" y="48" width="190" height="40" rx="8" fill="#1565c0"/>
  <text x="375" y="73" text-anchor="middle" font-size="13" fill="#fff" font-weight="bold">Coordinator (TM)</text>

  <!-- Participants -->
  <rect x="40" y="140" width="140" height="40" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="110" y="165" text-anchor="middle" font-size="11" fill="#1565c0" font-weight="bold">Participant A</text>

  <rect x="300" y="140" width="140" height="40" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="370" y="165" text-anchor="middle" font-size="11" fill="#1565c0" font-weight="bold">Participant B</text>

  <rect x="560" y="140" width="140" height="40" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="630" y="165" text-anchor="middle" font-size="11" fill="#1565c0" font-weight="bold">Participant C</text>

  <!-- Phase 1: Prepare -->
  <rect x="20" y="50" width="100" height="25" rx="4" fill="#fff3e0" stroke="#e65100"/>
  <text x="70" y="67" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">PHASE 1</text>

  <line x1="325" y1="88" x2="110" y2="135" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrow2PC)"/>
  <line x1="375" y1="88" x2="370" y2="135" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrow2PC)"/>
  <line x1="425" y1="88" x2="630" y2="135" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrow2PC)"/>
  <text x="210" y="107" font-size="9" fill="#1565c0">PREPARE?</text>
  <text x="535" y="107" font-size="9" fill="#1565c0">PREPARE?</text>

  <!-- Votes -->
  <line x1="110" y1="185" x2="325" y2="220" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrow2PCr)"/>
  <line x1="370" y1="185" x2="375" y2="220" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrow2PCr)"/>
  <line x1="630" y1="185" x2="425" y2="220" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrow2PCr)"/>
  <text x="210" y="210" font-size="9" fill="#2e7d32">YES (prepared)</text>
  <text x="535" y="210" font-size="9" fill="#2e7d32">YES (prepared)</text>

  <!-- Decision -->
  <rect x="290" y="220" width="170" height="35" rx="8" fill="#c8e6c9" stroke="#2e7d32" stroke-width="2"/>
  <text x="375" y="243" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">All YES → COMMIT</text>

  <!-- Phase 2: Commit -->
  <rect x="20" y="220" width="100" height="25" rx="4" fill="#fff3e0" stroke="#e65100"/>
  <text x="70" y="237" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">PHASE 2</text>

  <line x1="325" y1="258" x2="110" y2="280" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrow2PCr)"/>
  <line x1="375" y1="258" x2="370" y2="280" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrow2PCr)"/>
  <line x1="425" y1="258" x2="630" y2="280" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrow2PCr)"/>
  <text x="215" y="278" font-size="9" fill="#2e7d32">COMMIT!</text>
  <text x="540" y="278" font-size="9" fill="#2e7d32">COMMIT!</text>

  <!-- Abort case -->
  <rect x="530" y="220" width="200" height="35" rx="6" fill="#ffebee" stroke="#c62828" stroke-width="1.5"/>
  <text x="630" y="243" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">Any NO → ABORT all</text>

  <!-- Blocking problem -->
  <rect x="100" y="295" width="550" height="22" rx="4" fill="#fff8e1" stroke="#f9a825"/>
  <text x="375" y="311" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">Problem: If coordinator crashes after PREPARE, participants are BLOCKED (holding locks, waiting).</text>
</svg>

### Three-Phase Commit (3PC)

3PC adds a **pre-commit** phase to avoid blocking:

| Phase | Action |
|---|---|
| **1. Prepare** | Coordinator asks "Can you commit?" — participants respond YES/NO |
| **2. Pre-Commit** | If all YES → coordinator sends PRE-COMMIT. Participants acknowledge. |
| **3. Commit** | Coordinator sends COMMIT. Participants apply changes. |

**Advantage:** If the coordinator crashes after pre-commit, participants can independently decide to commit (they know everyone voted YES).
**Disadvantage:** Still not partition-tolerant — network partitions can cause inconsistency. Rarely used in practice.

### Modern Alternatives to 2PC

| Approach | Description | Used By |
|---|---|---|
| **Saga pattern** | Chain of local transactions with compensating actions | Microservices (Temporal, Cadence) |
| **Percolator** | Optimistic 2PC using timestamps | Google BigTable |
| **Calvin** | Deterministic ordering; no 2PC needed | FaunaDB |
| **Spanner commit** | Paxos-replicated 2PC with TrueTime | Google Spanner |

---

## 10. Real-World Architectures

### Google Spanner — Globally Consistent

| Component | Role |
|---|---|
| **TrueTime** | GPS + atomic clocks give globally synchronised timestamps (uncertainty < 7 ms) |
| **Paxos groups** | Each shard replicated via Paxos across zones |
| **2PC** | Cross-shard transactions use Paxos-replicated 2PC |
| **External consistency** | Transactions appear to execute in real-time order |
| **CAP class** | CP — sacrifices availability during partitions; PC/EC in PACELC |

### Amazon DynamoDB — Always Available

| Component | Role |
|---|---|
| **Consistent hashing** | Data partitioned across nodes on a hash ring with virtual nodes |
| **Quorum** | Configurable: eventual (fast) or strong (wait for all replicas) |
| **Vector clocks** | Detect conflicts in concurrent writes |
| **Anti-entropy** | Merkle trees compare replica state for repair |
| **CAP class** | AP by default; PA/EL in PACELC |

### CockroachDB — Open-Source Spanner

| Component | Role |
|---|---|
| **Raft** | Each range (64 MB shard) replicated via Raft |
| **MVCC** | Serialisable isolation using multi-version timestamps |
| **Distributed SQL** | Full SQL layer over a KV store |
| **Lease-based reads** | Range leaseholder serves reads without consensus round-trip |
| **CAP class** | CP; PC/EC in PACELC |

---

## 11. Worked Examples

### Example 1 — Beginner: Choosing CP vs AP for a Student System

```
Scenario: University grade management system.

Question: Should this be CP or AP?

Analysis:
- Grades must be accurate (no student sees wrong final grade)
- Temporary unavailability (during server maintenance) is acceptable
- Inconsistent grades could lead to wrong transcripts, legal issues

Decision: CP
- Use PostgreSQL with synchronous streaming replication
- During a network partition, reject writes rather than risk inconsistency
- Availability sacrifice: students retry in a few minutes
```

### Example 2 — Beginner: Replication Lag Monitoring

```sql
-- PostgreSQL: Check replication lag on the primary
SELECT client_addr,
       application_name,
       state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) / 1024.0 / 1024.0 AS lag_mb
  FROM pg_stat_replication;

-- On the standby: check how far behind
SELECT CASE WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn()
            THEN 0
            ELSE EXTRACT(EPOCH FROM NOW() - pg_last_xact_replay_timestamp())
       END AS replication_lag_seconds;
```

| client_addr | application_name | state | lag_bytes | lag_mb |
|---|---|---|---|---|
| 10.0.1.12 | standby1 | streaming | 4096 | 0.004 |
| 10.0.1.13 | standby2 | streaming | 1048576 | 1.000 |

### Example 3 — Intermediate: Application-Level Sharding

```python
# Python: route queries to the correct shard based on user_id
import hashlib

SHARD_CONFIG = {
    0: "postgresql://shard0.example.com/mydb",
    1: "postgresql://shard1.example.com/mydb",
    2: "postgresql://shard2.example.com/mydb",
}

NUM_SHARDS = len(SHARD_CONFIG)

def get_shard(user_id: int) -> int:
    """Consistent hash-based shard assignment."""
    key = str(user_id).encode()
    hash_val = int(hashlib.md5(key).hexdigest(), 16)
    return hash_val % NUM_SHARDS

def get_connection(user_id: int):
    shard_id = get_shard(user_id)
    return connect(SHARD_CONFIG[shard_id])

# Usage
user_id = 42
conn = get_connection(user_id)
cursor = conn.cursor()
cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))

# Cross-shard query: scatter-gather
def get_all_orders_total():
    total = 0
    for shard_id, dsn in SHARD_CONFIG.items():
        conn = connect(dsn)
        cursor = conn.cursor()
        cursor.execute("SELECT SUM(amount) FROM orders")
        result = cursor.fetchone()[0] or 0
        total += result
    return total
```

### Example 4 — Intermediate: Quorum Reads/Writes in Cassandra

```cql
-- Cassandra: creating a keyspace with replication factor 3
CREATE KEYSPACE ecommerce
    WITH REPLICATION = {
        'class': 'NetworkTopologyStrategy',
        'us-east': 3,
        'eu-west': 3
    };

USE ecommerce;

CREATE TABLE orders (
    order_id    UUID PRIMARY KEY,
    customer_id INT,
    total       DECIMAL,
    status      TEXT,
    created_at  TIMESTAMP
);

-- Strong consistency: QUORUM write + QUORUM read (W + R > N → 2 + 2 > 3 ✓)
CONSISTENCY QUORUM;
INSERT INTO orders (order_id, customer_id, total, status, created_at)
VALUES (uuid(), 42, 199.99, 'confirmed', toTimestamp(now()));

-- Fast reads (eventual consistency): ONE
CONSISTENCY ONE;
SELECT * FROM orders WHERE order_id = ?;

-- Maximum durability: ALL
CONSISTENCY ALL;
INSERT INTO orders (...) VALUES (...);
-- Waits for ALL 3 replicas — slowest but strongest guarantee
```

### Example 5 — Advanced: Saga Pattern for Cross-Service Transactions

```python
# Saga pattern: book a trip (flight + hotel + car) without 2PC
# Each step has a compensating action for rollback

class TripBookingSaga:
    def execute(self, trip_request):
        steps = [
            (self.book_flight, self.cancel_flight),
            (self.book_hotel,  self.cancel_hotel),
            (self.book_car,    self.cancel_car),
            (self.charge_payment, self.refund_payment),
        ]

        completed = []
        for action, compensate in steps:
            try:
                result = action(trip_request)
                completed.append((compensate, result))
            except Exception as e:
                # Compensate in reverse order
                for comp_action, comp_data in reversed(completed):
                    comp_action(comp_data)
                raise SagaFailure(f"Step failed: {e}") from e

        return TripConfirmation(completed)

    def book_flight(self, req):
        # Local transaction on flight service DB
        return flight_service.book(req.flight_id, req.passenger)

    def cancel_flight(self, booking):
        # Compensating transaction
        flight_service.cancel(booking.confirmation_id)

    # ... similar for hotel, car, payment
```

```
Saga execution timeline:

  book_flight ──✓──► book_hotel ──✓──► book_car ──✗ FAIL
                                           │
                     cancel_hotel ◄──── cancel_flight
                          │
                       ROLLED BACK (compensating transactions)
```

### Example 6 — Advanced: Distributed Leader Election with PostgreSQL Advisory Locks

```sql
-- PostgreSQL: simple leader election using advisory locks
-- Only one connection can hold a given advisory lock at a time

-- Each node tries to become leader:
SELECT pg_try_advisory_lock(12345) AS am_i_leader;
-- Returns true for exactly one connection; false for all others

-- Leader performs work:
DO $$
DECLARE
    v_is_leader BOOLEAN;
BEGIN
    SELECT pg_try_advisory_lock(12345) INTO v_is_leader;

    IF v_is_leader THEN
        RAISE NOTICE 'I am the leader — processing jobs';
        -- Process queued jobs, run maintenance, etc.
        PERFORM process_pending_jobs();
    ELSE
        RAISE NOTICE 'Not the leader — standing by';
    END IF;
END;
$$;

-- Release leadership (on graceful shutdown)
SELECT pg_advisory_unlock(12345);
```

---

## 12. Common Interview Questions

### Q1: Explain the CAP theorem.

**Answer:** The CAP theorem (Brewer, 2000) states that a distributed data store cannot simultaneously guarantee **Consistency** (all nodes see the same data), **Availability** (every request gets a response), and **Partition tolerance** (system works despite network failures). Since network partitions are unavoidable in distributed systems, the real choice is between CP (consistent but may be unavailable) and AP (available but may return stale data).

### Q2: Is CAP a binary choice? Can you have "mostly CP"?

**Answer:** In practice, CAP is a **spectrum**, not a binary switch. Systems like Cassandra offer **tunable consistency** — you choose per-query: `CONSISTENCY ONE` (AP-like, fast) vs `CONSISTENCY QUORUM` (CP-like, strong) vs `CONSISTENCY ALL` (strongest, slowest). Even CP systems are available most of the time — they only sacrifice availability during actual partitions, which are rare. The choice is really about what happens during that rare event.

### Q3: What is the PACELC theorem?

**Answer:** PACELC extends CAP: **P**artition → choose **A** or **C**; **E**lse (normal) → choose **L**atency or **C**onsistency. This captures the normal-operation trade-off that CAP ignores. Cassandra is PA/EL (prioritises availability and latency), Spanner is PC/EC (prioritises consistency everywhere). PACELC explains why systems with the same CAP classification behave differently in normal operation.

### Q4: What is the difference between leader-follower and leaderless replication?

**Answer:**
- **Leader-follower**: All writes go to a single leader, which replicates to followers. Simple, no write conflicts, but the leader is a bottleneck and SPOF (until failover). Used by PostgreSQL, MySQL, MongoDB.
- **Leaderless**: Any node accepts writes. Uses quorum (W + R > N) to ensure reads see the latest write. No single point of failure, higher write throughput, but requires conflict resolution. Used by Cassandra, DynamoDB, Riak.

### Q5: What is a quorum and why does W + R > N guarantee consistency?

**Answer:** In a system with **N** replicas, **W** is the number of nodes a write must reach, and **R** is the number of nodes a read must query. If W + R > N, then the read set and write set **must overlap** — at least one node has the latest write. Example: N=3, W=2, R=2 → any 2-of-3 read includes at least one node that participated in the 2-of-3 write. Common configurations: W=1, R=N (fast writes, slow reads); W=N, R=1 (slow writes, fast reads); W=R=(N+1)/2 (balanced).

### Q6: What is the two-phase commit (2PC) and what is its main weakness?

**Answer:** 2PC is a protocol for atomic cross-node transactions. Phase 1: coordinator asks participants to **prepare** (vote YES/NO). Phase 2: if all vote YES, coordinator sends **COMMIT**; otherwise **ABORT**. Main weakness: **blocking** — if the coordinator crashes after Phase 1 (participants voted YES), participants hold locks and wait indefinitely, unable to commit or abort on their own. This makes 2PC unsuitable for high-availability systems.

### Q7: How does sharding differ from replication?

**Answer:**
- **Sharding**: splits data — each shard holds a different subset of rows. Purpose: scale storage and write throughput.
- **Replication**: copies data — each replica holds the same data. Purpose: availability and read throughput.

Most production systems use **both**: shard the data, then replicate each shard for fault tolerance. Example: DynamoDB shards by partition key and replicates each shard across 3 AZs.

### Q8: What is consistent hashing?

**Answer:** A technique where both data keys and server nodes are mapped onto a hash ring (0 to 2^32). A key is stored on the first node clockwise from its hash position. Adding or removing a node only affects keys between the node and its predecessor — not the entire dataset. **Virtual nodes** (each physical node appears multiple times on the ring) ensure even distribution. Used by Cassandra, DynamoDB, and Riak.

### Q9: What are vector clocks?

**Answer:** A conflict detection mechanism for concurrent writes in distributed systems. Each node maintains a vector of logical clocks `[N1:3, N2:5, N3:2]`. On each write, the local counter increments. When comparing two versions: if all entries in V1 ≤ V2, V1 is an ancestor (no conflict). If neither dominates, they're concurrent (conflict — needs resolution). DynamoDB and Riak use vector clocks; Cassandra uses last-write-wins (LWW) timestamps instead.

### Q10: What is the Saga pattern?

**Answer:** A pattern for managing distributed transactions without 2PC. A saga is a sequence of local transactions, each with a **compensating action** (undo). If step 3 fails, you run compensating actions for steps 2, 1 (in reverse). Two styles: **choreography** (each service triggers the next) and **orchestration** (a central coordinator manages the flow). Trade-off: no atomicity (intermediate states are visible), but no blocking. Used in microservices architectures (Temporal, Cadence).

### Q11: How does Google Spanner achieve global consistency?

**Answer:** Spanner uses **TrueTime**, a globally synchronised clock backed by GPS receivers and atomic clocks in every data centre. Uncertainty is bounded to <7 ms. Transactions are assigned a timestamp, and the coordinator waits out the uncertainty interval before committing. This guarantees **external consistency**: if transaction T1 commits before T2 starts (in real-time), T1's timestamp < T2's timestamp. Combined with Paxos replication, this makes Spanner a globally consistent, CP system.

### Q12: When would you choose an AP system over a CP system?

**Answer:** Choose AP when:
- Brief staleness is acceptable (social media feeds, product recommendations, view counters)
- Availability is critical (shopping cart — losing a cart update is worse than showing a stale one)
- Low latency matters more than strict ordering (real-time analytics, IoT sensor data)
- Geographic distribution requires local reads

Choose CP when:
- Correctness is non-negotiable (banking, inventory, seat booking, auction bidding)
- Reading stale data causes real harm (medical records, financial ledgers)
- Regulatory requirements demand consistency

---

## 13. Key Takeaways

<svg viewBox="0 0 750 520" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">Phase 17 — Key Takeaways</text>

  <!-- Card 1 -->
  <rect x="20" y="50" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="40" y="75" font-size="13" font-weight="bold" fill="#1565c0">CAP: Pick Two (Really Pick One)</text>
  <text x="40" y="95" font-size="11" fill="#333">P is mandatory. During a partition,</text>
  <text x="40" y="112" font-size="11" fill="#333">choose Consistency or Availability.</text>

  <!-- Card 2 -->
  <rect x="390" y="50" width="340" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="410" y="75" font-size="13" font-weight="bold" fill="#2e7d32">PACELC Extends CAP</text>
  <text x="410" y="95" font-size="11" fill="#333">Normal ops: Latency vs Consistency.</text>
  <text x="410" y="112" font-size="11" fill="#333">PA/EL = fast. PC/EC = correct.</text>

  <!-- Card 3 -->
  <rect x="20" y="150" width="340" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="40" y="175" font-size="13" font-weight="bold" fill="#e65100">Replication = Copies for HA</text>
  <text x="40" y="195" font-size="11" fill="#333">Leader-follower, multi-leader, leaderless.</text>
  <text x="40" y="212" font-size="11" fill="#333">Sync vs async = durability vs latency.</text>

  <!-- Card 4 -->
  <rect x="390" y="150" width="340" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="410" y="175" font-size="13" font-weight="bold" fill="#6a1b9a">Sharding = Splits for Scale</text>
  <text x="410" y="195" font-size="11" fill="#333">Range, hash, or directory-based.</text>
  <text x="410" y="212" font-size="11" fill="#333">Cross-shard queries are expensive.</text>

  <!-- Card 5 -->
  <rect x="20" y="250" width="340" height="80" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="40" y="275" font-size="13" font-weight="bold" fill="#c62828">Consensus = Agreement</text>
  <text x="40" y="295" font-size="11" fill="#333">Raft/Paxos ensure all nodes agree.</text>
  <text x="40" y="312" font-size="11" fill="#333">Leader election + log replication.</text>

  <!-- Card 6 -->
  <rect x="390" y="250" width="340" height="80" rx="10" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="410" y="275" font-size="13" font-weight="bold" fill="#f9a825">2PC Blocks, Sagas Don't</text>
  <text x="410" y="295" font-size="11" fill="#333">2PC: atomic but blocking.</text>
  <text x="410" y="312" font-size="11" fill="#333">Saga: compensate on failure (no locks).</text>

  <!-- Card 7 -->
  <rect x="20" y="350" width="340" height="80" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="40" y="375" font-size="13" font-weight="bold" fill="#00695c">Quorum: W + R > N</text>
  <text x="40" y="395" font-size="11" fill="#333">Overlap guarantees read sees latest write.</text>
  <text x="40" y="412" font-size="11" fill="#333">Tune per query: fast vs consistent.</text>

  <!-- Card 8 -->
  <rect x="390" y="350" width="340" height="80" rx="10" fill="#ffccbc" stroke="#bf360c" stroke-width="2"/>
  <text x="410" y="375" font-size="13" font-weight="bold" fill="#bf360c">Most Systems Use Both</text>
  <text x="410" y="395" font-size="11" fill="#333">Shard for scale + replicate each shard</text>
  <text x="410" y="412" font-size="11" fill="#333">for availability. Real-world is hybrid.</text>

  <!-- Decision guide -->
  <rect x="80" y="455" width="590" height="50" rx="10" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="375" y="478" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Decision Guide</text>
  <text x="375" y="496" text-anchor="middle" font-size="10" fill="#555">Money/booking/inventory → CP | Social/analytics/IoT → AP | Single region → CA is fine</text>
</svg>

---

## 14. Download

<a href="17_cap_theorem_distributed.md" download="17_cap_theorem_distributed.md" style="display:inline-block;padding:14px 28px;background:linear-gradient(135deg,#1a237e,#0d47a1);color:#fff;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;box-shadow:0 4px 15px rgba(26,35,126,0.3);margin:10px 0;">Download 17_cap_theorem_distributed.md</a>
