# Phase 19 — NoSQL Databases

> **NoSQL** ("Not Only SQL") is a family of database systems that break away from the relational model to handle specific workloads — massive scale, flexible schemas, low-latency access patterns, and data structures that don't fit neatly into rows and columns.

---

## Table of Contents

1. [Why NoSQL?](#1-why-nosql)
2. [NoSQL Categories Overview](#2-nosql-categories)
3. [Document Stores — MongoDB](#3-document-stores)
4. [Key-Value Stores — Redis](#4-key-value-stores)
5. [Column-Family Stores — Cassandra](#5-column-family-stores)
6. [Graph Databases — Neo4j](#6-graph-databases)
7. [Time-Series Databases — InfluxDB](#7-time-series-databases)
8. [Data Modeling in NoSQL](#8-data-modeling)
9. [SQL vs NoSQL — Decision Guide](#9-sql-vs-nosql)
10. [Polyglot Persistence](#10-polyglot-persistence)
11. [Worked Examples](#11-worked-examples)
12. [Common Interview Questions](#12-interview-questions)
13. [Key Takeaways](#13-key-takeaways)
14. [Download](#14-download)

---

## 1. Why NoSQL?

### The Relational Model's Limits

| Challenge | Relational Approach | Problem |
|---|---|---|
| **Massive write throughput** | Single-leader writes | Leader bottleneck |
| **Flexible / evolving schema** | ALTER TABLE + migrations | Downtime, complexity |
| **Hierarchical / nested data** | Normalised into many tables + JOINs | JOIN overhead, impedance mismatch |
| **Horizontal scaling** | Sharding is bolted on | Complex, application-managed |
| **Sub-millisecond reads** | Disk-based B-Tree lookups | Not fast enough for caching |
| **Relationship traversal** | Recursive JOINs | Exponential cost at depth |

### What NoSQL Offers

| Benefit | How |
|---|---|
| **Horizontal scalability** | Built-in sharding, auto-rebalancing |
| **Flexible schema** | Schemaless or schema-on-read |
| **High throughput** | Optimised for specific access patterns |
| **Low latency** | In-memory, key-based lookups |
| **Geographic distribution** | Multi-region replication by design |

### Real-World Analogy

A relational database is like a **filing cabinet** with rigid folders and labels — perfect when everything fits the template. NoSQL databases are like different kinds of storage: a **dictionary** (key-value), a **filing box that accepts any document shape** (document store), a **spreadsheet with dynamic columns** (column-family), or a **mind map** (graph). You pick the storage that matches how you actually use the data.

---

## 2. NoSQL Categories Overview

<svg viewBox="0 0 750 420" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">NoSQL Database Categories</text>

  <!-- Document -->
  <rect x="20" y="55" width="210" height="160" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="125" y="80" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Document Store</text>
  <text x="125" y="100" text-anchor="middle" font-size="10" fill="#333">JSON/BSON documents</text>
  <text x="125" y="118" text-anchor="middle" font-size="10" fill="#333">Flexible nested structure</text>
  <text x="125" y="136" text-anchor="middle" font-size="10" fill="#333">Rich queries on fields</text>
  <rect x="40" y="148" width="170" height="22" rx="4" fill="#bbdefb"/>
  <text x="125" y="164" text-anchor="middle" font-size="9" fill="#333">MongoDB, CouchDB, Firestore</text>
  <rect x="40" y="175" width="170" height="30" rx="4" fill="#e8eaf6"/>
  <text x="125" y="190" text-anchor="middle" font-size="9" fill="#555">Use: CMS, catalogs, user profiles</text>
  <text x="125" y="202" text-anchor="middle" font-size="9" fill="#555">e-commerce, mobile backends</text>

  <!-- Key-Value -->
  <rect x="250" y="55" width="210" height="160" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="355" y="80" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">Key-Value Store</text>
  <text x="355" y="100" text-anchor="middle" font-size="10" fill="#333">Simple key → value pairs</text>
  <text x="355" y="118" text-anchor="middle" font-size="10" fill="#333">Blazing fast O(1) lookups</text>
  <text x="355" y="136" text-anchor="middle" font-size="10" fill="#333">No query on value content</text>
  <rect x="270" y="148" width="170" height="22" rx="4" fill="#c8e6c9"/>
  <text x="355" y="164" text-anchor="middle" font-size="9" fill="#333">Redis, Memcached, DynamoDB</text>
  <rect x="270" y="175" width="170" height="30" rx="4" fill="#e8f5e9"/>
  <text x="355" y="190" text-anchor="middle" font-size="9" fill="#555">Use: caching, sessions, rate</text>
  <text x="355" y="202" text-anchor="middle" font-size="9" fill="#555">limiting, leaderboards</text>

  <!-- Column-Family -->
  <rect x="480" y="55" width="250" height="160" rx="12" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="605" y="80" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Column-Family Store</text>
  <text x="605" y="100" text-anchor="middle" font-size="10" fill="#333">Wide rows, dynamic columns</text>
  <text x="605" y="118" text-anchor="middle" font-size="10" fill="#333">Sorted by clustering key</text>
  <text x="605" y="136" text-anchor="middle" font-size="10" fill="#333">Optimised for write-heavy</text>
  <rect x="500" y="148" width="210" height="22" rx="4" fill="#ffe0b2"/>
  <text x="605" y="164" text-anchor="middle" font-size="9" fill="#333">Cassandra, HBase, ScyllaDB</text>
  <rect x="500" y="175" width="210" height="30" rx="4" fill="#fff3e0"/>
  <text x="605" y="190" text-anchor="middle" font-size="9" fill="#555">Use: IoT, time-series, messaging,</text>
  <text x="605" y="202" text-anchor="middle" font-size="9" fill="#555">activity feeds, logs</text>

  <!-- Graph -->
  <rect x="80" y="240" width="250" height="160" rx="12" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="205" y="265" text-anchor="middle" font-size="13" font-weight="bold" fill="#6a1b9a">Graph Database</text>
  <text x="205" y="285" text-anchor="middle" font-size="10" fill="#333">Nodes + edges + properties</text>
  <text x="205" y="303" text-anchor="middle" font-size="10" fill="#333">Traversal queries (Cypher, Gremlin)</text>
  <text x="205" y="321" text-anchor="middle" font-size="10" fill="#333">Index-free adjacency</text>
  <rect x="100" y="333" width="210" height="22" rx="4" fill="#e1bee7"/>
  <text x="205" y="349" text-anchor="middle" font-size="9" fill="#333">Neo4j, Amazon Neptune, JanusGraph</text>
  <rect x="100" y="360" width="210" height="30" rx="4" fill="#f3e5f5"/>
  <text x="205" y="375" text-anchor="middle" font-size="9" fill="#555">Use: social networks, fraud detection,</text>
  <text x="205" y="387" text-anchor="middle" font-size="9" fill="#555">recommendations, knowledge graphs</text>

  <!-- Time-Series -->
  <rect x="400" y="240" width="250" height="160" rx="12" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="525" y="265" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">Time-Series Database</text>
  <text x="525" y="285" text-anchor="middle" font-size="10" fill="#333">Timestamp as primary axis</text>
  <text x="525" y="303" text-anchor="middle" font-size="10" fill="#333">High-ingest, automatic rollup</text>
  <text x="525" y="321" text-anchor="middle" font-size="10" fill="#333">Retention policies, downsampling</text>
  <rect x="420" y="333" width="210" height="22" rx="4" fill="#ffcdd2"/>
  <text x="525" y="349" text-anchor="middle" font-size="9" fill="#333">InfluxDB, TimescaleDB, Prometheus</text>
  <rect x="420" y="360" width="210" height="30" rx="4" fill="#fce4ec"/>
  <text x="525" y="375" text-anchor="middle" font-size="9" fill="#555">Use: monitoring, metrics, IoT sensors,</text>
  <text x="525" y="387" text-anchor="middle" font-size="9" fill="#555">financial ticks, DevOps observability</text>
</svg>

---

## 3. Document Stores — MongoDB

### Data Model

Documents are JSON-like objects (BSON in MongoDB) that can contain nested objects, arrays, and varied structures. Documents are grouped into **collections** (analogous to tables), but each document can have a different shape.

```json
// A product document — no rigid schema
{
  "_id": ObjectId("664a1b2c3d4e5f6a7b8c9d0e"),
  "name": "Wireless Headphones",
  "brand": "AudioMax",
  "price": 79.99,
  "categories": ["electronics", "audio"],
  "specs": {
    "driver_size": "40mm",
    "battery_hours": 30,
    "noise_cancelling": true,
    "bluetooth": "5.3"
  },
  "reviews": [
    { "user": "alice", "rating": 5, "text": "Amazing sound!" },
    { "user": "bob",   "rating": 4, "text": "Good value" }
  ],
  "in_stock": true,
  "created_at": ISODate("2026-04-15T10:30:00Z")
}
```

### CRUD Operations

```javascript
// MongoDB Shell / Node.js driver

// INSERT
db.products.insertOne({
  name: "Wireless Headphones",
  brand: "AudioMax",
  price: 79.99,
  categories: ["electronics", "audio"]
});

// FIND (query)
db.products.find({
  price: { $lt: 100 },
  categories: "audio"
}).sort({ price: 1 });

// FIND with projection (select specific fields)
db.products.find(
  { brand: "AudioMax" },
  { name: 1, price: 1, _id: 0 }
);

// UPDATE
db.products.updateOne(
  { _id: ObjectId("664a1b2c3d4e5f6a7b8c9d0e") },
  {
    $set: { price: 69.99 },
    $push: { categories: "sale" }
  }
);

// DELETE
db.products.deleteMany({ in_stock: false });

// AGGREGATION PIPELINE
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $unwind: "$items" },
  { $group: {
      _id: "$items.product_id",
      total_revenue: { $sum: "$items.price" },
      total_sold: { $sum: "$items.quantity" }
  }},
  { $sort: { total_revenue: -1 } },
  { $limit: 10 }
]);
```

### MongoDB Schema Design Patterns

| Pattern | Description | Use When |
|---|---|---|
| **Embed** | Nest related data inside the document | 1:1 or 1:few relationships, always queried together |
| **Reference** | Store an ObjectId pointing to another collection | 1:many or many:many, independent access patterns |
| **Bucket** | Group time-series data into fixed-size docs | IoT readings, metrics (reduces document count) |
| **Subset** | Embed only frequently-accessed subset, full data elsewhere | Large arrays where only top-N shown |
| **Computed** | Pre-compute aggregates, store in document | Dashboards, counters to avoid expensive queries |

### Embed vs Reference Decision

```
Embed (denormalise):
┌─ User ───────────────────────┐
│ name: "Alice"                │
│ address: {                   │  ← Embedded subdocument
│   street: "123 Main St",    │
│   city: "Portland"          │
│ }                            │
│ orders: [                    │  ← Embedded array (if small)
│   { product: "Book", ... }, │
│   { product: "Pen", ... }   │
│ ]                            │
└──────────────────────────────┘

Reference (normalise):
┌─ User ──────────┐     ┌─ Order ─────────────┐
│ _id: 42          │     │ _id: 1001           │
│ name: "Alice"    │     │ user_id: 42         │ ← Reference
│ address_id: 99   │──►  │ product: "Book"     │
└──────────────────┘     │ total: 29.99        │
                         └─────────────────────┘
```

### MongoDB Indexing

```javascript
// Single field index
db.products.createIndex({ price: 1 });

// Compound index
db.products.createIndex({ brand: 1, price: -1 });

// Text index (full-text search)
db.products.createIndex({ name: "text", description: "text" });

// TTL index (auto-expire documents)
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 });

// Partial index (only index matching docs)
db.products.createIndex(
  { price: 1 },
  { partialFilterExpression: { in_stock: true } }
);
```

---

## 4. Key-Value Stores — Redis

### Data Model

Every piece of data is accessed via a unique **key**. The value can be a string, list, set, hash, sorted set, stream, or other data structure. No queries on the value — you must know the key.

### Redis Data Structures

<svg viewBox="0 0 750 330" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Redis Data Structures</text>

  <!-- String -->
  <rect x="20" y="50" width="210" height="80" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="125" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">String</text>
  <text x="125" y="90" text-anchor="middle" font-size="10" fill="#333">key → "value"</text>
  <text x="125" y="106" text-anchor="middle" font-size="9" fill="#555">GET, SET, INCR, DECR</text>
  <text x="125" y="122" text-anchor="middle" font-size="9" fill="#888">Cache, counters, flags</text>

  <!-- Hash -->
  <rect x="250" y="50" width="210" height="80" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="355" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Hash</text>
  <text x="355" y="90" text-anchor="middle" font-size="10" fill="#333">key → {field: value, ...}</text>
  <text x="355" y="106" text-anchor="middle" font-size="9" fill="#555">HGET, HSET, HGETALL</text>
  <text x="355" y="122" text-anchor="middle" font-size="9" fill="#888">User profiles, config</text>

  <!-- List -->
  <rect x="480" y="50" width="250" height="80" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="605" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">List</text>
  <text x="605" y="90" text-anchor="middle" font-size="10" fill="#333">key → [val1, val2, val3, ...]</text>
  <text x="605" y="106" text-anchor="middle" font-size="9" fill="#555">LPUSH, RPOP, LRANGE</text>
  <text x="605" y="122" text-anchor="middle" font-size="9" fill="#888">Queues, recent items, feeds</text>

  <!-- Set -->
  <rect x="20" y="150" width="210" height="80" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="125" y="172" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">Set</text>
  <text x="125" y="190" text-anchor="middle" font-size="10" fill="#333">key → {unique1, unique2, ...}</text>
  <text x="125" y="206" text-anchor="middle" font-size="9" fill="#555">SADD, SISMEMBER, SUNION</text>
  <text x="125" y="222" text-anchor="middle" font-size="9" fill="#888">Tags, unique visitors</text>

  <!-- Sorted Set -->
  <rect x="250" y="150" width="210" height="80" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="355" y="172" text-anchor="middle" font-size="12" font-weight="bold" fill="#c62828">Sorted Set</text>
  <text x="355" y="190" text-anchor="middle" font-size="10" fill="#333">key → {(score,val), ...}</text>
  <text x="355" y="206" text-anchor="middle" font-size="9" fill="#555">ZADD, ZRANGE, ZRANK</text>
  <text x="355" y="222" text-anchor="middle" font-size="9" fill="#888">Leaderboards, priority queues</text>

  <!-- Stream -->
  <rect x="480" y="150" width="250" height="80" rx="8" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="605" y="172" text-anchor="middle" font-size="12" font-weight="bold" fill="#f9a825">Stream</text>
  <text x="605" y="190" text-anchor="middle" font-size="10" fill="#333">key → append-only log</text>
  <text x="605" y="206" text-anchor="middle" font-size="9" fill="#555">XADD, XREAD, XREADGROUP</text>
  <text x="605" y="222" text-anchor="middle" font-size="9" fill="#888">Event sourcing, message bus</text>

  <!-- Performance note -->
  <rect x="80" y="255" width="590" height="55" rx="8" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="278" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Performance: All Operations O(1) or O(log N)</text>
  <text x="375" y="298" text-anchor="middle" font-size="10" fill="#555">In-memory storage → sub-millisecond latency. Optional disk persistence (RDB snapshots / AOF log).</text>
</svg>

### Redis Commands

```redis
-- Strings (caching)
SET user:42:name "Alice" EX 3600     -- expires in 1 hour
GET user:42:name                      -- "Alice"
INCR page_views:homepage              -- atomic counter

-- Hashes (user profile)
HSET user:42 name "Alice" email "alice@example.com" plan "pro"
HGET user:42 name                     -- "Alice"
HGETALL user:42                       -- all fields

-- Lists (job queue)
LPUSH queue:emails "job_123"          -- push left
RPOP queue:emails                     -- pop right (FIFO)
BRPOP queue:emails 30                 -- blocking pop (wait 30s)

-- Sets (unique visitors)
SADD visitors:2026-05-04 "user:42" "user:17" "user:99"
SCARD visitors:2026-05-04             -- count: 3
SISMEMBER visitors:2026-05-04 "user:42"  -- 1 (true)

-- Sorted Sets (leaderboard)
ZADD leaderboard 9500 "alice" 8700 "bob" 9200 "carol"
ZREVRANGE leaderboard 0 2 WITHSCORES -- top 3 descending
ZRANK leaderboard "bob"              -- rank (0-indexed)

-- Streams (event log)
XADD events:orders * product "laptop" amount "999.99"
XRANGE events:orders - +             -- read all entries
```

### Common Redis Use Cases

| Use Case | Data Structure | Pattern |
|---|---|---|
| **Session store** | String or Hash | `SET session:<token> <json> EX 1800` |
| **Cache** | String | `SET cache:<key> <value> EX 300` |
| **Rate limiter** | String + INCR | `INCR rate:<ip>:<minute>` + `EXPIRE` |
| **Leaderboard** | Sorted Set | `ZADD` for scores, `ZREVRANGE` for top-N |
| **Job queue** | List | `LPUSH` to enqueue, `BRPOP` to dequeue |
| **Pub/Sub** | Pub/Sub or Stream | `PUBLISH channel msg`, `SUBSCRIBE channel` |
| **Distributed lock** | String + NX | `SET lock:<resource> <owner> NX EX 10` |

---

## 5. Column-Family Stores — Cassandra

### Data Model

Data is organised into **tables** with a **partition key** (determines which node stores the data) and optional **clustering columns** (determine sort order within a partition). Each row can have different columns — a "wide row" model.

```
Partition Key    Clustering Key      Columns
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
user:42          2026-04-01 10:00    {event: "login",  ip: "10.0.1.5"}
user:42          2026-04-01 14:30    {event: "purchase", amount: 99.99}
user:42          2026-04-02 09:15    {event: "login",  ip: "10.0.1.5"}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
user:17          2026-04-01 11:00    {event: "signup", source: "google"}
```

### CQL (Cassandra Query Language)

```sql
-- Create keyspace (replication config)
CREATE KEYSPACE ecommerce
  WITH REPLICATION = {
    'class': 'NetworkTopologyStrategy',
    'us-east': 3,
    'eu-west': 3
  };

USE ecommerce;

-- Create table (partition key + clustering columns)
CREATE TABLE user_activity (
    user_id     UUID,
    event_time  TIMESTAMP,
    event_type  TEXT,
    details     MAP<TEXT, TEXT>,
    PRIMARY KEY ((user_id), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);

-- Insert
INSERT INTO user_activity (user_id, event_time, event_type, details)
VALUES (
    550e8400-e29b-41d4-a716-446655440000,
    '2026-04-15 10:30:00',
    'purchase',
    {'product': 'Laptop', 'amount': '999.99'}
);

-- Query (must include partition key!)
SELECT * FROM user_activity
 WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
   AND event_time > '2026-04-01'
 LIMIT 20;

-- ❌ This will FAIL (no partition key)
SELECT * FROM user_activity WHERE event_type = 'purchase';
-- Cassandra requires the partition key for efficient queries
```

### Cassandra Data Modeling Rules

| Rule | Reason |
|---|---|
| **Model tables around queries, not entities** | Each query pattern gets its own table |
| **Denormalise aggressively** | No JOINs — duplicate data across tables |
| **Partition key = access pattern** | All data for one query must be in one partition |
| **Clustering columns = sort order** | Data within a partition is sorted on disk |
| **Avoid large partitions** | Keep partitions under ~100 MB for performance |
| **Use materialized views or manual denorm for multiple access patterns** | Different queries need different table structures |

---

## 6. Graph Databases — Neo4j

### Data Model

Data is stored as **nodes** (entities), **relationships** (edges connecting nodes), and **properties** (key-value pairs on both nodes and relationships). Relationships are first-class citizens with their own types and properties.

<svg viewBox="0 0 700 280" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowG" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <path d="M0,0 L10,3.5 L0,7 Z" fill="#6a1b9a"/>
    </marker>
  </defs>

  <text x="350" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Graph Data Model — Social Network</text>

  <!-- Alice node -->
  <circle cx="120" cy="100" r="40" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="120" y="95" text-anchor="middle" font-size="11" font-weight="bold" fill="#1565c0">Alice</text>
  <text x="120" y="110" text-anchor="middle" font-size="9" fill="#555">:Person</text>

  <!-- Bob node -->
  <circle cx="350" cy="80" r="40" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="350" y="75" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">Bob</text>
  <text x="350" y="90" text-anchor="middle" font-size="9" fill="#555">:Person</text>

  <!-- Carol node -->
  <circle cx="580" cy="100" r="40" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="580" y="95" text-anchor="middle" font-size="11" font-weight="bold" fill="#e65100">Carol</text>
  <text x="580" y="110" text-anchor="middle" font-size="9" fill="#555">:Person</text>

  <!-- Neo4j node -->
  <rect x="285" y="200" width="130" height="50" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="350" y="222" text-anchor="middle" font-size="11" font-weight="bold" fill="#6a1b9a">Neo4j</text>
  <text x="350" y="238" text-anchor="middle" font-size="9" fill="#555">:Technology</text>

  <!-- Edges -->
  <line x1="160" y1="90" x2="305" y2="82" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowG)"/>
  <text x="225" y="76" text-anchor="middle" font-size="9" fill="#6a1b9a" font-weight="bold">FRIENDS_WITH</text>
  <text x="225" y="88" text-anchor="middle" font-size="8" fill="#888">since: 2020</text>

  <line x1="393" y1="85" x2="537" y2="95" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowG)"/>
  <text x="465" y="80" text-anchor="middle" font-size="9" fill="#6a1b9a" font-weight="bold">WORKS_WITH</text>

  <line x1="140" y1="138" x2="290" y2="205" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowG)"/>
  <text x="190" y="185" text-anchor="middle" font-size="9" fill="#6a1b9a" font-weight="bold">USES</text>
  <text x="190" y="197" text-anchor="middle" font-size="8" fill="#888">skill: "expert"</text>

  <line x1="370" y1="118" x2="360" y2="198" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowG)"/>
  <text x="400" y="165" text-anchor="middle" font-size="9" fill="#6a1b9a" font-weight="bold">USES</text>

  <line x1="570" y1="138" x2="420" y2="210" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowG)"/>
  <text x="520" y="185" text-anchor="middle" font-size="9" fill="#6a1b9a" font-weight="bold">USES</text>
</svg>

### Cypher Query Language (Neo4j)

```cypher
// Create nodes and relationships
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 28})
CREATE (neo4j:Technology {name: 'Neo4j', type: 'GraphDB'})
CREATE (alice)-[:FRIENDS_WITH {since: 2020}]->(bob)
CREATE (alice)-[:USES {skill: 'expert'}]->(neo4j)
CREATE (bob)-[:USES]->(neo4j);

// Find Alice's friends
MATCH (alice:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend)
RETURN friend.name;

// Friends of friends (2 hops)
MATCH (alice:Person {name: 'Alice'})-[:FRIENDS_WITH*2]->(fof)
WHERE fof <> alice
RETURN DISTINCT fof.name;

// Shortest path between two people
MATCH path = shortestPath(
  (alice:Person {name: 'Alice'})-[*]-(carol:Person {name: 'Carol'})
)
RETURN path;

// Recommendation: people who use the same technology
MATCH (alice:Person {name: 'Alice'})-[:USES]->(tech)<-[:USES]-(other)
WHERE other <> alice
RETURN other.name, tech.name, COUNT(*) AS shared_tech
ORDER BY shared_tech DESC;

// Fraud detection: circular money transfers
MATCH path = (a:Account)-[:TRANSFER*3..6]->(a)
WHERE ALL(r IN relationships(path) WHERE r.amount > 10000)
RETURN path;
```

### Why Graph Databases Excel

| Query Type | SQL (Relational) | Cypher (Graph) |
|---|---|---|
| Direct friends | 1 JOIN | 1 hop — fast |
| Friends of friends | 2 JOINs | 2 hops — still fast |
| 6 degrees of separation | 6 JOINs (exponential) | 6 hops — **constant time per hop** |
| Shortest path | Recursive CTE (slow) | Built-in algorithm |
| Pattern matching | Complex self-joins | Natural pattern syntax |

Graph databases use **index-free adjacency** — each node directly stores pointers to its neighbours. Traversal cost is proportional to the path length, not the total data size.

---

## 7. Time-Series Databases — InfluxDB

### Data Model

Time-series databases are optimised for data that arrives as a continuous stream of timestamped observations: metrics, sensor readings, financial ticks, logs.

```
Measurement   Tags (indexed)         Fields (values)           Timestamp
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
cpu_usage     host=web01,region=us   value=72.5,idle=27.5      2026-04-15T10:30:00Z
cpu_usage     host=web01,region=us   value=68.2,idle=31.8      2026-04-15T10:31:00Z
cpu_usage     host=web02,region=eu   value=45.1,idle=54.9      2026-04-15T10:30:00Z
```

### InfluxDB / Flux Queries

```flux
// Query: CPU usage for web01 in the last hour
from(bucket: "monitoring")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage" and r.host == "web01")
  |> mean()

// 5-minute moving average
from(bucket: "monitoring")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "cpu_usage")
  |> movingAverage(n: 5)

// Downsample: hourly aggregates
from(bucket: "monitoring")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "cpu_usage")
  |> aggregateWindow(every: 1h, fn: mean)
```

### TimescaleDB (PostgreSQL Extension)

```sql
-- Create a hypertable (time-partitioned table)
CREATE TABLE sensor_data (
    time        TIMESTAMPTZ NOT NULL,
    sensor_id   INT,
    temperature DOUBLE PRECISION,
    humidity    DOUBLE PRECISION
);

SELECT create_hypertable('sensor_data', 'time');

-- Insert (normal SQL)
INSERT INTO sensor_data VALUES
    (NOW(), 1, 22.5, 45.0),
    (NOW(), 2, 23.1, 42.3);

-- Time-bucket aggregation
SELECT time_bucket('1 hour', time) AS hour,
       sensor_id,
       AVG(temperature) AS avg_temp,
       MAX(temperature) AS max_temp
  FROM sensor_data
 WHERE time > NOW() - INTERVAL '24 hours'
 GROUP BY hour, sensor_id
 ORDER BY hour DESC;

-- Continuous aggregate (materialised view auto-refresh)
CREATE MATERIALIZED VIEW hourly_temps
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS hour,
       sensor_id,
       AVG(temperature) AS avg_temp
  FROM sensor_data
 GROUP BY hour, sensor_id;

-- Retention policy: drop data older than 30 days
SELECT add_retention_policy('sensor_data', INTERVAL '30 days');
```

### Time-Series DB Comparison

| Feature | InfluxDB | TimescaleDB | Prometheus |
|---|---|---|---|
| **Model** | Custom (tags + fields) | PostgreSQL extension | Pull-based metrics |
| **Query language** | Flux | SQL | PromQL |
| **Schema** | Schemaless | SQL schema | Fixed (metric name + labels) |
| **Retention** | Built-in policies | Built-in policies | Built-in |
| **Best for** | IoT, application metrics | When you need SQL + time-series | DevOps monitoring |
| **Joins** | Limited | Full SQL JOINs | No |

---

## 8. Data Modeling in NoSQL

### The Fundamental Shift

| Relational (SQL) | NoSQL |
|---|---|
| Model the **entities** (normalise) | Model the **queries** (denormalise) |
| Schema-first, queries adapt | Query-first, schema follows |
| Normalise to reduce redundancy | Duplicate to avoid joins |
| JOINs are cheap | JOINs are expensive or impossible |
| One schema fits many queries | One table per query pattern |

### Denormalisation Example

**Relational model** (normalised):
```
users: id, name, email
orders: id, user_id, total, status
order_items: id, order_id, product_id, quantity, price
products: id, name, category, price
```

**Document model** (denormalised for "get user's orders"):
```json
{
  "_id": "user:42",
  "name": "Alice",
  "email": "alice@example.com",
  "orders": [
    {
      "order_id": "ord:1001",
      "total": 149.97,
      "status": "shipped",
      "items": [
        { "product": "Wireless Mouse", "quantity": 1, "price": 29.99 },
        { "product": "Keyboard", "quantity": 1, "price": 79.99 },
        { "product": "USB Cable", "quantity": 2, "price": 19.99 }
      ]
    }
  ]
}
```

One read. No joins. Everything for the "user orders" page in a single document.

### Anti-Patterns to Avoid

| Anti-Pattern | Problem |
|---|---|
| **Using NoSQL as a relational DB** | Trying to normalise and join in app code |
| **Unbounded arrays** | Document grows beyond 16 MB limit (MongoDB) |
| **Not designing for queries** | Writing data without knowing read patterns |
| **Generic key names** | `key123` instead of `user:42:profile` — hard to manage |
| **Ignoring consistency needs** | Using AP database for financial data |

---

## 9. SQL vs NoSQL — Decision Guide

<svg viewBox="0 0 750 500" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">SQL vs NoSQL — Decision Flowchart</text>

  <!-- Start -->
  <rect x="280" y="45" width="190" height="35" rx="8" fill="#37474f"/>
  <text x="375" y="68" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">What does your data look like?</text>

  <!-- Branch: Structured -->
  <line x1="325" y1="80" x2="160" y2="110" stroke="#1565c0" stroke-width="2"/>
  <rect x="60" y="110" width="200" height="35" rx="6" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="160" y="132" text-anchor="middle" font-size="10" fill="#1565c0" font-weight="bold">Structured, relational?</text>

  <!-- Branch: Flexible -->
  <line x1="425" y1="80" x2="590" y2="110" stroke="#2e7d32" stroke-width="2"/>
  <rect x="490" y="110" width="200" height="35" rx="6" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="590" y="132" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">Flexible, hierarchical?</text>

  <!-- SQL Path -->
  <line x1="160" y1="145" x2="160" y2="170" stroke="#1565c0" stroke-width="1.5"/>
  <rect x="50" y="170" width="220" height="30" rx="6" fill="#e3f2fd" stroke="#1565c0"/>
  <text x="160" y="190" text-anchor="middle" font-size="10" fill="#333">Need ACID transactions?</text>

  <line x1="110" y1="200" x2="80" y2="225" stroke="#1565c0" stroke-width="1.5"/>
  <text x="75" y="218" font-size="9" fill="#1565c0">Yes</text>
  <rect x="20" y="225" width="140" height="40" rx="8" fill="#1565c0"/>
  <text x="90" y="243" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">PostgreSQL</text>
  <text x="90" y="258" text-anchor="middle" font-size="9" fill="#bbdefb">MySQL, SQL Server</text>

  <line x1="210" y1="200" x2="240" y2="225" stroke="#1565c0" stroke-width="1.5"/>
  <text x="235" y="218" font-size="9" fill="#1565c0">+ Global scale</text>
  <rect x="190" y="225" width="140" height="40" rx="8" fill="#0d47a1"/>
  <text x="260" y="243" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Spanner</text>
  <text x="260" y="258" text-anchor="middle" font-size="9" fill="#bbdefb">CockroachDB</text>

  <!-- NoSQL Path -->
  <line x1="590" y1="145" x2="590" y2="170" stroke="#2e7d32" stroke-width="1.5"/>
  <rect x="490" y="170" width="200" height="30" rx="6" fill="#e8f5e9" stroke="#2e7d32"/>
  <text x="590" y="190" text-anchor="middle" font-size="10" fill="#333">Primary access pattern?</text>

  <!-- Document -->
  <line x1="530" y1="200" x2="440" y2="240" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="450" y="230" font-size="9" fill="#2e7d32">Rich queries</text>
  <rect x="370" y="240" width="120" height="40" rx="8" fill="#2e7d32"/>
  <text x="430" y="258" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">MongoDB</text>
  <text x="430" y="272" text-anchor="middle" font-size="9" fill="#c8e6c9">Document store</text>

  <!-- Key-Value -->
  <line x1="590" y1="200" x2="590" y2="240" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="605" y="230" font-size="9" fill="#2e7d32">Key lookups</text>
  <rect x="530" y="240" width="120" height="40" rx="8" fill="#e65100"/>
  <text x="590" y="258" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Redis</text>
  <text x="590" y="272" text-anchor="middle" font-size="9" fill="#ffe0b2">Key-Value</text>

  <!-- Wide Column -->
  <line x1="650" y1="200" x2="700" y2="240" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="710" y="230" font-size="9" fill="#2e7d32">Write-heavy</text>
  <rect x="660" y="240" width="80" height="40" rx="8" fill="#c62828"/>
  <text x="700" y="258" text-anchor="middle" font-size="9" fill="#fff" font-weight="bold">Cassandra</text>
  <text x="700" y="272" text-anchor="middle" font-size="8" fill="#ffcdd2">Column-family</text>

  <!-- Special cases -->
  <rect x="100" y="310" width="550" height="30" rx="6" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="330" text-anchor="middle" font-size="11" fill="#333" font-weight="bold">Special Data Structures:</text>

  <rect x="50" y="355" width="160" height="45" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="130" y="375" text-anchor="middle" font-size="10" font-weight="bold" fill="#6a1b9a">Relationships?</text>
  <text x="130" y="392" text-anchor="middle" font-size="9" fill="#555">→ Neo4j (Graph)</text>

  <rect x="240" y="355" width="170" height="45" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="325" y="375" text-anchor="middle" font-size="10" font-weight="bold" fill="#c62828">Time-stamped data?</text>
  <text x="325" y="392" text-anchor="middle" font-size="9" fill="#555">→ TimescaleDB / InfluxDB</text>

  <rect x="440" y="355" width="160" height="45" rx="8" fill="#e0f2f1" stroke="#00695c" stroke-width="1.5"/>
  <text x="520" y="375" text-anchor="middle" font-size="10" font-weight="bold" fill="#00695c">Full-text search?</text>
  <text x="520" y="392" text-anchor="middle" font-size="9" fill="#555">→ Elasticsearch</text>

  <!-- Bottom note -->
  <rect x="100" y="420" width="550" height="55" rx="8" fill="#fff8e1" stroke="#f9a825" stroke-width="1.5"/>
  <text x="375" y="442" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">Golden Rule</text>
  <text x="375" y="460" text-anchor="middle" font-size="10" fill="#333">Start with PostgreSQL. Add specialised databases only when you have a specific</text>
  <text x="375" y="474" text-anchor="middle" font-size="10" fill="#333">problem that PostgreSQL can't solve at the required scale or latency.</text>
</svg>

### Comparison Table

| Factor | SQL (Relational) | Document | Key-Value | Column-Family | Graph |
|---|---|---|---|---|---|
| **Schema** | Fixed | Flexible | None | Semi-flexible | Property-based |
| **Query** | SQL (rich) | JSON queries | Get/Set by key | CQL (limited) | Cypher/Gremlin |
| **JOINs** | Native, optimised | Limited ($lookup) | None | None | Traversals |
| **Transactions** | Full ACID | Document-level ACID | Limited | Row-level | Node/edge ACID |
| **Scaling** | Vertical (mostly) | Horizontal (sharding) | Horizontal | Horizontal | Vertical (mostly) |
| **Best for** | General, OLTP | Variable schemas | Caching, speed | Write-heavy, IoT | Relationships |

---

## 10. Polyglot Persistence

### Concept

Use the **right database for each job** rather than forcing all data into one system. A modern application might use:

<svg viewBox="0 0 750 300" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowPP" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#666"/>
    </marker>
  </defs>

  <text x="375" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Polyglot Persistence — E-Commerce Platform</text>

  <!-- Application layer -->
  <rect x="240" y="42" width="270" height="35" rx="8" fill="#37474f"/>
  <text x="375" y="65" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">Application / API Layer</text>

  <!-- Lines from app to databases -->
  <line x1="290" y1="77" x2="100" y2="115" stroke="#666" stroke-width="1.5" marker-end="url(#arrowPP)"/>
  <line x1="345" y1="77" x2="280" y2="115" stroke="#666" stroke-width="1.5" marker-end="url(#arrowPP)"/>
  <line x1="375" y1="77" x2="375" y2="115" stroke="#666" stroke-width="1.5" marker-end="url(#arrowPP)"/>
  <line x1="405" y1="77" x2="470" y2="115" stroke="#666" stroke-width="1.5" marker-end="url(#arrowPP)"/>
  <line x1="460" y1="77" x2="630" y2="115" stroke="#666" stroke-width="1.5" marker-end="url(#arrowPP)"/>

  <!-- PostgreSQL -->
  <rect x="20" y="115" width="160" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="100" y="138" text-anchor="middle" font-size="11" font-weight="bold" fill="#1565c0">PostgreSQL</text>
  <text x="100" y="155" text-anchor="middle" font-size="9" fill="#333">Orders, payments</text>
  <text x="100" y="168" text-anchor="middle" font-size="9" fill="#333">Inventory, users</text>
  <text x="100" y="183" text-anchor="middle" font-size="8" fill="#888">ACID transactions</text>

  <!-- MongoDB -->
  <rect x="200" y="115" width="160" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="280" y="138" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">MongoDB</text>
  <text x="280" y="155" text-anchor="middle" font-size="9" fill="#333">Product catalog</text>
  <text x="280" y="168" text-anchor="middle" font-size="9" fill="#333">Reviews, CMS</text>
  <text x="280" y="183" text-anchor="middle" font-size="8" fill="#888">Flexible schema</text>

  <!-- Redis -->
  <rect x="310" y="115" width="130" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="375" y="138" text-anchor="middle" font-size="11" font-weight="bold" fill="#e65100">Redis</text>
  <text x="375" y="155" text-anchor="middle" font-size="9" fill="#333">Sessions, cache</text>
  <text x="375" y="168" text-anchor="middle" font-size="9" fill="#333">Rate limiting</text>
  <text x="375" y="183" text-anchor="middle" font-size="8" fill="#888">Sub-ms latency</text>

  <!-- Elasticsearch -->
  <rect x="395" y="115" width="150" height="80" rx="10" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="470" y="138" text-anchor="middle" font-size="11" font-weight="bold" fill="#f9a825">Elasticsearch</text>
  <text x="470" y="155" text-anchor="middle" font-size="9" fill="#333">Product search</text>
  <text x="470" y="168" text-anchor="middle" font-size="9" fill="#333">Log analytics</text>
  <text x="470" y="183" text-anchor="middle" font-size="8" fill="#888">Full-text search</text>

  <!-- Neo4j -->
  <rect x="555" y="115" width="155" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="632" y="138" text-anchor="middle" font-size="11" font-weight="bold" fill="#6a1b9a">Neo4j</text>
  <text x="632" y="155" text-anchor="middle" font-size="9" fill="#333">Recommendations</text>
  <text x="632" y="168" text-anchor="middle" font-size="9" fill="#333">Fraud detection</text>
  <text x="632" y="183" text-anchor="middle" font-size="8" fill="#888">Relationship queries</text>

  <!-- CDC sync -->
  <rect x="100" y="215" width="550" height="30" rx="6" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="235" text-anchor="middle" font-size="10" fill="#555">CDC (Change Data Capture) / Event Streaming (Kafka) keeps databases in sync</text>

  <!-- Trade-offs -->
  <rect x="60" y="260" width="630" height="30" rx="6" fill="#ffebee" stroke="#c62828" stroke-width="1"/>
  <text x="375" y="280" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">Trade-off: operational complexity increases with each database. Only add one when the benefit justifies the cost.</text>
</svg>

---

## 11. Worked Examples

### Example 1 — Beginner: MongoDB CRUD for a Blog

```javascript
// Blog post collection
db.posts.insertOne({
  title: "Getting Started with MongoDB",
  slug: "getting-started-mongodb",
  author: { name: "Alice", email: "alice@blog.com" },
  content: "MongoDB is a document database...",
  tags: ["mongodb", "nosql", "tutorial"],
  published: true,
  views: 0,
  comments: [],
  created_at: new Date(),
  updated_at: new Date()
});

// Find published posts by tag, sorted by date
db.posts.find(
  { published: true, tags: "mongodb" }
).sort({ created_at: -1 }).limit(10);

// Add a comment (push to array)
db.posts.updateOne(
  { slug: "getting-started-mongodb" },
  {
    $push: { comments: {
      user: "bob",
      text: "Great article!",
      date: new Date()
    }},
    $inc: { views: 1 },
    $set: { updated_at: new Date() }
  }
);

// Aggregation: top 5 tags by post count
db.posts.aggregate([
  { $match: { published: true } },
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 5 }
]);
```

### Example 2 — Beginner: Redis Caching Pattern

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_user_profile(user_id: int) -> dict:
    cache_key = f"user:{user_id}:profile"

    # Check cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss — query database
    profile = db.query("SELECT * FROM users WHERE id = %s", (user_id,))

    # Store in cache (expire in 5 minutes)
    r.setex(cache_key, 300, json.dumps(profile))

    return profile

def invalidate_user_cache(user_id: int):
    r.delete(f"user:{user_id}:profile")

# Rate limiting
def check_rate_limit(ip: str, max_requests: int = 100, window: int = 60) -> bool:
    key = f"rate:{ip}:{int(time.time()) // window}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)
    return count <= max_requests
```

### Example 3 — Intermediate: Cassandra Time-Series for IoT

```sql
-- Sensor readings table
CREATE TABLE sensor_readings (
    sensor_id    UUID,
    day          DATE,
    reading_time TIMESTAMP,
    temperature  DOUBLE,
    humidity     DOUBLE,
    pressure     DOUBLE,
    PRIMARY KEY ((sensor_id, day), reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC)
  AND default_time_to_live = 2592000;  -- 30-day retention

-- Insert a reading
INSERT INTO sensor_readings (sensor_id, day, reading_time, temperature, humidity, pressure)
VALUES (
    550e8400-e29b-41d4-a716-446655440000,
    '2026-05-04',
    '2026-05-04 14:30:00+00',
    22.5, 45.0, 1013.25
);

-- Query: last 24 hours for a sensor (efficient — single partition per day)
SELECT reading_time, temperature, humidity, pressure
  FROM sensor_readings
 WHERE sensor_id = 550e8400-e29b-41d4-a716-446655440000
   AND day = '2026-05-04'
   AND reading_time > '2026-05-04 00:00:00+00'
 LIMIT 1440;

-- Compound partition key (sensor_id + day) keeps partitions bounded:
-- ~1440 readings/day (1 per minute) per sensor — well under 100 MB
```

### Example 4 — Intermediate: Neo4j Social Recommendations

```cypher
// Setup: social network
CREATE (alice:User {name: 'Alice', city: 'Portland'})
CREATE (bob:User {name: 'Bob', city: 'Portland'})
CREATE (carol:User {name: 'Carol', city: 'Seattle'})
CREATE (dave:User {name: 'Dave', city: 'Portland'})
CREATE (eve:User {name: 'Eve', city: 'Seattle'})

CREATE (alice)-[:FOLLOWS]->(bob)
CREATE (alice)-[:FOLLOWS]->(carol)
CREATE (bob)-[:FOLLOWS]->(dave)
CREATE (carol)-[:FOLLOWS]->(dave)
CREATE (carol)-[:FOLLOWS]->(eve)
CREATE (dave)-[:FOLLOWS]->(eve);

// "People you may know" — friends of friends that you don't already follow
MATCH (me:User {name: 'Alice'})-[:FOLLOWS]->()-[:FOLLOWS]->(suggestion)
WHERE NOT (me)-[:FOLLOWS]->(suggestion)
  AND suggestion <> me
RETURN suggestion.name, COUNT(*) AS mutual_connections
ORDER BY mutual_connections DESC;

// Result:
// | suggestion.name | mutual_connections |
// |---|---|
// | Dave | 2 |  (via Bob AND Carol)
// | Eve  | 1 |  (via Carol)

// "Users in your city who share interests"
MATCH (me:User {name: 'Alice'})-[:LIKES]->(interest)<-[:LIKES]-(other:User)
WHERE other.city = me.city AND other <> me
RETURN other.name, COLLECT(interest.name) AS shared_interests
ORDER BY SIZE(shared_interests) DESC;
```

### Example 5 — Intermediate: Redis Leaderboard

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Add/update player scores
r.zadd("game:leaderboard", {
    "alice": 9500,
    "bob": 8700,
    "carol": 9200,
    "dave": 7800,
    "eve": 9800
})

# Get top 3 players (descending)
top3 = r.zrevrange("game:leaderboard", 0, 2, withscores=True)
# [('eve', 9800.0), ('alice', 9500.0), ('carol', 9200.0)]

# Get a player's rank (0-indexed, highest first)
rank = r.zrevrank("game:leaderboard", "bob")  # 3 (4th place)

# Get player's score
score = r.zscore("game:leaderboard", "alice")  # 9500.0

# Increment a score (atomic)
r.zincrby("game:leaderboard", 500, "bob")  # bob: 8700 → 9200

# Get players between score range
mid_tier = r.zrangebyscore("game:leaderboard", 8000, 9000, withscores=True)

# Get total number of players
total = r.zcard("game:leaderboard")  # 5

# Get rank with surrounding players (for "your position" UI)
def get_player_context(player, context=2):
    rank = r.zrevrank("game:leaderboard", player)
    start = max(0, rank - context)
    end = rank + context
    return r.zrevrange("game:leaderboard", start, end, withscores=True)
```

### Example 6 — Advanced: Polyglot Persistence — Order Pipeline

```python
# E-commerce order pipeline using multiple databases

class OrderService:
    def __init__(self):
        self.pg = PostgresConnection()    # ACID for orders
        self.mongo = MongoConnection()    # Product catalog
        self.redis = RedisConnection()    # Cache + inventory locks
        self.es = ElasticsearchClient()   # Order search

    def place_order(self, user_id: int, cart_items: list) -> dict:
        # 1. Check inventory in Redis (fast, cached from PG)
        for item in cart_items:
            stock = self.redis.get(f"stock:{item['product_id']}")
            if int(stock or 0) < item['quantity']:
                raise InsufficientStock(item['product_id'])

        # 2. Acquire distributed locks (prevent overselling)
        locks = []
        for item in cart_items:
            lock_key = f"lock:stock:{item['product_id']}"
            lock = self.redis.set(lock_key, user_id, nx=True, ex=30)
            if not lock:
                self._release_locks(locks)
                raise ConcurrentOrderConflict()
            locks.append(lock_key)

        try:
            # 3. Get product details from MongoDB (flexible catalog)
            products = {}
            for item in cart_items:
                products[item['product_id']] = self.mongo.products.find_one(
                    {"_id": item['product_id']}
                )

            # 4. Create order in PostgreSQL (ACID transaction)
            with self.pg.transaction() as txn:
                order = txn.execute("""
                    INSERT INTO orders (user_id, total, status)
                    VALUES (%s, %s, 'confirmed')
                    RETURNING id
                """, (user_id, sum(
                    products[i['product_id']]['price'] * i['quantity']
                    for i in cart_items
                )))

                for item in cart_items:
                    txn.execute("""
                        INSERT INTO order_items (order_id, product_id, quantity, price)
                        VALUES (%s, %s, %s, %s)
                    """, (order['id'], item['product_id'],
                          item['quantity'], products[item['product_id']]['price']))

                    # Decrement inventory in PG (source of truth)
                    txn.execute("""
                        UPDATE inventory SET quantity = quantity - %s
                        WHERE product_id = %s AND quantity >= %s
                    """, (item['quantity'], item['product_id'], item['quantity']))

            # 5. Update Redis cache
            for item in cart_items:
                self.redis.decrby(f"stock:{item['product_id']}", item['quantity'])

            # 6. Index in Elasticsearch (async, for search)
            self.es.index(index="orders", body={
                "order_id": order['id'],
                "user_id": user_id,
                "products": [p['name'] for p in products.values()],
                "total": order['total'],
                "created_at": datetime.utcnow()
            })

            return {"order_id": order['id'], "status": "confirmed"}

        finally:
            self._release_locks(locks)
```

---

## 12. Common Interview Questions

### Q1: What is NoSQL and why was it created?

**Answer:** NoSQL ("Not Only SQL") is a category of databases that don't use the traditional relational table model. They were created to address limitations of relational databases at massive scale: horizontal scalability (sharding built-in), flexible schemas (no rigid ALTER TABLE), high write throughput (no single leader bottleneck), and specific data model fit (documents, graphs, time-series). The rise of web-scale applications (Google, Amazon, Facebook) drove the need for databases that could handle billions of reads/writes per day across thousands of servers.

### Q2: What are the main types of NoSQL databases?

**Answer:**
- **Document stores** (MongoDB, CouchDB): JSON-like documents, flexible schema, rich queries
- **Key-value stores** (Redis, DynamoDB): Simple key→value lookup, blazing fast, limited querying
- **Column-family stores** (Cassandra, HBase): Wide rows with dynamic columns, optimised for write-heavy workloads
- **Graph databases** (Neo4j, Neptune): Nodes + edges, optimised for relationship traversal
- **Time-series** (InfluxDB, TimescaleDB): Timestamp-indexed, high ingest rate, retention policies

### Q3: When would you choose MongoDB over PostgreSQL?

**Answer:** Choose MongoDB when:
- Your data is naturally hierarchical/nested (product catalogs with variable attributes)
- Schema evolves rapidly (early-stage startup, prototyping)
- You need horizontal scaling with built-in sharding
- Your query patterns align with document access (fetch full document by ID)
- You don't need complex multi-table JOINs or multi-document ACID transactions

Stick with PostgreSQL when you need strict ACID, complex JOINs, or your data is highly relational. PostgreSQL's JSONB column type can also handle semi-structured data, often eliminating the need for MongoDB.

### Q4: How is data modeled differently in NoSQL vs SQL?

**Answer:** In SQL, you model **entities** (normalise into tables, eliminate redundancy, use JOINs). In NoSQL, you model **queries** (denormalise, embed related data together, duplicate data across tables). This is called "query-driven design." Example: instead of `users`, `orders`, and `order_items` tables joined at query time, you embed orders with their items inside the user document — one read returns everything for the user's order page.

### Q5: What is Redis and when should you use it?

**Answer:** Redis is an in-memory key-value store supporting strings, hashes, lists, sets, sorted sets, and streams. It provides sub-millisecond latency for reads and writes. Use it for:
- **Caching** (database query results, API responses)
- **Session storage** (fast, TTL-based expiry)
- **Rate limiting** (INCR + EXPIRE per IP/user)
- **Leaderboards** (sorted sets with scores)
- **Job queues** (lists with BRPOP for blocking dequeue)
- **Distributed locks** (SET NX EX)
- **Pub/Sub** messaging

Don't use it as your primary database unless data loss is acceptable — it's in-memory first.

### Q6: What is Cassandra's partition key and why does it matter?

**Answer:** The partition key determines **which node** stores a row. Cassandra hashes the partition key to assign data to a specific node in the cluster. Every query **must** include the partition key — without it, Cassandra would need to scan all nodes (a full cluster scan). This means you must design tables around your query patterns: one table per query, with the partition key matching the `WHERE` clause. The clustering key then determines sort order within the partition.

### Q7: Why are graph databases better than SQL for relationship-heavy queries?

**Answer:** Graph databases use **index-free adjacency** — each node stores direct pointers to its neighbours. Traversing a relationship is a pointer dereference (O(1)), not a table lookup. In SQL, each hop requires a JOIN (O(n) or O(n log n) with indexes). For deep traversals (friends-of-friends at 6 degrees), SQL performance degrades exponentially while graph performance scales linearly with path length. Graph DBs also offer pattern matching (Cypher's `MATCH`) which is far more expressive than recursive CTEs for relationship queries.

### Q8: What is polyglot persistence?

**Answer:** Using multiple database technologies in a single application, each chosen for its strength: PostgreSQL for ACID transactions, Redis for caching, Elasticsearch for search, Neo4j for recommendations, Cassandra for time-series ingestion. Data flows between systems via CDC (Change Data Capture) or event streaming (Kafka). Trade-off: better fit per workload, but increased operational complexity — more systems to deploy, monitor, backup, and keep in sync.

### Q9: How does MongoDB handle transactions?

**Answer:** MongoDB supports **multi-document ACID transactions** since version 4.0 (single replica set) and 4.2 (sharded clusters). Within a transaction, you can read and write across multiple documents and collections with snapshot isolation. However, transactions are more expensive than single-document operations and should be used sparingly. MongoDB's best performance comes from designing documents so that a single operation (insert/update one document) accomplishes the work — leveraging the fact that single-document operations are inherently atomic.

### Q10: What are the trade-offs of denormalisation in NoSQL?

**Answer:**
- **Pro**: Fast reads (no joins), data locality (one query fetches everything), horizontal scalability
- **Con**: Data duplication (same info in multiple places), update complexity (change in one place must be propagated to all copies), storage cost, risk of inconsistency if updates are missed

The trade-off is: you optimise for **read performance** at the cost of **write complexity**. This makes sense when reads vastly outnumber writes (which is true for most web applications).

### Q11: How does Cassandra achieve high write throughput?

**Answer:** Cassandra's write path is optimised for speed:
1. Write goes to an **in-memory memtable** (very fast)
2. Simultaneously written to a **commit log** on disk (sequential write — fast)
3. When the memtable fills, it's flushed to disk as an **SSTable** (sorted string table)
4. SSTables are merged in the background via **compaction**

Writes never require reading existing data (no read-before-write). This append-only, log-structured design gives Cassandra write throughput that can exceed 100k writes/second per node.

### Q12: Can NoSQL databases replace SQL databases entirely?

**Answer:** No — and they shouldn't try. Each has strengths:
- **SQL**: Complex queries, JOINs, ACID transactions, ad-hoc analytics, data integrity constraints
- **NoSQL**: Specific access patterns, massive scale, flexible schemas, specialised data models

Most production systems use **both** (polyglot persistence). The "start with PostgreSQL" advice is sound — it handles 90% of use cases. Add NoSQL databases when you hit a specific problem: sub-ms caching (Redis), flexible catalog (MongoDB), massive write ingest (Cassandra), graph traversals (Neo4j). Never choose NoSQL just because it's trendy.

---

## 13. Key Takeaways

<svg viewBox="0 0 750 520" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">Phase 19 — Key Takeaways</text>

  <!-- Card 1 -->
  <rect x="20" y="50" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="40" y="75" font-size="13" font-weight="bold" fill="#1565c0">NoSQL = Right Tool for the Job</text>
  <text x="40" y="95" font-size="11" fill="#333">Not a replacement for SQL. A complement</text>
  <text x="40" y="112" font-size="11" fill="#333">for specific workloads and data shapes.</text>

  <!-- Card 2 -->
  <rect x="390" y="50" width="340" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="410" y="75" font-size="13" font-weight="bold" fill="#2e7d32">Documents for Flexibility</text>
  <text x="410" y="95" font-size="11" fill="#333">MongoDB: nested JSON, variable schema.</text>
  <text x="410" y="112" font-size="11" fill="#333">Embed for speed, reference for scale.</text>

  <!-- Card 3 -->
  <rect x="20" y="150" width="340" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="40" y="175" font-size="13" font-weight="bold" fill="#e65100">Key-Value for Speed</text>
  <text x="40" y="195" font-size="11" fill="#333">Redis: sub-ms latency, in-memory.</text>
  <text x="40" y="212" font-size="11" fill="#333">Caching, sessions, leaderboards, queues.</text>

  <!-- Card 4 -->
  <rect x="390" y="150" width="340" height="80" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="410" y="175" font-size="13" font-weight="bold" fill="#c62828">Column-Family for Write Scale</text>
  <text x="410" y="195" font-size="11" fill="#333">Cassandra: partition key + clustering.</text>
  <text x="410" y="212" font-size="11" fill="#333">Design tables for queries, not entities.</text>

  <!-- Card 5 -->
  <rect x="20" y="250" width="340" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="40" y="275" font-size="13" font-weight="bold" fill="#6a1b9a">Graphs for Relationships</text>
  <text x="40" y="295" font-size="11" fill="#333">Neo4j: index-free adjacency, Cypher.</text>
  <text x="40" y="312" font-size="11" fill="#333">O(1) per hop vs exponential JOINs.</text>

  <!-- Card 6 -->
  <rect x="390" y="250" width="340" height="80" rx="10" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="410" y="275" font-size="13" font-weight="bold" fill="#f9a825">Time-Series for Metrics</text>
  <text x="410" y="295" font-size="11" fill="#333">InfluxDB / TimescaleDB: high ingest,</text>
  <text x="410" y="312" font-size="11" fill="#333">automatic retention, downsampling.</text>

  <!-- Card 7 -->
  <rect x="20" y="350" width="340" height="80" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="40" y="375" font-size="13" font-weight="bold" fill="#00695c">Model Queries, Not Entities</text>
  <text x="40" y="395" font-size="11" fill="#333">Denormalise. Embed. Duplicate.</text>
  <text x="40" y="412" font-size="11" fill="#333">One read should answer one question.</text>

  <!-- Card 8 -->
  <rect x="390" y="350" width="340" height="80" rx="10" fill="#ffccbc" stroke="#bf360c" stroke-width="2"/>
  <text x="410" y="375" font-size="13" font-weight="bold" fill="#bf360c">Start with PostgreSQL</text>
  <text x="410" y="395" font-size="11" fill="#333">Add specialised DBs only when you hit</text>
  <text x="410" y="412" font-size="11" fill="#333">a real scale, latency, or model problem.</text>

  <!-- Decision guide -->
  <rect x="80" y="455" width="590" height="50" rx="10" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="375" y="478" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Decision Guide</text>
  <text x="375" y="496" text-anchor="middle" font-size="10" fill="#555">Flexible docs → MongoDB | Speed → Redis | Write-heavy → Cassandra | Relationships → Neo4j | Metrics → TimescaleDB</text>
</svg>

---

## 14. Download

<a href="19_nosql.md" download="19_nosql.md" style="display:inline-block;padding:14px 28px;background:linear-gradient(135deg,#e65100,#bf360c);color:#fff;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;box-shadow:0 4px 15px rgba(230,81,0,0.3);margin:10px 0;">Download 19_nosql.md</a>
