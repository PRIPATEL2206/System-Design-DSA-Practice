# SQL & PySpark Practice Problems

> These are the types of SQL/PySpark questions asked in implementation rounds for Data/ML Engineer roles.

---

## SQL Problems

---

### SQL-1: Find Top 3 Products Per Region by Revenue (Window Functions)

> Given a sales table, find the top 3 products by revenue in each region.

```sql
-- Schema:
-- sales(sale_id, product_id, region, amount, sale_date)

-- Try it yourself first!
```

<details>
<summary>Solution</summary>

```sql
WITH ranked_products AS (
    SELECT 
        region,
        product_id,
        SUM(amount) AS total_revenue,
        RANK() OVER (PARTITION BY region ORDER BY SUM(amount) DESC) AS rank
    FROM sales
    GROUP BY region, product_id
)
SELECT region, product_id, total_revenue, rank
FROM ranked_products
WHERE rank <= 3
ORDER BY region, rank;

-- Alternative with DENSE_RANK (includes ties)
-- Use ROW_NUMBER if you want exactly 3 rows per region (no ties)
```

**Interview tip:** Know the difference between RANK, DENSE_RANK, and ROW_NUMBER:
- `RANK`: 1, 2, 2, 4 (skips after tie)
- `DENSE_RANK`: 1, 2, 2, 3 (no skip)
- `ROW_NUMBER`: 1, 2, 3, 4 (no ties)
</details>

---

### SQL-2: Find Users Who Were Active on 3+ Consecutive Days

> From a login table, find users who logged in on at least 3 consecutive days.

```sql
-- Schema:
-- user_logins(user_id, login_date)
-- Note: A user may login multiple times in a day

-- Try it yourself first!
```

<details>
<summary>Solution</summary>

```sql
-- Approach: Use the "island" technique
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date
    FROM user_logins
),
grouped AS (
    SELECT 
        user_id,
        login_date,
        login_date - INTERVAL '1 day' * ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) AS grp
    FROM distinct_logins
)
SELECT DISTINCT user_id
FROM grouped
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;

-- How it works:
-- If dates are consecutive, subtracting row_number gives the same "group" value
-- Example: dates 2025-05-17, 2025-05-18, 2025-05-19
--   row_num: 1, 2, 3
--   date - row_num: 2025-05-16, 2025-05-16, 2025-05-16 → same group!
```

**Interview tip:** The "date - row_number" trick is a classic pattern for finding consecutive sequences. Works for dates, numbers, etc.
</details>

---

### SQL-3: Running Total with Reset (Complex Window Function)

> Calculate a running total of expenses per department, but reset to 0 when a "budget_reset" event occurs.

```sql
-- Schema:
-- expenses(id, department, amount, expense_date, is_budget_reset BOOLEAN)

-- Try it yourself first!
```

<details>
<summary>Solution</summary>

```sql
WITH reset_groups AS (
    SELECT 
        *,
        SUM(CASE WHEN is_budget_reset THEN 1 ELSE 0 END) OVER (
            PARTITION BY department 
            ORDER BY expense_date
            ROWS UNBOUNDED PRECEDING
        ) AS reset_group
    FROM expenses
)
SELECT 
    department,
    expense_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY department, reset_group
        ORDER BY expense_date
        ROWS UNBOUNDED PRECEDING
    ) AS running_total
FROM reset_groups
ORDER BY department, expense_date;
```

**Interview tip:** The key insight is creating "groups" between resets using a cumulative sum of the reset flag, then computing the running total within each group.
</details>

---

### SQL-4: Detect Data Quality Issues (Practical ETL Problem)

> Write queries to detect common data quality problems in HCP records.

```sql
-- Schema:
-- hcp_records(hcp_id, name, dob, region, specialty, last_updated, source_system)
```

<details>
<summary>Solution</summary>

