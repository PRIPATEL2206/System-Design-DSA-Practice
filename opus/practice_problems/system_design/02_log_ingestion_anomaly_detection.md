# System Design: Log Ingestion & Anomaly Detection Pipeline

## Problem Statement
Design a large-scale log ingestion and anomaly detection pipeline for a fintech company:
- 50 million logs/day (~580 logs/sec average, 3000/sec peak)
- Anomaly alerts within 30 seconds
- Analysts query logs with filters (service, error_code, time range)
- 2-year retention
- Handle schema evolution (new fields added over time)

---

## My Initial Design (Attempt)

```
Logs → Load Balancer → Lambda → MongoDB (all logs)
                          ↓
                      RabbitMQ → Lambda → SageMaker → Email alert

Archival: Weekly Glue job moves logs > 2 years to S3
```

**My reasoning:**
- MongoDB handles schema evolution (schemaless)
- RabbitMQ triggers anomaly detection per log
- SageMaker ML model detects anomalies
- Glue archives old data

---

## 6 Mistakes Identified

### Mistake 1: MongoDB for 2 Years = Financial Disaster
```
50M logs/day × 730 days = 36.5 BILLION documents

MongoDB Atlas M50 cluster handles ~100M documents efficiently
36.5B documents would need:
  - 20+ shards
  - ~500GB index RAM
  - Cost: $15,000-30,000/month just for storage

S3 equivalent cost: $800/month for same data
```

### Mistake 2: SageMaker Per Log = $75K/month
```
50M logs/day = 580 calls/sec to SageMaker
  - 50M × $0.05/1000 = $2,500/day = $75,000/month
  
Reality: 95% of logs are normal (status 200, no errors)
Only ~5% need ML analysis

With pre-filtering:
  - 2.5M calls/day × $0.05/1000 = $125/day = $3,750/month
  - 20x cost reduction
```

### Mistake 3: Lambda Chain Too Slow for 30s SLA
```
My chain: Log → Lambda → MongoDB → RabbitMQ → Lambda → SageMaker → Email
Latency:  0  + 300ms  +  50ms  +   20ms   + 300ms  +  100ms   + 200ms
         = ~970ms per log (happy path)
         
But at 3000/sec peak:
  Lambda concurrency limit = 1000
  Incoming 3000/sec > 1000 concurrent = QUEUING
  Queued logs wait 2-5 seconds before processing starts
  Total: 3-6 seconds (OK for 30s, but fragile)
  
Real problem: if SageMaker slows down or Lambda retries,
the queue grows exponentially → alert delay exceeds 30s
```

### Mistake 4: RabbitMQ Loses Messages If Consumer Dies
```
RabbitMQ: message deleted after consumer ACKs
  - If anomaly-detection Lambda crashes AFTER reading but BEFORE processing
  - Message is gone forever
  - That anomaly is never detected

Kafka/Kinesis: consumer tracks offset
  - Lambda crashes → next Lambda starts from same offset
  - Message is re-processed
  - Zero data loss
```

### Mistake 5: Weekly Glue Job = 350M Extra Docs in MongoDB
```
Monday: Glue runs, cleans up
Tuesday-Sunday: 50M × 6 days = 300M docs accumulate
  - Extra index pressure
  - Slower queries all week
  - Costs more than needed

Fix: Daily job (or streaming pipeline with Kinesis Firehose)
```

### Mistake 6: No Query Layer for Analysts
```
Analyst asks: "All 500 errors from payment service last 7 days"
On MongoDB with 36B docs: 
  - Query time: 30 seconds to 5 minutes
  - Analysts know SQL, not MongoDB query syntax
  - No dashboard capability

Need: purpose-built query layer (Athena, OpenSearch)
```

---

## Corrected Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         LOG SOURCES                               │
│    App Servers    │    Microservices    │    AWS Services          │
└────────┬──────────────────┬───────────────────┬─────────────────┘
         │                  │                   │
         ▼                  ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│            KINESIS DATA STREAM (Ingestion Buffer)                 │
│    - 10 shards (handles 10MB/sec = ~50K logs/sec capacity)       │
│    - 7-day retention (replay window for failures)                │
│    - Absorbs 3000/sec peak without dropping                      │
└────────┬──────────────────────────────────┬─────────────────────┘
         │                                  │
         │ Consumer 1                       │ Consumer 2
         ▼                                  ▼
┌────────────────────┐         ┌──────────────────────────────┐
│ Lambda (Storage)   │         │ Lambda (Anomaly Detection)    │
│ - Batch: 100 logs  │         │ - Batch: 50 logs              │
│ - Write to:        │         │ - Stage 1: Rule Engine        │
│   OpenSearch (hot) │         │   • status >= 500?            │
│   S3 via Firehose  │         │   • error_rate > 5%/min?      │
│   (cold)           │         │   • response_time > 3x avg?   │
└────────┬───────────┘         │ - Stage 2: SageMaker          │
         │                     │   (only suspicious ~5%)        │
         ▼                     └──────────────┬────────────────┘
┌────────────────────┐                        │
│ HOT: OpenSearch    │                        ▼
│ - Last 30 days     │         ┌──────────────────────────────┐
│ - Full-text search │         │ ALERT PIPELINE                │
│ - Kibana dashboards│         │                               │
└────────────────────┘         │  Anomaly → SNS → Lambda      │
                               │  (dedup in Redis, 5-min window)│
┌────────────────────┐         │                               │
│ COLD: S3 (Parquet) │         │  LOW:    email digest          │
│ - 30 days → 2 years│         │  MEDIUM: Slack                 │
│ - Partitioned:     │         │  HIGH:   PagerDuty             │
│   /year/month/day/ │         └──────────────────────────────┘
│   /service/        │
│ - Query via Athena │
└────────────────────┘

