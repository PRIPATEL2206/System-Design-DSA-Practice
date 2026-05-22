# DevOps Phase 12 — Scaling & Production Systems
## High Availability · Horizontal Scaling · Distributed Systems · CAP Theorem · Reliability

---

> **Who this is for:** Beginners who want to understand how real-world production systems handle millions of users, stay online 24/7, and recover from failures automatically.

---

## Table of Contents

1. [Why Scaling Matters](#1-why-scaling-matters)
2. [Vertical vs Horizontal Scaling](#2-vertical-vs-horizontal-scaling)
3. [High Availability (HA)](#3-high-availability-ha)
4. [Distributed Systems Fundamentals](#4-distributed-systems-fundamentals)
5. [CAP Theorem](#5-cap-theorem)
6. [Database Scaling](#6-database-scaling)
7. [Caching Strategies](#7-caching-strategies)
8. [Message Queues & Async Processing](#8-message-queues--async-processing)
9. [Auto-Scaling](#9-auto-scaling)
10. [Reliability Engineering (SRE)](#10-reliability-engineering-sre)
11. [Disaster Recovery](#11-disaster-recovery)
12. [Interview Mastery](#12-interview-mastery)

---

## 1. Why Scaling Matters

### Beginner Explanation

Imagine you run a coffee shop. On a normal day, one barista handles all orders fine. But on a holiday, 100 customers show up at once. What do you do?

- **Scale up (vertical):** Replace the barista with a super-fast robot (bigger machine)
- **Scale out (horizontal):** Hire 10 more baristas (more machines)

In software, "scaling" means ensuring your system can handle increased load (more users, more data, more requests) without becoming slow or crashing.

### Real-World Scale Numbers

```
Small startup:        ~1,000 users, 10 requests/second
Medium SaaS:          ~100,000 users, 1,000 req/s
Large platform:       ~10M users, 100,000 req/s
Netflix:              ~250M users, millions of req/s
Google Search:        ~8.5 billion searches/day = ~100,000 req/s

At scale, everything breaks differently:
- 10 req/s:   Single server is fine
- 1,000 req/s: Need load balancer + multiple servers
- 100K req/s:  Need distributed system, caching, CDN
- 1M+ req/s:   Need global distribution, sharding, custom infrastructure
```

### What Happens When You Don't Scale

```mermaid
flowchart TD
    A[Traffic increases] --> B{System scaled?}
    B -->|No| C[Response time increases\n200ms → 5000ms]
    C --> D[Timeouts start occurring]
    D --> E[Connection pool exhausted]
    E --> F[Cascading failure\nDependencies timeout too]
    F --> G[Complete outage\nAll users affected]
    B -->|Yes| H[System handles load\nResponse time stays < 200ms]
```

---

## 2. Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

```
Before:                     After:
┌─────────────────┐        ┌─────────────────────────┐
│  2 CPU cores    │        │  32 CPU cores           │
│  4 GB RAM      │  ───►  │  128 GB RAM             │
│  100 GB SSD    │        │  2 TB NVMe SSD          │
│  1 Gbps NIC    │        │  25 Gbps NIC            │
└─────────────────┘        └─────────────────────────┘
  t3.medium ($30/mo)         r6i.8xlarge ($1,500/mo)
```

**Advantages:**
- Simple — no code changes needed
- No distributed system complexity
- Single point of management

**Disadvantages:**
- Hardware limits (you can't buy a 10,000 core machine)
- Single point of failure (one machine dies = total outage)
- Cost grows exponentially (2x CPU ≠ 2x cost, usually 3-4x)
- Downtime during upgrade (must stop, resize, start)

### Horizontal Scaling (Scale Out)

```
Before:                     After:
┌───────────┐              ┌───────────┐
│  Server A │              │  Server A │
│  (all     │              ├───────────┤
│   traffic)│              │  Server B │
└───────────┘              ├───────────┤
                           │  Server C │
                           ├───────────┤
                           │  Server D │
                           └───────────┘
                           Load Balancer distributes
```

**Advantages:**
- Virtually unlimited scaling (add more machines)
- No single point of failure (one dies, others continue)
- Cost scales linearly (2x machines ≈ 2x cost)
- Scale incrementally (add 1 server at a time)
- Can scale down during off-peak (save money)

**Disadvantages:**
- Application must be designed for it (stateless)
- Distributed system complexity (consistency, coordination)
- Need load balancer, service discovery
- Data synchronization challenges

### When to Use Which

| Scenario | Recommended |
|---|---|
| Database (single write master) | Vertical first, then shard |
| Stateless web servers | Horizontal (auto-scale) |
| Cache (Redis) | Vertical first (RAM), then cluster |
| ML model inference | Horizontal (multiple GPU instances) |
| Legacy monolith | Vertical (can't easily distribute) |
| Microservices | Horizontal (designed for it) |

---

## 3. High Availability (HA)

### Beginner Explanation

High Availability means your system stays running even when things break. "Things" includes servers crashing, disks failing, data centers losing power, entire regions going offline.

**Availability is measured in "nines":**

| Availability | Downtime/year | Downtime/month | Typical SLA |
|---|---|---|---|
| 99% (two nines) | 3.65 days | 7.3 hours | Dev/staging |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes | Most SaaS |
| 99.95% | 4.38 hours | 21.9 minutes | AWS S3 |
| 99.99% (four nines) | 52.6 minutes | 4.38 minutes | Enterprise critical |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds | Financial systems |

**To calculate:** `Availability = (Total Time - Downtime) / Total Time × 100`

### HA Architecture Patterns

```mermaid
flowchart TB
    subgraph Region1 ["AWS Region: us-east-1"]
        subgraph AZ1 ["Availability Zone A"]
            A1[Web Server 1]
            D1[(DB Primary)]
        end
        subgraph AZ2 ["Availability Zone B"]
            A2[Web Server 2]
            D2[(DB Standby\nSync Replica)]
        end
        subgraph AZ3 ["Availability Zone C"]
            A3[Web Server 3]
            D3[(DB Read Replica)]
        end
        LB[Application\nLoad Balancer]
    end
    
    Users[Users] --> LB
    LB --> A1
    LB --> A2
    LB --> A3
    D1 -->|Sync Replication| D2
    D1 -->|Async Replication| D3
```

### Redundancy at Every Layer

```
Layer 1: DNS
  → Multiple nameservers, health-check-based failover (Route53)

Layer 2: Load Balancer
  → AWS ALB is inherently HA (multi-AZ)
  → Self-hosted: active-passive LB pair with VRRP/keepalived

Layer 3: Application
  → Multiple instances across availability zones
  → Auto-scaling group maintains minimum healthy count

Layer 4: Database
  → Primary + synchronous standby (automatic failover)
  → Read replicas for read scaling
  → Regular automated backups + point-in-time recovery

Layer 5: Storage
  → S3: 99.999999999% (11 nines) durability
  → EBS: replicated within AZ
  → EFS: replicated across AZs

Layer 6: Network
  → Multiple internet gateways
  → Redundant VPN connections
  → Multiple transit gateway attachments
```

### Active-Active vs Active-Passive

```
Active-Active:
┌──────────────────────────────────────────────────┐
│                                                  │
│  Region A (serving traffic)     Region B (serving traffic)  │
│  ┌──────────┐                   ┌──────────┐    │
│  │ App + DB │  ◄── sync ──►    │ App + DB │    │
│  └──────────┘                   └──────────┘    │
│                                                  │
│  Both regions serve ALL users simultaneously     │
│  + Best performance (nearest region)             │
│  + Zero downtime failover                        │
│  - Complex data synchronization                  │
│  - Conflict resolution needed                    │
└──────────────────────────────────────────────────┘

Active-Passive:
┌──────────────────────────────────────────────────┐
│                                                  │
│  Region A (PRIMARY)             Region B (STANDBY)          │
│  ┌──────────┐                   ┌──────────┐    │
│  │ App + DB │  ── replicate ──► │ App + DB │    │
│  └──────────┘                   └──────────┘    │
│                                                  │
│  Only Region A serves traffic                    │
│  Region B takes over if A fails (DNS switch)     │
│  + Simpler (no conflict resolution)              │
│  - Some downtime during failover (30s-5min)      │
│  - Region B cost without serving traffic         │
└──────────────────────────────────────────────────┘
```

### Kubernetes HA

```yaml
# Deployment with HA settings
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3        # Minimum 3 for HA
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # At most 1 pod down during update
      maxSurge: 1
  template:
    spec:
      # Spread pods across zones
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
      # Don't put all pods on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web-app
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:v1.2.3
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          periodSeconds: 15
          failureThreshold: 3
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# Pod Disruption Budget — ensure minimum available during maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2    # At least 2 pods must always be running
  selector:
    matchLabels:
      app: web-app
```

---

## 4. Distributed Systems Fundamentals

### Beginner Explanation

A distributed system is a group of computers working together that appears as a single system to users. When you use Google Search, your query hits not one computer but thousands working in coordination.

**Why distributed?** Because:
- No single machine can handle the load
- We need fault tolerance (if one dies, others continue)
- We need geographic distribution (serve users worldwide)

**The problem:** Distributed systems are HARD. Networks are unreliable, clocks drift, machines fail randomly, and messages arrive out of order.

### The Eight Fallacies of Distributed Computing

```
Developers assume (incorrectly) that:

1. The network is reliable        → Packets get dropped, connections fail
2. Latency is zero               → Network calls take 1-500ms, not 0ms
3. Bandwidth is infinite          → Data transfers have limits
4. The network is secure          → Traffic can be intercepted
5. Topology doesn't change        → Routes and IPs change dynamically
6. There is one administrator     → Multiple teams manage different parts
7. Transport cost is zero         → Cross-region data transfer costs $$
8. The network is homogeneous     → Different hardware, OS, versions
```

### Consistency Models

```mermaid
flowchart TD
    A[Consistency Models] --> B[Strong Consistency]
    A --> C[Eventual Consistency]
    A --> D[Causal Consistency]
    
    B --> B1["All reads see latest write\nExample: Bank transfers\nTrade-off: Higher latency"]
    C --> C1["Reads may see stale data\nbut will converge eventually\nExample: Social media likes\nTrade-off: Possible stale reads"]
    D --> D1["Related operations are ordered\nUnrelated may be out of order\nExample: Message threads\nTrade-off: Middle ground"]
```

**Strong consistency example (banking):**
```
Account balance: $1000

Transaction A: Withdraw $500     Transaction B: Withdraw $700
─────────────────────────────    ─────────────────────────────
Read balance: $1000              Read balance: $1000
Check: 500 ≤ 1000 ✓             Check: 700 ≤ 1000 ✓
Deduct: $1000 - $500 = $500     Deduct: $1000 - $700 = $300  ← PROBLEM!

Without strong consistency: both succeed → overdraft of $200
With strong consistency: one blocks until the other completes → second fails
```

**Eventual consistency example (social media likes):**
```
Post gets liked.
Server A: shows 1001 likes    (updated immediately)
Server B: shows 1000 likes    (hasn't received update yet)
Server C: shows 1001 likes    (received update)

5 seconds later: ALL servers show 1001 likes (converged)
This is fine — nobody cares if like count is briefly inconsistent.
```

### Common Distributed System Patterns

#### Leader Election
```
Multiple nodes → one is elected LEADER → leader coordinates writes

If leader dies → election happens → new leader takes over

Used by: ZooKeeper, etcd, Kafka (partition leader), PostgreSQL (primary)

Implementation: Raft consensus algorithm (etcd/K8s uses this)
```

#### Circuit Breaker Pattern
```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure threshold exceeded
    Open --> HalfOpen: Timeout expires
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
    
    note right of Closed: Normal operation\nRequests pass through
    note right of Open: Requests fail immediately\nDon't overwhelm failing service
    note right of HalfOpen: Test with limited requests\nIf success → close circuit
```

```python
# Circuit breaker implementation concept
class CircuitBreaker:
    CLOSED = "closed"      # Normal — requests flow
    OPEN = "open"          # Failing — reject requests immediately
    HALF_OPEN = "half_open"  # Testing — allow limited requests

    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.state = self.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None

    def call(self, func):
        if self.state == self.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = self.HALF_OPEN
            else:
                raise CircuitOpenError("Service unavailable")

        try:
            result = func()
            if self.state == self.HALF_OPEN:
                self.state = self.CLOSED
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = self.OPEN
            raise
```

#### Bulkhead Pattern

```
Without bulkhead:
┌─────────────────────────────────────────┐
│  Shared thread pool (100 threads)       │
│  Service A calls ─── 70 threads (slow)  │ ← Service A slowdown
│  Service B calls ─── 20 threads         │    blocks ALL services
│  Service C calls ─── 10 threads         │
│  ← No threads left! Everything blocked  │
└─────────────────────────────────────────┘

With bulkhead (isolated pools):
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Service A pool   │ │ Service B pool   │ │ Service C pool   │
│ (40 threads max) │ │ (30 threads max) │ │ (30 threads max) │
│ ← If full, A     │ │ ← Still works!   │ │ ← Still works!   │
│   fails alone    │ │                  │ │                  │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

#### Retry with Exponential Backoff

```bash
# Don't retry immediately — you'll overwhelm the failing service!

Attempt 1: fail → wait 1 second
Attempt 2: fail → wait 2 seconds
Attempt 3: fail → wait 4 seconds
Attempt 4: fail → wait 8 seconds
Attempt 5: fail → wait 16 seconds + give up

Add JITTER (random offset) to prevent thundering herd:
Attempt 1: wait 1s + random(0, 500ms)
Attempt 2: wait 2s + random(0, 1000ms)
...
```

```python
import time
import random

def retry_with_backoff(func, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.5)
            time.sleep(delay + jitter)
```

---

## 5. CAP Theorem

### Beginner Explanation

The CAP theorem states that a distributed system can only guarantee **two out of three** properties simultaneously:

- **C**onsistency — Every read returns the most recent write
- **A**vailability — Every request receives a response (even if stale)
- **P**artition Tolerance — System continues working when network splits occur

Since network partitions ALWAYS happen in real systems, you're really choosing between **CP** (consistent but may reject requests) or **AP** (available but may return stale data).

```mermaid
flowchart TD
    A[CAP Theorem] --> B[During Normal Operation:\nYou can have all three C + A + P]
    A --> C[During Network Partition:\nYou must choose]
    C --> D[CP: Consistent + Partition Tolerant\nReject requests if can't guarantee consistency\nExample: Banking, Inventory]
    C --> E[AP: Available + Partition Tolerant\nServe stale data rather than reject\nExample: Social media, DNS, CDN]
```

### Visual Explanation

```
Normal operation (no partition):
┌─────────────┐     ┌─────────────┐
│  Node A     │◄───►│  Node B     │
│  Data: X=5  │     │  Data: X=5  │
└─────────────┘     └─────────────┘
Both consistent, both available ✓ (C + A + P all work)

Network partition occurs (nodes can't communicate):
┌─────────────┐  ╳  ┌─────────────┐
│  Node A     │     │  Node B     │
│  Data: X=5  │     │  Data: X=5  │
└─────────────┘     └─────────────┘

Client writes X=10 to Node A:

Choice 1 — CP (Consistency):
┌─────────────┐  ╳  ┌─────────────┐
│  Node A     │     │  Node B     │
│  Data: X=10 │     │  REJECT     │ ← Returns error (unavailable)
└─────────────┘     │  requests   │   until partition heals
                    └─────────────┘
"I'd rather be unavailable than give you wrong data"

Choice 2 — AP (Availability):
┌─────────────┐  ╳  ┌─────────────┐
│  Node A     │     │  Node B     │
│  Data: X=10 │     │  Data: X=5  │ ← Returns stale data (inconsistent)
└─────────────┘     └─────────────┘   but still responds
"I'd rather give you possibly-stale data than no data at all"
```

### Real-World CAP Choices

| System | Choice | Why |
|---|---|---|
| PostgreSQL (single leader) | CP | Financial data must be consistent |
| MongoDB (with majority writes) | CP | Configurable, defaults to consistency |
| Cassandra | AP | Designed for availability, eventual consistency |
| DynamoDB | AP (default) / CP (optional) | Configurable via consistent reads |
| Redis Cluster | AP | Cache — stale data is acceptable |
| etcd (Kubernetes) | CP | Cluster state must be consistent |
| DNS | AP | Better to serve slightly stale record than nothing |
| ZooKeeper | CP | Coordination service requires consistency |

### PACELC Theorem (Extension of CAP)

```
If there's a Partition:
  choose Availability or Consistency (same as CAP)
Else (no partition, normal operation):
  choose Latency or Consistency

Full options:
PA/EL — High availability + low latency (sacrifice consistency)
        Example: Cassandra, DynamoDB (default)
        
PC/EC — Always consistent (sacrifice availability during partition, sacrifice latency normally)
        Example: PostgreSQL, traditional RDBMS

PA/EC — Available during partition, consistent otherwise
        Example: MongoDB (with certain settings)
```

---

## 6. Database Scaling

### Read Replicas

```mermaid
flowchart TB
    A[Application] --> B{Query Type?}
    B -->|Write| C[Primary DB\nAccepts writes]
    B -->|Read| D[Read Replica 1]
    B -->|Read| E[Read Replica 2]
    B -->|Read| F[Read Replica 3]
    C -->|Async Replication| D
    C -->|Async Replication| E
    C -->|Async Replication| F
```

```bash
# PostgreSQL streaming replication setup

# On PRIMARY (postgresql.conf):
wal_level = replica
max_wal_senders = 10
synchronous_commit = on     # For sync replica (CP choice)

# On REPLICA:
primary_conninfo = 'host=primary-db port=5432 user=replicator'
primary_slot_name = 'replica_1'

# Check replication lag:
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;
```

**Application-level read routing:**
```python
# Route reads to replicas, writes to primary
class DatabaseRouter:
    def __init__(self):
        self.primary = connect("primary-db:5432")
        self.replicas = [
            connect("replica-1:5432"),
            connect("replica-2:5432"),
        ]
        self.current_replica = 0

    def write(self, query, params):
        return self.primary.execute(query, params)

    def read(self, query, params):
        replica = self.replicas[self.current_replica]
        self.current_replica = (self.current_replica + 1) % len(self.replicas)
        return replica.execute(query, params)
```

### Database Sharding (Horizontal Partitioning)

```
Single database (limit ~10TB, can't scale further):
┌────────────────────────────────┐
│  users table: 500M rows        │ ← Too big for one server
└────────────────────────────────┘

Sharded across 4 databases (each handles a portion):
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Shard 0      │ │ Shard 1      │ │ Shard 2      │ │ Shard 3      │
│ user_id % 4  │ │ user_id % 4  │ │ user_id % 4  │ │ user_id % 4  │
│ = 0          │ │ = 1          │ │ = 2          │ │ = 3          │
│ 125M rows    │ │ 125M rows    │ │ 125M rows    │ │ 125M rows    │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

**Sharding strategies:**

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| Hash-based | `shard = hash(user_id) % num_shards` | Even distribution | Hard to add shards (resharding) |
| Range-based | Users 1-100M → shard 1, 100M-200M → shard 2 | Simple, range queries work | Hotspots (recent data accessed more) |
| Geographic | US users → US shard, EU → EU shard | Low latency, data locality | Uneven distribution |
| Tenant-based | Each customer on their own shard | Strong isolation | Uneven sizes |

**Challenges of sharding:**
```
1. Cross-shard queries: JOIN across shards is very expensive
   → Denormalize data or use application-level joins

2. Rebalancing: Adding/removing shards requires data migration
   → Use consistent hashing to minimize data movement

3. Distributed transactions: ACID across shards is hard
   → Use saga pattern or eventual consistency

4. Auto-increment IDs don't work: each shard has its own sequence
   → Use UUIDs, Snowflake IDs, or centralized ID generator
```

### Connection Pooling

```
Without pooling:
Each request opens new TCP connection → TCP handshake + TLS + auth = 50-100ms overhead
100 concurrent requests = 100 DB connections (database has limited connections!)

With pooling (PgBouncer):
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ App Instance │──conn──►│  PgBouncer   │──5 conn─►│  PostgreSQL  │
│ (50 threads) │         │  (pool)      │          │  (max_conn=  │
│              │         │              │          │   100)       │
├──────────────┤         │              │          │              │
│ App Instance │──conn──►│              │          │              │
│ (50 threads) │         └──────────────┘          └──────────────┘
└──────────────┘
100 app connections → 5 actual DB connections (multiplexed!)
```

```ini
# pgbouncer.ini
[databases]
mydb = host=db-primary port=5432 dbname=production

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5

pool_mode = transaction     # Share connections between transactions
max_client_conn = 1000      # Accept up to 1000 app connections
default_pool_size = 20      # But only keep 20 real DB connections
min_pool_size = 5           # Always keep 5 ready
reserve_pool_size = 5       # Extra pool for burst traffic
reserve_pool_timeout = 3    # Wait 3s before using reserve

# Monitoring
stats_period = 60
```

---

## 7. Caching Strategies

### Why Caching

```
Without cache:
User → App → Database (10-50ms per query)
1000 users reading same product page = 1000 identical DB queries

With cache:
User → App → Cache (0.1-1ms) → hit? return. miss? → Database → store in cache
1000 users reading same product page = 1 DB query + 999 cache hits
```

### Caching Layers

```mermaid
flowchart LR
    A[User] --> B[Browser Cache\nHTTP Cache-Control]
    B --> C[CDN Cache\nCloudflare / CloudFront]
    C --> D[Reverse Proxy Cache\nNginx / Varnish]
    D --> E[Application Cache\nRedis / Memcached]
    E --> F[Database Query Cache\nPg prepared statements]
    F --> G[(Database\nDisk)]
    
    style B fill:#90EE90
    style C fill:#90EE90
    style D fill:#90EE90
    style E fill:#90EE90
```

### Cache Patterns

#### Cache-Aside (Lazy Loading)

```python
def get_user(user_id):
    # 1. Check cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # 2. Cache miss — query database
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 3. Store in cache for future requests (with TTL)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user
```

**Pros:** Only caches what's actually requested. Cache failure doesn't break the system.
**Cons:** First request is always slow (cache miss). Data can become stale.

#### Write-Through Cache

```python
def update_user(user_id, data):
    # 1. Write to database
    db.execute("UPDATE users SET ... WHERE id = %s", user_id, data)
    
    # 2. Immediately update cache
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
```

**Pros:** Cache always has latest data (consistency).
**Cons:** Write latency increases (must write to both). May cache data never read.

#### Write-Behind (Write-Back) Cache

```python
def update_user(user_id, data):
    # 1. Write to cache only (fast!)
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    
    # 2. Async: queue write to database (happens later)
    queue.publish("db_write", {"user_id": user_id, "data": data})
```

**Pros:** Very fast writes. Batches DB writes.
**Cons:** Risk of data loss if cache crashes before DB write. Complex.

### Redis Configuration for Caching

```bash
# redis.conf for caching
maxmemory 4gb
maxmemory-policy allkeys-lru    # Evict least recently used when full

# Common eviction policies:
# noeviction     — return error when memory full (default)
# allkeys-lru    — remove least recently used key (best for cache)
# allkeys-lfu    — remove least frequently used
# volatile-lru   — remove LRU but only among keys with TTL set
# volatile-ttl   — remove keys with shortest TTL first
```

```yaml
# Redis Cluster for HA caching (Kubernetes)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6    # 3 masters + 3 replicas
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server"]
        args:
        - "--cluster-enabled"
        - "yes"
        - "--cluster-config-file"
        - "/data/nodes.conf"
        - "--maxmemory"
        - "2gb"
        - "--maxmemory-policy"
        - "allkeys-lru"
        ports:
        - containerPort: 6379
        - containerPort: 16379
        resources:
          requests:
            memory: 2Gi
            cpu: 500m
```

### Cache Invalidation (The Hard Problem)

```
"There are only two hard things in Computer Science:
 cache invalidation and naming things." — Phil Karlton
```

**Strategies:**

```
1. TTL-based (Time To Live):
   redis.setex("user:123", 300, data)  # Expires after 5 minutes
   Simple but allows stale data for up to TTL duration.

2. Event-driven invalidation:
   When data changes → publish event → delete cache entry
   More consistent but complex to implement.

3. Versioned keys:
   Key: "user:123:v5" → when data changes, increment version
   Old key expires naturally. No explicit deletion needed.

4. Write-through:
   Always update cache when updating DB.
   Most consistent but couples cache to write path.
```

**Cache stampede prevention:**
```python
def get_with_lock(key):
    value = redis.get(key)
    if value:
        return value
    
    # Only ONE request fetches from DB (others wait)
    lock = redis.set(f"lock:{key}", "1", nx=True, ex=10)
    if lock:
        # I got the lock — fetch from DB
        value = expensive_db_query()
        redis.setex(key, 300, value)
        redis.delete(f"lock:{key}")
        return value
    else:
        # Someone else is fetching — wait and retry
        time.sleep(0.1)
        return get_with_lock(key)
```

---

## 8. Message Queues & Async Processing

### Beginner Explanation

Instead of doing everything immediately (synchronously), you put tasks in a queue and process them later (asynchronously).

**Analogy:** Instead of waiting at the restaurant counter for your food (synchronous), you get a ticket number and sit down. They call your number when it's ready (asynchronous).

### Why Queues?

```
Without queue (synchronous):
User → API → Send email (2s) → Generate PDF (5s) → Upload to S3 (3s) → Response
Total: 10 seconds ← User waits for everything

With queue (asynchronous):
User → API → Queue(email, PDF, upload) → Response (50ms)
             └──► Worker: send email (whenever)
             └──► Worker: generate PDF (whenever)
             └──► Worker: upload to S3 (whenever)
Total user wait: 50ms ← Return immediately, background processing
```

```mermaid
flowchart LR
    A[Producer\nAPI Server] -->|Publish message| B[Message Queue\nRabbitMQ / Kafka / SQS]
    B -->|Consume| C[Worker 1]
    B -->|Consume| D[Worker 2]
    B -->|Consume| E[Worker 3]
    C --> F[(Process:\nSend emails)]
    D --> G[(Process:\nGenerate reports)]
    E --> H[(Process:\nVideo encoding)]
```

### Queue Use Cases in DevOps

| Use Case | Queue Technology | Why |
|---|---|---|
| Background jobs (email, notifications) | RabbitMQ, SQS | Simple producer-consumer |
| Event streaming (logs, metrics, clicks) | Kafka | High throughput, replay |
| Task distribution (video encoding) | RabbitMQ, Celery | Work distribution |
| Microservice communication | Kafka, NATS | Decoupling services |
| CI/CD job scheduling | SQS, Redis Streams | Job queue for runners |

### RabbitMQ Example

```yaml
# Docker Compose for RabbitMQ
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
    - "5672:5672"     # AMQP protocol
    - "15672:15672"   # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret
    volumes:
    - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

```python
# Producer (API server)
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()
channel.queue_declare(queue='email_queue', durable=True)

channel.basic_publish(
    exchange='',
    routing_key='email_queue',
    body=json.dumps({"to": "user@example.com", "subject": "Welcome!"}),
    properties=pika.BasicProperties(delivery_mode=2)  # Persistent message
)

# Consumer (Worker)
def process_email(ch, method, properties, body):
    email_data = json.loads(body)
    send_email(email_data["to"], email_data["subject"])
    ch.basic_ack(delivery_tag=method.delivery_tag)  # Acknowledge processed

channel.basic_qos(prefetch_count=1)  # Process one at a time
channel.basic_consume(queue='email_queue', on_message_callback=process_email)
channel.start_consuming()
```

### Apache Kafka (Event Streaming)

```
Kafka is different from traditional queues:
- Messages are PERSISTED (replay old messages)
- Multiple consumers can read independently (consumer groups)
- Extremely high throughput (millions of messages/sec)
- Ordered within a partition

Topic: "user-events" (like a category)
├── Partition 0: [msg1, msg4, msg7, msg10, ...]
├── Partition 1: [msg2, msg5, msg8, msg11, ...]
└── Partition 2: [msg3, msg6, msg9, msg12, ...]

Consumer Group A: reads all partitions (e.g., analytics)
Consumer Group B: reads all partitions independently (e.g., notifications)
```

```yaml
# Kafka on Kubernetes (Strimzi operator)
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-kafka
spec:
  kafka:
    replicas: 3
    listeners:
    - name: plain
      port: 9092
      type: internal
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      default.replication.factor: 3
      min.insync.replicas: 2
    storage:
      type: persistent-claim
      size: 100Gi
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 20Gi
```

---

## 9. Auto-Scaling

### Types of Auto-Scaling

```mermaid
flowchart TD
    A[Auto-Scaling Types] --> B[Horizontal Pod Autoscaler\nHPA]
    A --> C[Vertical Pod Autoscaler\nVPA]
    A --> D[Cluster Autoscaler\nCA]
    A --> E[KEDA\nEvent-Driven]
    
    B --> B1[Add/remove pod replicas\nbased on CPU/memory/custom metrics]
    C --> C1[Resize pod resources\nup/down based on actual usage]
    D --> D1[Add/remove nodes\nwhen pods can't be scheduled]
    E --> E1[Scale based on queue length,\nHTTP requests, cron, etc.]
```

### Kubernetes HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up more
      policies:
      - type: Pods
        value: 4                        # Add max 4 pods at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 25                       # Remove max 25% of pods at a time
        periodSeconds: 60
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70          # Scale up when avg CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"            # Scale up when > 1000 req/s per pod
```

**Check HPA status:**
```bash
kubectl get hpa
kubectl describe hpa web-app-hpa

# Output shows:
# Metrics:        cpu at 75% / 70% target (ABOVE target, will scale up)
# Min replicas:   3
# Max replicas:   50
# Current:        8 pods
# Desired:        12 pods (scaling up)
```

### KEDA — Event-Driven Autoscaling

```yaml
# Scale based on RabbitMQ queue length
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: email-worker-scaler
spec:
  scaleTargetRef:
    name: email-worker
  minReplicaCount: 1
  maxReplicaCount: 30
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
  - type: rabbitmq
    metadata:
      host: amqp://admin:secret@rabbitmq:5672/
      queueName: email_queue
      queueLength: "5"   # Each pod handles 5 messages at a time

---
# Scale based on AWS SQS queue
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-processor
spec:
  scaleTargetRef:
    name: sqs-worker
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456/my-queue
      queueLength: "10"
      awsRegion: us-east-1
    authenticationRef:
      name: aws-credentials

---
# Scale to zero during off-hours (cron)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: batch-processor
spec:
  scaleTargetRef:
    name: batch-worker
  minReplicaCount: 0     # Scale to zero!
  triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: "0 8 * * 1-5"    # 8 AM weekdays
      end: "0 18 * * 1-5"     # 6 PM weekdays
      desiredReplicas: "5"
```

### AWS Auto Scaling Group

```hcl
# Terraform ASG configuration
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  min_size            = 3
  max_size            = 20
  desired_capacity    = 3
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 75
    }
  }

  tag {
    key                 = "Name"
    value               = "app-server"
    propagate_at_launch = true
  }
}

# Scale up policy
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "scale-up"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# Predictive scaling (uses ML to predict demand)
resource "aws_autoscaling_policy" "predictive" {
  name                   = "predictive"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "PredictiveScaling"
  
  predictive_scaling_configuration {
    mode                          = "ForecastAndScale"
    scheduling_buffer_time        = 300   # Scale 5 min before predicted spike
    max_capacity_breach_behavior  = "HonorMaxCapacity"
    
    metric_specification {
      target_value = 70
      predefined_scaling_metric_specification {
        predefined_metric_type = "ASGAverageCPUUtilization"
      }
      predefined_load_metric_specification {
        predefined_metric_type = "ASGTotalCPUUtilization"
      }
    }
  }
}
```

---

## 10. Reliability Engineering (SRE)

### Beginner Explanation

Site Reliability Engineering (SRE) is Google's approach to running reliable production systems. An SRE team ensures that services stay up, perform well, and recover quickly when things go wrong.

Key concept: **100% availability is impossible and not even desirable** (it would mean zero deployments and zero changes). Instead, define an acceptable error budget.

### SLI, SLO, SLA

```
SLI (Service Level Indicator):
  A measurable metric of service quality.
  Example: "99.2% of requests complete in < 200ms"
  Example: "99.95% of requests return non-5xx responses"

SLO (Service Level Objective):
  Internal target for an SLI.
  Example: "99.9% of requests must succeed" (internal goal)
  If violated → team focuses on reliability over features.

SLA (Service Level Agreement):
  External contract with customers (with penalties).
  Example: "99.95% uptime guaranteed, or customer gets credit"
  Always less aggressive than SLO (buffer for safety).
```

```
Relationship:
SLI measures → SLO targets → SLA promises

Example:
  SLI: "Current availability is 99.97%"
  SLO: "Target is 99.95%"        ← We're meeting it
  SLA: "We guarantee 99.9%"      ← We have buffer
```

### Error Budget

```
If SLO = 99.9% availability per month:
  Error budget = 0.1% of month = 43.8 minutes of downtime allowed

Month so far: used 10 minutes of downtime
Remaining budget: 33.8 minutes

If budget remaining → deploy features, take risks
If budget exhausted → freeze changes, focus on reliability

This balances innovation speed with reliability.
```

```mermaid
flowchart LR
    A[Error Budget\nRemaining?] -->|Yes, plenty| B[Ship features faster\nAccept more risk\nRun experiments]
    A -->|Almost exhausted| C[Slow down deploys\nFocus on reliability\nAdd more testing]
    A -->|Exhausted| D[Feature freeze\nReliability sprint\nHarden systems]
```

### Observability — The Three Pillars

```
1. METRICS (numbers over time)
   - CPU usage: 75%
   - Request rate: 10,000 req/s
   - Error rate: 0.05%
   - Latency P99: 250ms
   Tool: Prometheus + Grafana

2. LOGS (event records)
   - "2024-01-15 10:30:01 ERROR: Database connection timeout after 30s"
   - "2024-01-15 10:30:05 WARN: Retry attempt 3 for user 12345"
   Tool: ELK Stack, Loki

3. TRACES (request journey across services)
   - User request → API gateway (5ms) → Auth service (20ms) → DB (150ms) → Response
   - Shows WHERE the bottleneck is
   Tool: Jaeger, Zipkin, OpenTelemetry
```

### Incident Management

```
Severity Levels:
┌──────────┬─────────────────────────────────────────────────────────┐
│ SEV 1    │ Complete outage. All users affected. Revenue impact.     │
│ (P1)     │ Response: Immediate. All-hands incident call.            │
│          │ Target MTTR: < 1 hour                                    │
├──────────┼─────────────────────────────────────────────────────────┤
│ SEV 2    │ Major degradation. Many users affected.                  │
│ (P2)     │ Response: Within 15 minutes. Primary oncall.             │
│          │ Target MTTR: < 4 hours                                   │
├──────────┼─────────────────────────────────────────────────────────┤
│ SEV 3    │ Minor issue. Few users affected.                         │
│ (P3)     │ Response: Next business day.                             │
│          │ Target MTTR: < 24 hours                                  │
├──────────┼─────────────────────────────────────────────────────────┤
│ SEV 4    │ Cosmetic/minor. No user impact.                          │
│ (P4)     │ Response: Sprint backlog.                                │
└──────────┴─────────────────────────────────────────────────────────┘
```

**Incident response process:**
```
1. DETECT  → Monitoring alert fires (PagerDuty)
2. RESPOND → Oncall acknowledges within 5 minutes
3. TRIAGE  → Assess severity, impact, affected users
4. COMMUNICATE → Update status page, notify stakeholders
5. MITIGATE → Stop the bleeding (rollback, failover, scale up)
6. RESOLVE → Fix root cause
7. REVIEW  → Blameless postmortem within 48 hours
```

### Chaos Engineering

```
"Everything fails, all the time." — Werner Vogels (Amazon CTO)

Chaos engineering: INTENTIONALLY inject failures to test resilience.

Principle: It's better to break things in a controlled experiment
           than to be surprised at 3 AM.
```

**Tools:**
```bash
# Chaos Mesh (Kubernetes)
# Kill random pods
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
spec:
  action: pod-kill
  mode: one                    # Kill one random pod
  selector:
    namespaces: [production]
    labelSelectors:
      app: web-app
  scheduler:
    cron: "@every 2h"          # Every 2 hours

---
# Inject network delay
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: all
  selector:
    labelSelectors:
      app: api
  delay:
    latency: "500ms"
    jitter: "100ms"
  duration: "5m"
```

```bash
# Litmus Chaos (popular Kubernetes chaos tool)
# AWS: Chaos Monkey (Netflix) — randomly terminates instances
# Gremlin — commercial chaos engineering platform
```

---

## 11. Disaster Recovery

### RPO and RTO

```
RPO (Recovery Point Objective):
  How much DATA can you afford to lose?
  "We can lose at most 1 hour of data"
  → Backup every hour (at minimum)

RTO (Recovery Time Objective):
  How quickly must you RECOVER?
  "We must be back online within 4 hours"
  → Recovery process must complete in < 4 hours

Example scenarios:
┌──────────────────────┬───────┬───────┐
│ System               │ RPO   │ RTO   │
├──────────────────────┼───────┼───────┤
│ E-commerce platform  │ 5 min │ 1 hr  │
│ Internal wiki        │ 24 hr │ 8 hr  │
│ Financial trading    │ 0     │ 5 min │
│ Dev/staging          │ 24 hr │ 24 hr │
└──────────────────────┴───────┴───────┘
```

### DR Strategies (by cost and RTO)

```mermaid
flowchart TD
    A[Disaster Recovery Strategies] --> B[Backup & Restore\nRTO: hours\nCost: $]
    A --> C[Pilot Light\nRTO: 10-30 min\nCost: $$]
    A --> D[Warm Standby\nRTO: minutes\nCost: $$$]
    A --> E[Active-Active\nRTO: ~0\nCost: $$$$]
```

```
1. Backup & Restore (cheapest, slowest):
   - Regular backups stored in another region
   - On disaster: provision new infrastructure, restore from backup
   - RTO: 4-24 hours
   - RPO: depends on backup frequency

2. Pilot Light (minimal always-on):
   - Core infrastructure always running (DB replica, network)
   - On disaster: scale up compute, switch DNS
   - RTO: 10-30 minutes
   - RPO: minutes (continuous replication)

3. Warm Standby (scaled-down copy):
   - Full copy of production at reduced scale
   - On disaster: scale to full size, switch DNS
   - RTO: minutes
   - RPO: seconds (sync replication)

4. Active-Active (both regions serving):
   - Full production in multiple regions simultaneously
   - On disaster: nothing to do, other region absorbs traffic
   - RTO: ~0 (automatic)
   - RPO: ~0 (multi-master)
```

### Backup Strategies

```bash
# PostgreSQL automated backup
# Full backup daily + continuous WAL archiving

# Full backup (pg_dump)
pg_dump -Fc -h db-primary -d production > /backups/prod_$(date +%Y%m%d).dump

# Continuous archiving (Point-in-time recovery)
# postgresql.conf
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-wal-archive/%f'

# Restore to any point in time:
pg_restore --target-time="2024-01-15 10:30:00" -d production /backups/prod_20240115.dump

# Verify backup (CRITICAL — untested backups are not backups!)
# Restore to test environment weekly:
pg_restore -d test_restore /backups/prod_$(date +%Y%m%d).dump
```

```yaml
# Velero — Kubernetes cluster backup
# Backs up: resources (YAML), persistent volumes, secrets

# Install Velero
# velero install --provider aws --bucket my-velero-backup --secret-file ./credentials

# Backup entire namespace
velero backup create prod-backup --include-namespaces production

# Schedule daily backups (retain 30 days)
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 720h

# Restore
velero restore create --from-backup prod-backup

# Verify backup
velero backup describe prod-backup
velero backup logs prod-backup
```

### Multi-Region Architecture

```hcl
# Terraform: Multi-region setup
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

# RDS with cross-region read replica
resource "aws_db_instance" "primary" {
  provider             = aws.primary
  identifier           = "prod-db"
  engine               = "postgres"
  instance_class       = "db.r6g.xlarge"
  multi_az             = true    # HA within region
  backup_retention_period = 7
}

resource "aws_db_instance" "dr_replica" {
  provider             = aws.dr
  identifier           = "prod-db-dr"
  replicate_source_db  = aws_db_instance.primary.arn
  instance_class       = "db.r6g.large"   # Smaller (pilot light)
}

# S3 cross-region replication
resource "aws_s3_bucket_replication_configuration" "replicate" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  role     = aws_iam_role.replication.arn

  rule {
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

---

## 12. Interview Mastery

---

### Beginner Questions

---

**Q: What is the difference between vertical and horizontal scaling?**

**Perfect Answer:**
> "Vertical scaling means increasing the capacity of a single machine — more CPU, RAM, or faster storage. It's simple because no code changes are needed, but has hard limits (you can't infinitely upgrade hardware) and creates a single point of failure.
>
> Horizontal scaling means adding more machines and distributing work across them using a load balancer. It offers virtually unlimited scalability and fault tolerance, but requires your application to be stateless — meaning any instance can handle any request without relying on local state.
>
> In practice, I use horizontal scaling for stateless application servers (behind ALB with auto-scaling groups), and vertical scaling as a first step for databases (before implementing read replicas or sharding). The modern standard is to design for horizontal from day one — store sessions in Redis, use S3 for files, and keep application servers stateless."

---

**Q: What is high availability and how do you achieve it?**

**Perfect Answer:**
> "High availability means designing a system so it continues operating even when components fail. It's measured in 'nines' — 99.9% means about 43 minutes of downtime per month.
>
> I achieve HA through redundancy at every layer:
> - **Load balancer:** Distributes traffic, health checks remove failed instances
> - **Multiple instances:** Minimum 3 application pods across availability zones
> - **Database:** Primary with synchronous standby (automatic failover)
> - **Multi-AZ:** Infrastructure spread across physically separate data centers
>
> Key practices:
> - No single points of failure anywhere in the path
> - Health checks that detect failures within seconds
> - Automatic failover without human intervention
> - Pod Disruption Budgets to survive node maintenance
> - Topology spread constraints to distribute pods across zones
>
> The target availability determines the architecture cost. 99.9% is achievable with single-region multi-AZ. 99.99% typically requires multi-region active-active, which is significantly more complex and expensive."

---

**Q: Explain CAP theorem in simple terms.**

**Perfect Answer:**
> "CAP theorem states that during a network partition, a distributed system must choose between consistency and availability — it can't guarantee both simultaneously.
>
> - **Consistency:** Every read sees the latest write
> - **Availability:** Every request gets a response
> - **Partition tolerance:** System works despite network failures between nodes
>
> Since network partitions are inevitable in distributed systems, the real choice is between CP (consistent but may refuse requests during partition) and AP (always responds but may serve stale data).
>
> Practical example: A banking system is CP — if the network splits between two database nodes, it's better to return an error than to allow the same money to be spent twice. A social media like counter is AP — showing 999 likes instead of 1000 for a few seconds is fine if it means the page always loads.
>
> Most systems aren't purely CP or AP — they make this choice per operation. A shopping site might be CP for inventory (don't oversell) but AP for product recommendations (stale is fine)."

---

### Intermediate Questions

---

**Q: How would you design an auto-scaling strategy for a web application with unpredictable traffic?**

**Perfect Answer:**
> "I'd use a multi-signal auto-scaling approach to handle both gradual growth and sudden spikes:
>
> **1. Reactive scaling (HPA):**
> - Primary metric: CPU utilization, target 70%
> - Secondary: requests per second per pod
> - Scale-up: aggressive (respond within 60 seconds)
> - Scale-down: conservative (5-minute stabilization window, max 25% reduction at a time) to avoid flapping
>
> **2. Predictive scaling (if on AWS):**
> - Use predictive scaling policies that learn daily/weekly patterns
> - Pre-scales 5 minutes before predicted spike (e.g., morning traffic rush)
>
> **3. Schedule-based (known events):**
> - Before marketing campaigns or known events, pre-scale to expected baseline
> - KEDA CronTrigger to scale up before Black Friday, scale down after
>
> **4. Cluster autoscaler:**
> - Adds nodes when pods are unschedulable
> - Over-provisions by 20% (buffer for burst)
>
> **Safeguards:**
> - Set reasonable min (3) and max (100) replicas
> - Resource requests/limits on all pods so scheduler makes good decisions
> - Pod Disruption Budgets to prevent over-aggressive scale-down
>
> **Monitoring:**
> - Alert when at 80% of max replicas (capacity approaching limit)
> - Track scaling events in Grafana
> - Post-incident review if auto-scaling didn't keep up"

---

**Q: Explain the circuit breaker pattern. When would you use it?**

**Perfect Answer:**
> "The circuit breaker pattern prevents cascading failures when a downstream service is failing. It has three states:
>
> - **Closed** (normal): Requests flow through. Track failures.
> - **Open** (tripped): After N failures, reject requests immediately without calling the failing service. This prevents overwhelming it and lets it recover.
> - **Half-open** (testing): After a timeout, allow ONE request through. If it succeeds, close the circuit. If it fails, stay open.
>
> **When to use it:**
> - Calling external APIs that might be down or slow
> - Between microservices where one dependency failing shouldn't take down everything
> - Database connections during overload
>
> **Why it matters for DevOps:**
> Without a circuit breaker, when Service B goes down, Service A keeps sending requests, exhausting its thread pool waiting for timeouts. Now Service A is also down. Service C depends on A, so it goes down too — cascading failure.
>
> With a circuit breaker, Service A detects B is failing after 5 failed requests, opens the circuit, and immediately returns a fallback response (cached data, degraded experience, or a clear error). Service A stays healthy, Service B gets breathing room to recover.
>
> In Kubernetes, Istio's service mesh implements circuit breaking declaratively — you configure it in YAML rather than in application code."

---

**Q: What is the difference between RPO and RTO? How do they influence architecture decisions?**

**Perfect Answer:**
> "RPO (Recovery Point Objective) is how much data loss is acceptable — the maximum age of data you'd need to restore from. RTO (Recovery Time Objective) is how quickly the system must be back online after a disaster.
>
> They directly drive architecture decisions:
>
> **RPO drives backup strategy:**
> - RPO = 24 hours → daily backups are sufficient
> - RPO = 1 hour → continuous WAL archiving or hourly snapshots
> - RPO = 0 → synchronous cross-region replication (expensive!)
>
> **RTO drives recovery strategy:**
> - RTO = 24 hours → backup & restore (cheapest)
> - RTO = 30 minutes → pilot light (core infra always running)
> - RTO = 5 minutes → warm standby (scaled-down full copy)
> - RTO ≈ 0 → active-active multi-region (most expensive)
>
> The key insight: lower RPO/RTO = exponentially higher cost. A business decision must be made: what's the cost of downtime vs. the cost of prevention? For most B2B SaaS, RPO of 5 minutes and RTO of 30 minutes (warm standby) is the sweet spot. For financial systems, RPO=0/RTO=0 is non-negotiable despite the cost."

---

### Advanced Questions

---

**Q: Design a system to handle 100,000 requests per second for a product catalog API.**

**Perfect Answer:**
> "At 100K req/s, a single server and database cannot handle the load. Here's my architecture:
>
> **Layer 1: CDN + Edge Caching**
> - Product catalog data is relatively static — cache at CDN edge
> - Cache-Control: max-age=60 (refresh every minute)
> - This alone handles 80%+ of requests if hit rate is good
>
> **Layer 2: Application Tier**
> - Stateless API servers behind ALB, auto-scaling group
> - Target: each pod handles 2,000 req/s → need ~10-15 pods minimum (with headroom)
> - Deployed across 3 AZs for HA
>
> **Layer 3: Application-Level Cache (Redis Cluster)**
> - Cache product data in Redis (sub-millisecond reads)
> - 6-node Redis cluster (3 masters, 3 replicas)
> - Cache hit rate target: 95%+
> - Remaining 5% (5,000 req/s) hits database
>
> **Layer 4: Database**
> - Primary for writes + 3 read replicas
> - 5,000 read req/s distributed across replicas ≈ 1,600 req/s per replica (manageable)
> - Connection pooling (PgBouncer) to limit actual connections
>
> **Layer 5: Invalidation**
> - When product data updates → publish event to Kafka
> - Cache invalidation workers consume events → delete Redis keys
> - CDN cache-bust via API (purge specific URLs)
>
> **Capacity math:**
> ```
> 100K req/s total
> - 80K served by CDN (80% hit rate)
> - 20K hit application servers
> - 19K served by Redis cache (95% hit rate at app level)
> - 1K actually hits database
> ```
>
> **Monitoring:** Track cache hit rates, P99 latency, and error rates. Alert if CDN hit rate drops below 70% (something is wrong with caching headers)."

---

**Q: A service is experiencing cascading failures during peak traffic. Walk me through how you'd diagnose and fix it.**

**Perfect Answer:**
> "Cascading failures happen when one component's failure causes dependent components to fail. My diagnostic and remediation approach:
>
> **Immediate diagnosis (first 5 minutes):**
> 1. Check metrics dashboard: which service failed FIRST? (Look at the timeline)
> 2. Check the originating service: is it CPU-bound, memory-bound, or connection-bound?
> 3. Check for the classic cascade pattern: Service A timeout → A's callers pile up → their thread pools exhaust → they fail too
>
> **Common root causes and fixes:**
>
> **Cause 1: One slow service exhausting caller thread pools**
> - Fix now: Reduce timeout on calls to slow service (e.g., 30s → 3s)
> - Fix properly: Implement circuit breaker. When 5 requests fail, stop calling that service entirely for 30 seconds.
>
> **Cause 2: Database connection pool exhaustion**
> - Symptom: 'too many connections' errors cascade to all services using that DB
> - Fix now: Kill idle connections, increase pool size
> - Fix properly: Implement connection pooling (PgBouncer), add rate limiting, shed load
>
> **Cause 3: No backpressure**
> - Symptom: Under load, services accept all requests → internal queues grow → OOM
> - Fix: Implement load shedding — return 503 early when queue depth exceeds threshold. Serve the requests you CAN handle, reject the rest gracefully.
>
> **Cause 4: Retry storms**
> - Symptom: Service A retries 3x → Service B sees 3x normal traffic → B overloads → A retries more → positive feedback loop
> - Fix: Exponential backoff with jitter. Circuit breakers. Retry budgets (max 10% of requests can be retries).
>
> **Systemic prevention:**
> - Bulkhead isolation between services
> - Graceful degradation (serve cached/partial data vs. error)
> - Auto-scaling with appropriate scale-up speed
> - Load testing with chaos injection (regular game days)"

---

**Q: How would you implement a blue-green deployment with zero-downtime database migration?**

**Perfect Answer:**
> "This requires careful coordination because the database is shared — you can't just swap traffic if the new version needs a different schema.
>
> **Strategy: Expand-Contract Migration**
>
> **Phase 1: Expand (backwards-compatible schema change)**
> ```sql
> -- Add new column without breaking existing code
> ALTER TABLE users ADD COLUMN email_verified boolean DEFAULT false;
> -- Both old and new app versions work with this schema
> ```
>
> **Phase 2: Deploy Green (new version)**
> - Green environment uses the expanded schema
> - Run Green alongside Blue — both work because schema is backwards-compatible
> - Route a small percentage of traffic to Green (canary)
> - Monitor for errors
>
> **Phase 3: Switch traffic**
> - If Green is healthy, switch 100% of traffic from Blue to Green
> - Keep Blue running for rollback (10-30 minutes)
>
> **Phase 4: Contract (remove old schema)**
> - Once Blue is decommissioned and we're confident Green works:
> ```sql
> -- Only now remove the old column/change (since nothing uses it)
> ALTER TABLE users DROP COLUMN old_email_format;
> ```
>
> **Key rules for zero-downtime DB migrations:**
> 1. Never rename a column — add new, migrate data, drop old
> 2. Never change a column type directly — add new column, backfill, switch
> 3. Never add NOT NULL constraint without default — add with DEFAULT, then add constraint
> 4. Never drop a column that any live code uses — deploy code change first, then drop
> 5. Always make migrations idempotent (safe to run twice)
>
> **Infrastructure setup:**
> ```
> ALB ──► Blue (v1, port 8080) ← current production
>    └──► Green (v2, port 8081) ← new version (weighted routing)
>    
> Both connect to same database (with expanded schema)
> Switch ALB target group weights: 100/0 → 90/10 → 0/100
> ```"

---

### Scenario-Based Questions

---

**Q: Your application's database is at 90% CPU. Users are reporting slow page loads. What do you do?**

**Perfect Answer:**
> "This is a capacity emergency. My approach, in order of speed:
>
> **Immediate (0-10 minutes):**
> 1. Check slow query log: is there one horrible query causing this?
>    ```sql
>    SELECT * FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC;
>    ```
> 2. Check if there's a runaway query or lock: kill it if safe
>    ```sql
>    SELECT pg_terminate_backend(pid) FROM pg_stat_activity 
>    WHERE duration > interval '5 minutes' AND state = 'active';
>    ```
> 3. Check connections: are we at max_connections?
>
> **Short-term (10-60 minutes):**
> 4. If reads are the problem → route read traffic to read replicas
> 5. If one query is hot → add a missing index or cache the result in Redis
> 6. Scale up the database instance (vertical, requires brief restart for RDS)
>
> **Medium-term (days):**
> 7. Add application-level caching for most-queried data
> 8. Implement connection pooling (PgBouncer) to reduce connection overhead
> 9. Optimize top 5 slow queries (EXPLAIN ANALYZE)
> 10. Consider read replicas if read-heavy workload
>
> **Long-term (weeks):**
> 11. Implement proper cache-aside pattern (Redis) to reduce DB load by 80%+
> 12. Review data model — is N+1 query problem happening?
> 13. Consider sharding if single-node limits are approaching
>
> The key insight: caching solves most DB scaling problems. A properly cached application hits the database for only 5-10% of requests."

---

**Q: How would you ensure your system can survive the loss of an entire AWS availability zone?**

**Perfect Answer:**
> "Multi-AZ resilience requires no single dependency on any one AZ. Here's my checklist:
>
> **Compute:**
> - Auto Scaling Group spans 3 AZs
> - Minimum 2 instances per AZ (so losing one AZ leaves 4 instances in 2 AZs)
> - Kubernetes: topology spread constraints ensure pods are evenly distributed
>
> **Load Balancer:**
> - ALB is inherently multi-AZ — AWS handles this
> - Cross-zone load balancing enabled (distributes evenly regardless of AZ)
>
> **Database:**
> - RDS Multi-AZ: synchronous standby in different AZ, automatic failover in ~60 seconds
> - Or Aurora: storage replicated 6 ways across 3 AZs
>
> **Cache:**
> - ElastiCache Redis with Multi-AZ enabled
> - Automatic failover to replica in different AZ
>
> **Storage:**
> - S3: inherently multi-AZ (11 nines durability)
> - EBS: single-AZ only! Use EFS for shared storage that must survive AZ failure
>
> **Networking:**
> - Subnets in each AZ (public + private)
> - NAT Gateways per AZ (don't share — if AZ dies, NAT goes too)
>
> **Testing:**
> - Quarterly AZ failover drill: artificially remove one AZ from the load balancer and verify the system stays healthy
> - Chaos engineering: randomly terminate instances in one AZ
>
> **Capacity planning:**
> - Size for N-1 AZs. If running in 3 AZs, each AZ must be able to handle 50% of traffic (not 33%), because losing one AZ means the remaining 2 absorb everything."

---

### FAANG-Style Conceptual Questions

---

**Q: Design a global content delivery system that serves 1 billion requests per day with 99.99% availability.**

**Perfect Answer:**
> "1 billion requests/day = ~11,500 req/s average, with peaks likely 3-5x higher (~50K req/s peak).
>
> **Architecture:**
>
> ```
> Users worldwide
>     │
>     ▼
> Anycast DNS (Route53 / Cloudflare)
>     │ → resolves to nearest PoP
>     ▼
> CDN Edge (200+ PoPs globally)
>     │ → 95%+ cache hit rate (static + cacheable dynamic)
>     │ → ~500 req/s per PoP on average
>     ▼ (cache miss only)
> Regional Origin Shield (one per continent)
>     │ → Second cache layer, collapses duplicate requests
>     │ → Reduces origin hits by another 80%
>     ▼ (origin fetch)
> Origin Clusters (3 regions: US, EU, APAC)
>     │ → Active-active, each handles its own region
>     ▼
> Storage (S3 + Database)
> ```
>
> **Achieving 99.99%:**
> - 99.99% = 52 minutes downtime/year max
> - CDN edge is inherently HA (Anycast reroutes around failed PoPs automatically)
> - Origin: multi-region active-active, each region handles 33% + can absorb 50% overflow
> - Database: Aurora Global Database with cross-region replication (RPO < 1 second)
> - All components have automated failover (no human in the loop)
>
> **Caching strategy:**
> - Static assets: CDN cache with TTL 1 year + versioned filenames (cache busting via filename)
> - Dynamic cacheable (e.g., product pages): CDN cache 60s, origin-shield cache 300s
> - Personalized content: CDN per-user (cookie-based key) or bypass CDN, serve from regional origin
>
> **Failure modes and mitigations:**
> - PoP failure → Anycast automatically routes to next nearest PoP (zero-action)
> - Region failure → DNS health check removes region, others absorb (RTO < 30s)
> - Origin down → CDN serves stale content (stale-while-revalidate) for up to 1 hour
> - Database corruption → Aurora point-in-time recovery + cross-region replica
>
> **Cost optimization:**
> - CDN handles 95% of traffic: edge serving is 10x cheaper than origin
> - Origin shield reduces origin load by 80%: saves compute costs
> - Reserved capacity for baseline, spot/on-demand for peaks
>
> **Monitoring:**
> - Real-user monitoring (RUM) for actual user experience
> - Synthetic monitoring from 50+ global locations
> - Cache hit rate dashboards (alert if drops below 90%)
> - Per-PoP latency tracking"

---

**Q: You're the first DevOps hire at a startup that's growing 10x in the next year. How do you prepare the infrastructure?**

**Perfect Answer:**
> "Growing 10x means current architecture will break. I prioritize based on what fails first:
>
> **Month 1-2: Foundation (survive the next 3x)**
>
> 1. **Move to IaC** — Get everything in Terraform. No more clicking in console. Reproducible, reviewable, version-controlled.
>
> 2. **Container orchestration** — If still on bare EC2/VMs, move to EKS/GKE. Horizontal scaling needs orchestration.
>
> 3. **Auto-scaling** — HPA for pods, cluster autoscaler for nodes. The system should handle traffic without manual intervention.
>
> 4. **Observability** — Prometheus + Grafana + Loki. Can't scale what you can't measure. Need to see bottlenecks before users hit them.
>
> **Month 3-4: Reliability (survive the next 5x)**
>
> 5. **Database scaling** — Read replicas for read-heavy workloads. PgBouncer for connection pooling. Index optimization for top slow queries.
>
> 6. **Caching layer** — Redis for hot data. CDN for static assets. Target: reduce DB load by 80%.
>
> 7. **CI/CD hardening** — Fast, reliable pipeline. Automated testing. Rollback capability. Can't have 'deployment broke prod' incidents at 10x scale.
>
> **Month 5-6: Scalability (prepare for 10x)**
>
> 8. **Service decomposition** — Identify the monolith's hottest paths. Extract into services that can scale independently.
>
> 9. **Async processing** — Queue-based architecture for background work. Don't make users wait for emails, reports, etc.
>
> 10. **Multi-region readiness** — Not active-active yet, but DR strategy in place. Cross-region backups. Tested recovery procedure.
>
> **Throughout: Culture**
> - Incident response process (even with one engineer)
> - Runbooks for common problems
> - Load testing: regularly test at 2-3x current peak
> - Chaos engineering: monthly game days
>
> **What I explicitly defer:**
> - Multi-region active-active (complexity not justified until 50x+)
> - Custom sharding (wait until read replicas + cache aren't enough)
> - Microservices for everything (only extract what's a bottleneck)"

---

## Summary

```
Scaling & Production Systems — Key Takeaways:

Scaling:
  Vertical = bigger machine (simple, limited)
  Horizontal = more machines (complex, unlimited)
  Design stateless from day one.

High Availability:
  Redundancy at every layer.
  Multi-AZ minimum. Multi-region for critical systems.
  Availability target drives architecture cost.

Distributed Systems:
  CAP: Choose CP or AP per operation.
  Patterns: Circuit breaker, bulkhead, retry with backoff.
  The network WILL fail — design for it.

Database Scaling:
  Read replicas → Connection pooling → Caching → Sharding
  (each step only when the previous isn't enough)

Caching:
  CDN → Reverse proxy → Application (Redis) → DB cache
  Cache invalidation is the hardest problem.

Auto-Scaling:
  HPA for pods, Cluster Autoscaler for nodes.
  Scale up fast, scale down slow.
  Predictive + reactive for best results.

Reliability (SRE):
  SLI → SLO → SLA → Error Budget
  Chaos engineering: break things intentionally.
  Incident management: detect → respond → mitigate → review.

Disaster Recovery:
  RPO = acceptable data loss. RTO = acceptable downtime.
  Strategy: backup/restore → pilot light → warm standby → active-active.
  Untested backups are not backups.
```

---

[⬇️ Download This File](#)
