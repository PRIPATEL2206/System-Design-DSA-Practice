# AWS & Data Engineering Questions — Interview Ready

Based on your resume: AWS Certified SA-Associate, Glue, Lambda, Redshift, S3, Aurora, PySpark.

---

## Section A: AWS Service Deep-Dive Questions

### Q1: Explain how AWS Glue works internally. How did you optimize Glue jobs?

**Answer:**
```
AWS Glue Architecture:
- Serverless Apache Spark environment
- Data Catalog: Hive-compatible metastore (tables, schemas, partitions)
- Crawlers: Auto-discover schema from S3/JDBC sources
- ETL Jobs: PySpark/Python shell scripts on managed Spark cluster
- DPU (Data Processing Unit): 4 vCPU + 16GB RAM each

Optimizations I applied:
1. Pushdown predicates: Filter at source (Aurora) instead of after loading
   - Before: Load all → filter in Spark = slow + expensive
   - After: Push WHERE clause to JDBC source = 60% less data transferred

2. Job bookmarks: Track processed data, avoid reprocessing
   - Saves last processed timestamp/partition
   - On restart, only processes new data

3. Partition pruning: Only read relevant S3 partitions
   - Data partitioned by: year/month/day/region
   - Query for one region reads 1/50th of the data

4. DPU right-sizing: 
   - Started with 10 DPUs (default), profiled actual usage
   - Most jobs only needed 5-6 DPUs (35% cost reduction)
   - Used Glue metrics (CPU, memory utilization) to determine

5. Columnar format: Converted CSV → Parquet in landing zone
   - 10x compression, columnar reads for specific columns
   - Enabled predicate pushdown at file level (row groups)
```

---

### Q2: Lambda cold starts — how do you mitigate them in your pipeline?

**Answer:**
```
Cold Start causes:
- New container initialization (download code, init runtime, establish connections)
- Worse for VPC-attached Lambdas (ENI creation adds 5-10s)

Mitigation strategies:
1. Provisioned Concurrency: Pre-warm N instances (used for critical paths)
2. Smaller package size: Only include needed dependencies (layers for shared)
3. Connection reuse: Initialize DB connections outside handler
4. Avoid VPC when possible: Use VPC endpoints for S3/DynamoDB
5. SnapStart (Java) or keep-alive pings for critical functions

In my pipeline:
- ETL triggers (S3 → Lambda) tolerate cold starts (async, not user-facing)
- API-facing Lambdas use provisioned concurrency during business hours
- Used Lambda Layers for shared libraries (PySpark utils, validators)
```

---

### Q3: Redshift — How did you design your schema for 125M records?

**Answer:**
```
Design decisions:

1. Distribution Style:
   - DISTSTYLE KEY on hcp_id (most joins are on this key)
   - Ensures joined data is co-located on same node (no network shuffle)
   - Small dimension tables: DISTSTYLE ALL (replicated to all nodes)

2. Sort Keys:
   - Compound sort key: (date, region)
   - Most queries filter by date range + region
   - Enables zone map elimination (skip entire blocks)

3. Compression:
   - ENCODE AUTO on most columns
   - Manual encoding for known patterns (run-length for status columns)
   - ANALYZE COMPRESSION to find optimal encodings

4. Table Design:
   - Fact table: hcp_interactions (date, hcp_id, interaction_type, value)
   - Dimension tables: hcp_master, region, product
   - Star schema for analytics workload

5. Maintenance:
   - VACUUM FULL weekly (reclaim space from deletes)
   - ANALYZE after large loads (update statistics)
   - WLM queues: Separate for ETL loads vs analyst queries

Performance results:
- Dashboard queries: <5s for 90th percentile
- Full table scan (125M rows): ~45s with proper sort keys
- Without sort keys: ~180s (4x slower)
```

---

### Q4: S3 — How do you organize data in your data lake?