```sql
-- 1. Find duplicates (same person, different IDs)
SELECT name, dob, region, COUNT(*) AS duplicate_count
FROM hcp_records
GROUP BY name, dob, region
HAVING COUNT(*) > 1;

-- 2. Find records with suspicious data patterns
SELECT hcp_id, name, dob,
    CASE 
        WHEN dob > CURRENT_DATE THEN 'future_dob'
        WHEN dob < '1900-01-01' THEN 'too_old'
        WHEN name ~ '^\d' THEN 'name_starts_with_number'
        WHEN LENGTH(name) < 2 THEN 'name_too_short'
        WHEN region IS NULL THEN 'missing_region'
        WHEN last_updated > CURRENT_TIMESTAMP THEN 'future_timestamp'
    END AS issue_type
FROM hcp_records
WHERE dob > CURRENT_DATE
   OR dob < '1900-01-01'
   OR name ~ '^\d'
   OR LENGTH(name) < 2
   OR region IS NULL
   OR last_updated > CURRENT_TIMESTAMP;

-- 3. Find stale records (not updated in 90 days from active source)
SELECT hcp_id, name, last_updated, source_system,
       CURRENT_DATE - last_updated::date AS days_stale
FROM hcp_records
WHERE last_updated < CURRENT_DATE - INTERVAL '90 days'
  AND source_system IN ('active_feed_1', 'active_feed_2')
ORDER BY days_stale DESC;

-- 4. Source system reconciliation
SELECT 
    source_system,
    COUNT(*) AS total_records,
    COUNT(*) FILTER (WHERE last_updated >= CURRENT_DATE) AS today_updates,
    COUNT(*) FILTER (WHERE region IS NULL) AS null_region_count,
    ROUND(100.0 * COUNT(*) FILTER (WHERE region IS NULL) / COUNT(*), 2) AS null_pct
FROM hcp_records
GROUP BY source_system
ORDER BY null_pct DESC;

-- 5. Detect sudden volume changes (anomaly detection)
WITH daily_counts AS (
    SELECT 
        last_updated::date AS load_date,
        COUNT(*) AS record_count
    FROM hcp_records
    GROUP BY last_updated::date
),
stats AS (
    SELECT 
        load_date,
        record_count,
        AVG(record_count) OVER (ORDER BY load_date ROWS 7 PRECEDING) AS avg_7day,
        STDDEV(record_count) OVER (ORDER BY load_date ROWS 7 PRECEDING) AS std_7day
    FROM daily_counts
)
SELECT load_date, record_count, avg_7day,
       ROUND((record_count - avg_7day) / NULLIF(std_7day, 0), 2) AS z_score
FROM stats
WHERE ABS((record_count - avg_7day) / NULLIF(std_7day, 0)) > 2
ORDER BY load_date DESC;
```

**Interview tip:** These are the exact types of quality checks you should mention when discussing your ETL pipeline. Shows you think about data quality proactively.
</details>

---

### SQL-5: Slowly Changing Dimensions (Type 2)

> Implement SCD Type 2 merge logic: maintain full history of HCP record changes.

```sql
-- Schema:
-- hcp_dim(hcp_key SERIAL, hcp_id, name, specialty, region, 
--         effective_from DATE, effective_to DATE, is_current BOOLEAN)
-- hcp_staging(hcp_id, name, specialty, region)  -- new incoming data
```

<details>
<summary>Solution</summary>

```sql
-- Step 1: Expire changed records
UPDATE hcp_dim d
SET 
    effective_to = CURRENT_DATE - 1,
    is_current = FALSE
FROM hcp_staging s
WHERE d.hcp_id = s.hcp_id
  AND d.is_current = TRUE
  AND (d.name != s.name OR d.specialty != s.specialty OR d.region != s.region);

-- Step 2: Insert new versions of changed records
INSERT INTO hcp_dim (hcp_id, name, specialty, region, effective_from, effective_to, is_current)
SELECT 
    s.hcp_id, s.name, s.specialty, s.region,
    CURRENT_DATE,
    '9999-12-31',
    TRUE
FROM hcp_staging s
JOIN hcp_dim d ON s.hcp_id = d.hcp_id
WHERE d.effective_to = CURRENT_DATE - 1  -- just expired
  AND d.is_current = FALSE;

-- Step 3: Insert truly new records
INSERT INTO hcp_dim (hcp_id, name, specialty, region, effective_from, effective_to, is_current)
SELECT 
    s.hcp_id, s.name, s.specialty, s.region,
    CURRENT_DATE,
    '9999-12-31',
    TRUE
FROM hcp_staging s
WHERE NOT EXISTS (
    SELECT 1 FROM hcp_dim d WHERE d.hcp_id = s.hcp_id
);

-- Query: "What was the specialty of HCP-001 on 2025-03-15?"
SELECT * FROM hcp_dim
WHERE hcp_id = 'HCP-001'
  AND '2025-03-15' BETWEEN effective_from AND effective_to;
```

**Interview tip:** SCD Type 2 is fundamental for data warehouses. Know when to use Type 1 (overwrite) vs Type 2 (history) vs Type 3 (previous column).
</details>

---

