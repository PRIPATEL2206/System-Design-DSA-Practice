# Scalability & Reliability — Deep Dive

## Part 1: Understanding Scale

### What Does "Scale" Actually Mean?
Scale is not just about more users. It encompasses:

```
Volume    → More data (TB → PB)
Velocity  → Faster writes (1K → 1M RPS)
Variety   → More data types, schemas evolving
Users     → Global, diverse, concurrent
Geography → Multi-region, latency constraints
```

### The Bottleneck Always Moves
```
Stage 1: Single server handles everything (startup)
Stage 2: DB becomes bottleneck → add caching
Stage 3: App servers bottleneck → add load balancer + horizontal scale
Stage 4: DB write bottleneck → replicate reads, optimise queries
Stage 5: DB write still bottlenecks → shard
Stage 6: Cache bottleneck → Redis Cluster
Stage 7: Network bandwidth → CDN, edge computing
```

**Key principle:** Don't pre-optimise. Scale each layer when it actually becomes the bottleneck.

---

## Part 2: Horizontal vs Vertical — When Each Wins

### Vertical Scaling — Limits and Use Cases
```
Physical limits (2024 maximums):
  CPU:  192 cores (x86), 128 (ARM)
  RAM:  24 TB (specialized servers), 3 TB (commodity)
  NVMe: 64 TB per server

When vertical is correct:
  1. Relational DB primary node (avoid distributed complexity)
  2. Legacy monolith (too expensive to refactor)
  3. In-memory data grid (single huge node for cache coherence)
  4. Startup phase (< 10M users)

Cost curve: Not linear. 2x RAM ≠ 2x cost (usually 3-4x at high end).
```

### Horizontal Scaling — Making It Work
For horizontal scaling to work, services must be **stateless**:

```
Stateful (hard to scale):            Stateless (easy to scale):
  Server stores user session             Session stored in Redis
  Server stores uploaded file            File stored in S3
  Server caches computation results      Cache in Redis/Memcached
  
Rule: "Any server can handle any request"
```

### Amdahl's Law
Even with infinite servers, speedup is limited by the sequential portion:

```
Speedup(N) = 1 / (S + (1-S)/N)

Where S = fraction of work that must be sequential
      N = number of processors/servers

Example: 20% of work is sequential (S=0.2)
  10 servers:  speedup = 1 / (0.2 + 0.8/10) = 3.6x (not 10x!)
  100 servers: speedup = 1 / (0.2 + 0.8/100) = 4.9x

Implication: Reduce sequential bottlenecks (locks, single-threaded DB writes) first.
```

---

## Part 3: Reliability Engineering (SRE Concepts)

### Error Budgets
Google SRE invented the error budget concept:

```
SLO = 99.9% availability
Error budget = 0.1% of time = 43.8 minutes/month

If errors consume budget → reliability work takes priority over features.
If budget unspent → team can take more risks (fast releases, experiments).

This creates alignment between dev velocity and reliability.
```

### Measuring Reliability

#### The Four Golden Signals (Google SRE)
```
1. Latency    — How long do requests take? (normal vs error latency separately)
2. Traffic    — How much demand? (RPS, queries/sec)
3. Errors     — Rate of failed requests (5xx, timeouts, wrong results)
4. Saturation — How full is the system? (CPU%, disk%, queue depth)
```

#### The USE Method (Brendan Gregg)
For every resource (CPU, disk, network):
```
U — Utilization: % time resource is busy
S — Saturation: amount of extra work queued
E — Errors: count of error events
```

### Failure Modes

#### Cascade Failure
```
Service A → calls → Service B (slow)
                 → Service A threads fill up waiting for B
                 → Service A becomes slow
                 → Service C → calls → Service A (also slow now)
                 → Entire system collapses

Prevention:
  - Circuit Breakers (fail fast instead of waiting)
  - Bulkheads (dedicated thread pools per dependency)
  - Timeouts (never wait forever)
  - Retry budgets (limit total retries across system)
```

#### Split Brain
```
Network partition splits cluster into two groups.
Both groups think they are the leader.
Both accept writes → divergent data.

Prevention:
  - Require majority (quorum) to elect leader
  - Raft/Paxos ensure only one leader at a time
  - Fencing tokens to invalidate old leader
```

---

## Part 4: Distributed Systems Failure Taxonomy

### The 8 Fallacies of Distributed Computing (Peter Deutsch, 1994)
These assumptions programmers make that are always false:

```
1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.
```

Every distributed system design must account for these.

### Failure Types
```
Crash failures:    Node stops responding permanently (easiest to handle)
Omission failures: Node sometimes doesn't respond (network drop)
Timing failures:   Node responds but too slow (timeout handling)
Byzantine failures: Node responds with incorrect/malicious data (hardest)
```

