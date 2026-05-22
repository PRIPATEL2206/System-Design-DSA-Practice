# My Common System Design Mistakes — Pattern Recognition

## Purpose
These are mistakes I keep making across multiple system design problems. Before answering any system design question, scan this list to catch myself BEFORE repeating them.

---

## Mistake 1: One Database for All Data

### The Pattern
I keep putting ALL data in one database (usually MongoDB) regardless of access frequency.

### Why I Do It
- Simpler to think about
- "MongoDB is schemaless so it handles everything"
- Don't think about data lifecycle

### The Rule
**ALWAYS split storage by access pattern:**
```
Hot  (accessed daily)     → Fast DB (DynamoDB, OpenSearch, MongoDB)
Cold (accessed monthly)   → Cheap storage (S3 + Athena)  
Archive (compliance only) → Cheapest (S3 Glacier)
```

### Self-Check Question
> "Will I still be querying this data the same way in 6 months?"
> If NO → it needs a separate storage tier.

---

## Mistake 2: Defaulting to Lambda for Everything

### The Pattern
I use Lambda as the compute layer for everything without checking its limits.

### Why I Do It
- Familiar with Lambda
- "Serverless = scalable"
- Don't think about cold starts or concurrency

### The Rule
**Lambda is great UNLESS:**
```
- Package > 250MB (ML models, heavy dependencies) → ECS or SageMaker
- Runtime > 15 min (training, batch jobs) → ECS or Batch
- Steady 1000+ req/sec (cost) → ECS Fargate is cheaper
- Sub-50ms latency required → Lambda cold start kills this
- GPU needed → EC2 with GPU instances
```

### Self-Check Question
> "What happens at 3x my expected traffic with Lambda?"
> Calculate: peak_rps × avg_duration = concurrent Lambdas needed
> If > 1000: mention reserved concurrency or alternative.

---

## Mistake 3: RabbitMQ When I Need Kafka/Kinesis

### The Pattern
I use RabbitMQ (or SQS) for streaming use cases where messages need to be read by multiple consumers or replayed.

### Why I Do It
- Familiar with "queue" concept
- Don't distinguish between task queues and event logs
- Don't think about replay scenarios

### The Rule
```
Message = COMMAND (do this thing once)  → SQS / RabbitMQ
Message = EVENT (this happened, many may care) → Kafka / Kinesis

Key questions:
- Will 2+ services read the same message? → Kafka/Kinesis
- Do I need replay if consumer fails? → Kafka/Kinesis
- Is it a background task (send email, resize image)? → SQS
```

### Self-Check Question
> "If the consumer dies mid-processing, what happens to the message?"
> If answer is "it's lost" → wrong queue choice.

---

## Mistake 4: Frontend Generating IDs / Direct Queue Access

### The Pattern
I let the frontend generate critical identifiers or directly interact with backend infrastructure (Kafka, RabbitMQ).

### Why I Do It
- Simplifies the mental model
- Don't think about security implications
- Forget that browsers can't talk to infrastructure

### The Rule
```
NEVER expose to frontend:
- ID generation (security: spoofing, collision attacks)
- Message queues (Kafka, RabbitMQ, SQS are backend-only)
- Database connections (obviously)
- Internal service URLs

Frontend communication options:
- REST API (request-response)
- WebSocket (real-time push from server)
- SSE (one-way server stream)
- Polling (simplest, but wasteful)
```

### Self-Check Question
> "Can a malicious user abuse this by manipulating what the frontend sends?"
> If YES → move that logic server-side.

---

## Mistake 5: No Alert Deduplication

### The Pattern
Every anomaly/error triggers a separate notification. I don't think about what happens when 1000 errors fire in 5 minutes.

### Why I Do It
- Focus on happy path (one error → one alert)
- Don't think about error storms
- Forget that humans get alert fatigue

### The Rule
```
ALWAYS include in alerting design:
1. Deduplication window (group by service + error_type for 5 min)
2. Severity escalation (1 error=log, 10=slack, 100=pagerduty)
3. Rate limiting (max 1 alert per service per 5 min)
4. Digest mode (batch into summary instead of individual alerts)
```

### Self-Check Question
> "What happens if this component fails 1000 times in a row?"
> If answer is "1000 emails" → add deduplication.

---

## Mistake 6: Not Addressing "How Does the Response Get Back?"

### The Pattern
I design the write/processing path but forget the read/response path back to the user.

### Why I Do It
- Focus on the interesting part (processing)
- Assume "somehow the user gets the result"
- Don't think about async response delivery

### The Rule
```
Every async system needs an explicit response mechanism:

Synchronous flow: Client waits → Server processes → Server responds
                  (simple but blocks the client)

Asynchronous flow: Client sends → gets acknowledgment immediately
                   Then one of:
                   a) WebSocket push when done
                   b) Client polls GET /status/{id}
                   c) Webhook callback to client's URL
                   d) Email/notification when complete
```

### Self-Check Question
> "The user clicked submit. How do they know when it's done?"
> Must have a concrete answer with specific mechanism.

---

## Mistake 7: Ignoring Cost in Architecture

### The Pattern
I pick services without considering cost, then get challenged by interviewer on budget.

### Why I Do It
- Focus on "what works" not "what costs"
- Don't know approximate service pricing
- Don't do back-of-envelope cost estimation

### The Rule
```
Always do a 30-second cost estimate:
1. Estimate daily volume
2. Look up per-unit cost of chosen services
3. Multiply and express monthly

Quick cost references:
  Lambda: $0.20 per 1M requests + duration
  SQS: $0.40 per 1M requests
  Kinesis: $0.015/shard/hour (~$11/shard/month)
  S3: $0.023/GB/month (standard)
  DynamoDB: $1.25 per 1M writes, $0.25 per 1M reads
  SageMaker: ~$0.05 per 1000 predictions (ml.m5.large)
  RDS (db.r5.large): ~$200/month
  OpenSearch (r6g.large): ~$150/month per node
```

### Self-Check Question
> "If traffic doubles, does my cost double or 10x?"
> Linear scaling = good design. Exponential = redesign needed.

---

## Pre-Answer Checklist (Use Before Every System Design Answer)

```
□ Did I separate hot/cold/archive storage?
□ Did I check Lambda limits for my use case?
□ Am I using the right queue type? (task queue vs event log)
□ Is all sensitive logic server-side? (not frontend)
□ Do I have alert deduplication?
□ How does the response get back to the user?
□ Did I estimate cost?
□ What breaks at 10x traffic?
```
