# Microservices Patterns — Deep Dive

## Part 1: Microservices vs Monolith

### When Monolith is Better
```
Monolith advantages:
  - Simple deployment (one artifact)
  - Easy debugging (single process, same logs)
  - No distributed systems complexity
  - Fast local function calls (no network latency)
  - ACID transactions across the whole codebase
  
Start with a monolith when:
  - Team < 15 engineers
  - Domain not well-understood yet
  - Startup validating product
  - Traffic < 1M daily active users
```

### When to Move to Microservices
```
Signals it's time:
  1. Long build/deploy times (> 30 min)
  2. Scaling specific parts is wasteful (need to scale whole monolith)
  3. Teams step on each other's code constantly
  4. Different parts need different tech stacks
  5. Team is > 30 engineers → Conway's Law: architecture mirrors team structure
  
Trade-offs you accept:
  + Independent deployability
  + Independent scalability per service
  + Team autonomy
  - Distributed systems complexity
  - Network latency between services
  - Distributed tracing overhead
  - More operational burden
```

---

## Part 2: Service Decomposition Strategies

### Domain-Driven Design (DDD) Boundaries
```
Identify "Bounded Contexts" — areas where a specific domain model applies.

E-commerce bounded contexts:
  ├── Identity & Access (users, auth, permissions)
  ├── Catalog (products, categories, inventory)
  ├── Ordering (carts, orders, order lifecycle)
  ├── Payments (charges, refunds, billing)
  ├── Shipping (addresses, fulfillment, tracking)
  ├── Notifications (email, SMS, push)
  └── Analytics (events, reporting)

Each bounded context → one or few microservices.
Shared nothing: each service has its own database.
```

### Database Per Service
```
Each microservice owns its data exclusively.
No direct DB access from another service.
Only via API calls.

Why:
  + Services can evolve schema independently
  + Choose right DB per service (MySQL for orders, Redis for sessions, Cassandra for logs)
  + Services fail independently
  
Challenge: 
  Queries that used to be JOINs now require:
  - API calls (N+1 problem at scale)
  - Event-driven data denormalisation
  - CQRS read models
```

---

## Part 3: Communication Patterns

### Synchronous (REST/gRPC)
```
Service A → HTTP/gRPC → Service B → response

Use when:
  - Caller needs immediate result (user-facing request)
  - Natural request/response interaction
  
Problem: Temporal coupling — A and B must be up simultaneously.
         Failure cascades: B down → A fails → caller fails.

Solution: Circuit Breaker (see below) + Bulkhead.
```

### Asynchronous (Event-Driven)
```
Service A → Kafka event → Service B (eventually processes)

Use when:
  - Caller doesn't need immediate result ("fire and forget")
  - Fan-out (one event, many consumers)
  - Workload levelling (absorb spikes)
  
Benefit: Services can be down without cascade failure.
         B can process events when it recovers.
```

### Request-Reply over Kafka
```
When you need async but also need a response:

Producer side:
  Publish to "requests" topic with:
    - correlation_id: UUID
    - reply_to: "responses.service_a.{instance_id}"
  Subscribe to "responses.service_a.{instance_id}"
  
Consumer side (Service B):
  Process request
  Publish response to reply_to topic with same correlation_id

Service A matches correlation_id → resolves future/promise

Used by: Microservices needing async RPC without blocking
```

---

## Part 4: Resilience Patterns

### Circuit Breaker (Detailed)
```java
// Pseudocode
class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN }
    
    State state = CLOSED;
    int failureCount = 0;
    int failureThreshold = 5;
    long openTimeout = 30_000; // ms
    long lastFailureTime;
    
    <T> T execute(Supplier<T> operation) {
        if (state == OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > openTimeout) {
                state = HALF_OPEN;
            } else {
                throw new CircuitOpenException("Circuit is open");
            }
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    void onSuccess() {
        failureCount = 0;
        state = CLOSED;
    }
    
    void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        if (failureCount >= failureThreshold) {
            state = OPEN;
        }
    }
}
```

### Bulkhead Pattern
```
Isolate resources per dependency so one slow dependency doesn't drain all threads.

Without bulkhead:
  100 threads in service A
  Service B becomes slow → 100 threads waiting for B → Service A can't handle other requests

With bulkhead:
  10 threads dedicated to calls to Service B
  If all 10 busy → calls to B rejected immediately (fast fail)
  90 threads remain available for other work

Implementation: Separate thread pools or semaphores per downstream service.
Used by: Netflix Hystrix, Resilience4j
```

### Timeout Hierarchy
```
Gateway timeout: 30s
  └── Service A timeout to Service B: 10s
         └── Service B timeout to DB: 3s

Rule: Each layer's timeout < parent's timeout.
If DB timeout is longer than B's timeout to DB → B always times out before DB does → pointless.
```

