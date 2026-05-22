# Concurrency & Distributed Systems — Complete Deep Dive

## Why This Matters
Every system at scale is distributed. FAANG interviews expect you to understand: threads, locks, race conditions, distributed consensus, and failure handling.

---

## 1. Concurrency vs Parallelism

```
Concurrency: Multiple tasks making progress (may be on one CPU)
  → Time-slicing: CPU switches between tasks rapidly
  → Example: Async I/O, event loops (Node.js)

Parallelism: Multiple tasks executing SIMULTANEOUSLY (multiple CPUs)
  → True simultaneous execution on multiple cores
  → Example: MapReduce, parallel matrix multiplication

┌─── Concurrency (1 CPU) ──────┐    ┌─── Parallelism (4 CPUs) ──┐
│ Task A: ██░░██░░██           │    │ CPU 1: ██████████          │
│ Task B: ░░██░░██░░           │    │ CPU 2: ██████████          │
│ (interleaved on 1 core)     │    │ CPU 3: ██████████          │
└──────────────────────────────┘    │ CPU 4: ██████████          │
                                     └────────────────────────────┘
```

---

## 2. Threading Models

```
1:1 Model (Kernel threads):
  Each user thread maps to one kernel thread
  OS manages scheduling
  Used by: Java, Rust, C++ (std::thread)
  ✅ True parallelism on multi-core
  ❌ Heavy (each thread uses ~1-8MB stack)

N:1 Model (Green threads):
  Many user threads on one kernel thread
  Runtime manages scheduling
  Used by: Early Java, Ruby (GIL)
  ❌ No true parallelism

M:N Model (Hybrid):
  M user threads on N kernel threads
  Best of both worlds
  Used by: Go (goroutines), Erlang, Rust (tokio)
  ✅ Lightweight + true parallelism

Python's GIL (Global Interpreter Lock):
  Only ONE thread executes Python bytecode at a time!
  → Threading in Python doesn't help for CPU-bound tasks
  → Use multiprocessing or async for parallelism
  → GIL doesn't affect I/O-bound tasks (released during I/O)
```

---

## 3. Locks and Synchronization Primitives

```
Mutex (Mutual Exclusion):
  Lock/Unlock. Only one thread in critical section.
  ⚠️ Deadlock if thread dies while holding lock

Reentrant Lock (Recursive Mutex):
  Same thread can acquire multiple times (must release same number)
  Useful for recursive functions that need the lock

Read-Write Lock:
  Multiple readers simultaneously OR one writer exclusively
  ✅ Great for read-heavy workloads (read >> write)

Spinlock:
  Busy-wait (keep checking) instead of sleeping
  ✅ Fast for very short critical sections (avoids context switch)
  ❌ Wastes CPU cycles if lock held for long

Condition Variable:
  wait(): Release lock + sleep until signaled
  signal(): Wake one waiting thread
  broadcast(): Wake all waiting threads
  Pattern: while(!condition) { cond.wait(lock); }

Semaphore:
  Counting lock. Allows N concurrent access.
  sem.acquire() → count-- (block if count=0)
  sem.release() → count++

Compare-and-Swap (CAS) — Lock-free:
  Atomic: if (value == expected) { value = new; return true; }
  Foundation of lock-free data structures
  Used by: java.util.concurrent.atomic, Go sync/atomic
```

---

## 4. Common Concurrency Patterns

### Producer-Consumer with Bounded Buffer
```python
import threading
from queue import Queue

buffer = Queue(maxsize=10)

def producer():
    while True:
        item = produce_item()
        buffer.put(item)  # blocks if full

def consumer():
    while True:
        item = buffer.get()  # blocks if empty
        process(item)
        buffer.task_done()
```

### Thread Pool
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(process, task) for task in tasks]
    results = [f.result() for f in futures]

# Why: Creating threads is expensive. Pool reuses them.
# Used by: Web servers, database connection pools
```

### Async/Await (Cooperative Concurrency)
```python
import asyncio