**Answer:**
```
Folder structure (zone-based):

s3://data-lake-bucket/
├── raw/                          # Landing zone (immutable)
│   ├── hcp_records/
│   │   └── year=2025/month=05/day=19/
│   │       ├── file_001.parquet
│   │       └── _SUCCESS
│   └── partner_feeds/
│       └── source=pharma_co/date=2025-05-19/
├── curated/                      # Cleaned, deduplicated
│   ├── hcp_master/
│   │   └── year=2025/month=05/
│   └── interactions/
│       └── region=north/year=2025/
├── enriched/                     # Aggregated, ready for analytics
│   ├── daily_metrics/
│   └── ml_features/
└── archive/                      # Glacier, rarely accessed

Key decisions:
- Hive-style partitioning (key=value/) for Athena/Glue compatibility
- Parquet format (columnar, compressed, splittable)
- Partition by most common query filter (date + region)
- File size: Target 128MB-1GB per file (optimal for Spark/Redshift COPY)
- Small files problem: Use Glue job to compact small files daily

Lifecycle policies:
- raw/: Standard → IA after 30 days → Glacier after 90 days
- curated/: Standard (frequently accessed)
- enriched/: Standard (active analytics)
- archive/: Glacier Deep Archive
```

---

### Q5: How does your event-driven architecture work (S3 → Lambda → Processing)?

**Answer:**
```
Architecture:

S3 (file upload) → EventBridge/S3 Event → Lambda (validator) → SQS → Glue Job
                                                |
                                          [DLQ if invalid]

Flow:
1. Partner uploads file to s3://raw/partner_feeds/
2. S3 Event Notification triggers Lambda
3. Lambda validates: file size, format, schema, naming convention
4. If valid: sends message to SQS queue with file metadata
5. If invalid: sends to DLQ + alerts via SNS
6. Glue Job polls SQS (or triggered by CloudWatch Events on schedule)
7. Processes batch of files from queue
8. On completion: Step Function moves to next stage

Why SQS in between:
- Decouples ingestion rate from processing rate
- Natural backpressure (Glue processes at its own pace)
- Retry built-in (visibility timeout + redrive policy)
- Batching: Collect 10 files then process together (fewer Glue invocations)

Monitoring:
- CloudWatch metrics: files received, processed, failed per hour
- SQS queue depth alarm: If > 100 messages, alert (pipeline backing up)
- Lambda error rate alarm: If > 5% errors, investigate
```

---

## Section B: PySpark Specific Questions

### Q6: How do you optimize PySpark jobs for large datasets?

**Answer:**
```python
# 1. Avoid shuffles where possible
# BAD: groupByKey (shuffles all data)
rdd.groupByKey().mapValues(sum)
# GOOD: reduceByKey (partial aggregation before shuffle)
rdd.reduceByKey(lambda a, b: a + b)

# 2. Broadcast small tables for joins
from pyspark.sql.functions import broadcast
# BAD: Regular join (shuffles both sides)
large_df.join(small_df, "key")
# GOOD: Broadcast join (replicates small table to all nodes)
large_df.join(broadcast(small_df), "key")

# 3. Partition wisely
# Repartition by join key before multiple joins
df = df.repartition(200, "hcp_id")  # Colocate same keys

# 4. Cache intermediate results used multiple times
filtered_df = raw_df.filter(col("status") == "active")
filtered_df.cache()  # Stored in memory for reuse
result1 = filtered_df.groupBy("region").count()
result2 = filtered_df.groupBy("category").sum("value")
filtered_df.unpersist()

# 5. Use appropriate file formats
# Write as Parquet with partitioning
df.write.partitionBy("year", "month").parquet("s3://curated/output/")

# 6. Handle skew
# If one key has 10M rows and others have 1K:
# Salt the key to distribute load
from pyspark.sql.functions import concat, lit, rand
df_salted = df.withColumn("salted_key", concat(col("key"), lit("_"), (rand() * 10).cast("int")))

# 7. Predicate pushdown with JDBC
df = spark.read.jdbc(
    url=aurora_url,
    table="(SELECT * FROM hcp WHERE region = 'NORTH') AS subquery",
    properties={"driver": "org.postgresql.Driver"}
)
```

