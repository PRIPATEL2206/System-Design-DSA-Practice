# Lambda Limitations & When to Use Alternatives

## Lambda Limits That Break System Designs

```
┌─────────────────────────────────────────────────────────────────┐
│ LIMIT                    │ VALUE          │ WHY IT MATTERS       │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Max execution time       │ 15 minutes     │ Can't run training,  │
│                          │                │ long ETL, video proc │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Package size (zip)       │ 250 MB         │ ML models won't fit  │
│ Package size (container) │ 10 GB          │ Better, but still    │
│                          │                │ limited for large    │
│                          │                │ models               │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Memory                   │ 128MB - 10GB   │ Large DataFrames,    │
│                          │                │ in-memory processing │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Default concurrency      │ 1000/region    │ At 1000 req/sec with │
│                          │                │ 1s duration = maxed  │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Cold start               │ 200ms - 2s     │ Kills latency-       │
│                          │                │ sensitive paths       │
├──────────────────────────┼────────────────┼──────────────────────┤
│ No GPU                   │ N/A            │ Can't do inference    │
│                          │                │ on GPU models         │
├──────────────────────────┼────────────────┼──────────────────────┤
│ /tmp storage             │ 512MB (10GB    │ Can't process large  │
│                          │ with ephemeral)│ files locally         │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Payload size (sync)      │ 6 MB           │ Can't return large   │
│                          │                │ results directly      │
├──────────────────────────┼────────────────┼──────────────────────┤
│ Payload size (async)     │ 256 KB         │ Large events need    │
│                          │                │ S3 pointer pattern   │
└──────────────────────────┴────────────────┴──────────────────────┘
```

---

## Cold Start Deep Dive

### What causes cold start?
```
First request (or after ~5 min idle):
  1. AWS provisions a container        → 50-200ms
  2. Downloads your code/dependencies   → 50-500ms
  3. Initializes runtime (Python/Node)  → 50-200ms
  4. Runs your init code (imports, connections) → 50-500ms
  Total: 200ms - 2000ms BEFORE your handler runs

Subsequent requests (warm):
  Container already running → 0ms overhead
  Just runs your handler
```

### Factors that increase cold start
```
Language:     Java/C# = 1-3s  |  Python/Node = 200-600ms
Package size: 50MB+ = +200ms per 50MB
VPC:          +500ms (ENI attachment) — fixed in recent updates
Memory:       Higher memory = faster init (more CPU allocated)
```

### How to reduce cold starts
```
1. Provisioned Concurrency: keep N Lambdas always warm
   - Cost: $0.015/GB-hour (pay even when idle)
   - Use for: APIs with strict latency SLA

2. Keep functions warm: EventBridge ping every 5 min
   - Cost: minimal
   - Hack: doesn't scale, only keeps 1 instance warm

3. SnapStart (Java only): snapshot after init
   - Cold start: 1s → 200ms
   - Only for Java runtimes

4. Smaller packages: fewer dependencies = faster init
   - Use Lambda Layers for shared code
   - Only import what you need
```

---

## When Lambda Costs MORE Than Alternatives

### Cost Crossover Point
```
Lambda pricing:
  $0.20 per 1M requests
  $0.0000166667 per GB-second

ECS Fargate (0.25 vCPU, 0.5 GB):
  ~$9/month per task (always-on)

Break-even calculation:
  Lambda at 1M requests/month, 200ms each, 512MB:
  = 1M × $0.20/1M + (1M × 0.2s × 0.5GB × $0.0000166667)
  = $0.20 + $1.67 = $1.87/month  ← Lambda CHEAPER

  Lambda at 100M requests/month, 200ms each, 512MB:
  = $20 + $167 = $187/month  ← Lambda MORE EXPENSIVE
  
  ECS Fargate (2 tasks, handles 100M req/month easily):
  = 2 × $9 = $18/month  ← 10x CHEAPER than Lambda

Rule of thumb:
  < 1M requests/month → Lambda is cheapest
  > 10M requests/month steady → ECS Fargate is cheaper
  Spiky traffic (0 to 10K/sec) → Lambda (pay nothing at 0)
```

---

## Decision Matrix: Lambda vs Alternatives

### Lambda vs SageMaker (ML Inference)

| Factor | Lambda | SageMaker Endpoint |
|--------|--------|--------------------|
| Model size | <250MB (zip) or <10GB (container) | Unlimited |
| Cold start | 200ms - 2s | None (always warm) |
| GPU | No | Yes (ml.g4dn, ml.p3) |
| Auto-scale | Instant | 1-5 minutes |
| Batch inference | Manual loop | Built-in batch transform |
| Cost (low traffic) | Cheaper | ~$50/month minimum |
| Cost (high traffic) | Expensive | Cheaper per prediction |
| A/B testing | Manual | Built-in |
| Model monitoring | Manual | Built-in drift detection |

**Use SageMaker when:** Model > 250MB, need GPU, or > 100K predictions/day

### Lambda vs ECS Fargate (APIs & Services)

