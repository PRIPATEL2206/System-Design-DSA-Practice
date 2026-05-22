# System Design: Real-Time Fraud Detection System

## Problem Statement
Design a fraud detection system that evaluates payment transactions in real-time, determines if they are fraudulent, and notifies the user of the result.

**Requirements:**
- 10,000 transactions/sec at peak
- Fraud decision within 500ms
- Results pushed back to the user in real-time
- System must be horizontally scalable
- Store all transaction results for auditing

---

## My Initial Design (Attempt)

```
Frontend → Load Balancer → Kafka → Lambda → MongoDB
                                      ↓
                              RabbitMQ → Frontend
```

- Frontend generates transaction_id
- Lambda detects fraud
- MongoDB stores result (new document per txn)
- RabbitMQ broadcasts result
- Load Balancer scales Lambda

---

## Mistakes I Made

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Frontend generates transaction_id | Security risk: IDs can be guessed/spoofed | Server generates UUID after auth |
| Load Balancer scales Lambda | Lambda auto-scales. LB is for HTTP services | API Gateway + Lambda auto-concurrency |
| RabbitMQ for broadcasting to frontend | RabbitMQ deletes messages after ACK. Frontend can't "listen" to RabbitMQ directly | WebSocket (API Gateway WebSocket API) |
| Lambda for ML inference | 250MB package limit, cold starts | Lambda orchestrates, SageMaker does inference |
| MongoDB "new doc = no duplicates" | Lambda retries cause duplicates | Upsert with transaction_id as _id |

---

## Corrected Architecture

```
                    ┌─────────────────────────────────────────┐
                    │              CLIENT                       │
                    │                                           │
                    │  1. POST /pay → gets transaction_id       │
                    │  2. Opens WebSocket → waits for result    │
                    └───────────┬───────────────┬───────────────┘
                                │               │
                         HTTP POST          WebSocket
                                │               │
                    ┌───────────▼───────────┐   │
                    │    API Gateway (HTTP)  │   │
                    │    - Rate limiting     │   │
                    │    - Auth (JWT/API key)│   │
                    │    - Returns txn_id    │   │
                    └───────────┬────────────┘   │
                                │                │
                    ┌───────────▼────────────┐   │
                    │   Kafka / Kinesis       │   │
                    │   Topic: transactions   │   │
                    │   Partitioned by user_id│   │
                    └───────────┬────────────┘   │
                                │                │
                    ┌───────────▼────────────┐   │
                    │   Lambda (Consumer)     │   │
                    │   - Triggered by Kafka  │   │
                    │   - Calls SageMaker     │   │
                    │   - Writes to MongoDB   │   │
                    │   - Pushes to WebSocket │   │
                    └───────────┬────────────┘   │
                                │                │
                    ┌───────────▼────────────┐   │
                    │   SageMaker Endpoint    │   │
                    │   - Fraud ML model      │   │
                    │   - Returns score 0-1   │   │
                    └───────────┬────────────┘   │
                                │                │
              ┌─────────────────┼────────────────┘
              │                 │
    ┌─────────▼──────┐  ┌──────▼──────────────────┐
    │   MongoDB      │  │  API Gateway (WebSocket) │
    │   - upsert by  │  │  - Pushes result to user │
    │     txn_id     │  │  - Connection by user_id │
    │   - audit log  │  └─────────────────────────┘
    └────────────────┘
```

---

## Key Design Decisions

### Why Kafka over direct Lambda?
```
Direct: Client → Lambda → SageMaker → Response
Problem: At 10,000 req/sec, Lambda concurrency exhausted
         No buffer = requests dropped during spikes

Kafka:   Client → API Gateway → Kafka → Lambda (batched)
Benefit: Kafka absorbs 100K req/sec spikes without dropping
         Lambda processes at its own pace
         Replay capability if Lambda fails
```

### Why WebSocket over polling?
```
Polling: Client calls GET /result/{txn_id} every 500ms
         10,000 users × 2 polls/sec = 20,000 requests/sec (wasted)

WebSocket: Client opens once, server pushes when ready
           0 extra requests while waiting
           Result arrives in <100ms after processing
```

### Why MongoDB with upsert?
```python
# Handles Lambda retries (at-least-once delivery from Kafka)
db.transactions.update_one(
    {"_id": transaction_id},          # unique key
    {"$setOnInsert": {
        "user_id": user_id,
        "amount": amount,
        "is_fraud": is_fraud,
        "fraud_score": 0.87,
        "processed_at": datetime.utcnow()
    }},
    upsert=True
)
# Second Lambda retry = no-op (document already exists)
```

### Why SageMaker not Lambda for ML?
```
Lambda limits:
  - Package size: 250MB (zip) / 10GB (container)
  - Cold start: 200-500ms per invocation
  - Memory: 10GB max
  - No GPU

SageMaker endpoint:
  - Any model size
  - Always warm (no cold start)
  - GPU available (ml.g4dn for deep learning)
  - Auto-scales based on traffic
  - Inference: 5-20ms
```

---

## How to Handle 10x Traffic Spike

```
Normal: 10,000 txn/sec
Spike:  100,000 txn/sec (Diwali sale, flash sale)

Layer-by-layer:
1. API Gateway: auto-scales (AWS managed), throttle at 100K
2. Kafka: add partitions (pre-provision for known events)
3. Lambda: increase reserved concurrency from 1000 → 5000
4. SageMaker: auto-scaling policy (target: 1000 invocations/instance)
5. MongoDB: sharded by user_id (distributes writes)

What breaks first: Lambda concurrency limit
How to fix: Request increase from AWS + use Kafka batching
            (process 50 transactions per Lambda invocation)
```

---

## Cost Estimation (10K txn/sec)

```
Kafka (MSK):     3 brokers × $0.21/hr = ~$450/month
Lambda:          864M invocations/month × $0.20/1M = ~$170/month
SageMaker:       4 × ml.m5.xlarge = ~$1,400/month
MongoDB Atlas:   M50 cluster = ~$800/month
API Gateway WS:  864M messages × $1/1M = ~$860/month
─────────────────────────────────────────────────
Total: ~$3,700/month for 10K txn/sec system
```

---

## Interview Tips for This Problem

1. Always start with requirements + capacity estimation BEFORE drawing boxes
2. Never say "Lambda will handle it" without addressing concurrency limits
3. Always address: how does the response get back to the user?
4. Mention idempotency — interviewers love this at senior level
5. Show cost awareness — this separates senior from mid-level