---

### Q7: Explain PySpark execution model (asked frequently).

**Answer:**
```
Execution hierarchy:
Application → Jobs → Stages → Tasks

1. Application: Your Glue/Spark script
2. Job: Triggered by each ACTION (collect, save, count)
3. Stage: Separated by SHUFFLE boundaries (groupBy, join, repartition)
4. Task: One task per partition per stage (parallelism unit)

Key concepts:
- Lazy evaluation: Transformations are just plans until an action triggers execution
- DAG (Directed Acyclic Graph): Optimizer builds execution plan
- Catalyst optimizer: Rewrites logical plan → physical plan
- Tungsten: Memory management, binary encoding, code generation

Memory model:
- Execution memory: Shuffles, joins, sorts, aggregations
- Storage memory: Cached DataFrames/RDDs
- Unified memory pool: Shared between execution and storage (dynamic)

Common issues:
- OOM: Skewed partitions → one task gets too much data
- Slow stages: Shuffle-heavy operations (wide transformations)
- Small files: Too many partitions → overhead > computation
- Straggler tasks: Data skew → one task takes 100x longer
```

---

## Section C: Data Engineering Concepts

### Q8: Explain CDC (Change Data Capture) and how you'd implement it.

**Answer:**
```
CDC = Capture row-level changes (INSERT, UPDATE, DELETE) from source database

Methods:
1. Timestamp-based: Query WHERE updated_at > last_run
   - Simple but misses deletes
   - Requires timestamp column on every table

2. Log-based (recommended): Read database transaction log
   - Aurora → DMS → Kinesis/S3
   - Captures all changes including deletes
   - No impact on source DB performance
   - Maintains ordering

3. Trigger-based: DB triggers write changes to audit table
   - High overhead on source DB
   - Not recommended for high-volume

AWS Implementation:
Aurora PostgreSQL → DMS (CDC mode) → Kinesis Data Stream → Lambda → S3/Redshift

DMS CDC settings:
- Full load first (initial snapshot)
- Then continuous replication (ongoing changes)
- Change tables in target: includes operation type (I/U/D)

Handling in ETL:
- MERGE/UPSERT logic in Redshift (INSERT on new, UPDATE on existing)
- Maintain SCD Type 2 for slowly changing dimensions
- Tombstone records for deletes (soft delete with flag)
```

---

### Q9: What is data skew and how do you handle it?

**Answer:**
```
Data Skew: Uneven distribution of data across partitions
- Result: One task processes 10GB while others process 100MB
- Symptom: 199/200 tasks complete in 1 min, last task takes 30 min

Causes:
- Null keys (all nulls go to one partition)
- Hot keys (one customer_id has 10M records)
- Default partitioning on skewed column

Solutions:
1. Salting (add random suffix to hot key):
   hot_key → hot_key_0, hot_key_1, ... hot_key_9
   Join: Explode the small side to match all salted keys

2. Broadcast join (if one side is small):
   Replicate small table, avoid shuffle entirely

3. Adaptive Query Execution (Spark 3.0+):
   spark.conf.set("spark.sql.adaptive.enabled", "true")
   spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
   Automatically detects and splits skewed partitions

4. Handle nulls separately:
   Filter nulls → process separately → union back

5. Custom partitioning:
   Repartition by composite key or different column

6. Two-phase aggregation:
   Phase 1: Partial aggregation within partitions (no shuffle)
   Phase 2: Final aggregation (smaller shuffle)
```

---

### Q10: Explain exactly-once processing in your pipeline.

