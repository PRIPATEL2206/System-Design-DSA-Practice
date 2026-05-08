# Distributed Systems Concepts

## 1. Consistent Hashing

**Problem:** You have 5 cache servers. You hash requests: `server = hash(key) % 5`. Adding a 6th server changes the formula → ~83% of keys remapped → cache invalidated → thundering herd.

**Solution:** Consistent hashing maps both servers AND keys onto a ring. Adding/removing a server only remaps ~1/N keys.

```
         Ring (0 to 2^32)
              0
           /      \
     Server A    Server D
        |                |
     Server B    Server C
           \      /
             ...

Key X → hash(X) → position on ring → travel clockwise → first server = responsible server.

Add Server E between A and B:
  Only keys that were going to B (in A→B range) now go to E.
  Everything else unchanged.
```

### Virtual Nodes
**Problem:** With 3 servers on a ring, distribution is uneven (lucky/unlucky positions).

**Solution:** Each server occupies multiple positions (virtual nodes) on the ring. Default: 150 virtual nodes per server in Cassandra.

**Benefits:**
- Even distribution of keys
- When a server dies, its load is spread across many servers, not one

**Used by:** Amazon DynamoDB, Apache Cassandra, Memcached (ketama), Redis Cluster

---

## 2. Leader Election

**Why:** In a distributed cluster, one node needs to be the "primary" to coordinate work, avoid split-brain.

### Approaches

#### ZooKeeper / etcd (most common in practice)
```
All nodes try to create ephemeral node /leader.
Only one succeeds → becomes leader.
If leader dies → ephemeral node deleted → election starts again.
```

#### Raft Consensus Leader Election
```
1. All nodes start as Followers.
2. If a Follower doesn't hear from a Leader, it becomes a Candidate.
3. Candidate sends RequestVote to all nodes.
4. Majority vote → becomes Leader.
5. Leader sends heartbeats to maintain authority.
```

**Used by:** etcd (Kubernetes uses it), CockroachDB, TiKV, Consul

---

## 3. Consensus: Raft (Simplified)

**What it solves:** Get N distributed nodes to agree on a sequence of values (log entries), even when some nodes fail.

```
Log Replication:
1. Client sends write to Leader.
2. Leader appends to its log (uncommitted).
3. Leader sends AppendEntries to all Followers.
4. Once majority ACK → Leader commits.
5. Leader notifies Followers to commit.
6. Response returned to Client.

Safety property: A committed entry will never be lost.
System available as long as majority (N/2 + 1) of nodes are up.
```

**Used by:** etcd, CockroachDB, TiKV, RethinkDB

---

## 4. Distributed Transactions

**Problem:** How to update data across multiple services/databases atomically?

### Two-Phase Commit (2PC)
```
Phase 1 (Prepare):
  Coordinator → "Can you commit?" → all Participants
  Participants acquire locks, write to temp area, reply "Yes" or "No"

Phase 2 (Commit):
  If ALL said Yes → Coordinator → "Commit" → all Participants
  If ANY said No  → Coordinator → "Rollback" → all Participants

Pro: Atomic, consistent
Con: Blocking (coordinator failure = all participants stuck)
     Performance (two round trips, locks held)
Use: Only for tightly-coupled systems; not recommended for microservices
```

### Saga Pattern (Preferred for Microservices)
```
Break one distributed transaction into a sequence of local transactions,
each with a compensating transaction (rollback action).

Choreography (event-driven):
Order → [OrderCreated event] → Inventory → [StockReserved event] → Payment → [Charged event] → Order (complete)
                                          ← [PaymentFailed event] → Inventory → [StockReleased]

Orchestration (central coordinator):
[Saga Orchestrator] → calls each service in sequence
                    → on failure, calls compensating transactions in reverse

Pro: Loosely coupled, no locking, scales well
Con: Eventual consistency; complex rollback logic; harder to debug
```

**Interview tip:** For microservices, always suggest Saga over 2PC. Mention that each step must be idempotent.

---

## 5. Bloom Filter

**What it is:** A probabilistic data structure that answers "Is element X in the set?" with:
- **No** → Definitely not in the set
- **Yes** → Probably in the set (false positives possible; false negatives impossible)

```
Example: Check if a URL has been crawled before
- Bloom filter lookup: O(1), < 10 bytes per element
- Hash table lookup: O(1), ~50+ bytes per element

Trade-off: 1% false positive rate with ~10 bits per element
Use: Pre-check before expensive DB lookup
```

**Used by:** Cassandra (check if key exists before disk read), Chrome (safe browsing), Redis (as optional module)

---

## 6. Idempotency

**Definition:** An operation that can be applied multiple times with the same result as applying it once.

```
Idempotent:
  PUT /users/123 (replace with given data — same result every time)
  DELETE /users/123 (already deleted = same result)
  SET balance = 100 (same result)

NOT idempotent:
  POST /orders (creates a new order every time)
  PATCH balance += 10 (each call adds $10)
```

### Idempotency Keys
```
Client sends unique idempotency-key header with each request.
Server stores (idempotency_key → response) for 24 hours.
If same key received again → return cached response, don't re-process.

Used by: Stripe API, Amazon API, Twilio
```

---

## 7. Data Consistency Patterns

### Read-Your-Writes Consistency
```
Problem: User posts a tweet. Reads from replica → doesn't see their own tweet.
Solution: After write, route that user's reads to Primary for 1 minute.
Or: Write timestamp in session cookie; if replica is behind that timestamp, read from Primary.
```

### Monotonic Reads
```
Problem: User refreshes page. First load reads from Replica 1 (up-to-date).
Second load reads from Replica 2 (behind) → message disappears.
Solution: Route same user to same replica (sticky session / user ID hashing).
```

---

## 8. Service Patterns

### Circuit Breaker
```
State: CLOSED (normal) → OPEN (failing) → HALF-OPEN (probing)

CLOSED: Calls go through. Track failure rate.
OPEN:   Failure rate > threshold. All calls rejected immediately.
        After timeout → HALF-OPEN.
HALF-OPEN: Allow one test call. Success → CLOSED. Fail → OPEN.

Used by: Netflix Hystrix, Resilience4j, AWS SDK
```

### Backpressure
```
Slow consumer → queue fills up → producer slows down or drops
Prevents runaway memory growth in pipelines.
Used by: Reactive streams, Kafka consumer lag monitoring
```

---

## Quick Reference

| Concept | One-line |
|---------|---------|
| Consistent hashing | Distribute keys so adding/removing nodes remaps minimal keys |
| Virtual nodes | Multiple ring positions per server for even distribution |
| Raft | Consensus algorithm for distributed log replication |
| 2PC | Atomic commit across nodes; blocks on failure |
| Saga | Distributed transaction via sequence of local transactions + compensations |
| Bloom filter | Space-efficient: "definitely no" or "probably yes" |
| Idempotency key | Deduplicate retried requests |
| Circuit breaker | Stop calling failing service; let it recover |
