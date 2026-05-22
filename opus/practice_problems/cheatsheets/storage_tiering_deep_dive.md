# Storage Tiering Deep Dive — When to Use What

## The Core Principle

> Data has a lifecycle. Its value and access frequency DECREASE over time.
> Your storage cost should decrease proportionally.

```
Day 1: Log written ──────── queried 100x/day ──── HOT (fast, expensive)
Day 30: Same log ─────────── queried 2x/month ──── WARM (moderate)
Day 365: Same log ────────── queried 0x ─────────── COLD (cheap, slow)
Year 3: Same log ─────────── legal holds it ──────── ARCHIVE (cheapest)
```

---

## AWS Storage Cost Comparison (per GB/month)

```
┌─────────────────────────────────────────────────────────────┐
│ Service                          │ Cost/GB/month │ Latency   │
├──────────────────────────────────┼───────────────┼───────────┤
│ ElastiCache (Redis)              │ $10.00+       │ <1ms      │
│ DynamoDB (provisioned)           │ $0.25         │ <10ms     │
│ RDS / Aurora                     │ $0.10         │ 1-5ms     │
│ OpenSearch                       │ $0.10         │ 10-100ms  │
│ MongoDB Atlas (M50)              │ $0.10         │ 5-20ms    │
│ EBS (gp3)                        │ $0.08         │ <1ms      │
│ S3 Standard                      │ $0.023        │ 50-100ms  │
│ S3 Infrequent Access             │ $0.0125       │ 50-100ms  │
│ S3 Glacier Instant Retrieval     │ $0.004        │ ms        │
│ S3 Glacier Flexible              │ $0.0036       │ 3-5 hours │
│ S3 Glacier Deep Archive          │ $0.00099      │ 12 hours  │
└──────────────────────────────────┴───────────────┴───────────┘

Example: 10TB of data
  All in MongoDB:            $1,000/month
  All in S3 Standard:        $230/month    (4x cheaper)
  S3 Glacier Deep Archive:   $10/month     (100x cheaper)
```

---

## Tiering Strategy by Use Case

### Logs & Events (50M/day)
```
Tier 1 — HOT (last 7-30 days):
  Store: OpenSearch
  Why: Full-text search, real-time dashboards (Kibana)
  Query: "show me all 500 errors from payment service today"
  Latency: <100ms
  Cost: ~$1,200/month for 1.5TB

Tier 2 — WARM (30 days - 2 years):
  Store: S3 (Parquet, partitioned by date/service)
  Query via: Athena (serverless SQL)
  Why: Cheap, queryable, no infra to manage
  Query: "aggregate error rates by service for last 6 months"
  Latency: 2-30 seconds
  Cost: ~$830/month for 36TB

Tier 3 — COLD (>2 years):
  Store: S3 Glacier
  Why: Compliance requirement, rarely accessed
  Latency: 3-5 hours to retrieve
  Cost: ~$130/month for 36TB

Transition: S3 Lifecycle Policy (automatic, set once)
```

### User-Generated Content (Images, Files)
```
Tier 1 — HOT (recently uploaded, frequently viewed):
  Store: S3 Standard + CloudFront CDN
  Why: Fast global delivery
  
Tier 2 — WARM (>90 days, rarely viewed):
  Store: S3 Infrequent Access
  Why: Same latency, 50% cheaper
  Note: Per-retrieval charge ($0.01/1000 requests)
  
Tier 3 — COLD (user deleted account but legal hold):
  Store: S3 Glacier
  Why: Cheapest, meets compliance
```

### ML Training Data
```
Tier 1 — ACTIVE (current model training):
  Store: EBS or FSx for Lustre (attached to training instances)
  Why: High IOPS needed during training
  
Tier 2 — REFERENCE (past datasets, may retrain):
  Store: S3 Standard
  Why: Easily accessible for new experiments
  
Tier 3 — ARCHIVED (old model versions + their data):
  Store: S3 Glacier
  Why: Reproducibility requirement, but rarely needed
```

---

## Partitioning Strategy for S3 + Athena

### Bad Partitioning (Slow Queries)
```
s3://logs/data.parquet              ← one giant file, scans everything
s3://logs/2024-01-15-data.parquet   ← flat, Athena scans all files
```

### Good Partitioning (Fast Queries)
```
s3://logs/year=2024/month=01/day=15/service=payment/data.parquet
s3://logs/year=2024/month=01/day=15/service=auth/data.parquet
s3://logs/year=2024/month=01/day=16/service=payment/data.parquet
```