### SQL-6: Funnel Analysis (Common in Product/Analytics Roles)

> Calculate conversion rates through a multi-step funnel: Visit → SignUp → FirstPurchase → RepeatPurchase

```sql
-- Schema:
-- events(user_id, event_type, event_date)
-- event_type IN ('visit', 'signup', 'first_purchase', 'repeat_purchase')
```

<details>
<summary>Solution</summary>

```sql
WITH funnel AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event_type = 'visit' THEN user_id END) AS visits,
        COUNT(DISTINCT CASE WHEN event_type = 'signup' THEN user_id END) AS signups,
        COUNT(DISTINCT CASE WHEN event_type = 'first_purchase' THEN user_id END) AS first_purchases,
        COUNT(DISTINCT CASE WHEN event_type = 'repeat_purchase' THEN user_id END) AS repeat_purchases
    FROM events
    WHERE event_date BETWEEN '2025-01-01' AND '2025-03-31'
)
SELECT 
    visits,
    signups,
    ROUND(100.0 * signups / NULLIF(visits, 0), 2) AS visit_to_signup_pct,
    first_purchases,
    ROUND(100.0 * first_purchases / NULLIF(signups, 0), 2) AS signup_to_purchase_pct,
    repeat_purchases,
    ROUND(100.0 * repeat_purchases / NULLIF(first_purchases, 0), 2) AS repeat_rate_pct
FROM funnel;

-- Time-aware funnel (user must do steps in order)
WITH ordered_events AS (
    SELECT 
        user_id,
        event_type,
        event_date,
        LEAD(event_type) OVER (PARTITION BY user_id ORDER BY event_date) AS next_event,
        LEAD(event_date) OVER (PARTITION BY user_id ORDER BY event_date) AS next_date
    FROM events
)
SELECT 
    event_type AS step,
    next_event AS next_step,
    COUNT(*) AS transitions,
    AVG(next_date - event_date) AS avg_days_between
FROM ordered_events
WHERE next_event IS NOT NULL
GROUP BY event_type, next_event
ORDER BY transitions DESC;
```
</details>

---

## PySpark Problems

---

### PySpark-1: Deduplicate and Merge Records from Multiple Sources

> You receive HCP data from 3 sources. Merge them into a single golden record, preferring the most recent update.

```python
# Try it yourself first!
# Input: 3 DataFrames with overlapping records
# Output: 1 DataFrame with best version of each record
```

<details>
<summary>Solution</summary>

```python
from pyspark.sql import SparkSession, Window
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("HCP_Dedup").getOrCreate()

# Simulate 3 sources
source1 = spark.createDataFrame([
    ("HCP001", "Dr. Patel", "Cardiology", "NORTH", "2025-05-01", "source_a"),
    ("HCP002", "Dr. Shah", "Neurology", "SOUTH", "2025-04-15", "source_a"),
], ["hcp_id", "name", "specialty", "region", "last_updated", "source"])

source2 = spark.createDataFrame([
    ("HCP001", "Dr. Prince Patel", "Cardiology", "NORTH", "2025-05-10", "source_b"),
    ("HCP003", "Dr. Kumar", "Orthopedics", "WEST", "2025-05-05", "source_b"),
], ["hcp_id", "name", "specialty", "region", "last_updated", "source"])

source3 = spark.createDataFrame([
    ("HCP001", "Dr. P. Patel", "Interventional Cardiology", "NORTH", "2025-05-15", "source_c"),
    ("HCP002", "Dr. Shah", "Neurology", "SOUTH", "2025-05-12", "source_c"),
], ["hcp_id", "name", "specialty", "region", "last_updated", "source"])

# Union all sources
all_records = source1.union(source2).union(source3)

# Deduplicate: Keep most recent record per hcp_id
window = Window.partitionBy("hcp_id").orderBy(col("last_updated").desc())

golden_records = (
    all_records
    .withColumn("rank", row_number().over(window))
    .filter(col("rank") == 1)
    .drop("rank")
)

golden_records.show()
# HCP001 → "Dr. P. Patel" from source_c (most recent: 2025-05-15)
# HCP002 → "Dr. Shah" from source_c (most recent: 2025-05-12)
# HCP003 → "Dr. Kumar" from source_b (only source)

# Advanced: Merge fields from different sources (take best non-null)
def merge_golden_record(df):
    """Take the best non-null value for each field, prioritized by recency."""
    window_per_field = Window.partitionBy("hcp_id").orderBy(col("last_updated").desc())
    
    # For each field, get first non-null value ordered by recency
    return (
        df
        .withColumn("name_rank", row_number().over(
            Window.partitionBy("hcp_id")
            .orderBy(when(col("name").isNotNull(), col("last_updated")).desc())
        ))
        .withColumn("specialty_rank", row_number().over(
            Window.partitionBy("hcp_id")
            .orderBy(when(col("specialty").isNotNull(), col("last_updated")).desc())
        ))
        .groupBy("hcp_id")
        .agg(
            first("name", ignorenulls=True).alias("name"),
            first("specialty", ignorenulls=True).alias("specialty"),
            first("region", ignorenulls=True).alias("region"),
            max("last_updated").alias("last_updated"),
            collect_set("source").alias("contributing_sources")
        )
    )
```

