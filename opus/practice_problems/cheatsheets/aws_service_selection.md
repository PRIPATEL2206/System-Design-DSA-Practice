# AWS Service Selection Cheatsheet

## The Decision Framework

Before picking any AWS service, ask these 3 questions:
1. **What's the access pattern?** (real-time vs batch, read-heavy vs write-heavy)
2. **What's the scale?** (requests/sec, data volume, concurrent users)
3. **Who manages it?** (serverless vs provisioned, cost vs control)

---

## Message Queues & Streaming

```
┌──────────────────────────────────────────────────────────────────────┐
│ "I need to process messages"                                          │
│                                                                       │
│ Q: Does the consumer need to replay old messages?                     │
│    NO  → Q: Simple task queue? ────────────── YES → SQS              │
│    │        Complex routing rules? ─────────── YES → RabbitMQ (EC2)  │
│    │        Fan out to multiple targets? ──── YES → SNS + SQS        │
│    │                                                                  │
│    YES → Q: Already on AWS, team < 5 people? ─ YES → Kinesis         │
│            Need Kafka ecosystem (Connect, Streams)? → MSK (Managed   │
│            Need multi-region replication? ─────────── MSK or Kafka    │
│            Budget-sensitive, < 1MB/sec? ───────────── Kinesis         │
└──────────────────────────────────────────────────────────────────────┘
```

### Quick Reference

| Service | Use Case | Throughput | Retention | Cost Model |
|---------|----------|-----------|-----------|------------|
| **SQS** | Task queue, decoupling | Unlimited | 14 days max | Per request ($0.40/1M) |
| **SNS** | Fan-out notifications | Unlimited | No retention | Per publish ($0.50/1M) |
| **Kinesis** | Log streaming, clickstream | 1MB/sec/shard | 24hr - 365 days | Per shard-hour ($0.015/hr) |
| **MSK** | Full Kafka (Connect, Streams) | Unlimited | Configurable | Per broker-hour (~$0.21/hr) |
| **EventBridge** | Event routing between services | Limited | 24hr replay | Per event ($1/1M) |

### Common Mistakes
```
❌ Using RabbitMQ when SQS works (unnecessary ops burden)
❌ Using Kafka for simple task queues (overkill)
❌ Using SQS when you need replay (SQS deletes after read)
❌ Using SNS when you need persistence (SNS is fire-and-forget)
❌ Using Kinesis with >100 consumers (1 shard = max 5 reads/sec)
```

---

## Databases & Storage

```
┌──────────────────────────────────────────────────────────────────────┐
│ "I need to store data"                                                │
│                                                                       │
│ Q: What's the access pattern?                                         │
│                                                                       │
│ Key-value lookups (user profiles, sessions)                           │
│   → Low latency (<10ms)? ──────────────── DynamoDB                   │
│   → Need caching? ────────────────────── ElastiCache (Redis)          │
│                                                                       │
│ Relational (joins, transactions, ACID)                                │
│   → < 64TB, standard SQL? ────────────── RDS (PostgreSQL/MySQL)       │
│   → Petabyte scale, analytics? ────────── Redshift                    │
│   → Serverless, auto-scale? ──────────── Aurora Serverless            │
│                                                                       │
│ Document (flexible schema, nested JSON)                               │
│   → General purpose? ─────────────────── MongoDB Atlas / DocumentDB   │
│   → Full-text search + analytics? ────── OpenSearch                   │
│                                                                       │
│ Time-series (logs, metrics, IoT)                                      │
│   → Metrics/monitoring? ──────────────── Timestream                   │
│   → Logs with search? ───────────────── OpenSearch                    │
│   → Cold logs, SQL queries? ─────────── S3 + Athena                   │
│                                                                       │
│ File/Object storage                                                   │
│   → Any unstructured data, backups? ──── S3                           │
│   → Archive (rare access)? ──────────── S3 Glacier                    │
└──────────────────────────────────────────────────────────────────────┘
```

### Database Selection Table

