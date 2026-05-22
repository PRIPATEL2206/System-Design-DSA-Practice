# Phase 1 — Introduction to Database Management Systems (DBMS)

> **Goal**: Understand what a DBMS is, why it replaced file systems, the different families of databases, how the ANSI/SPARC 3-tier architecture works, what data independence means, and how the major production systems compare.

---

## Table of Contents

1. [What is a Database?](#1-what-is-a-database)
2. [What is a DBMS?](#2-what-is-a-dbms)
3. [Brief History of Databases](#3-brief-history-of-databases)
4. [DBMS vs File System](#4-dbms-vs-file-system)
5. [Components of a DBMS](#5-components-of-a-dbms)
6. [Types of DBMS](#6-types-of-dbms)
7. [ANSI/SPARC 3-Tier Architecture](#7-ansisparc-3-tier-architecture)
8. [Data Independence](#8-data-independence)
9. [Database Users and Roles](#9-database-users-and-roles)
10. [Popular DBMS Systems — Deep Comparison](#10-popular-dbms-systems--deep-comparison)
11. [Worked Examples — Beginner, Intermediate, Advanced](#11-worked-examples--beginner-intermediate-advanced)
12. [Common Interview Questions](#12-common-interview-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. What is a Database?

### Definition

A **database** is an organized collection of logically related data, designed to serve the information needs of one or more applications.

> **Real-world analogy — Phone contacts:**  
> Your phone's contacts list is a tiny database. Each contact is a *record*, each field (name, phone number, email) is an *attribute*, and the list itself is a *table*. If you needed contacts for an entire organization with departments, managers, and shared resources, you'd need something far more structured — a DBMS.

### What makes "data" a "database"?

Raw data becomes a database when it has:

| Property | Meaning | Example |
|---|---|---|
| **Structure** | Data follows a defined format | Every student record has `id`, `name`, `dept` |
| **Relationships** | Data items are connected meaningfully | Student ↔ Course ↔ Instructor |
| **Integrity** | Rules prevent invalid data | GPA cannot be 5.0 on a 4.0 scale |
| **Persistence** | Data survives program termination | Saved to disk, not just in RAM |

---

## 2. What is a DBMS?

### Definition

A **Database Management System (DBMS)** is a software system that enables users and applications to **define, create, maintain, query, and control access** to a database.

It acts as an intermediary — a "gatekeeper" — between users/programs and the raw data on disk.

### Real-World Analogies

| Analogy | Without DBMS | With DBMS |
|---|---|---|
| **Library** | Books scattered on the floor. Want one? Search every corner. | Catalogued by title, author, subject. Librarian (DBMS) fetches it in seconds. |
| **Bank vault** | Money in boxes with no labels. Anyone walks in. No log of who took what. | Every deposit/withdrawal logged, access controlled, records audited. |
| **Hospital** | Patient records in random folders. Doctors lose files, duplicates everywhere. | Central system: any doctor retrieves records, no duplicates, controlled access. |

### What Does a DBMS Provide?

| Capability | Description | Without DBMS |
|---|---|---|
| **Data Definition (DDL)** | Define tables, columns, types, constraints | Manually write file format specs |
| **Data Manipulation (DML)** | INSERT, UPDATE, DELETE records | Write custom file-handling code |
| **Data Querying (DQL)** | Complex retrieval with SQL | Write custom search/filter logic per query |
| **Access Control** | GRANT/REVOKE permissions per user/role | OS-level file permissions only |
| **Concurrency Control** | Multiple users safely read/write simultaneously | File locking or corruption |
| **Transaction Management** | All-or-nothing operations (ACID) | Partial writes on crash |
| **Crash Recovery** | Automatic restoration after failures | Manual rebuild from backups |
| **Data Integrity** | Enforce rules (PK, FK, CHECK, UNIQUE) | Manually validate in every app |
| **Backup & Restore** | Scheduled backups, point-in-time recovery | Manual copy of files |

---

## 3. Brief History of Databases

Understanding where databases came from helps you see *why* each generation of systems was built.

| Era | Decade | Model | Key Idea | Example Systems |
|---|---|---|---|---|
| **File-based** | 1950s–60s | Flat files | Application owns its data | COBOL file systems |
| **Hierarchical** | 1960s | Tree (parent-child) | One-to-many only | IBM IMS (still runs banking!) |
| **Network** | 1960s–70s | Graph (CODASYL) | Many-to-many | IDMS |
| **Relational** | 1970s–now | Tables + SQL | Data independence, set theory | Oracle, PostgreSQL, MySQL |
| **Object-oriented** | 1980s–90s | Objects + methods | Complex data types | ObjectDB, db4o |
| **NoSQL** | 2000s–now | Various (doc, KV, graph) | Horizontal scale, schema-free | MongoDB, Redis, Neo4j |
| **NewSQL** | 2010s–now | Distributed SQL | ACID + horizontal scale | Spanner, CockroachDB |

### The Relational Revolution

In 1970, **Edgar F. Codd** at IBM published *"A Relational Model of Data for Large Shared Data Banks"* — the paper that founded relational databases. His key insight: separate the logical description of data from its physical storage. This idea is the foundation of **data independence**, which we'll cover in Section 8.

---

## 4. DBMS vs File System

### The File System Approach

In a file system, **each application owns its own data files**. There is no shared manager.

```
┌──────────┐     ┌────────────────┐
│  App A   │────▶│ students.txt   │
│ (Python) │────▶│ courses.txt    │
└──────────┘     └────────────────┘

┌──────────┐     ┌────────────────┐
│  App B   │────▶│ students.csv   │  ← same data, different file!
│  (Java)  │────▶│ grades.dat     │
└──────────┘     └────────────────┘
```

### Problems with File Systems (Detailed)

#### Problem 1: Data Redundancy and Inconsistency
The same student record exists in `students.txt` AND `students.csv`. When Alice changes her department, App A updates its file but App B doesn't — now the two files disagree.

```
# App A's file (updated):
101, Alice, Physics, 3.9

# App B's file (stale):
101, Alice, CS, 3.9         ← INCONSISTENT!
```

#### Problem 2: Difficulty in Accessing Data
Want to find "all CS students with GPA > 3.5 enrolled in more than 3 courses"? In a file system, you write a custom program for **every new query**. There's no SQL, no query optimizer.

```python
# File system: custom code for every query
results = []
with open("students.txt") as f:
    for line in f:
        fields = line.strip().split(",")
        if fields[2] == "CS" and float(fields[3]) > 3.5:
            # now cross-reference with enrollments file...
            with open("enrollments.txt") as e:
                count = sum(1 for row in e if row.startswith(fields[0]))
            if count > 3:
                results.append(fields)
```

vs. DBMS:
```sql
-- DBMS: one declarative query
SELECT s.name, s.gpa, COUNT(e.course_id) AS courses
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
WHERE s.dept = 'CS' AND s.gpa > 3.5
GROUP BY s.student_id, s.name, s.gpa
HAVING COUNT(e.course_id) > 3;
```

#### Problem 3: Data Isolation
Data is scattered across multiple files in multiple formats (`.txt`, `.csv`, `.dat`, `.xml`). Joining them requires writing custom parsing logic for each format.

#### Problem 4: Integrity Problems
Suppose a rule says "GPA must be between 0.0 and 4.0". In a file system, **every application** must independently check this rule. If one forgets, invalid data enters the system.

```
# Oops, App B didn't validate:
101, Alice, CS, 5.3     ← invalid GPA, but nobody stopped it
```

In a DBMS:
```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    dept       VARCHAR(50),
    gpa        DECIMAL(3,2) CHECK (gpa >= 0.0 AND gpa <= 4.0)
);
-- Now ANY insert/update from ANY application is validated automatically
```

#### Problem 5: Atomicity Problems
"Transfer $500 from Account A to Account B." In a file system, if the system crashes after debiting A but before crediting B, $500 vanishes. A DBMS wraps both operations in a **transaction** — either both happen or neither does.

#### Problem 6: Concurrent Access Anomalies
Two users try to withdraw from the same bank account at the same time. Without concurrency control, both read the same balance and both succeed — the bank loses money.

#### Problem 7: Security Problems
File systems offer only OS-level security: read/write/execute per user. You can't say "User X can see student names but not their GPAs." A DBMS supports column-level, row-level, and role-based security.

### Head-to-Head Comparison

| Criterion | File System | DBMS |
|---|---|---|
| **Data redundancy** | High (duplicated across apps) | Low (single source of truth) |
| **Data consistency** | Hard to maintain | Enforced by constraints |
| **Query capability** | Custom code per query | SQL (declarative, optimized) |
| **Concurrency** | None or manual locking | Built-in (locks, MVCC) |
| **Security** | OS-level file permissions | GRANT/REVOKE, RLS, RBAC |
| **Crash recovery** | Manual | Automatic (WAL, checkpoints) |
| **Data integrity** | Every app validates independently | Centralized constraints |
| **Data independence** | None | Logical + Physical |
| **Cost** | Free (just files) | Software cost, DBA needed |
| **Overhead** | Minimal | Memory, CPU for DBMS engine |

> **When is a file system still appropriate?** For simple, single-user, read-mostly data (config files, logs, small datasets). When multiple applications share data, need queries, or require reliability — use a DBMS.

### SVG: DBMS vs File System

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 400" font-family="Segoe UI, Arial, sans-serif">
  <rect width="800" height="400" fill="#f8f9fa" rx="12"/>

  <!-- File System Side -->
  <rect x="20" y="20" width="360" height="360" fill="#fff3f3" rx="10" stroke="#e74c3c" stroke-width="2"/>
  <text x="200" y="50" text-anchor="middle" font-size="16" font-weight="bold" fill="#c0392b">FILE SYSTEM</text>

  <rect x="60" y="70" width="110" height="40" rx="6" fill="#e74c3c" opacity="0.85"/>
  <text x="115" y="94" text-anchor="middle" fill="white" font-size="13">App A (Python)</text>
  <rect x="230" y="70" width="110" height="40" rx="6" fill="#e74c3c" opacity="0.85"/>
  <text x="285" y="94" text-anchor="middle" fill="white" font-size="13">App B (Java)</text>

  <line x1="100" y1="110" x2="70" y2="150" stroke="#e74c3c" stroke-width="1.5"/>
  <line x1="130" y1="110" x2="140" y2="150" stroke="#e74c3c" stroke-width="1.5"/>
  <line x1="130" y1="110" x2="210" y2="150" stroke="#e74c3c" stroke-width="1.5" stroke-dasharray="4,3"/>
  <line x1="270" y1="110" x2="210" y2="150" stroke="#e74c3c" stroke-width="1.5"/>
  <line x1="300" y1="110" x2="290" y2="150" stroke="#e74c3c" stroke-width="1.5"/>

  <rect x="35" y="150" width="90" height="35" rx="4" fill="#fadbd8" stroke="#e6b0aa"/>
  <text x="80" y="172" text-anchor="middle" font-size="11" fill="#922b21">students.txt</text>
  <rect x="135" y="150" width="90" height="35" rx="4" fill="#fadbd8" stroke="#e6b0aa"/>
  <text x="180" y="172" text-anchor="middle" font-size="11" fill="#922b21">courses.csv</text>
  <rect x="250" y="150" width="90" height="35" rx="4" fill="#fadbd8" stroke="#e6b0aa"/>
  <text x="295" y="172" text-anchor="middle" font-size="11" fill="#922b21">grades.dat</text>

  <text x="55" y="220" font-size="12" fill="#c0392b">&#x2717; Data scattered across files</text>
  <text x="55" y="242" font-size="12" fill="#c0392b">&#x2717; Redundancy &amp; inconsistency</text>
  <text x="55" y="264" font-size="12" fill="#c0392b">&#x2717; No query language</text>
  <text x="55" y="286" font-size="12" fill="#c0392b">&#x2717; No crash recovery</text>
  <text x="55" y="308" font-size="12" fill="#c0392b">&#x2717; No concurrency control</text>
  <text x="55" y="330" font-size="12" fill="#c0392b">&#x2717; No fine-grained security</text>
  <text x="55" y="352" font-size="12" fill="#c0392b">&#x2717; No integrity constraints</text>

  <!-- DBMS Side -->
  <rect x="420" y="20" width="360" height="360" fill="#f0fff4" rx="10" stroke="#27ae60" stroke-width="2"/>
  <text x="600" y="50" text-anchor="middle" font-size="16" font-weight="bold" fill="#1e8449">DBMS</text>

  <rect x="460" y="70" width="110" height="40" rx="6" fill="#27ae60" opacity="0.85"/>
  <text x="515" y="94" text-anchor="middle" fill="white" font-size="13">App A (Python)</text>
  <rect x="630" y="70" width="110" height="40" rx="6" fill="#27ae60" opacity="0.85"/>
  <text x="685" y="94" text-anchor="middle" fill="white" font-size="13">App B (Java)</text>

  <line x1="515" y1="110" x2="575" y2="140" stroke="#27ae60" stroke-width="2"/>
  <line x1="685" y1="110" x2="625" y2="140" stroke="#27ae60" stroke-width="2"/>

  <rect x="505" y="140" width="190" height="60" rx="8" fill="#27ae60"/>
  <text x="600" y="164" text-anchor="middle" fill="white" font-size="14" font-weight="bold">DBMS Engine</text>
  <text x="600" y="184" text-anchor="middle" fill="white" font-size="10">Parser | Optimizer | Executor | Recovery</text>

  <line x1="600" y1="200" x2="600" y2="230" stroke="#27ae60" stroke-width="2"/>

  <rect x="520" y="230" width="160" height="45" rx="6" fill="#d5f5e3" stroke="#27ae60"/>
  <text x="600" y="250" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a5c38">Unified</text>
  <text x="600" y="266" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a5c38">Database</text>

  <text x="455" y="305" font-size="12" fill="#1e8449">&#x2713; Single source of truth</text>
  <text x="455" y="325" font-size="12" fill="#1e8449">&#x2713; SQL queries (declarative)</text>
  <text x="455" y="345" font-size="12" fill="#1e8449">&#x2713; ACID transactions &amp; recovery</text>
  <text x="455" y="365" font-size="12" fill="#1e8449">&#x2713; Concurrency, Security, Integrity</text>
</svg>

---

## 5. Components of a DBMS

A DBMS is not a monolith — it is made up of cooperating components.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 480" font-family="Segoe UI, Arial, sans-serif">
  <rect width="720" height="480" fill="#f8f9fa" rx="12"/>
  <text x="360" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Internal Components of a DBMS</text>

  <!-- Users -->
  <rect x="80" y="45" width="100" height="35" rx="6" fill="#5dade2"/>
  <text x="130" y="67" text-anchor="middle" fill="white" font-size="12">End Users</text>
  <rect x="210" y="45" width="100" height="35" rx="6" fill="#5dade2"/>
  <text x="260" y="67" text-anchor="middle" fill="white" font-size="12">Applications</text>
  <rect x="340" y="45" width="100" height="35" rx="6" fill="#5dade2"/>
  <text x="390" y="67" text-anchor="middle" fill="white" font-size="12">DBA</text>

  <!-- Lines down to Query Processor -->
  <line x1="130" y1="80" x2="250" y2="110" stroke="#5dade2" stroke-width="1.5"/>
  <line x1="260" y1="80" x2="280" y2="110" stroke="#5dade2" stroke-width="1.5"/>
  <line x1="390" y1="80" x2="310" y2="110" stroke="#5dade2" stroke-width="1.5"/>

  <!-- Query Processor box -->
  <rect x="140" y="110" width="270" height="70" rx="8" fill="#2980b9"/>
  <text x="275" y="135" text-anchor="middle" fill="white" font-size="14" font-weight="bold">Query Processor</text>
  <text x="275" y="155" text-anchor="middle" fill="white" font-size="10">DDL Compiler | DML Compiler | Query Optimizer</text>
  <text x="275" y="170" text-anchor="middle" fill="white" font-size="10">Execution Engine</text>

  <!-- Arrow down -->
  <line x1="275" y1="180" x2="275" y2="210" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="269,208 281,208 275,218" fill="#7f8c8d"/>

  <!-- Transaction Manager + Concurrency -->
  <rect x="50" y="220" width="200" height="55" rx="8" fill="#27ae60"/>
  <text x="150" y="245" text-anchor="middle" fill="white" font-size="12" font-weight="bold">Transaction Manager</text>
  <text x="150" y="262" text-anchor="middle" fill="white" font-size="10">ACID | Concurrency Control</text>

  <rect x="300" y="220" width="200" height="55" rx="8" fill="#e67e22"/>
  <text x="400" y="245" text-anchor="middle" fill="white" font-size="12" font-weight="bold">Recovery Manager</text>
  <text x="400" y="262" text-anchor="middle" fill="white" font-size="10">WAL | Checkpoints | Undo/Redo</text>

  <rect x="540" y="220" width="150" height="55" rx="8" fill="#8e44ad"/>
  <text x="615" y="245" text-anchor="middle" fill="white" font-size="12" font-weight="bold">Authorization</text>
  <text x="615" y="262" text-anchor="middle" fill="white" font-size="10">GRANT | REVOKE | RLS</text>

  <!-- Arrow down -->
  <line x1="275" y1="275" x2="275" y2="305" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="269,303 281,303 275,313" fill="#7f8c8d"/>

  <!-- Storage Engine -->
  <rect x="100" y="315" width="350" height="65" rx="8" fill="#2c3e50"/>
  <text x="275" y="340" text-anchor="middle" fill="white" font-size="14" font-weight="bold">Storage Engine</text>
  <text x="275" y="360" text-anchor="middle" fill="white" font-size="10">Buffer Pool | File Manager | Disk Manager | Index Manager</text>
  <text x="275" y="372" text-anchor="middle" fill="white" font-size="10">Page Management | Space Allocation</text>

  <!-- Arrow down -->
  <line x1="275" y1="380" x2="275" y2="410" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="269,408 281,408 275,418" fill="#7f8c8d"/>

  <!-- Disk -->
  <rect x="170" y="420" width="210" height="45" rx="8" fill="#f39c12"/>
  <text x="275" y="448" text-anchor="middle" font-size="14" font-weight="bold" fill="white">Physical Storage (Disk/SSD)</text>

  <!-- Catalog / Data Dictionary (side) -->
  <rect x="510" y="315" width="170" height="65" rx="8" fill="#1abc9c"/>
  <text x="595" y="342" text-anchor="middle" fill="white" font-size="12" font-weight="bold">Data Dictionary</text>
  <text x="595" y="358" text-anchor="middle" fill="white" font-size="10">(System Catalog)</text>
  <text x="595" y="373" text-anchor="middle" fill="white" font-size="10">Schema metadata</text>
  <line x1="510" y1="347" x2="450" y2="347" stroke="#1abc9c" stroke-width="1.5" stroke-dasharray="4,3"/>
</svg>

### Component Details

| Component | Role | Key Details |
|---|---|---|
| **Query Processor** | Parses SQL, optimizes, and executes | Contains the parser, optimizer (cost-based), and execution engine |
| **DDL Compiler** | Processes schema definitions | `CREATE TABLE`, `ALTER TABLE` — updates the data dictionary |
| **DML Compiler** | Processes data manipulation | `INSERT`, `UPDATE`, `DELETE` — generates execution plans |
| **Query Optimizer** | Finds the most efficient execution plan | Considers indexes, statistics, join order — the "brain" of performance |
| **Transaction Manager** | Ensures ACID properties | Manages BEGIN, COMMIT, ROLLBACK, savepoints |
| **Concurrency Control** | Handles simultaneous access | Locking (2PL), MVCC, timestamp ordering |
| **Recovery Manager** | Restores after crashes | Write-Ahead Logging (WAL), ARIES algorithm |
| **Authorization Manager** | Enforces access control | Checks every query against GRANT/REVOKE rules |
| **Storage Engine** | Manages disk I/O | Buffer pool (caches pages in RAM), reads/writes disk blocks |
| **Data Dictionary** | Stores metadata | Table definitions, column types, indexes, constraints, statistics |

---

## 6. Types of DBMS

### 6.1 Relational DBMS (RDBMS)

Data is organized into **tables (relations)** with **rows (tuples)** and **columns (attributes)**. Tables are connected through **keys** (primary keys, foreign keys). Queried using **SQL**.

**Analogy**: A well-organized filing cabinet where each drawer is a table, each folder is a row, and labels are columns. A numbering system links folders across drawers.

#### The Relational Model in One Diagram

```
students                    enrollments                  courses
┌────┬──────┬──────┬─────┐  ┌────────────┬──────┬───┐   ┌──────┬───────────┬───┐
│ id │ name │ dept │ gpa │  │ student_id │ c_id │ G │   │ c_id │ title     │ cr│
├────┼──────┼──────┼─────┤  ├────────────┼──────┼───┤   ├──────┼───────────┼───┤
│  1 │Alice │ CS   │ 3.9 │  │     1      │ DB01 │ A │   │ DB01 │ Databases │  4│
│  2 │Bob   │ Math │ 3.5 │  │     1      │ OS02 │ B │   │ OS02 │ Op. Sys.  │  3│
│  3 │Carol │ CS   │ 3.7 │  │     2      │ DB01 │ A │   │ ML03 │ Machine L │  4│
└────┴──────┴──────┴─────┘  │     3      │ ML03 │ B │   └──────┴───────────┴───┘
       ▲                    └────────────┴──────┴───┘
       │ PK                   FK ▲    FK ▲
       └──────────────────────────┘      │
                                         └───────────── FK to courses
```

#### Characteristics

| Feature | Description |
|---|---|
| **Structure** | Fixed schema — every row has the same columns |
| **Query Language** | SQL (Structured Query Language) |
| **Integrity** | Primary keys, foreign keys, CHECK, UNIQUE, NOT NULL |
| **Transactions** | Full ACID support |
| **Joins** | Powerful relational operators to combine tables |
| **Scaling** | Primarily vertical (bigger machine); read replicas for reads |

**Best for**: Banking, ERP, HR, healthcare — anywhere data relationships and consistency are critical.

**Examples**: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, SQLite

---

### 6.2 NoSQL DBMS

**"Not Only SQL"** — a family of databases that abandon the rigid table model for more flexible structures. Born out of the need to handle **massive scale**, **unstructured data**, and **developer agility** in web-era applications.

#### The Four Families of NoSQL

| Family | Data Model | Analogy | Key Players | Ideal Use Case |
|---|---|---|---|---|
| **Document** | JSON/BSON documents | A filing cabinet where each folder can contain different papers | MongoDB, CouchDB, Firestore | User profiles, CMS, catalogs |
| **Key-Value** | Key → Value dictionary | A giant hash map | Redis, DynamoDB, Riak | Caching, sessions, leaderboards |
| **Column-Family** | Rows with dynamic column groups | A spreadsheet where each row can have different columns | Cassandra, HBase, ScyllaDB | IoT, time-series, analytics |
| **Graph** | Nodes + Edges + Properties | A social network diagram | Neo4j, Neptune, ArangoDB | Social graphs, fraud detection, recommendations |

#### Document Store Example (MongoDB)

```javascript
// No fixed schema — documents in the same collection can differ
db.students.insertMany([
    {
        _id: 1,
        name: "Alice",
        dept: "CS",
        gpa: 3.9,
        courses: ["DB101", "OS202"],
        address: { city: "New York", zip: "10001" }
    },
    {
        _id: 2,
        name: "Bob",
        dept: "Math",
        gpa: 3.5,
        // no 'address' field — that's fine in NoSQL!
        advisor: "Dr. Smith"
    }
]);

// Query: find CS students with GPA > 3.5
db.students.find({ dept: "CS", gpa: { $gt: 3.5 } });
```

#### Key-Value Store Example (Redis)

```redis
SET session:user:101 '{"name":"Alice","role":"student","login":"2026-04-24T10:30:00Z"}'
EXPIRE session:user:101 3600   -- auto-delete after 1 hour

GET session:user:101
-- Returns: '{"name":"Alice","role":"student","login":"2026-04-24T10:30:00Z"}'

INCR leaderboard:alice:score   -- atomic increment
```

#### Graph Database Example (Neo4j / Cypher)

```cypher
// Create nodes and relationships
CREATE (alice:Student {name: "Alice", dept: "CS"})
CREATE (bob:Student {name: "Bob", dept: "Math"})
CREATE (db:Course {id: "DB101", title: "Databases"})
CREATE (alice)-[:ENROLLED_IN {grade: "A"}]->(db)
CREATE (bob)-[:ENROLLED_IN {grade: "B"}]->(db)
CREATE (alice)-[:FRIENDS_WITH]->(bob)

// Query: "friends of Alice who are in the same course"
MATCH (alice:Student {name: "Alice"})-[:FRIENDS_WITH]->(friend)-[:ENROLLED_IN]->(course)
WHERE (alice)-[:ENROLLED_IN]->(course)
RETURN friend.name, course.title
```

---

### 6.3 NewSQL DBMS

NewSQL databases solve the "impossible triangle": **SQL interface + ACID guarantees + horizontal scalability**.

```
                        ┌──────────────┐
                        │   NewSQL     │
                        │ ACID + Scale │
                        └──────┬───────┘
                               │
               ┌───────────────┼───────────────┐
               │               │               │
         ┌─────┴─────┐  ┌─────┴─────┐  ┌──────┴──────┐
         │ SQL/ACID   │  │ Horizontal│  │ Distributed │
         │ Interface  │  │  Scaling  │  │ Consensus   │
         └───────────┘  └───────────┘  │ (Raft/Paxos)│
                                        └─────────────┘
```

**How it works**: Data is **sharded** across nodes. Each transaction uses a **distributed consensus protocol** (like Raft or Paxos) to ensure all nodes agree. Global transactions use **2-phase commit** or similar.

**Examples**: Google Spanner (globally consistent, GPS clocks!), CockroachDB, TiDB, YugabyteDB, VoltDB

**Use cases**: Global financial systems, multi-region e-commerce, telecom billing at scale

---

### 6.4 In-Memory DBMS

All data lives in **RAM** rather than disk. Disk is only used for persistence (snapshots, logs).

```
Traditional DB:               In-Memory DB:
┌─────────┐   seek   ┌────┐  ┌─────────┐  direct  ┌─────┐
│ App     │─────────▶│Disk│  │ App     │────────▶│ RAM │──▶ (optional disk backup)
└─────────┘  ~5-10ms └────┘  └─────────┘  ~0.1ms └─────┘
```

| Metric | Disk-based | In-Memory |
|---|---|---|
| Read latency | 1–10 ms | 0.01–0.1 ms |
| Write latency | 1–50 ms | 0.01–0.5 ms |
| Throughput | ~10K–100K ops/s | ~1M–10M ops/s |
| Durability | Inherent (on disk) | Requires snapshots/AOF |

**Trade-off**: Data is volatile — power loss means data loss UNLESS persistence is configured. Redis, for example, offers:
- **RDB snapshots**: periodic full dump to disk
- **AOF (Append-Only File)**: log every write operation

**Examples**: Redis, Memcached, VoltDB, SAP HANA, Apache Ignite

**Use cases**: Session storage, real-time leaderboards, caching layers, real-time analytics

---

### 6.5 Other Specialized Types

| Type | Description | Examples |
|---|---|---|
| **Time-Series** | Optimized for timestamped data (append-heavy, range queries) | InfluxDB, TimescaleDB, Prometheus |
| **Object-Oriented** | Stores objects directly (classes, inheritance) | ObjectDB, db4o |
| **Object-Relational** | RDBMS + object features (custom types, inheritance) | PostgreSQL (supports custom types!) |
| **Spatial/GIS** | Optimized for geographic data | PostGIS (Postgres extension), SpatiaLite |
| **Embedded** | Runs inside the application process | SQLite, LevelDB, RocksDB |
| **Multimodel** | Supports multiple models (doc + graph + KV) | ArangoDB, Cosmos DB, SurrealDB |

### SVG: DBMS Types Overview

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 750 330" font-family="Segoe UI, Arial, sans-serif">
  <rect width="750" height="330" fill="#fafafa" rx="10"/>
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Types of DBMS</text>

  <!-- Relational -->
  <rect x="15" y="45" width="170" height="265" rx="10" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="100" y="70" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">Relational</text>
  <text x="100" y="88" text-anchor="middle" font-size="11" fill="#555">Tables + SQL</text>
  <line x1="35" y1="96" x2="165" y2="96" stroke="#aed6f1" stroke-width="1"/>
  <rect x="30" y="105" width="140" height="22" rx="4" fill="#aed6f1"/>
  <text x="100" y="120" text-anchor="middle" font-size="11" fill="#1a5276">PostgreSQL</text>
  <rect x="30" y="132" width="140" height="22" rx="4" fill="#aed6f1"/>
  <text x="100" y="147" text-anchor="middle" font-size="11" fill="#1a5276">MySQL / MariaDB</text>
  <rect x="30" y="159" width="140" height="22" rx="4" fill="#aed6f1"/>
  <text x="100" y="174" text-anchor="middle" font-size="11" fill="#1a5276">Oracle / SQL Server</text>
  <rect x="30" y="186" width="140" height="22" rx="4" fill="#aed6f1"/>
  <text x="100" y="201" text-anchor="middle" font-size="11" fill="#1a5276">SQLite</text>
  <text x="100" y="235" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; ACID</text>
  <text x="100" y="255" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Complex queries</text>
  <text x="100" y="275" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Strong consistency</text>
  <text x="100" y="295" text-anchor="middle" font-size="11" fill="#e74c3c">&#x2717; Hard to scale out</text>

  <!-- NoSQL -->
  <rect x="200" y="45" width="170" height="265" rx="10" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="285" y="70" text-anchor="middle" font-size="14" font-weight="bold" fill="#f39c12">NoSQL</text>
  <text x="285" y="88" text-anchor="middle" font-size="11" fill="#555">Flexible Models</text>
  <line x1="220" y1="96" x2="350" y2="96" stroke="#fdebd0" stroke-width="1"/>
  <rect x="215" y="105" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="285" y="120" text-anchor="middle" font-size="11" fill="#784212">MongoDB (Document)</text>
  <rect x="215" y="132" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="285" y="147" text-anchor="middle" font-size="11" fill="#784212">Redis (Key-Value)</text>
  <rect x="215" y="159" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="285" y="174" text-anchor="middle" font-size="11" fill="#784212">Cassandra (Column)</text>
  <rect x="215" y="186" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="285" y="201" text-anchor="middle" font-size="11" fill="#784212">Neo4j (Graph)</text>
  <text x="285" y="235" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Scales horizontally</text>
  <text x="285" y="255" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Flexible schema</text>
  <text x="285" y="275" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; High throughput</text>
  <text x="285" y="295" text-anchor="middle" font-size="11" fill="#e74c3c">&#x2717; Eventual consistency</text>

  <!-- NewSQL -->
  <rect x="385" y="45" width="170" height="265" rx="10" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="470" y="70" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">NewSQL</text>
  <text x="470" y="88" text-anchor="middle" font-size="11" fill="#555">Best of Both</text>
  <line x1="405" y1="96" x2="535" y2="96" stroke="#a9dfbf" stroke-width="1"/>
  <rect x="400" y="105" width="140" height="22" rx="4" fill="#a9dfbf"/>
  <text x="470" y="120" text-anchor="middle" font-size="11" fill="#145a32">Google Spanner</text>
  <rect x="400" y="132" width="140" height="22" rx="4" fill="#a9dfbf"/>
  <text x="470" y="147" text-anchor="middle" font-size="11" fill="#145a32">CockroachDB</text>
  <rect x="400" y="159" width="140" height="22" rx="4" fill="#a9dfbf"/>
  <text x="470" y="174" text-anchor="middle" font-size="11" fill="#145a32">TiDB</text>
  <rect x="400" y="186" width="140" height="22" rx="4" fill="#a9dfbf"/>
  <text x="470" y="201" text-anchor="middle" font-size="11" fill="#145a32">YugabyteDB</text>
  <text x="470" y="235" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; ACID</text>
  <text x="470" y="255" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Scales out</text>
  <text x="470" y="275" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; SQL interface</text>
  <text x="470" y="295" text-anchor="middle" font-size="11" fill="#e74c3c">&#x2717; Complex ops</text>

  <!-- In-Memory -->
  <rect x="570" y="45" width="165" height="265" rx="10" fill="#f4ecf7" stroke="#8e44ad" stroke-width="2"/>
  <text x="652" y="70" text-anchor="middle" font-size="14" font-weight="bold" fill="#8e44ad">In-Memory</text>
  <text x="652" y="88" text-anchor="middle" font-size="11" fill="#555">RAM-first</text>
  <line x1="590" y1="96" x2="715" y2="96" stroke="#d7bde2" stroke-width="1"/>
  <rect x="585" y="105" width="135" height="22" rx="4" fill="#d7bde2"/>
  <text x="652" y="120" text-anchor="middle" font-size="11" fill="#4a235a">Redis</text>
  <rect x="585" y="132" width="135" height="22" rx="4" fill="#d7bde2"/>
  <text x="652" y="147" text-anchor="middle" font-size="11" fill="#4a235a">Memcached</text>
  <rect x="585" y="159" width="135" height="22" rx="4" fill="#d7bde2"/>
  <text x="652" y="174" text-anchor="middle" font-size="11" fill="#4a235a">SAP HANA</text>
  <rect x="585" y="186" width="135" height="22" rx="4" fill="#d7bde2"/>
  <text x="652" y="201" text-anchor="middle" font-size="11" fill="#4a235a">VoltDB</text>
  <text x="652" y="235" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Sub-ms latency</text>
  <text x="652" y="255" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Millions ops/sec</text>
  <text x="652" y="275" text-anchor="middle" font-size="11" fill="#27ae60">&#x2713; Great for caching</text>
  <text x="652" y="295" text-anchor="middle" font-size="11" fill="#e74c3c">&#x2717; Volatile by default</text>
</svg>

---

## 7. ANSI/SPARC 3-Tier Architecture

The **ANSI/SPARC** architecture (proposed in 1975) divides a database system into **three abstraction levels** to achieve **data independence**. This is the foundational architecture concept in DBMS theory.

### The Three Levels

| Level | Also Called | Managed By | Contains | Analogy |
|---|---|---|---|---|
| **External** | View Level | Application developers | Individual user views | The menu at a restaurant (each customer sees what's relevant) |
| **Conceptual** | Logical Level | DBA | Complete logical schema | The full recipe book in the kitchen |
| **Internal** | Physical Level | DBMS engine | Physical storage details | The pantry organization — where ingredients are actually stored |

### External Level (View Level)

The external level defines **what each user or application sees**. Different users have different views of the same underlying data.

**Why?**
- **Security**: Hide sensitive columns (salaries, SSN)
- **Simplicity**: Show only what's relevant to each user
- **Independence**: Applications break less when the schema changes

```sql
-- The underlying table has 8 columns
CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(200),
    dept       VARCHAR(50),
    salary     DECIMAL(10,2),
    ssn        CHAR(11),
    hire_date  DATE,
    manager_id INT REFERENCES employees(emp_id)
);

-- VIEW 1: HR department sees everything
CREATE VIEW hr_full_view AS
SELECT * FROM employees;

-- VIEW 2: Employee directory — public info only (no salary, no SSN)
CREATE VIEW employee_directory AS
SELECT emp_id, name, email, dept FROM employees;

-- VIEW 3: Finance sees salary info but not SSN
CREATE VIEW finance_view AS
SELECT emp_id, name, dept, salary FROM employees;

-- VIEW 4: Manager sees only their direct reports
CREATE VIEW manager_team_view AS
SELECT emp_id, name, dept, hire_date
FROM employees
WHERE manager_id = CURRENT_USER_ID();  -- pseudo-function for illustration
```

### Conceptual Level (Logical Level)

The **single, unified logical structure** of the entire database. This is the schema a DBA designs.

- Defines all tables, columns, data types, relationships, and constraints
- Independent of how data is physically stored
- Independent of individual user views
- This is what `CREATE TABLE`, `ALTER TABLE`, foreign keys, etc. define

```sql
-- Conceptual schema for a university database
CREATE TABLE departments (
    dept_id    CHAR(4) PRIMARY KEY,
    dept_name  VARCHAR(100) NOT NULL UNIQUE,
    building   VARCHAR(50),
    budget     DECIMAL(12,2) CHECK (budget > 0)
);

CREATE TABLE professors (
    prof_id    INT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    dept_id    CHAR(4) REFERENCES departments(dept_id),
    tenure     BOOLEAN DEFAULT FALSE
);

CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    dept_id    CHAR(4) REFERENCES departments(dept_id),
    gpa        DECIMAL(3,2) CHECK (gpa >= 0 AND gpa <= 4.0),
    enrolled   DATE NOT NULL
);

CREATE TABLE courses (
    course_id  CHAR(8) PRIMARY KEY,
    title      VARCHAR(200) NOT NULL,
    credits    INT CHECK (credits BETWEEN 1 AND 6),
    dept_id    CHAR(4) REFERENCES departments(dept_id),
    prof_id    INT REFERENCES professors(prof_id)
);

CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id  CHAR(8) REFERENCES courses(course_id),
    semester   CHAR(6),       -- e.g., '2026S1'
    grade      CHAR(2),       -- e.g., 'A+', 'B-'
    PRIMARY KEY (student_id, course_id, semester)
);
```

### Internal Level (Physical Level)

Describes **how data is physically stored** on the storage medium. Users and application programmers never interact with this level. It's managed by the DBMS engine.

| Aspect | Example |
|---|---|
| **File organization** | `students` table stored as a heap file at `/var/lib/postgresql/base/16384/18201` |
| **Indexes** | B-Tree index on `student_id` (primary key), hash index on `email` |
| **Page layout** | 8 KB pages, slotted page format, ~40 student records per page |
| **Compression** | Dictionary encoding on `dept_id`, TOAST for large text fields |
| **Partitioning** | `enrollments` table range-partitioned by `semester` |
| **Buffer pool** | 256 MB of shared buffers caching frequently accessed pages |

```
-- These are INTERNAL decisions — users don't write this code.
-- The DBA may influence them with configuration or DDL hints:

-- Create an index (DBA influencing internal level)
CREATE INDEX idx_students_dept ON students(dept_id);

-- Create a partitioned table (DBA influencing physical layout)
CREATE TABLE enrollments (
    student_id INT,
    course_id  CHAR(8),
    semester   CHAR(6),
    grade      CHAR(2)
) PARTITION BY RANGE (semester);

CREATE TABLE enrollments_2025 PARTITION OF enrollments
    FOR VALUES FROM ('2025S1') TO ('2025S3');
CREATE TABLE enrollments_2026 PARTITION OF enrollments
    FOR VALUES FROM ('2026S1') TO ('2026S3');
```

### SVG: 3-Tier Architecture Diagram

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 500" font-family="Segoe UI, Arial, sans-serif">
  <rect width="720" height="500" fill="#f8f9fa" rx="12"/>
  <text x="360" y="30" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">ANSI/SPARC 3-Tier Architecture</text>

  <!-- External Level -->
  <rect x="40" y="50" width="640" height="115" rx="10" fill="#ebf5fb" stroke="#2980b9" stroke-width="2.5"/>
  <text x="70" y="75" font-size="15" font-weight="bold" fill="#2980b9">EXTERNAL LEVEL (View Level)</text>
  <text x="70" y="95" font-size="12" fill="#555">What individual users and applications see. Multiple views of the same data.</text>
  <text x="70" y="112" font-size="12" fill="#555">Managed by: Application Developers</text>

  <rect x="60" y="122" width="120" height="30" rx="5" fill="#aed6f1"/>
  <text x="120" y="142" text-anchor="middle" font-size="11" fill="#1a5276">Student View</text>
  <rect x="195" y="122" width="120" height="30" rx="5" fill="#aed6f1"/>
  <text x="255" y="142" text-anchor="middle" font-size="11" fill="#1a5276">Faculty View</text>
  <rect x="330" y="122" width="120" height="30" rx="5" fill="#aed6f1"/>
  <text x="390" y="142" text-anchor="middle" font-size="11" fill="#1a5276">Admin View</text>
  <rect x="465" y="122" width="120" height="30" rx="5" fill="#aed6f1"/>
  <text x="525" y="142" text-anchor="middle" font-size="11" fill="#1a5276">Finance View</text>

  <!-- Mapping 1 -->
  <rect x="260" y="172" width="200" height="26" rx="6" fill="#d4efdf" stroke="#27ae60" stroke-width="1.5"/>
  <text x="360" y="189" text-anchor="middle" font-size="11" font-weight="bold" fill="#1e8449">External/Conceptual Mapping</text>

  <!-- Conceptual Level -->
  <rect x="40" y="206" width="640" height="115" rx="10" fill="#eafaf1" stroke="#27ae60" stroke-width="2.5"/>
  <text x="70" y="232" font-size="15" font-weight="bold" fill="#27ae60">CONCEPTUAL LEVEL (Logical Level)</text>
  <text x="70" y="252" font-size="12" fill="#555">Complete logical schema — tables, relationships, constraints, domains.</text>
  <text x="70" y="269" font-size="12" fill="#555">Managed by: Database Administrator (DBA)</text>

  <rect x="60" y="280" width="105" height="28" rx="5" fill="#a9dfbf"/>
  <text x="112" y="299" text-anchor="middle" font-size="10" fill="#145a32">departments</text>
  <rect x="175" y="280" width="105" height="28" rx="5" fill="#a9dfbf"/>
  <text x="227" y="299" text-anchor="middle" font-size="10" fill="#145a32">students</text>
  <rect x="290" y="280" width="105" height="28" rx="5" fill="#a9dfbf"/>
  <text x="342" y="299" text-anchor="middle" font-size="10" fill="#145a32">courses</text>
  <rect x="405" y="280" width="105" height="28" rx="5" fill="#a9dfbf"/>
  <text x="457" y="299" text-anchor="middle" font-size="10" fill="#145a32">enrollments</text>
  <rect x="520" y="280" width="105" height="28" rx="5" fill="#a9dfbf"/>
  <text x="572" y="299" text-anchor="middle" font-size="10" fill="#145a32">FK constraints</text>

  <!-- Mapping 2 -->
  <rect x="260" y="328" width="200" height="26" rx="6" fill="#fdebd0" stroke="#e67e22" stroke-width="1.5"/>
  <text x="360" y="345" text-anchor="middle" font-size="11" font-weight="bold" fill="#a04000">Conceptual/Internal Mapping</text>

  <!-- Internal Level -->
  <rect x="40" y="362" width="640" height="120" rx="10" fill="#fef9e7" stroke="#f39c12" stroke-width="2.5"/>
  <text x="70" y="388" font-size="15" font-weight="bold" fill="#e67e22">INTERNAL LEVEL (Physical Level)</text>
  <text x="70" y="408" font-size="12" fill="#555">How data is physically stored — files, indexes, pages, buffer pools, disk layout.</text>
  <text x="70" y="425" font-size="12" fill="#555">Managed by: DBMS Engine (automatic) + DBA (tuning)</text>

  <rect x="60" y="438" width="100" height="25" rx="4" fill="#fdebd0"/>
  <text x="110" y="455" text-anchor="middle" font-size="10" fill="#784212">Heap Files</text>
  <rect x="170" y="438" width="100" height="25" rx="4" fill="#fdebd0"/>
  <text x="220" y="455" text-anchor="middle" font-size="10" fill="#784212">B-Tree Indexes</text>
  <rect x="280" y="438" width="100" height="25" rx="4" fill="#fdebd0"/>
  <text x="330" y="455" text-anchor="middle" font-size="10" fill="#784212">Page Buffers</text>
  <rect x="390" y="438" width="100" height="25" rx="4" fill="#fdebd0"/>
  <text x="440" y="455" text-anchor="middle" font-size="10" fill="#784212">Disk Blocks</text>
  <rect x="500" y="438" width="100" height="25" rx="4" fill="#fdebd0"/>
  <text x="550" y="455" text-anchor="middle" font-size="10" fill="#784212">Partitions</text>
</svg>

### Mappings Between Levels

The two mappings (External↔Conceptual and Conceptual↔Internal) are what enable **data independence**:

| Mapping | What It Does | Example |
|---|---|---|
| **External/Conceptual** | Translates between a user's view and the logical schema | `student_directory` view is mapped to columns `emp_id, name, email, dept` from the `employees` table |
| **Conceptual/Internal** | Translates between logical tables and physical files/pages | The `students` table maps to heap file at `/data/students.dat` with a B-Tree on `student_id` |

---

## 8. Data Independence

**Data independence** is the ability to change the schema at one level of the 3-tier architecture **without requiring changes at the levels above it**. It is one of the most important goals of any DBMS and one of the foundational reasons we use DBMS instead of file systems.

### 8.1 Logical Data Independence

> **Definition**: The ability to modify the **conceptual schema** without having to change the **external views** or application programs.

This is **harder to achieve** than physical data independence because applications may depend closely on the logical structure.

#### Examples

**Adding a column:**
```sql
-- BEFORE: conceptual schema
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name       VARCHAR(100),
    dept       VARCHAR(50)
);

-- CHANGE: add email column (conceptual level change)
ALTER TABLE students ADD COLUMN email VARCHAR(200);

-- RESULT: existing views are UNAFFECTED
-- This view still works perfectly:
CREATE VIEW student_directory AS
SELECT student_id, name, dept FROM students;  -- doesn't mention 'email'
```

**Splitting a table:**
```sql
-- BEFORE: one table
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name   VARCHAR(100),
    dept   VARCHAR(50),
    salary DECIMAL(10,2)
);

-- CHANGE: split into two tables (conceptual restructuring)
CREATE TABLE emp_info (emp_id INT PRIMARY KEY, name VARCHAR(100), dept VARCHAR(50));
CREATE TABLE emp_salary (emp_id INT PRIMARY KEY, salary DECIMAL(10,2));

-- PRESERVE: create a view that mimics the old table
CREATE VIEW employees AS
SELECT i.emp_id, i.name, i.dept, s.salary
FROM emp_info i JOIN emp_salary s ON i.emp_id = s.emp_id;

-- Applications that queried 'employees' continue working unchanged!
```

**Renaming a column:**
```sql
-- CHANGE: rename 'dept' to 'department'
ALTER TABLE students RENAME COLUMN dept TO department;

-- PRESERVE: update the view to alias back to old name
CREATE OR REPLACE VIEW legacy_student_view AS
SELECT student_id, name, department AS dept FROM students;
-- Old apps querying 'dept' through this view: still work
```

### 8.2 Physical Data Independence

> **Definition**: The ability to modify the **internal schema** (storage structure, indexes, file organization) without having to change the **conceptual schema** or application programs.

This is **easier to achieve** and is well-supported by modern DBMS.

#### Examples

**Adding an index:**
```sql
-- Application query (doesn't change at all):
SELECT * FROM students WHERE dept = 'CS';

-- DBA creates an index (physical change)
CREATE INDEX idx_students_dept ON students(dept);

-- Same query runs, now uses index scan instead of sequential scan — faster!
-- Application code: ZERO changes needed
```

**Changing storage engine:**
```sql
-- MySQL: change from MyISAM to InnoDB (physical change)
ALTER TABLE orders ENGINE = InnoDB;

-- All queries on 'orders' continue to work identically
-- But now the table supports transactions — that's a physical-level upgrade
```

**Moving to different disk / tablespace:**
```sql
-- PostgreSQL: move a table to a faster SSD tablespace
ALTER TABLE students SET TABLESPACE fast_ssd;

-- Applications notice nothing — just faster performance
```

**Partitioning a table:**
```sql
-- DBA partitions the enrollments table by semester (physical reorganization)
-- Conceptual schema stays the same — 'enrollments' still looks like one table
-- But internally, data is split into per-semester files for faster range scans
```

### SVG: Data Independence

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 380" font-family="Segoe UI, Arial, sans-serif">
  <rect width="720" height="380" fill="#fafafa" rx="10"/>
  <text x="360" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Data Independence</text>

  <!-- External -->
  <rect x="220" y="50" width="280" height="55" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="360" y="73" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">External Level</text>
  <text x="360" y="93" text-anchor="middle" font-size="11" fill="#555">Views / Applications / User Interfaces</text>

  <!-- Logical Independence bracket -->
  <rect x="530" y="85" width="170" height="70" rx="6" fill="#d4efdf" stroke="#27ae60" stroke-width="1.5"/>
  <text x="615" y="108" text-anchor="middle" font-size="12" font-weight="bold" fill="#1e8449">Logical Data</text>
  <text x="615" y="125" text-anchor="middle" font-size="12" font-weight="bold" fill="#1e8449">Independence</text>
  <text x="615" y="145" text-anchor="middle" font-size="10" fill="#555">Harder to achieve</text>

  <!-- Arrow -->
  <line x1="360" y1="105" x2="360" y2="150" stroke="#27ae60" stroke-width="2.5"/>
  <polygon points="354,148 366,148 360,160" fill="#27ae60"/>

  <!-- Conceptual -->
  <rect x="220" y="163" width="280" height="55" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="360" y="186" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">Conceptual Level</text>
  <text x="360" y="206" text-anchor="middle" font-size="11" fill="#555">Tables / Schema / Constraints / Relationships</text>

  <!-- Physical Independence bracket -->
  <rect x="530" y="198" width="170" height="70" rx="6" fill="#fdebd0" stroke="#e67e22" stroke-width="1.5"/>
  <text x="615" y="221" text-anchor="middle" font-size="12" font-weight="bold" fill="#a04000">Physical Data</text>
  <text x="615" y="238" text-anchor="middle" font-size="12" font-weight="bold" fill="#a04000">Independence</text>
  <text x="615" y="258" text-anchor="middle" font-size="10" fill="#555">Easier to achieve</text>

  <!-- Arrow -->
  <line x1="360" y1="218" x2="360" y2="263" stroke="#e67e22" stroke-width="2.5"/>
  <polygon points="354,261 366,261 360,273" fill="#e67e22"/>

  <!-- Internal -->
  <rect x="220" y="276" width="280" height="55" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="360" y="299" text-anchor="middle" font-size="14" font-weight="bold" fill="#e67e22">Internal Level</text>
  <text x="360" y="319" text-anchor="middle" font-size="11" fill="#555">Disk / Files / Indexes / Partitions / Buffer Pool</text>

  <!-- Left annotations -->
  <text x="90" y="105" text-anchor="middle" font-size="11" fill="#2980b9" font-style="italic">Add column?</text>
  <text x="90" y="120" text-anchor="middle" font-size="11" fill="#2980b9" font-style="italic">Views don't break.</text>
  <line x1="155" y1="112" x2="218" y2="112" stroke="#2980b9" stroke-width="1" stroke-dasharray="3,3"/>

  <text x="90" y="255" text-anchor="middle" font-size="11" fill="#e67e22" font-style="italic">Add index?</text>
  <text x="90" y="270" text-anchor="middle" font-size="11" fill="#e67e22" font-style="italic">Schema doesn't break.</text>
  <line x1="155" y1="262" x2="218" y2="262" stroke="#e67e22" stroke-width="1" stroke-dasharray="3,3"/>

  <!-- Summary box -->
  <rect x="120" y="345" width="480" height="28" rx="6" fill="#2c3e50"/>
  <text x="360" y="364" text-anchor="middle" font-size="12" fill="white">Each level is insulated from changes in the level below it</text>
</svg>

### Why Does Data Independence Matter?

| Without Data Independence | With Data Independence |
|---|---|
| Adding a column breaks 10 applications | Applications are insulated by views |
| Creating an index requires rewriting queries | Queries work the same, just faster |
| Moving data to a new server means rewriting code | Transparent to all applications |
| Schema changes require coordinated deployment of all apps | Each app evolves independently |

---

## 9. Database Users and Roles

Different people interact with a DBMS in different ways.

| User Type | What They Do | Tools They Use | Example |
|---|---|---|---|
| **Database Administrator (DBA)** | Designs schema, manages security, monitors performance, handles backup/recovery | pgAdmin, SQL, EXPLAIN, config files | "Create the production schema, set up replication, tune slow queries" |
| **Application Programmer** | Writes applications that use the database | ORM (SQLAlchemy, Hibernate), SQL, API | "Build the REST API that reads student records" |
| **End User (Naive)** | Uses pre-built interfaces — doesn't write SQL | Web forms, dashboards, reports | "Look up my grades on the student portal" |
| **End User (Sophisticated)** | Writes SQL queries directly for ad-hoc analysis | SQL client (psql, DBeaver), BI tools | "Analyst runs a complex join to find at-risk students" |
| **Database Designer** | Designs the conceptual schema and ER diagrams | ERD tools, UML, SQL DDL | "Design the schema for the new e-commerce platform" |
| **System Analyst** | Determines data requirements and workflows | Interviews, modeling tools | "Figure out what data the payroll system needs" |

---

## 10. Popular DBMS Systems — Deep Comparison

### PostgreSQL

**The "Swiss Army Knife" of databases.** Open-source, object-relational, extensible.

| Attribute | Detail |
|---|---|
| **Type** | Object-Relational |
| **License** | PostgreSQL License (very permissive, BSD-like) |
| **ACID** | Full |
| **Concurrency** | MVCC (Multi-Version Concurrency Control) |
| **Standout Features** | JSONB, CTEs, window functions, full-text search, PostGIS, custom types, table inheritance, partitioning, logical replication |
| **Used By** | Apple, Instagram, Spotify, Twitch, Reddit, the US Federal Aviation Administration |

```sql
-- PostgreSQL showcase: JSONB + window functions + CTE
WITH ranked_products AS (
    SELECT
        name,
        details->>'category' AS category,
        (details->>'price')::DECIMAL AS price,
        RANK() OVER (
            PARTITION BY details->>'category'
            ORDER BY (details->>'price')::DECIMAL DESC
        ) AS rank_in_category
    FROM products
    WHERE details @> '{"in_stock": true}'  -- JSONB containment operator
)
SELECT name, category, price
FROM ranked_products
WHERE rank_in_category <= 3;
```

### MySQL

**The world's most popular web database.** Powers most of the internet.

| Attribute | Detail |
|---|---|
| **Type** | Relational |
| **License** | GPL (open-source) + Commercial (Oracle) |
| **ACID** | Yes (with InnoDB engine; MyISAM is not ACID) |
| **Concurrency** | MVCC (InnoDB) |
| **Standout Features** | Simplicity, speed for read-heavy workloads, replication, wide hosting support |
| **Used By** | Facebook, YouTube, Twitter, WordPress (powers ~40% of the web), Uber |

```sql
-- MySQL: JSON, generated columns, and full-text search
CREATE TABLE articles (
    id        INT AUTO_INCREMENT PRIMARY KEY,
    title     VARCHAR(200),
    body      TEXT,
    metadata  JSON,
    -- Virtual generated column from JSON:
    author    VARCHAR(100) GENERATED ALWAYS AS (metadata->>'$.author') VIRTUAL,
    FULLTEXT INDEX ft_body (body)
) ENGINE=InnoDB;

-- Full-text search:
SELECT title, MATCH(body) AGAINST('database optimization' IN NATURAL LANGUAGE MODE) AS score
FROM articles
WHERE MATCH(body) AGAINST('database optimization' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;
```

### Oracle Database

**The enterprise heavyweight.** Dominant in banking, telecom, government for 40+ years.

| Attribute | Detail |
|---|---|
| **Type** | Relational (object-relational features) |
| **License** | Commercial (very expensive) |
| **ACID** | Full |
| **Concurrency** | MVCC + Read Consistency |
| **Standout Features** | RAC (Real Application Clusters), Data Guard, Flashback, PL/SQL, partitioning, Advanced Security |
| **Used By** | Major banks worldwide, telecom (AT&T, Vodafone), SAP, government agencies |

```sql
-- Oracle: Flashback query (see data AS OF a past time!)
SELECT * FROM accounts
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR)
WHERE account_id = 12345;

-- Oracle: PL/SQL procedure
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_acct  IN NUMBER,
    p_to_acct    IN NUMBER,
    p_amount     IN NUMBER
) AS
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from_acct;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to_acct;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/
```

### MongoDB

**The leading document database.** Schema-free, developer-friendly, horizontally scalable.

| Attribute | Detail |
|---|---|
| **Type** | Document Store (NoSQL) |
| **License** | SSPL (Server Side Public License) |
| **ACID** | Multi-document transactions since 4.0 (2018) |
| **Concurrency** | Document-level locking + WiredTiger storage |
| **Standout Features** | Flexible schema, aggregation pipeline, sharding, change streams, Atlas (managed cloud) |
| **Used By** | LinkedIn, Forbes, eBay, Electronic Arts, CERN |

```javascript
// MongoDB: aggregation pipeline — average GPA by department
db.students.aggregate([
    { $match: { gpa: { $gte: 2.0 } } },           // filter
    { $group: {                                      // group
        _id: "$dept",
        avg_gpa: { $avg: "$gpa" },
        count: { $sum: 1 }
    }},
    { $sort: { avg_gpa: -1 } },                     // sort
    { $project: {                                    // reshape
        department: "$_id",
        avg_gpa: { $round: ["$avg_gpa", 2] },
        count: 1,
        _id: 0
    }}
]);
```

### Head-to-Head Matrix

| Feature | PostgreSQL | MySQL | Oracle | MongoDB |
|---|---|---|---|---|
| **Type** | Object-Relational | Relational | Object-Relational | Document (NoSQL) |
| **Cost** | Free | Free / Commercial | $$$$ | Free / Commercial |
| **ACID** | Full | Full (InnoDB) | Full | Multi-doc (4.0+) |
| **SQL Standard** | Excellent compliance | Good | Good (+ PL/SQL) | MQL (not SQL) |
| **JSON Support** | Native JSONB | JSON type | JSON type | Native (BSON) |
| **Scaling** | Vertical + read replicas | Vertical + replication | RAC + Data Guard | Horizontal (sharding) |
| **Concurrency** | MVCC | MVCC (InnoDB) | MVCC + Read Consistency | Doc-level locking |
| **Best For** | Complex queries, extensibility | High-read web apps | Enterprise, banking | Flexible schema, rapid dev |
| **Learning Curve** | Moderate | Easy | Steep | Easy (for developers) |

---

## 11. Worked Examples — Beginner, Intermediate, Advanced

### Beginner Example: Student-Course Database

**Scenario**: A university tracks students, courses, and enrollments.

```sql
-- Create the schema
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    dept        VARCHAR(50),
    gpa         DECIMAL(3,2)
);

CREATE TABLE courses (
    course_id   CHAR(6) PRIMARY KEY,
    title       VARCHAR(200) NOT NULL,
    credits     INT
);

CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(6) REFERENCES courses(course_id),
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);

-- Insert sample data
INSERT INTO students VALUES (1, 'Alice', 'CS', 3.9);
INSERT INTO students VALUES (2, 'Bob', 'Math', 3.5);
INSERT INTO students VALUES (3, 'Carol', 'CS', 3.7);

INSERT INTO courses VALUES ('DB101', 'Databases', 4);
INSERT INTO courses VALUES ('OS202', 'Operating Systems', 3);

INSERT INTO enrollments VALUES (1, 'DB101', 'A');
INSERT INTO enrollments VALUES (1, 'OS202', 'B+');
INSERT INTO enrollments VALUES (2, 'DB101', 'A-');
INSERT INTO enrollments VALUES (3, 'DB101', 'B');

-- Query: students in the Databases course with their grades
SELECT s.name, s.dept, e.grade
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
WHERE e.course_id = 'DB101';
```

**Result:**
| name | dept | grade |
|---|---|---|
| Alice | CS | A |
| Bob | Math | A- |
| Carol | CS | B |

---

### Intermediate Example: E-Commerce Database

**Scenario**: An online store tracks customers, products, orders, and order items.

```sql
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(200) UNIQUE NOT NULL,
    city          VARCHAR(100),
    joined_date   DATE DEFAULT CURRENT_DATE
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    category      VARCHAR(50),
    price         DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock         INT NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INT NOT NULL REFERENCES customers(customer_id),
    order_date    TIMESTAMP DEFAULT NOW(),
    status        VARCHAR(20) DEFAULT 'pending'
                  CHECK (status IN ('pending', 'shipped', 'delivered', 'cancelled')),
    total_amount  DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id      INT REFERENCES orders(order_id),
    product_id    INT REFERENCES products(product_id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    unit_price    DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Query: top 5 customers by total spend
SELECT c.name, c.email, SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.name, c.email
ORDER BY total_spent DESC
LIMIT 5;

-- Query: products that have never been ordered
SELECT p.name, p.category, p.price
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.product_id IS NULL;
```

---

### Advanced Example: Banking System

**Scenario**: A bank manages accounts, transactions, and enforces strict integrity (money can't appear or vanish).

```sql
CREATE TABLE branches (
    branch_id     INT PRIMARY KEY,
    branch_name   VARCHAR(100) NOT NULL,
    city          VARCHAR(100) NOT NULL,
    total_assets  DECIMAL(15,2) DEFAULT 0
);

CREATE TABLE accounts (
    account_id    SERIAL PRIMARY KEY,
    branch_id     INT NOT NULL REFERENCES branches(branch_id),
    holder_name   VARCHAR(100) NOT NULL,
    account_type  VARCHAR(20) CHECK (account_type IN ('checking', 'savings', 'loan')),
    balance       DECIMAL(15,2) NOT NULL DEFAULT 0,
    opened_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE transactions (
    txn_id        SERIAL PRIMARY KEY,
    from_account  INT REFERENCES accounts(account_id),
    to_account    INT REFERENCES accounts(account_id),
    amount        DECIMAL(15,2) NOT NULL CHECK (amount > 0),
    txn_type      VARCHAR(20) NOT NULL,
    txn_time      TIMESTAMP DEFAULT NOW(),
    description   VARCHAR(500),
    CHECK (from_account != to_account)
);

CREATE TABLE audit_log (
    log_id        SERIAL PRIMARY KEY,
    txn_id        INT REFERENCES transactions(txn_id),
    action        VARCHAR(50) NOT NULL,
    old_balance   DECIMAL(15,2),
    new_balance   DECIMAL(15,2),
    logged_at     TIMESTAMP DEFAULT NOW()
);

-- TRANSACTION: transfer $500 from account 1001 to account 1002
-- This MUST be atomic — both succeed or both fail
BEGIN;

    -- Debit sender
    UPDATE accounts
    SET balance = balance - 500.00
    WHERE account_id = 1001 AND balance >= 500.00 AND is_active = TRUE;

    -- Check that exactly one row was updated (sufficient funds, account exists)
    -- In a real procedure, you'd check @@ROWCOUNT or GET DIAGNOSTICS

    -- Credit receiver
    UPDATE accounts
    SET balance = balance + 500.00
    WHERE account_id = 1002 AND is_active = TRUE;

    -- Log the transaction
    INSERT INTO transactions (from_account, to_account, amount, txn_type, description)
    VALUES (1001, 1002, 500.00, 'transfer', 'Rent payment April 2026');

COMMIT;

-- Query: detect suspicious activity — accounts with > 10 transactions in the last hour
SELECT a.account_id, a.holder_name, COUNT(t.txn_id) AS txn_count
FROM accounts a
JOIN transactions t ON a.account_id = t.from_account
WHERE t.txn_time > NOW() - INTERVAL '1 hour'
GROUP BY a.account_id, a.holder_name
HAVING COUNT(t.txn_id) > 10
ORDER BY txn_count DESC;

-- Query: daily account balance report for branch 1
SELECT
    a.account_id,
    a.holder_name,
    a.account_type,
    a.balance AS current_balance,
    COALESCE(SUM(CASE WHEN t.to_account = a.account_id THEN t.amount ELSE 0 END), 0) AS credits_today,
    COALESCE(SUM(CASE WHEN t.from_account = a.account_id THEN t.amount ELSE 0 END), 0) AS debits_today
FROM accounts a
LEFT JOIN transactions t
    ON (a.account_id = t.from_account OR a.account_id = t.to_account)
    AND t.txn_time::DATE = CURRENT_DATE
WHERE a.branch_id = 1
GROUP BY a.account_id, a.holder_name, a.account_type, a.balance
ORDER BY a.account_id;
```

---

## 12. Common Interview Questions

### Q1: What is a DBMS? Why do we need it?
> A DBMS is software that manages databases — it handles storage, retrieval, security, concurrency, and recovery. We need it because file systems suffer from data redundancy, inconsistency, lack of security, no concurrency control, and no crash recovery. A DBMS centralizes all of this.

### Q2: What is the difference between a database, a DBMS, and a database system?
> - **Database**: The organized collection of data itself  
> - **DBMS**: The software that manages the database (PostgreSQL, MySQL, etc.)  
> - **Database System**: The complete package — database + DBMS + applications + users

### Q3: List 5 advantages of DBMS over file systems.
> 1. **Reduced redundancy** — normalization eliminates duplicate data  
> 2. **Data consistency** — constraints enforce correctness  
> 3. **Data sharing** — multiple users/apps access the same data concurrently  
> 4. **Security** — fine-grained access control (column-level, row-level)  
> 5. **Crash recovery** — WAL and checkpoints restore data after failures

### Q4: What are the disadvantages of a DBMS?
> 1. **Cost** — commercial licenses (Oracle), hardware, DBA salary  
> 2. **Complexity** — requires trained personnel to administer  
> 3. **Overhead** — DBMS consumes memory and CPU for its services  
> 4. **Single point of failure** — if the DBMS crashes, all apps are affected (mitigated by replication)

### Q5: Explain the ANSI/SPARC 3-level architecture.
> The three levels are:  
> - **External** (View): What individual users see — personalized, restricted views  
> - **Conceptual** (Logical): The complete logical schema — all tables, constraints, relationships  
> - **Internal** (Physical): How data is stored on disk — files, indexes, pages  
> The separation enables **data independence**: changes at one level don't ripple to others.

### Q6: What is data independence? Explain both types with examples.
> **Logical data independence**: Change the conceptual schema without affecting views. Example: adding a column to a table doesn't break existing views.  
> **Physical data independence**: Change the internal schema without affecting the conceptual schema. Example: adding an index speeds up queries but requires zero changes to table definitions or application code.  
> Logical independence is harder to achieve because applications often depend closely on the schema.

### Q7: Compare RDBMS, NoSQL, and NewSQL.
> | | RDBMS | NoSQL | NewSQL |
> |---|---|---|---|
> | **Model** | Tables/SQL | Various (doc, KV, graph) | Tables/SQL |
> | **Schema** | Fixed | Flexible | Fixed |
> | **ACID** | Yes | Usually no (BASE) | Yes |
> | **Scale** | Vertical | Horizontal | Horizontal |
> | **Best for** | Structured data, complex queries | Unstructured data, massive scale | Both consistency and scale |

### Q8: When would you use a file system instead of a DBMS?
> - Very small datasets with a single user  
> - Simple, static data that rarely changes (config files)  
> - When DBMS overhead is not justified (embedded systems with tight resources)  
> - Log files that are append-only and processed by batch jobs

### Q9: What is the data dictionary (system catalog)?
> The data dictionary stores **metadata**: table definitions, column types, indexes, constraints, user permissions, statistics. The DBMS reads it to validate and optimize every query. In PostgreSQL, it's accessible via `information_schema` and `pg_catalog`.

### Q10: Name 5 components of a DBMS and describe their roles.
> 1. **Query Processor** — parses SQL, optimizes, generates execution plans  
> 2. **Storage Engine** — manages disk I/O, buffer pool, page management  
> 3. **Transaction Manager** — ensures ACID, manages BEGIN/COMMIT/ROLLBACK  
> 4. **Concurrency Control** — handles simultaneous access (locks, MVCC)  
> 5. **Recovery Manager** — restores data after crashes using WAL and checkpoints

### Q11: Explain MVCC in one paragraph.
> **Multi-Version Concurrency Control (MVCC)** allows readers and writers to work simultaneously without blocking each other. Instead of locking rows, the database keeps **multiple versions** of each row. A reader sees the version that was committed before its transaction started (a "snapshot"), while a writer creates a new version. This eliminates read-write contention and is used by PostgreSQL, MySQL (InnoDB), and Oracle.

### Q12: What is the difference between MySQL's InnoDB and MyISAM?
> | Feature | InnoDB | MyISAM |
> |---|---|---|
> | ACID/Transactions | Yes | No |
> | Row-level locking | Yes | Table-level only |
> | Foreign keys | Yes | No |
> | Crash recovery | Yes (WAL) | No (manual repair) |
> | Full-text search | Yes (5.6+) | Yes |
> | **Recommendation** | Default for all new tables | Legacy only |

---

## 13. Key Takeaways

| # | Concept | One-Line Summary |
|---|---|---|
| 1 | **Database** | An organized collection of logically related data |
| 2 | **DBMS** | Software that defines, manages, queries, and secures a database |
| 3 | **File System Problems** | Redundancy, inconsistency, no security, no concurrency, no recovery |
| 4 | **DBMS Advantages** | Solves all file system problems + adds SQL, ACID, and data independence |
| 5 | **RDBMS** | Tables + SQL + ACID — best for structured data with strong consistency |
| 6 | **NoSQL** | Flexible models (doc, KV, graph, column) — best for scale and unstructured data |
| 7 | **NewSQL** | ACID + horizontal scale — best of both worlds, but complex |
| 8 | **In-Memory** | Sub-millisecond latency — best for caching and real-time |
| 9 | **3-Tier Architecture** | External (views) → Conceptual (schema) → Internal (physical storage) |
| 10 | **Logical Independence** | Change schema without breaking views — harder to achieve |
| 11 | **Physical Independence** | Change storage without breaking schema — well-supported |
| 12 | **Data Dictionary** | System catalog storing all metadata about the database |
| 13 | **PostgreSQL** | Open-source powerhouse — complex queries, JSONB, extensibility |
| 14 | **MySQL** | Web workhorse — simple, fast, ubiquitous |
| 15 | **Oracle** | Enterprise standard — banking, telecom, 40+ years of features |
| 16 | **MongoDB** | Leading NoSQL — flexible schema, aggregation pipeline, sharding |
| 17 | **DBA** | Manages schema, performance, security, backup, and recovery |
| 18 | **Codd's Insight** | Separate logical data from physical storage — the idea that started it all |

---

<p style="text-align: center; margin-top: 40px;">
  <a href="01_intro_to_dbms.md" download="01_intro_to_dbms.md"
     style="display: inline-block; padding: 14px 36px; font-size: 16px; font-weight: bold;
            color: #fff; background-color: #2980b9; border-radius: 8px;
            text-decoration: none; box-shadow: 0 4px 6px rgba(0,0,0,0.15);
            transition: background-color 0.3s;">
    Download 01_intro_to_dbms.md
  </a>
</p>

---

*Ready? Say **"next"** to proceed to **Phase 2 — Data Modeling & ER Diagrams**.*