**Interview tip:** This is a real ETL problem. Discuss: What if sources conflict? (Priority order, recency, data quality score). What about null handling? (Don't overwrite good data with null).
</details>

---

### PySpark-2: Detect Anomalies in Time-Series Data

> Find days where record counts deviate significantly from the 7-day rolling average.

```python
# Try it yourself first!
```

<details>
<summary>Solution</summary>

```python
from pyspark.sql import Window
from pyspark.sql.functions import *

# Daily record counts
daily_counts = spark.createDataFrame([
    ("2025-05-01", 1000000), ("2025-05-02", 980000), ("2025-05-03", 1050000),
    ("2025-05-04", 1020000), ("2025-05-05", 990000), ("2025-05-06", 1010000),
    ("2025-05-07", 1030000), ("2025-05-08", 500000),  # Anomaly!
    ("2025-05-09", 1100000), ("2025-05-10", 3000000),  # Anomaly!
], ["date", "record_count"])

daily_counts = daily_counts.withColumn("date", to_date("date"))

# Calculate rolling statistics
window_7day = Window.orderBy("date").rowsBetween(-7, -1)

anomalies = (
    daily_counts
    .withColumn("rolling_avg", avg("record_count").over(window_7day))
    .withColumn("rolling_std", stddev("record_count").over(window_7day))
    .withColumn("z_score", 
        (col("record_count") - col("rolling_avg")) / col("rolling_std"))
    .withColumn("is_anomaly", abs(col("z_score")) > 2)
    .withColumn("anomaly_type",
        when(col("z_score") > 2, "SPIKE")
        .when(col("z_score") < -2, "DROP")
        .otherwise("NORMAL"))
)

anomalies.filter(col("is_anomaly")).show()

# More sophisticated: Use IQR method (robust to outliers)
window_all = Window.orderBy("date").rowsBetween(-14, -1)

iqr_anomalies = (
    daily_counts
    .withColumn("q1", expr("percentile_approx(record_count, 0.25)").over(window_all))
    .withColumn("q3", expr("percentile_approx(record_count, 0.75)").over(window_all))
    .withColumn("iqr", col("q3") - col("q1"))
    .withColumn("lower_bound", col("q1") - 1.5 * col("iqr"))
    .withColumn("upper_bound", col("q3") + 1.5 * col("iqr"))
    .withColumn("is_anomaly", 
        (col("record_count") < col("lower_bound")) | 
        (col("record_count") > col("upper_bound")))
)
```
</details>

---

### PySpark-3: Implement SCD Type 2 in PySpark

> Merge incoming records into a dimension table maintaining full history.

```python
# Try it yourself first!
```

<details>
<summary>Solution</summary>

```python
from pyspark.sql.functions import *
from pyspark.sql.types import *

# Current dimension table
dim_df = spark.createDataFrame([
    (1, "HCP001", "Dr. Patel", "Cardiology", "2025-01-01", "9999-12-31", True),
    (2, "HCP002", "Dr. Shah", "Neurology", "2025-01-01", "9999-12-31", True),
    (3, "HCP003", "Dr. Kumar", "Orthopedics", "2025-02-01", "9999-12-31", True),
], ["surrogate_key", "hcp_id", "name", "specialty", "effective_from", "effective_to", "is_current"])

# New incoming records
staging_df = spark.createDataFrame([
    ("HCP001", "Dr. Prince Patel", "Interventional Cardiology"),  # Changed
    ("HCP002", "Dr. Shah", "Neurology"),                           # No change
    ("HCP004", "Dr. Verma", "Dermatology"),                       # New
], ["hcp_id", "name", "specialty"])

today = "2025-05-19"

# Step 1: Identify changed records
current_dim = dim_df.filter(col("is_current") == True)
changes = (
    staging_df.alias("s")
    .join(current_dim.alias("d"), "hcp_id", "left")
    .withColumn("change_type",
        when(col("d.hcp_id").isNull(), "NEW")
        .when(
            (col("s.name") != col("d.name")) | 
            (col("s.specialty") != col("d.specialty")),
            "CHANGED"
        )
        .otherwise("UNCHANGED")
    )
)

# Step 2: Expire old records for changed entries
changed_ids = changes.filter(col("change_type") == "CHANGED").select("hcp_id")
expired_records = (
    dim_df
    .join(changed_ids, "hcp_id", "inner")
    .filter(col("is_current") == True)
    .withColumn("effective_to", lit(today))
    .withColumn("is_current", lit(False))
)

# Step 3: Create new versions for changed records
new_versions = (
    changes
    .filter(col("change_type") == "CHANGED")
    .select(
        lit(None).alias("surrogate_key"),  # Auto-generate
        col("s.hcp_id"),
        col("s.name"),
        col("s.specialty"),
        lit(today).alias("effective_from"),
        lit("9999-12-31").alias("effective_to"),
        lit(True).alias("is_current")
    )
)

# Step 4: Insert truly new records
new_records = (
    changes
    .filter(col("change_type") == "NEW")
    .select(
        lit(None).alias("surrogate_key"),
        col("s.hcp_id"),
        col("s.name"),
        col("s.specialty"),
        lit(today).alias("effective_from"),
        lit("9999-12-31").alias("effective_to"),
        lit(True).alias("is_current")
    )
)

# Step 5: Combine all
unchanged_records = dim_df.join(changed_ids, "hcp_id", "left_anti")  # Records not in changed_ids
final_dim = (
    unchanged_records
    .union(expired_records)
    .union(new_versions)
    .union(new_records)
)

final_dim.orderBy("hcp_id", "effective_from").show(truncate=False)
```
</details>

---

### PySpark-4: Optimize a Skewed Join

> Table A has 100M rows. Table B has 50M rows. 80% of records in A have region="NORTH". Join them on region + date.

```python
# Try it yourself first!
# The naive join will be extremely slow due to skew on "NORTH"
```

<details>
<summary>Solution</summary>

```python
from pyspark.sql.functions import *

# Problem: NORTH partition is 80M rows → single task overwhelmed

# Solution 1: Salting (best general approach)
SALT_BUCKETS = 10

# Salt the large table
table_a_salted = (
    table_a
    .withColumn("salt", (rand() * SALT_BUCKETS).cast("int"))
    .withColumn("join_key", concat(col("region"), lit("_"), col("salt")))
)

# Explode the small table to match all salt values
table_b_exploded = (
    table_b
    .crossJoin(spark.range(SALT_BUCKETS).withColumnRenamed("id", "salt"))
    .withColumn("join_key", concat(col("region"), lit("_"), col("salt")))
)

# Join on salted key (distributes NORTH across 10 partitions)
result = table_a_salted.join(table_b_exploded, ["join_key", "date"])
result = result.drop("salt", "join_key")

# Solution 2: Broadcast join (if table_b fits in memory < 8GB)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "500m")
result = table_a.join(broadcast(table_b), ["region", "date"])

# Solution 3: Adaptive Query Execution (Spark 3.0+)
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256m")
# Spark automatically detects and splits skewed partitions

# Solution 4: Isolate hot key
north_a = table_a.filter(col("region") == "NORTH")
other_a = table_a.filter(col("region") != "NORTH")

north_b = table_b.filter(col("region") == "NORTH")
other_b = table_b.filter(col("region") != "NORTH")

# Broadcast the smaller NORTH partition of B
result_north = north_a.join(broadcast(north_b), ["region", "date"])
result_other = other_a.join(other_b, ["region", "date"])

result = result_north.union(result_other)
```

**Interview tip:** Always mention: 1) Identify the skew (Spark UI → stage with straggler tasks), 2) Choose strategy based on data characteristics. Salting is the universal fix; broadcast is simpler if data fits.
</details>