---

## Part 5: Data Consistency Patterns

### CQRS (Command Query Responsibility Segregation)
```
Separate the write model (Commands) from the read model (Queries).

Write side:
  Commands: PlaceOrder, CancelOrder, UpdateAddress
  Validates, applies business rules, emits events
  Stores in normalised DB (correctness)

Read side:
  Queries: GetOrderHistory, GetOrderDetails
  Denormalised read model optimised for specific queries
  Updated by consuming events from write side
  Can use different DB (Elasticsearch, Redis) per query need

Flow:
  [User] → Command → [Write Model] → Event → [Event Bus]
                                                   ↓
                                          [Read Model Updater]
                                                   ↓
                                          [Read Model DB]
  [User] → Query ─────────────────────> [Read Model]
```

### Event Sourcing
```
Store all state changes as an immutable sequence of events (not current state).

Traditional:
  Row: { id: 1, balance: 150 }  ← just the current state

Event Sourcing:
  Events: [
    { type: "AccountOpened", amount: 100, at: T1 },
    { type: "Deposited",      amount: 100, at: T2 },
    { type: "Withdrawn",      amount: 50,  at: T3 },
  ]
  Current state = fold(events)  → balance = 100 + 100 - 50 = 150

Benefits:
  Full audit trail (regulatory requirement for payments)
  Replay events to rebuild state at any point in time
  Easy to project same events into different read models
  
Challenges:
  Querying current state requires replaying events (slow without snapshots)
  Snapshots: periodically materialise current state, replay from last snapshot
```

---

## Part 6: API Gateway Patterns

### BFF (Backend for Frontend)
```
Problem: Mobile app, web app, and partner API have different needs.
  Mobile: lightweight payloads, battery-conscious
  Web:    rich data, desktop bandwidth
  Partner: structured, stable, versioned API

Solution: Separate API Gateways per client type.

[Mobile App] → [Mobile BFF] → [User Service]
                                [Order Service] (returns mobile-optimised data)

[Web App]    → [Web BFF]    → [User Service]
                                [Order Service]
                                [Recommendation Service]

[Partners]   → [Partner API] → [User Service]
                                [Order Service] (stable, versioned)

Each BFF owned by the frontend team — they control their own data aggregation.
```

---

## Part 7: Service Discovery

### Client-Side vs Server-Side
```
Client-Side Discovery:
  Client → asks Service Registry (Consul) → gets IP list → load-balances itself
  
  Pros: No extra hop
  Cons: Every client must implement discovery logic; multiple languages = multiple implementations

Server-Side Discovery (more common):
  Client → Load Balancer → LB asks Registry → routes to correct service
  
  Pros: Language-agnostic; clients are simple
  Cons: LB is an extra hop and potential SPOF (use active-passive LB)

Kubernetes Service Discovery:
  Every Service gets a DNS name: my-service.my-namespace.svc.cluster.local
  kube-proxy manages iptables rules for routing
  No external registry needed; built-in to Kubernetes
```

---

## Part 8: Distributed Tracing

### The Problem
```
Request fails. Error is in Service E.
But the user's request went through: API GW → A → B → C → D → E.
How do you find E's log? How do you understand the call chain?
```

### OpenTelemetry Solution
```
Each service:
  1. Generates or inherits a trace_id (UUID) at request entry
  2. Creates a span (unit of work) with start time, end time, metadata
  3. Propagates trace_id + span_id in headers to downstream services
  4. Reports spans to collector (Jaeger, Zipkin, Datadog)

Trace = tree of spans:
  [API GW: 100ms]
    └── [Service A: 80ms]
          └── [Service B: 60ms]
                ├── [Redis: 2ms]
                └── [Service D: 55ms]
                      └── [MySQL: 50ms]  ← bottleneck!

Sampling:
  Can't trace every request (overhead).
  Sample 1% (or 100% for errors) → still useful for bottleneck analysis.
```

---

## Key Takeaways

1. **Start with a monolith** — decompose when team/scale demands it.
2. **Bounded contexts** (DDD) define service boundaries; each owns its data.
3. **Database per service**: choose the right DB type per service's access pattern.
4. **Circuit Breaker + Bulkhead**: prevent cascade failures and thread exhaustion.
5. **CQRS**: separate read model lets you optimise reads independently of writes.
6. **Event Sourcing**: full audit trail + time travel; snapshot to avoid full replay.
7. **BFF**: separate API gateways per client type, owned by frontend teams.
8. **Distributed tracing** (OpenTelemetry) is non-negotiable for debugging microservices.
