# Architectural Patterns

## 1. Microservices vs Monolith

```
Monolith:                              Microservices:
┌─────────────────────┐               ┌──────┐ ┌──────┐ ┌──────┐
│                     │               │ Auth │ │Orders│ │ Pay  │
│   All code in one   │     →→→       │      │ │      │ │      │
│   deployable unit   │               └──┬───┘ └──┬───┘ └──┬───┘
│                     │                  │        │        │
└─────────────────────┘               ┌──▼───┐ ┌──▼───┐ ┌──▼───┐
                                      │ DB 1 │ │ DB 2 │ │ DB 3 │
                                      └──────┘ └──────┘ └──────┘

When to use Monolith:
- Early stage startup (move fast)
- Small team (< 10 engineers)
- Simple domain

When to use Microservices:
- Large team (need independent deployment)
- Different scaling needs per component
- Polyglot (different services need different tech)
- High availability requirements
```

---

## 2. Event-Driven Architecture

```
┌─────────┐    event    ┌──────────────┐    event    ┌─────────────┐
│ Service │───────────▶│  Event Bus   │───────────▶│  Service B  │
│    A    │            │  (Kafka)     │            │  (consumer) │
└─────────┘            └──────────────┘            └─────────────┘
                              │
                              │ event
                              ▼
                       ┌─────────────┐
                       │  Service C  │
                       │  (consumer) │
                       └─────────────┘

Benefits:
- Loose coupling (producers don't know consumers)
- Easy to add new consumers
- Natural audit log (events are immutable)
- Temporal decoupling (process at own pace)

Challenges:
- Eventual consistency
- Event ordering
- Debugging is harder (no single call stack)
- Schema evolution
```

---

## 3. CQRS (Command Query Responsibility Segregation)

```
Commands (writes):                    Queries (reads):
┌────────┐                           ┌────────┐
│ Client │                           │ Client │
└───┬────┘                           └───┬────┘
    │ POST/PUT/DELETE                     │ GET
    ▼                                    ▼
┌────────────┐                      ┌────────────┐
│ Command    │                      │ Query      │
│ Service    │                      │ Service    │
└────┬───────┘                      └────┬───────┘
     │                                   │
     ▼                                   ▼
┌──────────┐    sync/event    ┌─────────────────┐
│ Write DB │──────────────────▶│   Read DB       │
│(normalized)│                 │(denormalized,   │
│ PostgreSQL │                 │ optimized views) │
└───────────┘                  └─────────────────┘

When to use:
- Read/write patterns are very different
- Need different models optimized for each
- High read:write ratio (optimize reads independently)
- Complex domain with many query patterns

Example: E-commerce
- Write: Normalized order tables with strict ACID
- Read: Denormalized product catalog with search index
```

---

## 4. Event Sourcing

```
Traditional: Store CURRENT state
  user.balance = 500  (you don't know how you got here)

Event Sourcing: Store ALL events
  [Deposited $1000] → [Withdrew $200] → [Transferred $300] → balance = $500

┌─────────────────────────────────────────────────┐
│ Event Store (append-only log)                   │
│                                                  │
│ Event 1: {type: "Deposited", amount: 1000}     │
│ Event 2: {type: "Withdrew", amount: 200}        │
│ Event 3: {type: "Transferred", amount: 300}     │
│                                                  │
│ Current state = replay all events               │
└─────────────────────────────────────────────────┘

Benefits:
- Complete audit trail
- Can rebuild state at any point in time
- Natural fit for event-driven systems
- Easy to add new read models (replay events)

Challenges:
- Event schema evolution (versioning)
- Rebuilding state is slow (use snapshots)
- Eventually consistent read models
- Complexity in handling concurrent events
```

---

## 5. Strangler Fig Pattern (Migration)

```
Migrating from monolith to microservices incrementally:

Phase 1: All traffic to monolith
  ┌─────┐
  │Proxy│──▶ Monolith [handles everything]
  └─────┘

Phase 2: New feature in microservice, old in monolith  
  ┌─────┐──▶ New Service [/api/v2/orders]
  │Proxy│
  └─────┘──▶ Monolith [everything else]

Phase 3: Gradually move features
  ┌─────┐──▶ Orders Service
  │Proxy│──▶ Users Service
  └─────┘──▶ Monolith [shrinking]

Phase 4: Monolith gone
  ┌─────┐──▶ Orders Service
  │Proxy│──▶ Users Service
  └─────┘──▶ Payment Service
```

---

## 6. Circuit Breaker

```
States:
          success
    ┌────────────────────┐
    │                    │
    ▼                    │
┌────────┐  failures   ┌────────┐  timeout  ┌───────────┐
│ CLOSED │───────────▶│  OPEN  │──────────▶│HALF-OPEN  │
│(normal)│  > threshold│(reject)│           │(test one) │
└────────┘            └────────┘           └───────────┘
    ▲                                           │
    │              success                      │
    └───────────────────────────────────────────┘
              failure → back to OPEN

Use when:
- Downstream service might be down
- Want to fail fast instead of waiting for timeout
- Prevent cascading failures in microservices
```

---

## 7. Bulkhead Pattern

```
Isolate failures to prevent total system collapse:

Without bulkhead:
┌──────────────────────────────────────┐
│        Shared Thread Pool (100)       │
│                                       │
│  Service A calls ──▶ slow/down       │
│  Service B calls ──▶ can't get threads│ ← Affected!
│  Service C calls ──▶ can't get threads│ ← Affected!
└──────────────────────────────────────┘

With bulkhead:
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│Pool A (30)   │ │Pool B (40)   │ │Pool C (30)   │
│              │ │              │ │              │
│Service A: 💀│ │Service B: ✅│ │Service C: ✅│
│(isolated!)   │ │(unaffected) │ │(unaffected) │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## 8. Sidecar Pattern

```
┌─────────────────────────────────────┐
│            Pod / Container          │
│                                      │
│  ┌──────────────┐  ┌─────────────┐ │
│  │  Main App    │  │   Sidecar   │ │
│  │  (business   │  │  (logging,  │ │
│  │   logic)     │◀─▶│   proxy,   │ │
│  │              │  │  monitoring)│ │
│  └──────────────┘  └─────────────┘ │
│                                      │
└─────────────────────────────────────┘

Examples:
- Envoy proxy (service mesh sidecar)
- Log collector (Fluentd)
- Config updater
- Security proxy (mTLS)
```

---

## Pattern Selection Guide

```
Problem                          → Pattern
─────────────────────────────────────────────────────
Independent team deployment      → Microservices
Different read/write models      → CQRS
Need complete audit trail        → Event Sourcing
Decouple services               → Event-Driven
Migrate incrementally           → Strangler Fig
Prevent cascading failures      → Circuit Breaker
Isolate resource pools          → Bulkhead
Cross-cutting concerns          → Sidecar
Flexible API for multiple UIs   → BFF (Backend for Frontend)
```
