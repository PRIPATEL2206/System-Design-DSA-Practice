# Chapter 6: Message Queues & Async Processing

## Why Message Queues?

```
Without queue (synchronous):
User → App → [wait for email to send] → [wait for analytics] → Response (slow!)

With queue (asynchronous):
User → App → Response (fast!)
              └──▶ Queue → Email Worker → sends email
              └──▶ Queue → Analytics Worker → logs event
```

**Benefits:**
- **Decoupling** — Producer doesn't know/care about consumers
- **Buffering** — Handle traffic spikes without dropping requests
- **Async processing** — User doesn't wait for slow operations
- **Retry/reliability** — Failed jobs can be retried
- **Ordering** — Process events in sequence

---

## Message Queue vs Event Stream

```
┌──────────────────────┬────────────────────────┬────────────────────────┐
│                      │ Message Queue           │ Event Stream           │
│                      │ (RabbitMQ, SQS)        │ (Kafka, Kinesis)       │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Message lifecycle    │ Deleted after consume  │ Retained (days/weeks)  │
│ Consumer model       │ Competing consumers   │ Consumer groups         │
│ Replay              │ No                      │ Yes (seek to offset)   │
│ Ordering            │ Per-queue              │ Per-partition           │
│ Throughput          │ ~10K msg/sec           │ ~100K+ msg/sec         │
│ Best for            │ Task queues, RPC       │ Event sourcing, logs   │
└──────────────────────┴────────────────────────┴────────────────────────┘
```

---

## Apache Kafka

The go-to for high-throughput event streaming.

### Architecture
```
                    ┌─────────── Kafka Cluster ───────────┐
                    │                                      │
Producer ──msg──▶   │  Topic: "orders"                    │
                    │  ┌───────────────────────────────┐  │
                    │  │ Partition 0: [m1][m3][m5]     │──────▶ Consumer Group A
                    │  │ Partition 1: [m2][m4][m6]     │──────▶   (consumer 1)
                    │  │ Partition 2: [m7][m8]         │──────▶   (consumer 2)
                    │  └───────────────────────────────┘  │
                    │                                      │
                    │  Each partition replicated 3x        │
                    └──────────────────────────────────────┘

Key concepts:
─────────────
Topic       = Category of messages (like "orders", "user-events")
Partition   = Ordered, immutable sequence within a topic
Offset      = Position of message in partition
Consumer Group = Set of consumers that share partition load
```

### When to Use Kafka
- Event streaming (clicks, logs, metrics)
- Data pipeline (ETL from multiple sources)
- Event sourcing (rebuild state from events)
- Microservice communication (event-driven)
- Real-time analytics

### Kafka Guarantees
```
At-most-once:  May lose messages (producer doesn't retry)
At-least-once: May duplicate messages (consumer retries)
Exactly-once:  No loss, no duplicates (idempotent producer + transactions)
```

---

## RabbitMQ / SQS

Traditional message queue — job/task processing.

### Patterns

```
1. Work Queue (Competing Consumers)
   ─────────────────────────────────
   Producer → Queue → Worker 1 (processes job)
                    → Worker 2 (processes job)
                    → Worker 3 (processes job)
   
   Each message processed by exactly ONE worker.
   Use for: sending emails, image processing, PDF generation

2. Pub/Sub (Fan-out)
   ──────────────────
   Producer → Exchange → Queue 1 → Consumer A (email service)
                       → Queue 2 → Consumer B (notification service)
                       → Queue 3 → Consumer C (analytics service)
   
   Each message goes to ALL subscribed queues.
   Use for: broadcasting events to multiple services

3. Dead Letter Queue (DLQ)
   ────────────────────────
   Main Queue → Worker fails 3 times → DLQ
   
   Messages that can't be processed go to DLQ for investigation.
   ALWAYS set up DLQ — don't lose failed messages silently.
```

---

## Common Patterns in System Design

### 1. Request-Reply (Async RPC)
```
Client → Request Queue → Worker processes → Reply Queue → Client
(with correlation ID to match request/response)
```

### 2. Saga Pattern (Distributed Transactions)
```
Order Service → [Order Created Event] → Payment Service
                                              │
Payment Service → [Payment Completed] → Inventory Service
                                              │
Inventory Service → [Items Reserved] → Shipping Service

If any step fails → compensating transactions (undo previous steps)
```

### 3. CQRS + Event Sourcing
```
Write Command → Command Handler → Event Store (Kafka)
                                       │
                                  Event published
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                   ▼
              Read Model 1      Read Model 2        Analytics
              (user feed)       (search index)      (metrics)
```

### 4. Outbox Pattern
```
Problem: DB write + message publish must be atomic
         (what if DB succeeds but message fails?)

Solution:
1. Write to DB + write to "outbox" table in SAME transaction
2. Background worker polls outbox → publishes to queue → marks as sent

┌─────────────┐   same tx    ┌─────────────┐
│ Orders Table │◀────────────▶│ Outbox Table│
└─────────────┘              └──────┬──────┘
                                    │ poll
                              ┌─────▼─────┐
                              │  Publisher │──▶ Kafka
                              └───────────┘
```

---

## Choosing the Right Queue

```
Use Case                               → Technology
──────────────────────────────────────────────────────────
Simple task queue (email, PDF)         → SQS or RabbitMQ
High-throughput event streaming        → Kafka
Ordered processing (per entity)        → Kafka (partition by entity ID)
Fan-out to multiple services           → Kafka or RabbitMQ (exchanges)
Serverless event processing            → SQS + Lambda
Delayed/scheduled messages             → SQS (delay queues) or RabbitMQ (TTL)
Priority queue                         → RabbitMQ (priority queues)
```

---

## Key Numbers

```
Kafka single partition:     ~100K messages/sec
Kafka cluster:              Millions of messages/sec
SQS standard:              ~3000 messages/sec per queue (scales)
SQS FIFO:                  ~300 messages/sec (3000 with batching)
RabbitMQ:                  ~20K-50K messages/sec
```

---

## Interview Tips

1. **Always consider async** — "Does the user need to wait for this? If not → queue it"
2. **Mention failure handling** — "If the worker crashes, the message goes back to queue (visibility timeout)"
3. **Ordering** — "I'd partition by user_id so all events for one user are processed in order"
4. **Idempotency** — "Workers must be idempotent because messages can be delivered more than once"
5. **Backpressure** — "If consumers can't keep up, the queue acts as a buffer. We monitor queue depth and scale workers"
