# Chapter 9: Consistency & Consensus

## The Consistency Spectrum

```
Strong ◀──────────────────────────────────────────────▶ Weak

Linearizable → Sequential → Causal → Eventual

Linearizable:  Every read sees the most recent write. Period.
               (single-threaded illusion)
               
Sequential:    All operations appear in SOME total order
               consistent with each process's order.

Causal:        If A causes B, everyone sees A before B.
               Concurrent events may be seen in any order.

Eventual:      Given enough time with no new writes,
               all replicas converge to the same value.
               (no guarantees about WHEN)
```

---

## Strong Consistency

### What it means
Every read returns the most recent write, regardless of which replica you read from.

### How to achieve it
```
Option 1: Single leader, read from leader only
           Pros: Simple
           Cons: Leader is bottleneck, not fault-tolerant

Option 2: Synchronous replication (all replicas ACK before success)
           Pros: Any replica has latest data
           Cons: Slow (wait for slowest replica), unavailable if one replica is down

Option 3: Consensus protocol (Raft, Paxos)
           Pros: Fault-tolerant, consistent
           Cons: Complex, higher latency (majority must agree)
```

### When to use
- Financial transactions (bank balances)
- Inventory management (prevent overselling)
- User authentication (password changes must be immediately visible)
- Distributed locks

---

## Eventual Consistency

### What it means
Writes propagate asynchronously. Replicas may temporarily disagree, but will eventually converge.

### Typical propagation time
- Same datacenter: 10-100ms
- Cross-region: 100ms - several seconds

### When to use
- Social media feeds (slightly stale is OK)
- Like/view counts (approximate is fine)
- DNS records (TTL-based propagation)
- Shopping cart (merge conflicts manageable)

---

## Consensus Algorithms

### Raft (Easier to Understand)
```
Roles:
- Leader:    Handles all writes, replicates to followers
- Follower:  Accepts replicated entries from leader
- Candidate: Trying to become leader (during election)

Write Flow:
1. Client sends write to Leader
2. Leader appends to its log
3. Leader sends to all Followers
4. Majority (N/2 + 1) acknowledge → committed
5. Leader responds to client

Leader Election:
- If follower doesn't hear from leader (timeout) → becomes candidate
- Requests votes from all nodes
- First to get majority votes → new leader
- Split vote → increment term, try again

         ┌──────────┐
    ┌───▶│  Leader  │◀─── client writes
    │    └────┬─────┘
    │         │ replicate
    │    ┌────┼────┐
    │    ▼    ▼    ▼
    │  ┌──┐ ┌──┐ ┌──┐
    │  │F1│ │F2│ │F3│  (Followers)
    │  └──┘ └──┘ └──┘
    │         │
    └─── heartbeat ───┘
```

### Paxos (The OG — More Complex)
Used by: Google Spanner, Chubby
Same idea: majority agreement, but with Proposer/Acceptor/Learner roles.

---

## Distributed Transactions

### Two-Phase Commit (2PC)
```
Phase 1 (Prepare):
  Coordinator → "Can you commit?" → All participants
  All respond: "Yes, I can" (or "No, abort")

Phase 2 (Commit):
  If ALL said yes: Coordinator → "Commit!" → All participants
  If ANY said no:  Coordinator → "Abort!" → All participants

Problem: Coordinator crashes between phases → participants stuck (blocking!)
```

### Saga Pattern (Preferred for Microservices)
```
Instead of one big transaction:
  T1 → T2 → T3 (each is a local transaction)

If T3 fails:
  C2 → C1 (compensating transactions undo previous steps)

Example (Book a trip):
  Book Flight → Book Hotel → Book Car
  If Car fails:
    Cancel Hotel → Cancel Flight

Two flavors:
- Choreography: Each service publishes events, next service reacts
- Orchestration: Central orchestrator tells each service what to do
```

---

## CRDTs (Conflict-Free Replicated Data Types)

Data structures that can be merged without conflicts — perfect for eventually consistent systems.

```
Examples:
─────────
G-Counter:     Only increments. Merge = take max of each node's count.
PN-Counter:    Increment + Decrement (two G-Counters).
G-Set:         Only adds elements. Merge = union.
OR-Set:        Add + Remove with unique tags.
LWW-Register:  Last-Write-Wins based on timestamp.

Used by: Redis CRDT, Riak, collaborative editors
```

---

## Distributed Locks

### Redis-based Lock (Redlock)
```python
# Acquire
SET lock_key unique_value NX PX 30000
# NX = only if not exists, PX = expire in 30s

# Release (Lua script for atomicity)
if GET lock_key == unique_value:
    DEL lock_key
```

### ZooKeeper-based Lock
```
1. Create ephemeral sequential node: /locks/lock-0000001
2. Check if your node has the LOWEST sequence number
3. If yes → you have the lock
4. If no → watch the node just before yours
5. When it's deleted → check again
```

---

## Interview Application

### Choosing Consistency Level by Use Case

```
┌─────────────────────────┬───────────────────────┬─────────────────────────────┐
│ Use Case                │ Consistency Needed    │ Why                         │
├─────────────────────────┼───────────────────────┼─────────────────────────────┤
│ Bank transfer           │ Strong (linearizable) │ Can't show wrong balance    │
│ Inventory (last item)   │ Strong                │ Can't oversell              │
│ User profile update     │ Read-your-writes      │ User should see own changes │
│ Social media feed       │ Eventual              │ Slight delay acceptable     │
│ View/like counters      │ Eventual              │ Approximate is fine         │
│ Shopping cart           │ Eventual + merge      │ CRDTs can handle conflicts  │
│ Leader election         │ Strong (consensus)    │ Must agree on ONE leader    │
│ Config updates          │ Sequential            │ All nodes see same order    │
└─────────────────────────┴───────────────────────┴─────────────────────────────┘
```

### Magic Phrase for Interviews
> "For this component, I'd choose [eventual/strong] consistency because [specific reason]. The trade-off is [availability/latency], which is acceptable here because [business justification]."
