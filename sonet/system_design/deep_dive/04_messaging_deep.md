# Apache Kafka Internals — Deep Dive

## Part 1: Why Kafka Exists

### Before Kafka: Point-to-Point Chaos
```
Each source connected to each destination directly:
  [User Service] → [Email Service]
  [User Service] → [Analytics]
  [Order Service] → [Email Service]
  [Order Service] → [Inventory]
  [Order Service] → [Analytics]
  
N sources × M destinations = N×M integrations.
Schema changes break all consumers.
Any consumer slowdown affects the producer.
```

### After Kafka: Hub-and-Spoke
```
Everyone publishes to Kafka topics.
Everyone reads from Kafka topics.
N sources + M destinations = N + M integrations.

[User Service]  → TOPIC: user.events  → [Email Service]
[Order Service] → TOPIC: order.events → [Inventory Service]
                                      → [Analytics Pipeline]
                                      → [Fraud Detection]
```

---

## Part 2: Kafka Architecture Deep Dive

### Core Components
```
Broker:     Kafka server (stores logs, handles client connections)
Topic:      Named feed/category (e.g., "orders", "user_events")
Partition:  Ordered, immutable log segment within a topic
Offset:     Integer position within a partition (monotonically increasing)
Producer:   Writes records to a topic
Consumer:   Reads records from a topic
Consumer Group: Multiple consumers sharing partitions of a topic
ZooKeeper / KRaft: Cluster metadata management (ZK deprecated in Kafka 3.x)
```

### The Commit Log
```
Partition = append-only ordered log on disk:

Offset: 0    1    2    3    4    5    6    ...
Record: [A] [B] [C] [D] [E] [F] [G] ...

Key properties:
  - Immutable after write
  - Random reads by offset: O(1) (with index file)
  - Sequential writes: extremely fast (sequential I/O)
  - Replicated to N broker replicas for fault tolerance

Consumer tracks its own offset:
  Consumer group X reads from offset 3 onward
  Consumer group Y reads from offset 0 (different speed, independent)
  
Multiple consumer groups can read the SAME partition independently.
```

---

## Part 3: Replication — How Kafka Handles Failures

### ISR (In-Sync Replicas)
```
For each partition, one broker is the Leader, others are Followers.
  
Replication factor = 3:
  Broker 1: Leader
  Broker 2: Follower (ISR)
  Broker 3: Follower (ISR)

ISR = set of replicas that are "caught up" with leader.
A replica falls out of ISR if it falls > replica.lag.time.max.ms behind.

Write flow:
  Producer → Leader (writes to disk)
  Leader → Follower 2 (replicates)
  Leader → Follower 3 (replicates)
  Once all ISR ACK → Leader acknowledges producer

acks setting:
  acks=0: Don't wait for any ACK (fastest, can lose data)
  acks=1: Wait for leader ACK only (data loss if leader fails before followers replicate)
  acks=all: Wait for all ISR ACKs (safest, highest latency)
  
Production recommendation: acks=all + min.insync.replicas=2
```

### Leader Election on Failure
```
Broker 1 (leader) dies.
ZooKeeper/KRaft detects failure (missed heartbeats).
One of the ISR followers elected as new leader.
Producers/Consumers get metadata update → redirect to new leader.

If no ISR follower available (all replicas behind):
  unclean.leader.election.enable=false (default): wait, no data loss but unavailable
  unclean.leader.election.enable=true:  elect out-of-sync replica, may lose messages
```

---

## Part 4: Producers — Delivery and Batching

### Producer Record Path
```
Producer.send(record):
  1. Serialise key + value
  2. Determine partition:
     - Key specified: hash(key) % num_partitions → deterministic routing
     - No key: round-robin or sticky partitioning
  3. Add to RecordBatch for that topic-partition
  4. When batch full OR linger.ms elapsed → send batch to broker
  5. Await ACK (per acks setting)
```

### Batching Tuning
```
batch.size:      Max bytes per batch (default 16KB; increase for high-throughput)
linger.ms:       Wait N ms to fill batch before sending (default 0; increase for batching)
compression.type: none/gzip/snappy/lz4/zstd

High throughput config:
  batch.size=65536     (64KB)
  linger.ms=20         (wait 20ms to batch records)
  compression.type=lz4 (fast compression)
  
Impact: 10x–100x throughput improvement vs defaults.
```

### Exactly-Once Semantics (EOS)
```
Problem: Producer retries on timeout → broker may have already processed → duplicate.

Solution: Idempotent Producer
  Each producer gets a unique Producer ID (PID).
  Each message gets a sequence number.
  Broker deduplicates messages with same (PID, sequence_number).
  
Enable: enable.idempotence=true

For transactional (read-process-write atomic):
  Producer wraps in transaction:
    beginTransaction()
    read from input topic
    process
    produce to output topic
    commitTransaction()  ← atomic: either all commits or all rollback
  
  consumer: isolation.level=read_committed
    → Only reads committed (transactional) messages
```