---

### PySpark-5: Build a Feature Engineering Pipeline

> Create a feature engineering pipeline that computes features for an ML model from raw HCP interaction data.

```python
# Try it yourself first!
# Input: Raw interactions (hcp_id, interaction_type, value, date)
# Output: Feature vector per HCP for model training
```

<details>
<summary>Solution</summary>

```python
from pyspark.sql.functions import *
from pyspark.sql import Window

# Raw data
interactions = spark.createDataFrame([
    ("HCP001", "prescription", 500, "2025-05-01"),
    ("HCP001", "visit", 1, "2025-05-03"),
    ("HCP001", "prescription", 300, "2025-05-10"),
    ("HCP001", "conference", 1, "2025-04-15"),
    ("HCP002", "prescription", 200, "2025-05-02"),
    ("HCP002", "visit", 1, "2025-05-05"),
], ["hcp_id", "interaction_type", "value", "interaction_date"])

interactions = interactions.withColumn("interaction_date", to_date("interaction_date"))
reference_date = lit("2025-05-19")

# Feature 1: Aggregation features
agg_features = (
    interactions
    .groupBy("hcp_id")
    .agg(
        count("*").alias("total_interactions"),
        countDistinct("interaction_type").alias("unique_interaction_types"),
        sum("value").alias("total_value"),
        avg("value").alias("avg_value"),
        max("value").alias("max_value"),
        min("interaction_date").alias("first_interaction"),
        max("interaction_date").alias("last_interaction"),
    )
    .withColumn("days_since_last", datediff(reference_date, col("last_interaction")))
    .withColumn("tenure_days", datediff(col("last_interaction"), col("first_interaction")))
)

# Feature 2: Pivot features (interaction counts by type)
pivot_features = (
    interactions
    .groupBy("hcp_id")
    .pivot("interaction_type")
    .agg(count("*"))
    .fillna(0)
)

# Feature 3: Recency features (last 7 days, 30 days, 90 days)
def window_features(df, days, suffix):
    return (
        df
        .filter(datediff(reference_date, col("interaction_date")) <= days)
        .groupBy("hcp_id")
        .agg(
            count("*").alias(f"interactions_{suffix}"),
            sum("value").alias(f"total_value_{suffix}")
        )
    )

features_7d = window_features(interactions, 7, "7d")
features_30d = window_features(interactions, 30, "30d")
features_90d = window_features(interactions, 90, "90d")

# Feature 4: Trend features (is activity increasing or decreasing?)
monthly_activity = (
    interactions
    .withColumn("month", date_format("interaction_date", "yyyy-MM"))
    .groupBy("hcp_id", "month")
    .agg(count("*").alias("monthly_count"))
)

window_trend = Window.partitionBy("hcp_id").orderBy("month")
trend_features = (
    monthly_activity
    .withColumn("prev_month_count", lag("monthly_count").over(window_trend))
    .withColumn("trend", 
        when(col("monthly_count") > col("prev_month_count"), 1)
        .when(col("monthly_count") < col("prev_month_count"), -1)
        .otherwise(0))
    .groupBy("hcp_id")
    .agg(avg("trend").alias("activity_trend"))
)

# Combine all features
final_features = (
    agg_features
    .join(pivot_features, "hcp_id", "left")
    .join(features_7d, "hcp_id", "left")
    .join(features_30d, "hcp_id", "left")
    .join(features_90d, "hcp_id", "left")
    .join(trend_features, "hcp_id", "left")
    .fillna(0)
)

final_features.show(truncate=False)
```

