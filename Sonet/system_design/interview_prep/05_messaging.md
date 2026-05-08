# Messaging Queues & Event Streaming

## Why Message Queues?

**Problems they solve:**
```
1. Decoupling  — Service A doesn't need to know about Service B
2. Buffering   — Absorb traffic spikes (queue fills up, not DB)
3. Async work  — Return HTTP 202 immediately, process in background
4. Fan-out     — One event → multiple consumers
5. Retry logic — Failed processing retried without re-calling API
```

```
Without queue (tight coupling):
[Order Service] ──sync call──> [Email Service]  ← if email is down, order fails

With queue (loose coupling):
[Order Service] ──publish──> [Queue] ──consume──> [Email Service]
                                     ──consume──> [Analytics Service]
                                     ──consume──> [Inventory Service]
```

---

## Core Concepts

| Term | Definition |
|------|-----------|
| **Producer** | Service that sends messages |
| **Consumer** | Service that receives messages |
| **Broker** | The queue server (Kafka, RabbitMQ) |
| **Topic** | Named channel; producers write to topics, consumers read from topics |
| **Partition** | Sub-division of a topic for parallelism (Kafka) |
| **Consumer Group** | Multiple consumers sharing the work of consuming a topic |
| **Offset** | Position of a message in a partition (Kafka) |
| **Acknowledgement (ACK)** | Consumer tells broker "I processed this message" |

---

## Message Delivery Guarantees

| Guarantee | Meaning | Risk | Use Case |
|-----------|---------|------|---------|
| **At-most-once** | Message delivered 0 or 1 times | Can lose messages | Log shipping, metrics (loss OK) |
| **At-least-once** | Message delivered 1 or more times | Can duplicate messages | Most common; handle duplicates in consumer |
| **Exactly-once** | Message delivered exactly 1 time | Hardest to achieve | Financial transactions, inventory |

**Interview tip:** For most systems, use **at-least-once** and make consumers **idempotent** (safe to process same message twice).

```
Idempotency example:
"Charge user $10" — NOT idempotent (could charge twice)
"Set balance to $90 if current is $100" — IS idempotent
"Process order #12345 if not already processed" — IS idempotent (check DB)
```

---

## Kafka Architecture

Apache Kafka is the industry standard for high-throughput event streaming.

```
                        KAFKA CLUSTER
[Producer A] ─────>  Topic: "orders"
[Producer B] ─────>  ┌──────────────────────────────────────┐
                      │  Partition 0: [msg0] [msg1] [msg2]   │
                      │  Partition 1: [msg0] [msg1] [msg2]   │
                      │  Partition 2: [msg0] [msg1] [msg2]   │
                      └──────────────────────────────────────┘
                           |            |            |
                      [Consumer        [Consumer    [Consumer
                       Group A          Group A      Group A
                       Worker 1]        Worker 2]    Worker 3]
                      
[Consumer Group B] ────── reads independently (own offsets)
(e.g. Analytics)
```

### Key Properties
- **Durability:** Messages written to disk; survive broker restarts
- **Retention:** Messages kept for configurable time (default 7 days), not deleted after consumption
- **Ordered:** Within a partition, messages are strictly ordered
- **Scalability:** Add partitions to scale; each partition handled by one consumer per group
- **Replayability:** Consumers can rewind offset and re-read past messages

### Partition Key
```
Messages with the same key always go to the same partition → order guaranteed for that key.
Example: order_id as key → all events for one order are in order.
```

---

## RabbitMQ Architecture

Traditional message broker. Great for task queues and complex routing.

```
[Producer] ──> [Exchange] ──routing key──> [Queue 1] ──> [Consumer 1]
                         ──routing key──> [Queue 2] ──> [Consumer 2]
```

### Exchange Types
| Type | How it routes | Use Case |
|------|--------------|---------|
| **Direct** | Exact match on routing key | Task queues |
| **Topic** | Pattern match (e.g., `order.*`) | Event categorisation |
| **Fanout** | Broadcast to all bound queues | Notifications to multiple services |
| **Headers** | Match on message headers | Complex routing rules |

### Key Difference from Kafka
| Feature | Kafka | RabbitMQ |
|---------|-------|---------|
| Model | Event log (pull) | Message queue (push) |
| Message deletion | After TTL (consumers can replay) | After ACK (consumed and gone) |
| Ordering | Per partition | Per queue |
| Throughput | Millions/sec | Thousands/sec |
| Use case | Event streaming, audit log, analytics | Task queues, RPC, microservice messaging |

---

## Common Queue Patterns

### 1. Task Queue (Worker Pool)
```
[Web Server] ──publish──> [Queue] ──distribute──> [Worker 1]
                                               ──> [Worker 2]
                                               ──> [Worker 3]
Use: Image resize, email sending, PDF generation
```

### 2. Pub/Sub (Fan-out)
```
[Order Service] ──event: "order_placed"──> [Topic]
                                            ├──> [Email Consumer]
                                            ├──> [Inventory Consumer]
                                            └──> [Analytics Consumer]
Use: Decouple microservices, event-driven architecture
```

### 3. Dead Letter Queue (DLQ)
```
Message fails processing → retry 3x → move to DLQ
Ops team inspects DLQ → fix bug → replay messages
Use: Handle poison messages without losing them
```

### 4. Priority Queue
```
High priority messages processed before low priority.
Use: Urgent notifications, premium user requests
```

---

## When to Use a Queue

| Situation | Queue Helps? |
|-----------|-------------|
| Send email after user signs up | Yes — decouple, retry on failure |
| Write to DB on user click | No — synchronous is fine |
| Process uploaded video | Yes — async, time-consuming |
| Real-time stock price update | No — use WebSocket/SSE instead |
| Fan-out to 10 microservices | Yes — pub/sub |
| Rate limit writes to a slow service | Yes — queue as buffer |

---

## Quick Reference

| Concept | One-line |
|---------|---------|
| Kafka | High-throughput, durable, replayable event log |
| RabbitMQ | Traditional broker; push model; task queues |
| At-least-once | Most practical; make consumers idempotent |
| Consumer group | Multiple consumers share a topic's partitions |
| DLQ | Failed messages go here for inspection/retry |
| Partition key | Routes same-key messages to same partition (ordering) |
| Idempotent consumer | Safe to process same message multiple times |