| Service | Best For | Latency | Scale | Cost |
|---------|----------|---------|-------|------|
| **DynamoDB** | Key-value, session store | <10ms | Infinite (auto) | Per RCU/WCU |
| **RDS** | Transactions, joins | 1-5ms | 64TB max | Per instance |
| **Aurora** | High-throughput relational | 1-5ms | 128TB, 15 replicas | Per instance + I/O |
| **Redshift** | Analytics, OLAP | Seconds | Petabytes | Per node |
| **OpenSearch** | Full-text search, logs | 10-100ms | Petabytes | Per instance |
| **MongoDB** | Flexible schema, documents | 5-20ms | Terabytes (sharded) | Per instance |
| **ElastiCache** | Caching, sessions, counters | <1ms | 500 nodes | Per node |
| **S3 + Athena** | Historical queries, data lake | 2-30 sec | Unlimited | $5/TB scanned |

### Storage Tiering Pattern (Use This in Every Design)
```
HOT (frequent access, fast queries):
  → DynamoDB / OpenSearch / MongoDB
  → Window: last 7-30 days
  → Cost: $$$

WARM (occasional access, acceptable latency):
  → S3 Standard + Athena
  → Window: 30 days - 1 year
  → Cost: $$

COLD (rare access, compliance):
  → S3 Glacier / Glacier Deep Archive
  → Window: 1-7 years
  → Cost: $

Rule: NEVER store cold data in a hot database. 
      This is the #1 cost mistake in system design.
```

---

## Compute

```
┌──────────────────────────────────────────────────────────────────────┐
│ "I need to run code"                                                  │
│                                                                       │
│ Q: How long does it run?                                              │
│                                                                       │
│ < 15 minutes, event-driven:                                           │
│   → Simple function? ─────────────── Lambda                           │
│   → Needs container + dependencies? ── Lambda (container image)       │
│                                                                       │
│ Long-running (hours/days):                                            │
│   → Containerized service? ────────── ECS Fargate (serverless)        │
│   → Need GPUs? ───────────────────── ECS on EC2 (GPU instances)       │
│   → Batch processing? ────────────── AWS Batch                        │
│                                                                       │
│ Always-on API server:                                                 │
│   → Microservices? ───────────────── ECS Fargate + ALB                │
│   → Kubernetes required? ─────────── EKS                              │
│   → Simple web app? ─────────────── Elastic Beanstalk                 │
│                                                                       │
│ ML model serving:                                                     │
│   → Real-time inference? ─────────── SageMaker Endpoint               │
│   → Batch predictions? ──────────── SageMaker Batch Transform         │
│   → Lightweight model (<250MB)? ──── Lambda (if latency OK)           │
└──────────────────────────────────────────────────────────────────────┘
```

### Lambda vs ECS vs EC2 Decision

| Factor | Lambda | ECS Fargate | EC2 |
|--------|--------|-------------|-----|
| Max runtime | 15 min | Unlimited | Unlimited |
| Cold start | 200-500ms | 30-60 sec | N/A (always on) |
| Max memory | 10GB | 30GB | Any |
| Package size | 250MB zip / 10GB image | Any | Any |
| GPU | No | No (Fargate) | Yes |
| Scaling | Instant (1000 concurrent) | 1-2 min | 3-5 min |
| Cost at low traffic | Nearly free | ~$30/month min | ~$50/month min |
| Cost at high traffic | Expensive | Moderate | Cheapest |
| Best for | Event handlers, glue code | Microservices, APIs | ML training, GPUs |

---

## Data Processing