```sql
-- This query only scans 1 day × 1 service partition
-- Instead of scanning ALL data
SELECT * FROM logs
WHERE year = '2024' AND month = '01' AND day = '15'
  AND service = 'payment'
  AND status_code = 500;

-- Scans: ~50MB (one partition)
-- Cost: $0.00025 (fractions of a cent)
-- Without partitioning: scans 36TB, costs $180 per query!
```

### Choosing Partition Keys
```
Good partition keys:
  - date (year/month/day) → almost every query filters by time
  - service_name → common filter, high cardinality
  - region → if multi-region

Bad partition keys:
  - user_id → too many partitions (millions of tiny files)
  - request_id → unique per record, useless as partition
  - status_code → only 5 values, too few partitions

Rule: Partition on fields that appear in WHERE clause of most queries.
      Each partition should have 100MB-1GB of data (not too small, not too large).
```

---

## File Format Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│ Format    │ Best For            │ Compression │ Query Speed      │
├───────────┼─────────────────────┼─────────────┼──────────────────┤
│ JSON      │ Raw logs, debugging │ Poor        │ Slowest          │
│ CSV       │ Simple exports      │ OK          │ Slow             │
│ Parquet   │ Analytics queries   │ Excellent   │ Fast (columnar)  │
│ ORC       │ Hive/Spark workloads│ Excellent   │ Fast (columnar)  │
│ Avro      │ Schema evolution    │ Good        │ Medium           │
└───────────┴─────────────────────┴─────────────┴──────────────────┘

Always use Parquet for:
  - Athena queries (reads only needed columns)
  - Redshift Spectrum
  - Glue ETL output
  - Any analytical workload

Size comparison (same 1M log records):
  JSON:    ~1.2 GB
  CSV:     ~800 MB
  Parquet: ~150 MB (8x smaller than JSON!)
  
Athena cost = $5/TB scanned
  JSON scan: 1.2GB × $5/1000 = $0.006 per query
  Parquet:   150MB × $5/1000 = $0.00075 per query (8x cheaper!)
```

---

## Automated Tier Transitions

### S3 Lifecycle Policy (Set Once, Forget)
```json
{
  "Rules": [
    {
      "ID": "hot-to-warm",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 365, "StorageClass": "GLACIER_IR"},
        {"Days": 730, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 2555}
    }
  ]
}
```

### DynamoDB TTL (Auto-Delete Old Items)
```python
# Set TTL attribute when writing
item = {
    'pk': 'LOG#12345',
    'timestamp': int(time.time()),
    'ttl': int(time.time()) + (30 * 86400),  # 30 days from now
    'data': {...}
}
table.put_item(Item=item)

# DynamoDB automatically deletes item after TTL expires
# No Lambda, no Glue, no cron — it just happens
```

### OpenSearch Index Lifecycle Management
```json
{
  "policy": {
    "phases": {
      "hot":    {"min_age": "0d",  "actions": {"rollover": {"max_size": "50gb"}}},
      "warm":   {"min_age": "7d",  "actions": {"shrink": {"number_of_shards": 1}}},
      "cold":   {"min_age": "30d", "actions": {"freeze": {}}},
      "delete": {"min_age": "90d", "actions": {"delete": {}}}
    }
  }
}
```

---

## Decision Flowchart for Interview

```
Interviewer: "You need to store X data for Y time..."

Step 1: Calculate total volume
  daily_volume × retention_days = total_data

Step 2: Determine access patterns
  "How often is this queried after day 1? Day 30? Day 365?"

Step 3: Assign tiers
  Frequent → hot database (OpenSearch, DynamoDB, MongoDB)
  Occasional → S3 Standard + Athena
  Rare → S3 Glacier
  
Step 4: Define transitions
  "Data moves from hot to S3 after 30 days via [Glue/Firehose/DynamoDB Streams]"
  "S3 lifecycle policy moves to Glacier after 2 years automatically"

Step 5: Estimate cost
  Sum: hot_db_cost + S3_cost + Glacier_cost = total

Always mention this in interview — it shows cost awareness and senior thinking.
```

---

## Common Interview Question

> "You have 1 billion records. Some are queried 1000x/day, most are never queried. How do you store them?"

**Answer template:**
```
"I'd implement a tiered storage strategy:

1. Identify the hot set — typically 1-5% of data drives 95% of queries
   → Store in DynamoDB/Redis for <10ms latency

2. Warm data — accessed occasionally
   → Store in S3 Standard, query via Athena when needed

3. Cold data — never accessed but retained for compliance
   → S3 Glacier, automated via lifecycle policy

4. Transition mechanism — DynamoDB Streams or daily Glue job
   moves items from hot → S3 based on last_accessed timestamp

This gives us: fast queries on active data, minimal cost on cold data,
and a total storage bill ~10x lower than putting everything in DynamoDB."
```