Most production systems design for crash + omission failures.
Byzantine fault tolerance is rare (blockchain, military systems).

---

## Part 5: Timeouts, Retries, and Backoff

### Timeout Values — How to Choose
```
Rule: timeout < P99 latency of the dependency (not P50!)

If DB P99 = 100ms → set timeout = 200ms (give some buffer)
If timeout = 10ms and P99 = 100ms → 99% of requests will timeout!

Cascading timeouts:
  Service A timeout for B = 1000ms
  Service B timeout for C = 1000ms
  Actual timeout = 1000ms (not 2000ms — A gives up first)

Set timeouts per layer to be less than the upstream timeout.
```

### Retry Strategy

#### When to Retry
```
Retry:
  - Network timeouts (transient)
  - HTTP 503 Service Unavailable
  - HTTP 429 Too Many Requests (with backoff)
  
Never retry:
  - HTTP 400 Bad Request (client error — retrying won't help)
  - HTTP 401/403 Authentication errors
  - Non-idempotent operations without idempotency keys
```

#### Exponential Backoff with Jitter
```python
def retry_with_backoff(func, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return func()
        except RetryableError:
            if attempt == max_attempts - 1:
                raise
            sleep_time = (2 ** attempt) + random.uniform(0, 1)  # jitter
            time.sleep(sleep_time)

# Attempts sleep: ~1s, ~2s, ~4s, ~8s, ~16s
# Jitter prevents thundering herd (all retries at same time)
```

### Why Jitter Matters
```
Without jitter:
  1000 clients all get 503 at T=0
  All retry at T=1 second simultaneously
  → Another wave of 1000 requests → another 503
  → Retry storm, system never recovers

With jitter:
  Retries spread over [1s, 2s] window
  → Gradual ramp-up → system recovers
```

---

## Part 6: Load Balancing Algorithms — Deep Dive

### Round Robin — Why It's Not Always Best
```
Servers: [A(fast), B(slow), C(fast)]
Round robin sends equal requests to all.
B processes slower → builds up queue → latency spikes for those users.

Problem: Doesn't account for server capacity or current load.
```

### Weighted Least Connections
```
Route to server with fewest active connections, weighted by capacity.

Server A: 8 connections, weight 2  → score = 8/2 = 4
Server B: 3 connections, weight 1  → score = 3/1 = 3  ← route here
Server C: 10 connections, weight 4 → score = 10/4 = 2.5 ← or here

Best for: Long-lived connections, varying request complexity.
Used by: Nginx, HAProxy in production.
```

### Consistent Hashing for Load Balancers
```
Use case: Sticky sessions without storing state in LB.

hash(user_id) → always routes same user to same server
(as long as server is healthy)

When server added/removed: only 1/N users re-routed (not all).
Servers can cache user-specific data safely.

Used by: AWS ALB target groups, Nginx upstream module.
```

---

## Part 7: Chaos Engineering

Deliberately injecting failures to test resilience.

### Netflix Chaos Monkey Principles
```
Simian Army tools:
  Chaos Monkey:    Randomly terminates production instances
  Latency Monkey:  Introduces artificial latency between services
  Chaos Gorilla:   Simulates failure of entire AWS availability zone
  Chaos Kong:      Simulates failure of entire AWS region

Philosophy:
  "If it hurts, do it more often" — surface failure paths before they happen in prod.
  Build systems that survive expected failures.
```

### Game Days
```
Scheduled exercises:
  1. Define failure scenario: "US-East-1 becomes unavailable"
  2. Notify key engineers
  3. Simulate failure in staging or production
  4. Observe system behaviour, team response, runbooks
  5. Document findings, fix gaps
  6. Repeat
```

---

## Part 8: Real-World Reliability Numbers

```
AWS SLAs (2024):
  EC2: 99.99%
  S3:  99.99%
  RDS Multi-AZ: 99.95%
  DynamoDB: 99.999%
  
Google Cloud Storage: 99.999999999% (11 nines) durability
  
Netflix:
  Global availability: > 99.99%
  Achieved via: 3 AWS regions, CDN, aggressive retry
  
Amazon.com:
  1 hour of downtime = ~$34M lost revenue (estimated)
  → Justifies massive investment in availability
```

---

## Key Takeaways

1. **Profile before optimising** — measure to find the actual bottleneck.
2. **Stateless services** scale horizontally; state goes to Redis/S3/DB.
3. **Timeout + circuit breaker** prevents cascade failures.
4. **Jitter in retries** prevents thundering herd.
5. **Error budgets** align engineering priorities with product goals.
6. **Chaos testing** exposes failure modes before customers do.
7. **Amdahl's Law** — reduce sequential bottlenecks, not just add servers.