```
┌──────────────────────────────────────────────────────────────────────┐
│ "I need to process/transform data"                                    │
│                                                                       │
│ Real-time (stream processing):                                        │
│   → Simple transforms? ───────────── Lambda + Kinesis                 │
│   → SQL on streams? ──────────────── Kinesis Analytics                │
│   → Complex event processing? ────── Flink on MSK / KDA              │
│                                                                       │
│ Batch (scheduled, large datasets):                                    │
│   → ETL to data warehouse? ───────── Glue ETL                        │
│   → PySpark jobs? ────────────────── Glue / EMR                       │
│   → Simple SQL transforms? ───────── Athena CTAS                      │
│                                                                       │
│ Data movement:                                                        │
│   → Stream to S3/Redshift? ───────── Kinesis Firehose (zero code)     │
│   → Database replication? ────────── DMS                              │
│   → API to S3? ──────────────────── Lambda + EventBridge              │
└──────────────────────────────────────────────────────────────────────┘
```

---

## API & Networking

```
┌──────────────────────────────────────────────────────────────────────┐
│ "I need to expose an API"                                             │
│                                                                       │
│ REST API:                                                             │
│   → Serverless + Lambda? ─────────── API Gateway (REST)               │
│   → Container-based? ────────────── ALB + ECS                         │
│                                                                       │
│ WebSocket (real-time push):                                           │
│   → Serverless? ─────────────────── API Gateway WebSocket             │
│   → Self-managed? ───────────────── ALB + ECS (Socket.io/ws)          │
│                                                                       │
│ GraphQL:                                                              │
│   → Managed? ────────────────────── AppSync                           │
│   → Self-hosted? ────────────────── ECS + Apollo Server               │
│                                                                       │
│ CDN / Static content:                                                 │
│   → Global distribution? ─────────── CloudFront + S3                  │
│   → API caching? ────────────────── CloudFront or API Gateway cache   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## The "Senior Engineer" Patterns

### Pattern 1: Event-Driven Decoupling
```
Instead of: Service A → direct call → Service B
Use:        Service A → EventBridge/SNS → Service B
                                        → Service C (added later, zero change to A)
```

### Pattern 2: CQRS (Command Query Responsibility Segregation)
```
Writes: API → Lambda → DynamoDB (optimized for writes)
Reads:  API → Lambda → OpenSearch (optimized for search)
Sync:   DynamoDB Streams → Lambda → OpenSearch (eventual consistency)
```

### Pattern 3: Saga Pattern (Distributed Transactions)
```
Order Service → Payment Service → Inventory Service
     ↓ (if payment fails)
     Compensating transaction: refund + restore inventory
     
Implement with: Step Functions (orchestration) or EventBridge (choreography)
```

### Pattern 4: Circuit Breaker
```
Service A → Circuit Breaker → Service B
  - Closed: normal operation
  - Open: Service B unhealthy, fail fast (don't wait for timeout)
  - Half-open: test with 1 request, close if healthy
  
AWS: Use Lambda Destinations + DLQ for automatic retry/fallback
```

---

## Interview Cheatsheet: What to Say

| When interviewer asks... | Say this |
|--------------------------|----------|
| "How would you store this?" | "It depends on access pattern. For hot data I'd use X, for cold data Y..." |
| "How does this scale?" | "Kinesis/Kafka absorbs traffic spikes, Lambda/ECS auto-scales compute..." |
| "What about failures?" | "The queue provides durability. If consumer dies, messages replay from last offset..." |
| "Cost concerns?" | "Storage tiering keeps hot data fast and cold data cheap. S3 lifecycle handles transitions..." |
| "How do you monitor?" | "CloudWatch metrics + alarms, X-Ray for tracing, custom dashboards for business metrics..." |

---

## Anti-Patterns to NEVER Say in an Interview

```
❌ "We'll put everything in one database"
   → Always separate hot/cold, OLTP/OLAP

❌ "Lambda will handle it" (without discussing limits)
   → Always mention: concurrency, cold start, timeout limits

❌ "We'll use microservices for everything"  
   → Start monolith, split only when scale demands it

❌ "We'll figure out schema later"
   → At least mention: event versioning, backward compatibility

❌ "We'll scale vertically"
   → Senior engineers think horizontally first

❌ "RabbitMQ/SQS for event streaming"
   → These are task queues, not event logs. Use Kafka/Kinesis for streaming.
```