┌────────────────────┐
│ ARCHIVE: S3 Glacier│
│ - After 2 years    │
│ - S3 Lifecycle Rule│
│   (automatic, no   │
│    code needed)    │
└────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                     ANALYST QUERY LAYER                          │
│                                                                  │
│  Last 30 days → OpenSearch / Kibana (real-time dashboards)       │
│  Historical   → Athena SQL on S3 (any time range, any filter)   │
│  Cross-period → QuickSight (connects to both, unified view)     │
└────────────────────────────────────────────────────────────────┘
```

---

## Anomaly Detection: Two-Stage Pipeline (Detail)

```python
# Stage 1: Rule Engine (runs in Lambda, costs nothing)
def rule_engine(logs_batch):
    suspicious = []
    for log in logs_batch:
        if log['status_code'] >= 500:
            suspicious.append(log)
        elif log['response_time_ms'] > get_baseline(log['service']) * 3:
            suspicious.append(log)
        elif error_rate_last_5min(log['service']) > 0.05:
            suspicious.append(log)
    return suspicious  # typically ~5% of input

# Stage 2: SageMaker (only for suspicious logs)
def ml_anomaly_detection(suspicious_logs):
    features = extract_features(suspicious_logs)
    # Features: service_name, status_code, response_time,
    #           error_rate_5min, request_count_5min, hour_of_day
    response = sagemaker_client.invoke_endpoint(
        EndpointName='anomaly-detector',
        Body=json.dumps(features)
    )
    scores = json.loads(response['Body'].read())
    return [log for log, score in zip(suspicious_logs, scores) 
            if score > 0.85]  # threshold
```

---

## Alert Deduplication (Detail)

```python
import redis
import json

r = redis.Redis()

def deduplicate_and_alert(anomaly):
    key = f"alert:{anomaly['service']}:{anomaly['error_type']}"
    
    current = r.get(key)
    if current:
        data = json.loads(current)
        data['count'] += 1
        data['last_seen'] = anomaly['timestamp']
        r.setex(key, 300, json.dumps(data))  # refresh 5-min TTL
        
        # Escalate if count crosses threshold
        if data['count'] == 20:
            send_pagerduty(data)  # HIGH severity
        # Otherwise: already alerted, skip
    else:
        # First occurrence in this window
        data = {
            'service': anomaly['service'],
            'error_type': anomaly['error_type'],
            'count': 1,
            'first_seen': anomaly['timestamp'],
            'last_seen': anomaly['timestamp']
        }
        r.setex(key, 300, json.dumps(data))  # 5-minute window
        send_slack(data)  # MEDIUM severity for first occurrence

# Result: 500 anomalies in 5 minutes → 2 alerts (first + escalation)
#          instead of 500 emails
```

---

## Schema Evolution Strategy

```
Problem: Log format changes over time
  Week 1: {"service": "auth", "status": 200}
  Week 5: {"service": "auth", "status": 200, "trace_id": "abc123"}
  Week 10: {"service": "auth", "status": 200, "trace_id": "abc123", "region": "us-east-1"}

Solution by storage tier:

OpenSearch (hot):
  - Dynamic mapping: new fields auto-detected
  - No schema changes needed ever
  
S3 + Glue Catalog (cold):
  - Glue Crawler runs daily, detects new columns
  - Athena sees new columns automatically
  - Old data: new columns show as NULL (safe)
  
Key insight: schema evolution is a NON-PROBLEM if you use 
schemaless stores (OpenSearch) + columnar formats (Parquet + Glue Crawler)
```

---

## S3 Lifecycle Policy (Automatic Archival — No Code)

```json
{
  "Rules": [
    {
      "ID": "move-to-infrequent-after-90-days",
      "Status": "Enabled",
      "Transition": {
        "Days": 90,
        "StorageClass": "STANDARD_IA"
      }
    },
    {
      "ID": "move-to-glacier-after-2-years",
      "Status": "Enabled", 
      "Transition": {
        "Days": 730,
        "StorageClass": "GLACIER"
      }
    },
    {
      "ID": "delete-after-7-years",
      "Status": "Enabled",
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```
Set once. Runs forever. No Lambda, no Glue, no code.

---

## Capacity Estimation

```
Logs/sec:     580 average, 3000 peak
Log size:     ~1KB average
Daily volume: 50M × 1KB = 50GB/day
Monthly:      1.5TB
2-year total: 36TB

Kinesis: 10 shards × 1MB/sec = 10MB/sec capacity (handles 10K logs/sec)
         Cost: 10 × $0.015/hr = ~$110/month

OpenSearch: 30 days × 50GB = 1.5TB hot storage
            3 × r6g.xlarge = ~$1,200/month

S3: 36TB × $0.023/GB = ~$830/month
    After Glacier transition: ~$150/month

Athena: $5/TB scanned, ~$50/month for weekly analyst queries

Lambda: ~30M invocations/month (batched) = ~$6/month

SageMaker: 2 × ml.m5.large (5% of traffic) = ~$350/month

Total: ~$2,700/month for production-grade system
```

---

## What an Interviewer Would Ask Next

1. "How do you ensure exactly-once processing?" → Idempotent writes + Kinesis dedup
2. "What if OpenSearch goes down?" → Kinesis retains 7 days, replay when healthy
3. "How do you test the anomaly model?" → Shadow mode: run ML on all logs, alert only on confident scores, compare with manual labels
4. "What about multi-region?" → Active-passive with Kinesis cross-region replication
5. "How do you handle a bad deployment flooding logs?" → Circuit breaker at ingestion + auto-scaling Kinesis shards
