# Distributed Consensus — Deep Dive

## Part 1: The Problem of Agreement

### Why Consensus is Hard
```
Distributed Consensus: Get N nodes to agree on a value when some nodes may fail.

Impossibility result (FLP, 1985):
  In an asynchronous distributed system with even one faulty process,
  it is impossible to guarantee consensus in bounded time.

Practical solution:
  Use timeouts (make the system partially synchronous).
  Accept that consensus may occasionally stall but will eventually complete.
```

### The Two Generals Problem
```
Two armies need to coordinate an attack via messengers.
Messengers can be captured (messages can be lost).

Army A sends: "Attack at dawn."
Army B receives and sends: "Confirmed."
But Army B doesn't know if Army A received the confirmation.

Result: No protocol can guarantee agreement if messages can be lost.

This is why distributed systems can never guarantee exactly-once
without additional assumptions (persistent storage, retries, idempotency).
```

---

## Part 2: Raft Consensus Algorithm — Complete Walkthrough

Raft is designed to be understandable. It powers etcd (Kubernetes) and CockroachDB.

### Three Roles
```
Follower:   Passive; receives heartbeats; votes in elections.
Candidate:  Actively seeking votes to become leader.
Leader:     One per term; handles all client writes; sends heartbeats.

State transitions:
  Follower ──election timeout──> Candidate ──majority vote──> Leader
                                            ──split vote/timeout──> Candidate (retry)
  Leader ──discovers higher term──> Follower
```

### Terms — Kafka's "Epochs"
```
Time is divided into terms (numbered 1, 2, 3, ...).
Each term begins with an election.
At most one leader per term.

A node with a stale term (term < current term) immediately becomes Follower.
Prevents old leaders from causing split-brain after network partition heals.
```

### Leader Election — Step by Step
```
Initial state: All nodes are Followers.

Election trigger:
  Follower receives no heartbeat within election timeout (150–300ms, randomised).
  → Converts to Candidate
  → Increments term (e.g., term = 2)
  → Votes for itself
  → Sends RequestVote(term=2, lastLogIndex, lastLogTerm) to all other nodes

Voting rules:
  Node votes Yes if:
    1. Hasn't voted this term yet
    2. Candidate's log is at least as up-to-date as own log
    
  "Log up-to-date" means:
    Higher last log term, OR
    Same last log term AND longer log

Win:
  Candidate receives majority (N/2 + 1 votes) → becomes Leader
  Sends Heartbeat to all Followers immediately (prevent new elections)

Loss/Split:
  No majority (votes split among candidates) → election timeout → retry
  Randomised timeouts prevent repeated splits
```

### Log Replication — How Writes Work
```
Client → Leader: "Set x = 5"

Step 1: Leader appends to its log as uncommitted entry
  Log: [..., (term=2, index=47, cmd="SET x=5")]

Step 2: Leader sends AppendEntries to all Followers
  AppendEntries(term=2, prevLogIndex=46, prevLogTerm=2, entries=[...], leaderCommit=46)

Step 3: Followers validate and append
  Check: Does my log match prevLogIndex/prevLogTerm? Yes → append entry
  Reply: Success

Step 4: Leader commits
  Once majority (3/5) ACK → entry is committed (durable)
  Leader applies to state machine: x = 5
  Leader includes commitIndex in next AppendEntries → Followers apply

Step 5: Leader responds to client
  "Success: x = 5"

Safety guarantee:
  A committed entry will NEVER be lost.
  Any future leader will have that entry (election voting ensures this).
```

### Log Inconsistency and Repair
```
Scenario: Follower crashed and missed some entries.
  Leader log:   [1, 2, 3, 4, 5]
  Follower log: [1, 2]

Leader detects mismatch via AppendEntries failure.
Leader sends entries 3, 4, 5 to follower → follower catches up.

Scenario: Follower has conflicting entries (was a leader in old term).
  Leader log:   [1, 2, 3, 4, 5]  (current term)
  Follower log: [1, 2, 3, X, Y]  (from different old leader)

Leader tells follower: "Your log diverges at index 3. Delete from index 3 onward. Here's the correct log."
Follower overwrites with leader's log.

Rule: Leader's log ALWAYS wins. Followers always eventually converge to leader's log.
```

---

## Part 3: ZooKeeper — Distributed Coordination

### What ZooKeeper Provides
```
ZooKeeper is not a general-purpose database.
It provides primitives for distributed coordination:

1. Hierarchical namespace (like a filesystem)
   /kafka/brokers/ids/1
   /kafka/controller
   /service/leader
   
2. Watches (event notification)
   Watch a znode → get notified when it changes
   
3. Ephemeral znodes (disappear when session ends)
   Perfect for detecting failures (node crash = ephemeral node gone)
   
4. Sequential znodes (auto-numbered)
   /locks/request-000001, /locks/request-000002, ...
   Used to implement distributed queues and locks
```

### Distributed Lock with ZooKeeper
```
Algorithm:
  1. Create ephemeral sequential node: /locks/lock-000042
  2. Get all children of /locks
  3. If my node has the smallest number → I hold the lock
  4. Else → Watch the node just before mine
  5. When that node is deleted → I'm next → retry step 2-3

Why watch only the previous node (not smallest)?
  Herd effect prevention: If all watch smallest, deletion triggers N watches → thundering herd
  Watching predecessor: only one watcher triggered per lock release → O(1) notifications

Unlock:
  Delete my ephemeral sequential node.
  ZooKeeper notifies the next waiter.
```