**Interview tip:** This is exactly what a feature store computation looks like. Mention: Why each feature matters for the model, how you'd productionize this (schedule daily, store in feature store, serve online features from Redis).
</details>

---

## Summary — SQL & PySpark Problem Map

| # | Problem | Type | Difficulty | Key Concept |
|---|---------|------|-----------|-------------|
| SQL-1 | Top N per group | SQL | Medium | Window functions (RANK) |
| SQL-2 | Consecutive days | SQL | Medium | Island technique |
| SQL-3 | Running total with reset | SQL | Hard | Nested windows |
| SQL-4 | Data quality checks | SQL | Medium | Practical ETL |
| SQL-5 | SCD Type 2 | SQL | Hard | Dimensional modeling |
| SQL-6 | Funnel analysis | SQL | Medium | Conditional aggregation |
| PySpark-1 | Dedup & merge | PySpark | Medium | Window + union |
| PySpark-2 | Anomaly detection | PySpark | Medium | Rolling statistics |
| PySpark-3 | SCD Type 2 in Spark | PySpark | Hard | Merge logic |
| PySpark-4 | Skewed join | PySpark | Hard | Salting technique |
| PySpark-5 | Feature engineering | PySpark | Hard | ML pipeline |

**Practice order:** SQL-1 → SQL-2 → SQL-4 → PySpark-1 → PySpark-2 → SQL-5 → PySpark-4 → PySpark-5
