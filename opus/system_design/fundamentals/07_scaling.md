# Chapter 7: Load Balancing & Scaling

## Scaling Fundamentals

```
Vertical Scaling (Scale Up)          Horizontal Scaling (Scale Out)
─────────────────────────           ──────────────────────────────
Add more CPU/RAM to one machine     Add more machines

Pros:                               Pros:
- Simple (no code changes)          - No hardware limits
- No distributed complexity         - Fault tolerant
                                    - Linear cost scaling
Cons:                               Cons:
- Hardware limits (max ~128 cores)  - Distributed system complexity
- Single point of failure           - Need load balancing
- Expensive at high end             - Data consistency challenges
```

---

## Load Balancers

### What They Do
```
                         ┌──────────┐
                    ┌───▶│ Server 1 │
┌────────┐    ┌────┴───┐├──────────┤
│ Client │───▶│   LB   │┤ Server 2 │
└────────┘    └────┬───┘├──────────┤
                    └───▶│ Server 3 │
                         └──────────┘

Functions:
- Distribute traffic across servers
- Health checks (remove unhealthy servers)
- SSL termination
- Session affinity (sticky sessions)
```

### Load Balancing Algorithms

```
1. Round Robin
   ────────────
   Requests go to each server in turn: 1 → 2 → 3 → 1 → 2 → 3
   Simple but ignores server capacity and current load.

2. Weighted Round Robin
   ─────────────────────
   Server 1 (weight 3): gets 3 requests
   Server 2 (weight 1): gets 1 request
   Use when servers have different capacities.

3. Least Connections
   ──────────────────
   Send to server with fewest active connections.
   Best for long-lived connections (WebSocket, database).

4. Least Response Time
   ─────────────────────
   Send to server with fastest recent response.
   Best for heterogeneous server performance.

5. IP Hash
   ────────
   hash(client_ip) % num_servers → same client always hits same server.
   Use for: session affinity without cookies.

6. Consistent Hashing
   ───────────────────
   Servers on a ring, requests map to nearest clockwise server.
   Use for: distributed caches (minimal redistribution on server add/remove).
```

### Layer 4 vs Layer 7 Load Balancing

```
Layer 4 (Transport):
- Routes based on IP + port
- Very fast (no packet inspection)
- Can't route based on content
- Examples: AWS NLB, HAProxy (TCP mode)

Layer 7 (Application):
- Routes based on HTTP headers, URL path, cookies
- Can do content-based routing (/api → backend, /images → CDN)
- SSL termination, request modification
- Slower but more flexible
- Examples: AWS ALB, Nginx, HAProxy (HTTP mode)
```

---

## Scaling Patterns

### Stateless Services (Easy to Scale)
```
Rule: Keep servers stateless → any server can handle any request

Don't store:
- Session data (use Redis/DB instead)
- Uploaded files (use S3 instead)
- Cache (use Redis instead)

Then scaling = just add more servers behind LB
```

### Database Scaling Ladder
```
Step 1: Single DB + Query optimization + Indexing
         ↓ still slow
Step 2: Read replicas (separate read/write traffic)
         ↓ still not enough
Step 3: Caching layer (Redis) — absorb 80% of reads
         ↓ still growing
Step 4: Vertical scaling (bigger DB instance)
         ↓ hit limits
Step 5: Sharding (horizontal partitioning)
         ↓ some queries need multiple shards
Step 6: Denormalization + CQRS (specialized read/write models)
```

### Auto-Scaling
```
Metrics to scale on:
- CPU utilization > 70%
- Memory > 80%
- Request queue depth
- Response latency > threshold
- Custom metrics (business-specific)

Policies:
- Target tracking: "keep CPU at 60%"
- Step scaling: "if CPU > 80% for 5 min, add 2 servers"
- Scheduled: "scale up at 9am, down at 11pm"

Cool-down period: Wait X minutes before scaling again (avoid thrashing)
```

---

## Microservices Scaling

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
│  (rate limiting, auth, routing, request aggregation)            │
└────────┬───────────────┬──────────────────┬────────────────────┘
         │               │                  │
    ┌────▼────┐    ┌────▼────┐       ┌────▼────┐
    │ User    │    │ Order   │       │ Payment │
    │ Service │    │ Service │       │ Service │
    │ (5 inst)│    │ (10 inst)│      │ (3 inst)│
    └────┬────┘    └────┬────┘       └────┬────┘
         │               │                  │
    ┌────▼────┐    ┌────▼────┐       ┌────▼────┐
    │ User DB │    │ Order DB│       │Payment DB│
    └─────────┘    └─────────┘       └─────────┘

Each service scales independently based on its own load!
```

### Service Discovery
```
How does Service A find Service B?

1. Client-side discovery:
   Service → Service Registry (Consul/ZooKeeper) → get IP list → call directly

2. Server-side discovery:
   Service → Load Balancer → routes to healthy instance
   (simpler for the service, LB handles discovery)

3. DNS-based:
   Service → DNS lookup → returns IP(s)
   (simple but slow to update, TTL caching issues)
```

---

## Global Scaling

### Multi-Region Architecture
```
                    ┌──── DNS (GeoDNS) ────┐
                    │                       │
            ┌───────▼───────┐      ┌───────▼───────┐
            │  US Region    │      │  India Region  │
            │  ┌─────────┐  │      │  ┌─────────┐  │
            │  │   LB    │  │      │  │   LB    │  │
            │  └────┬────┘  │      │  └────┬────┘  │
            │  ┌────▼────┐  │      │  ┌────▼────┐  │
            │  │ Servers  │  │      │  │ Servers  │  │
            │  └────┬────┘  │      │  └────┬────┘  │
            │  ┌────▼────┐  │      │  ┌────▼────┐  │
            │  │   DB    │◀─┼──────┼─▶│   DB    │  │
            │  └─────────┘  │ sync │  └─────────┘  │
            └───────────────┘      └───────────────┘

Challenges:
- Data replication lag between regions
- Conflict resolution (two regions update same data)
- Compliance (data residency laws)
```

---

## Key Numbers for Interviews

```
Single Nginx:          ~100K concurrent connections
Single app server:     ~1K-10K QPS (depends on complexity)
Horizontal scaling:    Linear (10 servers ≈ 10x throughput)
Auto-scale reaction:   ~2-5 minutes to spin up new instances
Cross-region latency:  100-300ms (depends on distance)
Same-region latency:   1-5ms
```