### Leader Election with ZooKeeper
```
1. All nodes create ephemeral sequential node under /leader
   Node A creates: /leader/member-000001
   Node B creates: /leader/member-000002
   Node C creates: /leader/member-000003

2. Node with smallest sequence number → is the Leader

3. Other nodes watch the node just before them (same as lock above)

4. If leader (member-000001) crashes:
   Ephemeral node deleted automatically (session expires)
   Node B (member-000002) is notified → becomes leader
   
No external monitoring needed (ZooKeeper handles failure detection via sessions).
```

### ZooKeeper Limitations
```
1. Single-threaded processing (all requests serialised through leader)
   Max ~30,000 operations/second
   Not suitable for high-frequency data (Kafka uses it for metadata, NOT messages)

2. Ensemble size must be odd
   3 nodes: tolerate 1 failure
   5 nodes: tolerate 2 failures
   ZAB protocol (similar to Raft) requires majority to operate

3. Not suitable for storing large data
   Designed for small config/coordination data (< 1 MB per znode)
```

---

## Part 4: etcd (Kubernetes' Brain)

### What etcd Is
```
etcd is a distributed key-value store using Raft consensus.
Kubernetes stores ALL cluster state in etcd:
  - Pod definitions
  - Service configurations
  - Secrets
  - ConfigMaps
  - Namespace objects
  - RBAC policies
  
If etcd loses data → Kubernetes cluster state is gone.
Backup etcd before any major cluster operation.
```

### etcd vs ZooKeeper
| Feature | etcd | ZooKeeper |
|---------|------|-----------|
| Consensus | Raft | ZAB (Paxos-like) |
| API | gRPC / REST | Custom |
| Data model | Key-value with MVCC | Hierarchical (znodes) |
| Watch semantics | Range watches | Individual node watches |
| Performance | Higher (gRPC, binary) | Moderate |
| Language | Go | Java |
| Used by | Kubernetes, CoreDNS | Kafka (being replaced), HBase |

---

## Part 5: Two-Phase Commit (2PC) — Why It Fails in Practice

### The Algorithm
```
Coordinator sends:  "Prepare to commit transaction T"
Participants:       Lock resources, write to temp area, vote Yes/No

If ALL vote Yes:
  Coordinator:      "Commit"
  Participants:     Commit and release locks

If ANY votes No:
  Coordinator:      "Rollback"
  Participants:     Rollback and release locks
```

### Problems with 2PC

#### Blocking on Coordinator Failure
```
Phase 1 completes: all participants voted Yes, waiting for commit/rollback.
Coordinator crashes.

Participants:
  Cannot commit (might violate atomicity if not all committed)
  Cannot abort (coordinator might have already told others to commit)
  STUCK. Locks held. DB unavailable until coordinator recovers.

This is why 2PC is called a "blocking protocol."
```

#### Performance
```
2 round trips + locking = high latency.
For 5 microservices: 2 × 5 = 10 network round trips minimum.
During Phase 1: locks held → contention → throughput reduced.
```

#### Why Use It Anyway?
```
Internal to a single DB engine: 2PC is fine (fast local network, reliable).
MySQL XA transactions, PostgreSQL distributed transactions.

Between microservices: Use Saga pattern instead.
```

---

## Part 6: The Saga Pattern — Distributed Transactions Without 2PC

### Core Idea
```
Replace one distributed transaction with N local transactions,
each with a compensating transaction (the "undo").

Example: E-commerce order flow
  1. Order Service:     Create order (local)                   → Undo: Cancel order
  2. Inventory Service: Reserve items (local)                  → Undo: Release items
  3. Payment Service:   Charge card (local)                    → Undo: Refund charge
  4. Shipping Service:  Schedule shipment (local)              → Undo: Cancel shipment

If Step 3 (Payment) fails:
  Execute compensating transactions in reverse:
    Undo Step 2: Release inventory
    Undo Step 1: Cancel order
```

### Orchestration vs Choreography

#### Choreography (Event-Driven)
```
Each service listens for events and publishes events.
No central coordinator.

[Order Created] → Inventory Service → [Stock Reserved]
                                    → Payment Service → [Payment Charged]
                                                      → Shipping Service → [Shipped]

On failure:
[Payment Failed] → Inventory Service → [Stock Released]
                → Order Service → [Order Cancelled]

Pro: Loosely coupled, easy to add new services
Con: Hard to trace saga across services, implicit dependencies
```

#### Orchestration (Central Coordinator)
```
Saga Orchestrator explicitly calls each service:
  orchestrator.step1() → Order created
  orchestrator.step2() → Inventory reserved
  orchestrator.step3() → Payment charged (fails!)
  orchestrator.compensate(step2) → Inventory released
  orchestrator.compensate(step1) → Order cancelled

Pro: Explicit flow, easy to monitor and debug
Con: Orchestrator is a central dependency
```

---

## Key Takeaways

1. **FLP impossibility**: consensus cannot be deterministic in async systems → use timeouts.
2. **Raft**: leader election via timeouts; log replication requires majority ACK; leader log always wins.
3. **ZooKeeper**: ephemeral sequential znodes are the building block for locks, leader election, queues.
4. **etcd**: Raft-based KV store; entire Kubernetes state lives here — protect it.
5. **2PC blocks on coordinator failure** → not suitable for microservices.
6. **Saga**: each step has a compensating transaction; orchestration for complex flows.
7. **Idempotency is mandatory** for Saga — compensating transactions may be called multiple times.