**Answer:**
```
Challenge: Network failures, retries, crashes can cause duplicates or data loss

Three semantics:
- At-most-once: Fire and forget (may lose data)
- At-least-once: Retry until success (may duplicate)
- Exactly-once: Each record processed precisely once (hardest)

How I achieve exactly-once in batch ETL:

1. Idempotent writes:
   - Use MERGE/UPSERT instead of INSERT
   - Deduplication key: hash(source_id + record_id + version)
   - Same input → same output regardless of reruns

2. Job bookmarks (AWS Glue):
   - Track last processed file/timestamp
   - On restart, resume from bookmark (no reprocessing)
   - Bookmark committed AFTER successful write (not before)

3. Transactional writes:
   - Write to staging table first
   - Validate row counts match expected
   - SWAP staging → production in single atomic operation
   - If failure: staging is discarded, production unchanged

4. Checkpointing:
   - For streaming: Kafka offsets committed after processing
   - For batch: Manifest files list processed inputs

5. Deduplication layer:
   - Window function: ROW_NUMBER() PARTITION BY pk ORDER BY updated_at DESC
   - Keep only rank = 1 (latest version of each record)
```

---

## Section D: Quick-Fire Technical Questions

| Question | Key Points |
|----------|------------|
| Parquet vs CSV vs JSON? | Parquet: columnar, compressed, schema-embedded, best for analytics. CSV: row-based, human-readable, slow. JSON: semi-structured, flexible, large size. |
| OLTP vs OLAP? | OLTP: transactional, row-oriented, normalized (Aurora). OLAP: analytical, columnar, denormalized (Redshift). |
| Star vs Snowflake schema? | Star: fact + denormalized dimensions (faster queries, more storage). Snowflake: normalized dimensions (less storage, more joins). |
| What is a data mesh? | Decentralized data ownership. Domain teams own their data products. Central governance, distributed execution. |
| Batch vs Stream processing? | Batch: high throughput, higher latency (Glue/EMR). Stream: low latency, complex state management (Kinesis/Flink). |
| SCD Type 1 vs 2 vs 3? | Type1: overwrite (no history). Type2: new row with effective dates (full history). Type3: add previous_value column (limited history). |
| What is backpressure? | Consumer can't keep up with producer. Solutions: buffer (queue), drop, throttle producer. |
| Redshift vs Athena? | Redshift: provisioned, fast dashboards, complex joins. Athena: serverless, ad-hoc queries, pay per scan. |
| What is a materialized view? | Pre-computed query result stored physically. Faster reads, stale until refreshed. Good for dashboards. |
| Lambda vs Step Functions? | Lambda: single function (15 min max). Step Functions: orchestrate multiple Lambdas/services with state machine. |

---

## Section E: Scenario-Based Questions

### "A Glue job that normally takes 30 minutes suddenly takes 3 hours. How do you debug?"

```
Investigation steps:
1. Check CloudWatch metrics: DPU utilization, data volume processed
2. Check source data: Did input size spike? New partition? Schema change?
3. Check for data skew: One partition much larger than others?
4. Check for small files problem: Too many tiny files = high overhead
5. Check Spark UI: Which stage is slow? Shuffle spill to disk?
6. Check network: VPC/endpoint throttling? S3 throttling (503s)?
7. Check concurrent jobs: Resource contention on shared cluster?

Common root causes:
- Data volume spike (partner sent 10x normal data)
- Schema drift (new column caused type casting issues)
- Small files (1000 tiny files vs 10 normal files)
- Upstream table lock (JDBC read waiting on Aurora)
- Memory spill to disk (insufficient DPUs for data volume)
```

### "Data in Redshift doesn't match source Aurora. How do you investigate?"

```
Approach:
1. Row count comparison: source vs target
2. Checksum comparison: hash(key columns) aggregated
3. Sample random records and compare field-by-field
4. Check ETL logs: any skipped/failed records?
5. Check for timing issues: source updated after ETL ran?
6. Check deduplication logic: are we dropping valid records?
7. Check data types: truncation during load? (VARCHAR length)

Prevention:
- Automated reconciliation job after each load
- Alert when source_count != target_count (±1% tolerance)
- Audit table: log every load with counts and checksums
```