---

## Part 5: Consumers — The Hard Parts

### Consumer Group Mechanics
```
Topic "orders" with 6 partitions.
Consumer Group "notifications" has 3 consumers.

Assignment:
  Consumer 1 → Partition 0, 1
  Consumer 2 → Partition 2, 3
  Consumer 3 → Partition 4, 5

Key rule: At most 1 consumer per partition per group.
If consumers > partitions → some consumers are idle.
If consumers < partitions → some consumers handle multiple partitions.
```

### Rebalancing — The Painful Part
```
When a consumer joins or leaves the group → rebalance.

Stop-the-world rebalance:
  1. Group Coordinator notifies all consumers: "Rebalance starting"
  2. All consumers stop consuming and "rejoin" the group
  3. Coordinator assigns partitions to consumers
  4. Consumers resume from last committed offset
  
Problem: During rebalance, no consumer processes any message.
For 10K consumer group: rebalance takes seconds to minutes!

Better: Incremental Cooperative Rebalance (Kafka 2.4+)
  Only reassign partitions that need to move.
  Other consumers keep consuming unaffected partitions.
  Rebalance is gradual, no full stop.
```

### Offset Management
```
Committed offset = last successfully processed message position.

Commit strategies:
  Auto-commit (default):
    Consumer commits every auto.commit.interval.ms (default 5s)
    Risk: commit before processing done → message loss
    Risk: crash after commit but before processing → message not retried
    
  Manual commit after processing:
    consumer.commitSync()  after successfully processing batch
    Guaranteed at-least-once (no message loss, possible duplicates on restart)
    
  Idempotent processing + at-least-once:
    Best combination for most applications.
    
Offset storage:
  Stored in internal topic: __consumer_offsets
  (Not ZooKeeper anymore since Kafka 0.9)
```

---

## Part 6: Log Compaction

### What It Is
```
Normal topic: messages kept for N days, then deleted.
Compacted topic: keeps only the LATEST value per key.

Use case: "Change data capture" — current state of each record.
  User profile topic: always have latest profile per user_id.
  
Compaction example:
  Before:  [user:1, {name: Alice}], [user:2, {name: Bob}], [user:1, {name: Alice Smith}]
  After:   [user:2, {name: Bob}], [user:1, {name: Alice Smith}]

Tombstone: Write message with key but null value → deletes that key.
```

### When to Use Log Compaction
```
Use compaction for:
  - Configuration/settings topics
  - User preferences
  - Database CDC (Change Data Capture) — current DB state in Kafka

Use retention (TTL) for:
  - Event logs (clicks, views)
  - Metrics
  - Audit logs (keep all history)
```

---

## Part 7: Kafka vs RabbitMQ — Detailed Comparison

| Feature | Kafka | RabbitMQ |
|---------|-------|---------|
| Model | Pull (consumer polls) | Push (broker pushes to consumer) |
| Message retention | Configurable time/size (days) | Deleted after ACK |
| Replay | Yes (seek to any offset) | No (consumed = gone) |
| Ordering | Per partition (strict) | Per queue |
| Throughput | Millions/sec | Tens of thousands/sec |
| Consumer groups | Multiple independent groups | Competing consumers |
| Routing | Partition key only | Exchange types (direct, topic, fanout, headers) |
| Persistence | Always (log-based) | Optional (durable queues) |
| Message priority | No | Yes |
| Delay messages | No (native) | Yes (with TTL + DLQ trick) |
| Best for | Event streaming, audit, analytics | Task queues, RPC, complex routing |

---

## Part 8: Kafka Streams and ksqlDB

### Kafka Streams (Java Library)
```
Process events as they arrive, maintaining state:

// Count orders per region in 5-minute windows
KStream<String, Order> orders = builder.stream("orders");
orders
  .groupBy((key, order) -> order.region)
  .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
  .count()
  .toStream()
  .to("region-order-counts");
```

### ksqlDB (SQL on Kafka)
```sql
-- Create stream from Kafka topic
CREATE STREAM orders (
  order_id VARCHAR, region VARCHAR, amount DOUBLE
) WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='JSON');

-- Aggregate in streaming SQL
SELECT region, SUM(amount) AS total
FROM orders
WINDOW TUMBLING (SIZE 5 MINUTES)
GROUP BY region
EMIT CHANGES;
```

---

## Key Takeaways

1. **Partitions** are the unit of parallelism and ordering — choose count carefully (16-64 for most topics).
2. **ISR + acks=all** gives durable, consistent writes at cost of some latency.
3. **Consumer group rebalances** are painful — use incremental cooperative rebalance and minimise restarts.
4. **Commit after processing** for at-least-once; pair with idempotent consumers.
5. **Log compaction** for "current state" topics; TTL retention for event history.
6. **Batching** (linger.ms + batch.size) dramatically increases throughput.
7. **Kafka is not a message queue** — it's a distributed commit log. Messages persist after consumption.