async def fetch_data(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def main():
    # Run 100 requests concurrently (NOT 100 threads!)
    tasks = [fetch_data(url) for url in urls]
    results = await asyncio.gather(*tasks)

# Single thread handles thousands of I/O operations
# Event loop switches between coroutines at await points
```

---

## 5. Distributed Systems Fundamentals

### CAP Theorem
```
In a distributed system experiencing a network partition,
you can only guarantee TWO of:

Consistency:  Every read returns the most recent write
Availability: Every request gets a response (may not be latest)
Partition Tolerance: System works despite network failures

Since partitions ARE inevitable in distributed systems:
  CP: Consistent + Partition-tolerant (sacrifice availability)
      → HBase, MongoDB (in strict mode), ZooKeeper
  AP: Available + Partition-tolerant (sacrifice consistency)
      → Cassandra, DynamoDB, CouchDB

In practice: It's a spectrum, not binary.
  Most systems are "mostly consistent" or "mostly available"
```

### PACELC Theorem (extends CAP)
```
If Partition:  choose Availability or Consistency (CAP)
Else:          choose Latency or Consistency

Even without partitions, there's a consistency-latency trade-off!

Example: DynamoDB
  Partition: Available (AP)
  No partition: Low Latency (sacrifice consistency — eventual)
```

### Consistency Models
```
Strong:       Everyone sees the same data at the same time
              (linearizability — single global order)

Sequential:   All operations in SOME total order consistent
              with each process's local order

Causal:       If A caused B, everyone sees A before B
              Unrelated events may be in any order

Eventual:     Given enough time, all replicas converge
              No guarantees on WHEN (could be seconds or minutes)

Read-your-writes: You always see your own writes
              (others might not immediately)
```

---

## 6. Distributed Consensus

### Raft Protocol
```
Roles: Leader, Follower, Candidate

Normal operation:
  Client → Leader → replicates to Followers → majority ACK → committed

Leader election:
  1. Follower times out (no heartbeat from leader)
  2. Becomes Candidate, increments term, votes for self
  3. Requests votes from all nodes
  4. Majority votes → becomes Leader
  5. Sends heartbeats to maintain authority

Split brain prevention:
  - Only one leader per term
  - Need majority to elect → two leaders impossible with same term
  - If stale leader contacts node with higher term → steps down

Used by: etcd, CockroachDB, TiKV, Consul
```

### Two-Phase Commit (2PC)
```
Distributed transaction across multiple nodes.

Phase 1 (PREPARE):
  Coordinator: "Can you all commit?"
  All nodes: Lock resources, write to WAL, respond YES/NO

Phase 2 (COMMIT/ABORT):
  If ALL said YES: Coordinator: "COMMIT!"
  If ANY said NO: Coordinator: "ABORT!"

⚠️ Problem: Coordinator crashes between phases → blocking!
Solution: Three-Phase Commit (3PC) or Saga pattern
```

---

## 7. Failure Modes in Distributed Systems

```
Fail-stop:      Node crashes and stays down (detectable)
Fail-slow:      Node becomes slow (hardest to detect!)
Byzantine:      Node behaves arbitrarily (malicious/buggy)
Network:        Messages lost, delayed, or reordered

Handling strategies:
─────────────────────
Timeouts:       "If no response in 5s, assume failed"
Retries:        With exponential backoff + jitter
Idempotency:    Safe to retry (same result on repeat)
Circuit Breaker: Stop calling failing service (fail fast)
Bulkhead:       Isolate failures (one service down ≠ all down)
Redundancy:     Replicate everything (data, services, regions)
Checksums:      Detect corrupted data
Heartbeats:     Detect dead nodes
Quorum:         Majority agreement (tolerates minority failures)
```

---

## 8. Clock and Time in Distributed Systems

```
Problem: No global clock! Each machine's clock drifts differently.

Physical clocks (NTP):
  Accuracy: ~1-10ms within datacenter, ~100ms across internet
  ⚠️ Not reliable for ordering events!

Logical clocks:
  Lamport timestamps: Counter incremented on each event
    If A→B then timestamp(A) < timestamp(B)
    But: timestamp(A) < timestamp(B) does NOT mean A caused B

  Vector clocks: [clock_per_node]
    Detects causality AND concurrency
    If VC(A) < VC(B): A happened before B
    If neither < other: A and B are concurrent (conflict!)
    Used by: DynamoDB (conflict detection)

Hybrid Logical Clocks (HLC):
  Physical time + logical counter
  Best of both: meaningful timestamps + causal ordering
  Used by: CockroachDB, YugabyteDB
```

---

## 9. Interview Questions (Top 15)

1. What is a race condition? How to prevent it?
2. Mutex vs Semaphore — when to use which?
3. Explain deadlock. How to prevent/detect?
4. What is the CAP theorem? Give examples of CP and AP systems.
5. Explain eventual consistency. When is it acceptable?
6. How does Raft consensus work?
7. What is a distributed lock? How to implement with Redis?
8. Explain the difference between optimistic and pessimistic locking.
9. What is a vector clock? Why can't we use wall-clock time?
10. How would you design a distributed counter (millions of increments/sec)?
11. Explain the Saga pattern for distributed transactions.
12. What is split-brain? How to prevent it?
13. Python GIL — what is it and how to work around it?
14. Explain async/await. How is it different from threading?
15. What is idempotency? Why is it critical in distributed systems?