| Factor | Lambda | ECS Fargate |
|--------|--------|-------------|
| Cold start | Yes (200ms-2s) | No (always running) |
| Max runtime | 15 min | Unlimited |
| Max memory | 10 GB | 30 GB |
| Scaling speed | Instant | 1-2 minutes |
| Persistent connections | No (dies between requests) | Yes (WebSocket, DB pools) |
| Cost at 0 traffic | $0 | ~$9/month/task |
| Cost at high traffic | High | Lower |
| Deployment | Simple (zip/container) | More complex (task defs, services) |

**Use ECS when:** Need always-on, persistent connections, >15min processing, or high steady traffic

### Lambda vs Glue/EMR (Data Processing)

| Factor | Lambda | Glue | EMR |
|--------|--------|------|-----|
| Max runtime | 15 min | 48 hours | Unlimited |
| Max data size | ~10GB (memory bound) | Terabytes | Petabytes |
| PySpark | No | Yes | Yes |
| Cost model | Per invocation | Per DPU-hour ($0.44) | Per instance-hour |
| Setup complexity | Low | Medium | High |
| Best for | Small transforms, routing | ETL jobs, catalog | Heavy ML, custom Spark |

**Use Glue when:** Processing > 10GB, need PySpark, scheduled ETL
**Use EMR when:** Need full Spark control, custom libs, interactive notebooks

---

## Lambda Patterns That Work Well

### Pattern 1: Event Router (Small, Fast)
```
S3 upload → Lambda (route by file type) → SQS queue A or B
                                         → Step Functions
                                         → Another Lambda

Lambda code: 5 lines of routing logic, <100ms
This is Lambda's sweet spot.
```

### Pattern 2: Glue Code (Connect Services)
```
API Gateway → Lambda → DynamoDB
                     → SQS
                     → SNS
                     → Step Functions

Lambda code: validate input, transform, write. <50 lines.
```

### Pattern 3: Kinesis/SQS Consumer (Batched)
```
Kinesis → Lambda (batch of 100 records per invocation)
        → Process all 100, write results in bulk

Why batching helps:
  Without: 1M events = 1M Lambda invocations = $0.20 + cold starts
  With:    1M events = 10K Lambda invocations = $0.002 + warm
  100x cheaper, much faster
```

### Pattern 4: Step Functions Orchestration
```
Step Function:
  State 1: Lambda (validate)     → 200ms
  State 2: Lambda (process)      → 2s  
  State 3: Lambda (notify)       → 300ms
  
Total: 2.5s, but each Lambda is small and focused
If step 2 takes > 15min → replace with ECS task in the Step Function
```

---

## Lambda Anti-Patterns (Things That Break)

### Anti-Pattern 1: Lambda for Real-Time ML
```
❌ API → Lambda → load 2GB model → predict → respond
   Problem: cold start loads 2GB every time = 5-10 seconds

✓ API → Lambda → call SageMaker endpoint → respond
   SageMaker keeps model in memory, Lambda is just glue
```

### Anti-Pattern 2: Lambda for Long-Running Batch
```
❌ S3 (1TB file) → Lambda → process for 2 hours
   Problem: Lambda times out at 15 minutes

✓ S3 (1TB file) → Glue job (PySpark) → process → S3 output
   Or: S3 → Lambda (splits file into 100 chunks) → 100 Lambdas (parallel)
```

### Anti-Pattern 3: Lambda for WebSocket Server
```
❌ Client → Lambda WebSocket → keep connection open
   Problem: Lambda dies between invocations, can't hold connections

✓ Client → API Gateway WebSocket (managed) → Lambda (on message)
   API Gateway holds the connection, Lambda handles each message
```

### Anti-Pattern 4: Lambda for Database-Heavy Work
```
❌ Lambda (1000 concurrent) → RDS (max 100 connections)
   Problem: 1000 Lambdas each open a connection = RDS overwhelmed

✓ Lambda → RDS Proxy → RDS
   RDS Proxy pools connections: 1000 Lambdas share 50 connections
```

---

## Interview Talking Points

### When asked "Why not Lambda?"
```
"Lambda is excellent for event-driven, short-lived compute.
But in this case, [specific reason]:

- Our model is 2GB → exceeds Lambda's package limit → SageMaker
- Processing takes 2 hours → exceeds 15-min timeout → ECS/Glue
- We need 5000 concurrent at steady state → cost-prohibitive → Fargate
- We need sub-50ms latency → cold starts make this unreliable → Fargate
- We need GPU for inference → Lambda has no GPU → SageMaker/EC2

Lambda works great as the orchestration layer — calling SageMaker,
writing to DynamoDB, routing events — just not as the compute-heavy layer."
```

### When asked "Why Lambda?"
```
"Lambda is the right choice here because:
- Traffic is spiky (0 to 1000/sec) → pay nothing at idle
- Each invocation is <5 seconds → well within limits
- Code is <50MB → fast cold starts
- No persistent state → stateless by design
- Native integration with [Kinesis/S3/API Gateway] → less glue code
- Team doesn't want to manage containers → serverless = zero ops"
```
