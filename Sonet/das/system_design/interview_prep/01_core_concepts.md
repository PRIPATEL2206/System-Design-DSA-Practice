# Core Concepts

## 1. Scalability

**Definition:** The ability to handle growing load by adding resources.

### Vertical vs Horizontal Scaling

| Aspect | Vertical (Scale Up) | Horizontal (Scale Out) |
|--------|--------------------|-----------------------|
| How | Bigger machine (more CPU/RAM) | More machines |
| Limit | Hard ceiling (largest machine) | Theoretically unlimited |
| Cost | Expensive, diminishing returns | Cheaper commodity hardware |
| Complexity | Simple, no code changes | Needs load balancer, stateless design |
| Downtime | Requires restart | No downtime (rolling deploy) |
| Use when | DB primary node, quick fix | App servers, stateless services |

**Rule of thumb:** Start vertical, go horizontal when you hit the ceiling.

---

## 2. Latency vs Throughput

| Term | Definition | Example |
|------|-----------|---------|
| **Latency** | Time for one request to complete | 50ms to load a page |
| **Throughput** | Requests per second the system handles | 10,000 RPS |
| **Bandwidth** | Max data transfer rate of a link | 1 Gbps network card |

**Key insight:** You can have high throughput but also high latency (batch processing).
Optimising for both requires careful design.

---

## 3. Availability & Reliability

### Availability = Uptime / Total Time

| SLA | Allowed Downtime/Year | Used By |
|-----|----------------------|---------|
| 99% (two nines) | 3.65 days | Acceptable for internal tools |
| 99.9% (three nines) | 8.7 hours | Standard web services |
| 99.99% (four nines) | 52 minutes | Payment systems, healthcare |
| 99.999% (five nines) | 5.2 minutes | Telecom, critical infrastructure |

### How to Achieve High Availability
```
1. Eliminate single points of failure (SPOF)
2. Replicate everything (DB, cache, app servers)
3. Use health checks + auto-failover
4. Deploy across multiple Availability Zones (AZs)
5. Use circuit breakers to prevent cascade failures
```

### SLI, SLO, SLA (Google SRE model)
- **SLI** (Indicator): Actual measurement (e.g., % of requests < 200ms)
- **SLO** (Objective): Internal target (e.g., 99.9% of requests < 200ms)
- **SLA** (Agreement): Contract with customers (e.g., 99.5% uptime or refund)

---

## 4. CAP Theorem

**"In a distributed system, you can only guarantee 2 of 3: Consistency, Availability, Partition Tolerance."**

```
         Consistency
            /\
           /  \
          / CA \   ← Not realistic in network; partitions always happen
         /      \
        /--------\
  CP   /    ?    \ AP
      /            \
Availability ---- Partition Tolerance
```

| Choice | Meaning | Examples |
|--------|---------|---------|
| **CP** (Consistent + Partition-tolerant) | Returns error or waits rather than stale data | HBase, Zookeeper, etcd, MongoDB (default config) |
| **AP** (Available + Partition-tolerant) | Returns possibly stale data during partition | Cassandra, DynamoDB, CouchDB |
| **CA** (Consistent + Available) | Only possible if there are no network partitions (single node) | Traditional RDBMS on single node |

**Interview tip:** Partitions WILL happen in the real world. You're always choosing between CP and AP. Ask: "Is it okay to show stale data? Or must all reads be up-to-date?"

---

## 5. Consistency Models

Ordered from strongest to weakest:

| Model | Guarantee | Latency | Example |
|-------|-----------|---------|---------|
| **Linearisability** (Strong) | Every read sees the most recent write | High | Single-node DB, etcd |
| **Sequential** | All operations appear in a single order | Medium-High | — |
| **Causal** | Causally-related operations are ordered | Medium | MongoDB causal sessions |
| **Eventual** | If no new writes, all replicas converge | Low | DynamoDB, Cassandra |

**Common question:** "Which consistency model does your design use and why?"

---

## 6. ACID vs BASE

### ACID (Relational Databases)
| Property | Meaning |
|----------|---------|
| **A**tomicity | All operations in a transaction succeed or all fail |
| **C**onsistency | DB moves from one valid state to another |
| **I**solation | Concurrent transactions don't see each other's intermediate state |
| **D**urability | Committed data survives crashes (written to disk) |

### BASE (NoSQL / Distributed Systems)
| Property | Meaning |
|----------|---------|
| **B**asically **A**vailable | System guarantees availability (may be stale) |
| **S**oft state | System state may change over time even without input |
| **E**ventual consistency | System will become consistent given enough time |

**When to use ACID:** Financial transactions, inventory, anything where correctness > speed.  
**When to use BASE:** Social feeds, analytics, shopping carts, anything where speed > perfect accuracy.

---

## 7. Fault Tolerance Patterns

### Replication
```
Primary ──writes──> Replica 1
         ──writes──> Replica 2  (reads served from replicas)
```

### Circuit Breaker
```
State Machine:
CLOSED (normal) ──failures exceed threshold──> OPEN (reject all calls)
OPEN ──after timeout──> HALF-OPEN (try one request)
HALF-OPEN ──success──> CLOSED
HALF-OPEN ──failure──> OPEN
```

### Retry with Exponential Backoff
```
Attempt 1: fail → wait 1s
Attempt 2: fail → wait 2s
Attempt 3: fail → wait 4s
Attempt 4: fail → wait 8s + jitter
```

### Bulkhead Pattern
Isolate services so one failure doesn't bring down everything.
```
[User Service Pool]   [Order Service Pool]   [Payment Pool]
   Thread pool 1         Thread pool 2         Thread pool 3
   (isolated)             (isolated)            (isolated)
```

---

## Quick Reference

| Concept | One-line Answer |
|---------|----------------|
| Horizontal scaling | Add more servers |
| Latency | Time per request |
| Throughput | Requests per second |
| CAP theorem | Pick 2: Consistency, Availability, Partition Tolerance |
| ACID | Strong guarantees for relational DBs |
| BASE | Relaxed guarantees for NoSQL / distributed systems |
| Eventual consistency | Replicas will converge, but may be temporarily stale |
| Circuit breaker | Stop calling a failing service to prevent cascade failure |
