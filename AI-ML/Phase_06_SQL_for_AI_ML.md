# Phase 6 — SQL for AI/ML

## Complete Learning & Interview Mastery Guide

---

## Table of Contents

1. [Why SQL Matters for AI/ML Engineers](#why-sql-matters-for-aiml-engineers)
2. [Database Fundamentals](#database-fundamentals)
3. [SELECT — Querying Data](#select--querying-data)
4. [WHERE — Filtering Data](#where--filtering-data)
5. [GROUP BY & HAVING — Aggregations](#group-by--having--aggregations)
6. [JOINs — Combining Tables](#joins--combining-tables)
7. [Subqueries](#subqueries)
8. [CTEs — Common Table Expressions](#ctes--common-table-expressions)
9. [Window Functions](#window-functions)
10. [SQL for Feature Engineering](#sql-for-feature-engineering)
11. [SQL Pipelines for ML](#sql-pipelines-for-ml)
12. [Data Warehouse Concepts](#data-warehouse-concepts)
13. [Performance Optimization](#performance-optimization)
14. [Interview Mastery](#interview-mastery)

---

## Why SQL Matters for AI/ML Engineers

### Beginner Explanation

SQL (Structured Query Language) is the universal language for talking to databases. Nearly all real-world data lives in databases — customer records, transaction logs, user behavior, sensor readings. Before you can train any ML model, you need to **extract, clean, and transform** that data. SQL is the primary tool for doing this at scale.

Think of SQL as the foundation of every data pipeline: data flows from a database → through SQL transformations → into your Python/Pandas environment → into your model.

### Why Every ML Engineer Must Know SQL

```
90% of companies store their data in SQL databases.
60-80% of an ML engineer's time is data preparation.
Most production ML pipelines are SQL-first.
Data scientists who know SQL get hired faster and paid more.
```

### Where SQL Fits in the ML Workflow

```mermaid
graph TD
    A[Raw Data in Database] --> B[SQL: Extract & Filter]
    B --> C[SQL: Clean & Transform]
    C --> D[SQL: Feature Engineering]
    D --> E[Python/Pandas: Final Processing]
    E --> F[Model Training]
    F --> G[SQL: Store Predictions]
    G --> H[SQL: Monitor Model Performance]

    style A fill:#ff6b6b,color:#fff
    style B fill:#4ecdc4,color:#fff
    style C fill:#4ecdc4,color:#fff
    style D fill:#4ecdc4,color:#fff
    style E fill:#45b7d1,color:#fff
    style F fill:#96ceb4,color:#fff
    style G fill:#4ecdc4,color:#fff
    style H fill:#4ecdc4,color:#fff
```

### SQL vs Pandas for Data Work

| Task | SQL | Pandas |
|------|-----|--------|
| Query stored data | Native | Requires loading first |
| Data at scale (TB+) | Excellent | Limited by memory |
| Real-time pipelines | Excellent | Not designed for this |
| Complex transformations | Good | Excellent |
| ML feature engineering | Good | Excellent |
| When to use | Data extraction, warehousing | Prototyping, local analysis |

---

## Database Fundamentals

### What is a Relational Database?

A relational database stores data in **tables** (relations) with rows and columns, just like spreadsheets — but with strict structure, relationships between tables, and powerful query capabilities.

```
Table: customers
+----+--------+-----+----------+
| id | name   | age | city     |
+----+--------+-----+----------+
|  1 | Alice  |  28 | New York |
|  2 | Bob    |  34 | Chicago  |
|  3 | Carol  |  22 | New York |
+----+--------+-----+----------+

Table: orders
+----+-------------+---------+-------+
| id | customer_id | product | price |
+----+-------------+---------+-------+
|  1 |           1 | Laptop  |  1200 |
|  2 |           1 | Phone   |   800 |
|  3 |           2 | Tablet  |   400 |
+----+-------------+---------+-------+
```

### Key Concepts

| Concept | Definition | Example |
|---------|-----------|---------|
| **Table** | A collection of rows and columns | `customers`, `orders` |
| **Row / Record** | A single data entry | One customer's info |
| **Column / Field** | A single attribute | `name`, `age` |
| **Primary Key (PK)** | Unique row identifier | `id` in customers |
| **Foreign Key (FK)** | Links to another table's PK | `customer_id` in orders |
| **Schema** | Structure definition of tables | Column names + types |
| **Index** | Speed-up structure for lookups | Index on `customer_id` |

### SQL Data Types (Essential for ML)

```sql
-- Numeric types (for ML features)
INT, BIGINT          -- Integer counts, IDs
FLOAT, DOUBLE        -- Continuous values
DECIMAL(10, 2)       -- Precise monetary values

-- String types
VARCHAR(255)         -- Variable-length text
TEXT                 -- Long-form text
CHAR(10)             -- Fixed-length

-- Date/time (critical for time-series ML)
DATE                 -- 2024-01-15
DATETIME             -- 2024-01-15 10:30:00
TIMESTAMP            -- Unix timestamp

-- Boolean
BOOLEAN              -- TRUE/FALSE

-- JSON (modern databases)
JSON, JSONB          -- Semi-structured data (PostgreSQL)
```

### SQL Execution Order (Critical Concept)

This is one of the most misunderstood SQL concepts. SQL clauses are **written** in one order but **executed** in a different order:

```
Written order:        Execution order:
1. SELECT         →   1. FROM
2. FROM           →   2. JOIN
3. JOIN           →   3. WHERE
4. WHERE          →   4. GROUP BY
5. GROUP BY       →   5. HAVING
6. HAVING         →   6. SELECT
7. ORDER BY       →   7. DISTINCT
8. LIMIT          →   8. ORDER BY
                  →   9. LIMIT
```

**Why this matters:** You cannot use a SELECT alias in a WHERE clause because WHERE executes before SELECT.

```sql
-- WRONG: alias used before it's defined
SELECT price * 0.9 AS discounted_price
FROM products
WHERE discounted_price < 100;   -- Error! alias doesn't exist yet

-- CORRECT: repeat the expression
SELECT price * 0.9 AS discounted_price
FROM products
WHERE price * 0.9 < 100;

-- OR use a subquery/CTE
SELECT * FROM (
    SELECT price * 0.9 AS discounted_price FROM products
) sub
WHERE discounted_price < 100;
```

---

## SELECT — Querying Data

### Basic SELECT Syntax

```sql
SELECT column1, column2, ...
FROM table_name;
```

### The Full SELECT Toolkit

```sql
-- Select all columns
SELECT * FROM customers;

-- Select specific columns
SELECT name, age, city FROM customers;

-- Select with alias (rename output)
SELECT name AS customer_name,
       age  AS customer_age
FROM customers;

-- Select with expression
SELECT name,
       age,
       age * 365 AS age_in_days,
       UPPER(name) AS name_upper,
       CONCAT(name, ' from ', city) AS full_description
FROM customers;

-- Select distinct values (remove duplicates)
SELECT DISTINCT city FROM customers;

-- Select with ordering
SELECT name, age
FROM customers
ORDER BY age DESC;          -- descending

SELECT name, age
FROM customers
ORDER BY age ASC, name ASC; -- multiple columns

-- Limit results
SELECT name, age
FROM customers
ORDER BY age DESC
LIMIT 10;                   -- top 10 oldest

-- Offset (for pagination)
SELECT name, age
FROM customers
ORDER BY age DESC
LIMIT 10 OFFSET 20;         -- records 21-30
```

### Conditional Logic in SELECT

```sql
-- CASE WHEN (SQL's if/else — critical for ML feature creation)
SELECT
    name,
    age,
    CASE
        WHEN age < 25 THEN 'Young'
        WHEN age BETWEEN 25 AND 40 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group,

    -- Binary feature for ML
    CASE WHEN age > 30 THEN 1 ELSE 0 END AS is_over_30,

    -- Numeric mapping (label encoding in SQL)
    CASE city
        WHEN 'New York' THEN 1
        WHEN 'Chicago'  THEN 2
        WHEN 'Boston'   THEN 3
        ELSE 0
    END AS city_encoded
FROM customers;
```

### NULL Handling

```sql
-- Check for NULL
SELECT * FROM customers WHERE age IS NULL;
SELECT * FROM customers WHERE age IS NOT NULL;

-- Replace NULL with a value (COALESCE)
SELECT
    name,
    COALESCE(age, 0) AS age,         -- replace NULL with 0
    COALESCE(city, 'Unknown') AS city -- replace NULL with 'Unknown'
FROM customers;

-- NULLIF — return NULL if two values are equal (avoid divide by zero)
SELECT
    total_revenue / NULLIF(num_orders, 0) AS avg_order_value
FROM sales;

-- IFNULL (MySQL) / NVL (Oracle) — equivalent to COALESCE for 2 values
SELECT IFNULL(age, 0) FROM customers;
```

### String Functions (for NLP feature engineering)

```sql
SELECT
    UPPER(name),              -- 'ALICE'
    LOWER(name),              -- 'alice'
    LENGTH(name),             -- character count
    TRIM('  hello  '),        -- 'hello'
    LTRIM, RTRIM,             -- trim left/right only
    SUBSTRING(name, 1, 3),   -- first 3 chars
    REPLACE(text, 'old', 'new'),
    CONCAT(first, ' ', last), -- join strings
    LIKE '%pattern%',         -- pattern matching
    name REGEXP '^[A-Z]'      -- regex matching (MySQL/PostgreSQL)
FROM table_name;
```

### Date/Time Functions (critical for time-series ML)

```sql
SELECT
    NOW(),                             -- current datetime
    CURDATE(),                         -- current date
    YEAR(created_at),                  -- extract year
    MONTH(created_at),                 -- extract month
    DAY(created_at),                   -- extract day
    DAYOFWEEK(created_at),             -- 1=Sunday, 7=Saturday
    HOUR(created_at),                  -- extract hour
    DATEDIFF(end_date, start_date),    -- days between dates
    DATE_ADD(date, INTERVAL 30 DAY),   -- add 30 days
    DATE_FORMAT(date, '%Y-%m'),        -- format as 'YYYY-MM'
    EXTRACT(YEAR FROM created_at),     -- ANSI SQL version
    DATE_TRUNC('month', created_at)    -- PostgreSQL: truncate to month
FROM events;
```

---

## WHERE — Filtering Data

### Basic Filtering

```sql
-- Comparison operators
SELECT * FROM products WHERE price > 100;
SELECT * FROM products WHERE price >= 100;
SELECT * FROM products WHERE price = 100;
SELECT * FROM products WHERE price != 100;
SELECT * FROM products WHERE price <> 100; -- same as !=

-- Logical operators
SELECT * FROM products WHERE price > 100 AND category = 'Electronics';
SELECT * FROM products WHERE price < 50 OR category = 'Books';
SELECT * FROM products WHERE NOT (price > 1000);
```

### Range and List Filtering

```sql
-- BETWEEN (inclusive on both ends)
SELECT * FROM products WHERE price BETWEEN 50 AND 200;

-- IN (match any value in a list)
SELECT * FROM customers WHERE city IN ('New York', 'Chicago', 'Boston');
SELECT * FROM orders WHERE status NOT IN ('cancelled', 'refunded');

-- LIKE (pattern matching)
SELECT * FROM customers WHERE name LIKE 'A%';    -- starts with A
SELECT * FROM customers WHERE name LIKE '%son';  -- ends with son
SELECT * FROM customers WHERE name LIKE '%li%';  -- contains li
SELECT * FROM customers WHERE name LIKE '_ob';   -- 3 chars, ends in ob

-- Regular expressions (PostgreSQL / MySQL)
SELECT * FROM customers WHERE email ~ '^[a-z]+@gmail\.com$';  -- PostgreSQL
SELECT * FROM customers WHERE email REGEXP '^[a-z]+@gmail';   -- MySQL
```

### NULL Handling in WHERE

```sql
-- NEVER use = NULL (always returns empty!)
SELECT * FROM customers WHERE age = NULL;   -- WRONG
SELECT * FROM customers WHERE age IS NULL;  -- CORRECT
SELECT * FROM customers WHERE age IS NOT NULL;
```

### Filtering for ML Data Preparation

```sql
-- Remove outliers
SELECT *
FROM transactions
WHERE amount BETWEEN 1 AND 10000      -- remove extreme outliers
  AND amount IS NOT NULL              -- remove nulls
  AND user_id IS NOT NULL;            -- remove missing labels

-- Select training data (exclude test period)
SELECT *
FROM user_events
WHERE event_date < '2024-01-01'       -- train on data before this date
  AND label IS NOT NULL;              -- only labeled samples

-- Filter for class balance analysis
SELECT label, COUNT(*) as count
FROM ml_dataset
WHERE label IN (0, 1)
GROUP BY label;
```

---

## GROUP BY & HAVING — Aggregations

### Understanding GROUP BY

GROUP BY collapses multiple rows that share the same value in a column into a single summary row. This is the SQL equivalent of Pandas `.groupby()`.

```
Before GROUP BY:              After GROUP BY city:
+--------+-------+           +----------+----------+
| city   | price |           | city     | avg_price|
+--------+-------+           +----------+----------+
|New York|  100  |           |New York  |  125.0   |
|New York|  150  |           |Chicago   |   75.0   |
|Chicago |   75  |           +----------+----------+
+--------+-------+
```

### Aggregate Functions

```sql
SELECT
    city,
    COUNT(*)              AS total_customers,     -- count rows
    COUNT(age)            AS non_null_age_count,  -- count non-NULLs
    COUNT(DISTINCT city)  AS unique_cities,        -- count distinct
    SUM(purchase_amount)  AS total_revenue,
    AVG(purchase_amount)  AS avg_purchase,
    MIN(age)              AS youngest,
    MAX(age)              AS oldest,
    STDDEV(purchase_amount) AS purchase_std,       -- standard deviation
    VARIANCE(purchase_amount) AS purchase_var
FROM customers
GROUP BY city;
```

### GROUP BY Rules

```sql
-- Every non-aggregated column in SELECT must be in GROUP BY
SELECT city, age, COUNT(*)     -- WRONG: age not in GROUP BY
FROM customers
GROUP BY city;

SELECT city, COUNT(*)          -- CORRECT
FROM customers
GROUP BY city;

-- GROUP BY multiple columns
SELECT city, age_group, COUNT(*) AS count
FROM customers
GROUP BY city, age_group;

-- GROUP BY with expression
SELECT
    DATE_FORMAT(created_at, '%Y-%m') AS month,
    COUNT(*) AS events_count
FROM user_events
GROUP BY DATE_FORMAT(created_at, '%Y-%m')
ORDER BY month;
```

### HAVING — Filtering After Aggregation

WHERE filters rows **before** grouping. HAVING filters groups **after** aggregation.

```sql
-- Find cities with more than 100 customers
SELECT city, COUNT(*) AS customer_count
FROM customers
GROUP BY city
HAVING COUNT(*) > 100;

-- Find high-value customer segments
SELECT
    city,
    AVG(lifetime_value) AS avg_ltv,
    COUNT(*) AS customer_count
FROM customers
GROUP BY city
HAVING AVG(lifetime_value) > 1000
   AND COUNT(*) >= 50
ORDER BY avg_ltv DESC;
```

### WHERE vs HAVING

```sql
-- WHERE: filter rows before grouping (faster — reduces data early)
-- HAVING: filter groups after aggregation

-- Example: avg purchase for active customers in major cities
SELECT city, AVG(purchase_amount) AS avg_purchase
FROM customers
WHERE status = 'active'           -- WHERE: filter rows first
GROUP BY city
HAVING AVG(purchase_amount) > 500 -- HAVING: filter aggregated result
ORDER BY avg_purchase DESC;
```

### ML Use Case — Class Distribution Analysis

```sql
-- Check class imbalance
SELECT
    label,
    COUNT(*) AS count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM training_data
GROUP BY label
ORDER BY label;

-- Feature statistics per class (EDA in SQL)
SELECT
    label,
    COUNT(*)        AS n,
    AVG(feature_1)  AS mean_f1,
    STDDEV(feature_1) AS std_f1,
    MIN(feature_1)  AS min_f1,
    MAX(feature_1)  AS max_f1
FROM training_data
GROUP BY label;
```

---

## JOINs — Combining Tables

### Why JOINs Matter for ML

In real ML projects, your features live in **multiple tables**. You need to JOIN them to create the training dataset. A poorly written JOIN can silently duplicate rows, drop valid samples, or skew your training data — leading to bad models.

### Types of JOINs — Visual Guide

```
Table A (customers):          Table B (orders):
+----+-------+               +----+-----+--------+
| id | name  |               | id | cid | amount |
+----+-------+               +----+-----+--------+
|  1 | Alice |               |  1 |  1  |  100   |
|  2 | Bob   |               |  2 |  1  |  200   |
|  3 | Carol |               |  3 |  2  |   50   |
|  4 | Dave  |               |  4 |  5  |  300   | <- cid=5 not in A
+----+-------+               +----+-----+--------+

INNER JOIN (only matching rows):
+----+-------+--------+
|  1 | Alice |  100   |
|  1 | Alice |  200   |
|  2 | Bob   |   50   |

LEFT JOIN (all A + matching B):
+----+-------+--------+
|  1 | Alice |  100   |
|  1 | Alice |  200   |
|  2 | Bob   |   50   |
|  3 | Carol |  NULL  | <- Carol has no orders
|  4 | Dave  |  NULL  | <- Dave has no orders

RIGHT JOIN (matching A + all B):
|  1 | Alice |  100   |
|  1 | Alice |  200   |
|  2 | Bob   |   50   |
| NULL| NULL |  300   | <- order with no customer

FULL OUTER JOIN (everything):
All rows from both sides, NULLs where no match
```

```mermaid
graph LR
    subgraph INNER
        A1((A)) --> |"∩"| B1((B))
    end
    subgraph LEFT
        A2((A)) --> |"A + ∩"| B2((B))
    end
    subgraph RIGHT
        A3((A)) --> |"∩ + B"| B3((B))
    end
    subgraph FULL
        A4((A)) --> |"A ∪ B"| B4((B))
    end
```

### JOIN Syntax

```sql
-- INNER JOIN — most common, returns only matching rows
SELECT c.name, o.amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- LEFT JOIN — all customers even without orders
SELECT c.name, COALESCE(SUM(o.amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- RIGHT JOIN (rare — usually rewritten as LEFT JOIN)
SELECT c.name, o.amount
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;

-- FULL OUTER JOIN (PostgreSQL / SQL Server)
SELECT c.name, o.amount
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;

-- CROSS JOIN (every row of A with every row of B — be careful!)
SELECT a.feature_1, b.feature_2
FROM table_a a
CROSS JOIN table_b b;
-- If A has 1000 rows and B has 1000 rows → result has 1,000,000 rows!

-- SELF JOIN (join a table to itself)
SELECT e1.name AS employee, e2.name AS manager
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.id;
```

### Multi-Table JOINs for ML Feature Assembly

```sql
-- Assembling ML training dataset from multiple tables
SELECT
    u.user_id,
    u.age,
    u.city,
    u.signup_date,

    -- Purchase features
    COALESCE(p.total_purchases, 0) AS total_purchases,
    COALESCE(p.avg_order_value, 0) AS avg_order_value,
    COALESCE(p.days_since_last_purchase, 999) AS days_since_last_purchase,

    -- Behavioral features
    COALESCE(b.page_views_30d, 0) AS page_views_30d,
    COALESCE(b.sessions_30d, 0) AS sessions_30d,

    -- Label
    CASE WHEN churn.user_id IS NOT NULL THEN 1 ELSE 0 END AS churned

FROM users u
LEFT JOIN (
    SELECT
        user_id,
        COUNT(*) AS total_purchases,
        AVG(amount) AS avg_order_value,
        DATEDIFF(NOW(), MAX(order_date)) AS days_since_last_purchase
    FROM orders
    WHERE order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
    GROUP BY user_id
) p ON u.user_id = p.user_id

LEFT JOIN (
    SELECT
        user_id,
        SUM(page_views) AS page_views_30d,
        COUNT(DISTINCT session_id) AS sessions_30d
    FROM user_sessions
    WHERE session_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    GROUP BY user_id
) b ON u.user_id = b.user_id

LEFT JOIN churned_users churn ON u.user_id = churn.user_id
WHERE u.signup_date < DATE_SUB(NOW(), INTERVAL 30 DAY); -- only mature users
```

### Common JOIN Pitfalls in ML

```sql
-- PITFALL 1: Row multiplication (fan-out)
-- If a user has 5 orders, INNER JOIN multiplies their user row 5 times
-- This skews feature statistics!

-- WRONG: user features are duplicated 5 times
SELECT u.age, o.amount
FROM users u
JOIN orders o ON u.user_id = o.user_id;
-- Result: 5 rows per user → if you AVG(age), it's wrong!

-- CORRECT: aggregate first, then join
SELECT u.age, order_stats.total_orders
FROM users u
JOIN (
    SELECT user_id, COUNT(*) AS total_orders FROM orders GROUP BY user_id
) order_stats ON u.user_id = order_stats.user_id;

-- PITFALL 2: LEFT JOIN removes rows due to WHERE on right table
-- WRONG: converts LEFT JOIN into INNER JOIN
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 100;  -- Customers without orders get filtered out!

-- CORRECT: move filter into JOIN condition
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.amount > 100;
```

---

## Subqueries

### What is a Subquery?

A subquery is a SQL query **nested inside** another query. It runs first and its result is used by the outer query.

```sql
-- Find customers who have placed orders above average amount
SELECT name
FROM customers
WHERE id IN (
    SELECT customer_id
    FROM orders
    WHERE amount > (SELECT AVG(amount) FROM orders)
);
```

### Types of Subqueries

#### 1. Scalar Subquery — returns a single value

```sql
-- Find products priced above the average
SELECT name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Use in SELECT
SELECT
    name,
    price,
    price - (SELECT AVG(price) FROM products) AS diff_from_avg
FROM products;
```

#### 2. Row Subquery — returns a single row

```sql
SELECT * FROM products
WHERE (category, price) = (
    SELECT category, MAX(price) FROM products WHERE category = 'Electronics'
);
```

#### 3. Table Subquery (Derived Table) — returns multiple rows/columns

```sql
-- Subquery in FROM creates a virtual table
SELECT
    city,
    avg_age,
    CASE WHEN avg_age > 30 THEN 'Older' ELSE 'Younger' END AS segment
FROM (
    SELECT city, AVG(age) AS avg_age
    FROM customers
    GROUP BY city
) city_stats
WHERE avg_age > 25;
```

#### 4. Correlated Subquery — references the outer query

```sql
-- Find customers whose last order was their largest order
SELECT c.name, o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount = (
    SELECT MAX(o2.amount)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id  -- references outer query!
);

-- Correlated subqueries are slow on large data — prefer window functions
```

#### 5. EXISTS / NOT EXISTS Subquery

```sql
-- Find customers who have made at least one purchase
SELECT name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Find customers who have NEVER purchased (for churn modeling)
SELECT name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### Subqueries for ML Data Preparation

```sql
-- Percentile-based outlier detection
SELECT *
FROM transactions
WHERE amount BETWEEN
    (SELECT PERCENTILE_CONT(0.01) WITHIN GROUP (ORDER BY amount) FROM transactions)
    AND
    (SELECT PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY amount) FROM transactions);

-- Create features using correlated subquery (slow, prefer window functions)
SELECT
    user_id,
    order_date,
    amount,
    (SELECT COUNT(*) FROM orders o2
     WHERE o2.user_id = o1.user_id
       AND o2.order_date < o1.order_date) AS prior_orders_count
FROM orders o1;
```

---

## CTEs — Common Table Expressions

### What is a CTE?

A CTE (Common Table Expression) is a **named temporary result set** defined with `WITH`. Think of it as creating a named variable that holds a query result, which you can then reference multiple times. CTEs make complex queries readable and modular.

```sql
WITH cte_name AS (
    SELECT ...     -- define the CTE
)
SELECT ...
FROM cte_name;    -- use it
```

### Basic CTE vs Subquery

```sql
-- Subquery version (hard to read)
SELECT city, avg_age
FROM (
    SELECT city, AVG(age) AS avg_age
    FROM customers
    GROUP BY city
) city_stats
WHERE avg_age > 30;

-- CTE version (clean and readable)
WITH city_stats AS (
    SELECT city, AVG(age) AS avg_age
    FROM customers
    GROUP BY city
)
SELECT city, avg_age
FROM city_stats
WHERE avg_age > 30;
```

### Multiple CTEs (Chained)

```sql
-- ML pipeline: compute features step by step
WITH
-- Step 1: Purchase features
purchase_features AS (
    SELECT
        user_id,
        COUNT(*) AS total_orders,
        SUM(amount) AS lifetime_value,
        AVG(amount) AS avg_order_value,
        MAX(order_date) AS last_order_date
    FROM orders
    GROUP BY user_id
),

-- Step 2: Recency features
recency_features AS (
    SELECT
        user_id,
        DATEDIFF(CURRENT_DATE, last_order_date) AS days_since_last_order,
        CASE
            WHEN DATEDIFF(CURRENT_DATE, last_order_date) <= 30  THEN 'Active'
            WHEN DATEDIFF(CURRENT_DATE, last_order_date) <= 90  THEN 'At Risk'
            ELSE 'Churned'
        END AS recency_segment
    FROM purchase_features
),

-- Step 3: Join everything into training dataset
final_dataset AS (
    SELECT
        u.user_id,
        u.age,
        u.city,
        pf.total_orders,
        pf.lifetime_value,
        pf.avg_order_value,
        rf.days_since_last_order,
        rf.recency_segment,
        CASE WHEN rf.recency_segment = 'Churned' THEN 1 ELSE 0 END AS label
    FROM users u
    LEFT JOIN purchase_features pf ON u.user_id = pf.user_id
    LEFT JOIN recency_features rf ON u.user_id = rf.user_id
)

SELECT * FROM final_dataset;
```

### Recursive CTEs — Hierarchical Data

```sql
-- Traverse organizational hierarchy (employee → manager chain)
WITH RECURSIVE org_hierarchy AS (
    -- Base case: start with top-level employees
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: find direct reports
    SELECT e.id, e.name, e.manager_id, h.level + 1
    FROM employees e
    JOIN org_hierarchy h ON e.manager_id = h.id
)
SELECT * FROM org_hierarchy ORDER BY level, name;

-- Generate a date series (useful for time-series ML gaps)
WITH RECURSIVE date_series AS (
    SELECT '2024-01-01'::DATE AS dt
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < '2024-12-31'
)
SELECT dt FROM date_series;
```

### CTEs vs Subqueries vs Temp Tables

| Feature | CTE | Subquery | Temp Table |
|---------|-----|----------|------------|
| Readability | Excellent | Poor for complex | Good |
| Reusability | Yes (within query) | No | Yes (across queries) |
| Performance | Same as subquery | Baseline | Better for large data |
| Recursive | Yes | No | No |
| Indexable | No | No | Yes |
| When to use | Complex multi-step logic | Simple one-off | Repeated use, large data |

---

## Window Functions

### What are Window Functions?

Window functions perform calculations **across a set of rows related to the current row** — without collapsing them into one row (unlike GROUP BY). They are the most powerful SQL feature for ML feature engineering.

```
Regular aggregation (GROUP BY):     Window function:
+------+-------+                    +------+-------+----------+
| city | count |                    | name | city  | city_cnt |
+------+-------+                    +------+-------+----------+
| NY   |   2   |                    | Alice| NY    |    2     | ← keeps row
| LA   |   1   |                    | Bob  | NY    |    2     | ← keeps row
+------+-------+                    | Carol| LA    |    1     | ← keeps row
                                    +------+-------+----------+
```

### Window Function Syntax

```sql
function_name() OVER (
    PARTITION BY column1, column2  -- group rows (like GROUP BY but keeps rows)
    ORDER BY column3               -- order within the window
    ROWS/RANGE frame_spec          -- define the frame size
)
```

### Ranking Functions

```sql
SELECT
    name,
    department,
    salary,

    -- ROW_NUMBER: unique rank, no ties (1,2,3,4...)
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,

    -- RANK: tied values get same rank, gaps in sequence (1,1,3...)
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_num,

    -- DENSE_RANK: tied values same rank, NO gaps (1,1,2...)
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank,

    -- PERCENT_RANK: relative rank as percentage (0.0 to 1.0)
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary) AS pct_rank,

    -- NTILE: divide rows into N buckets
    NTILE(4) OVER (ORDER BY salary) AS quartile,  -- 1,2,3,4 quartile

    -- CUME_DIST: cumulative distribution (0.0 to 1.0)
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_pct

FROM employees;
```

### Lag and Lead — Critical for Time-Series ML

```sql
SELECT
    user_id,
    event_date,
    amount,

    -- LAG: previous row value
    LAG(amount) OVER (PARTITION BY user_id ORDER BY event_date)
        AS prev_amount,

    -- LEAD: next row value
    LEAD(amount) OVER (PARTITION BY user_id ORDER BY event_date)
        AS next_amount,

    -- LAG with offset (2 periods back)
    LAG(amount, 2) OVER (PARTITION BY user_id ORDER BY event_date)
        AS amount_2_periods_ago,

    -- LAG with default value if no previous row
    LAG(amount, 1, 0) OVER (PARTITION BY user_id ORDER BY event_date)
        AS prev_amount_default_0,

    -- Compute change (momentum feature)
    amount - LAG(amount) OVER (PARTITION BY user_id ORDER BY event_date)
        AS amount_change,

    -- Percent change
    (amount - LAG(amount) OVER (PARTITION BY user_id ORDER BY event_date))
    / NULLIF(LAG(amount) OVER (PARTITION BY user_id ORDER BY event_date), 0)
        AS pct_change

FROM transactions
ORDER BY user_id, event_date;
```

### Aggregate Window Functions

```sql
SELECT
    user_id,
    order_date,
    amount,

    -- Running total
    SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,

    -- Running average
    AVG(amount) OVER (PARTITION BY user_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_avg,

    -- Rolling 7-day average (preceding 6 rows + current row = 7)
    AVG(amount) OVER (PARTITION BY user_id ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7d_avg,

    -- Rolling 30-day sum
    SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS rolling_30d_sum,

    -- Total for the partition (no ORDER BY = no frame limit)
    SUM(amount) OVER (PARTITION BY user_id) AS user_total,

    -- What % of user's total is this order?
    amount / SUM(amount) OVER (PARTITION BY user_id) AS pct_of_user_total,

    -- Grand total
    SUM(amount) OVER () AS grand_total

FROM orders;
```

### Window Frame Specification

```sql
-- ROWS BETWEEN: physical row offsets
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- from start to now
ROWS BETWEEN 3 PRECEDING AND CURRENT ROW          -- last 3 rows + current
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING  -- from now to end
ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING          -- centered window

-- RANGE BETWEEN: logical range (by value)
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW  -- last 7 days by date
```

### Window Functions for ML Feature Engineering

```sql
-- Full feature engineering pipeline using window functions
WITH user_features AS (
    SELECT
        user_id,
        event_date,
        amount,

        -- Recency feature: days since previous purchase
        DATEDIFF(event_date,
            LAG(event_date) OVER (PARTITION BY user_id ORDER BY event_date)
        ) AS days_since_prev_purchase,

        -- Frequency feature: rolling 30d order count
        COUNT(*) OVER (PARTITION BY user_id ORDER BY event_date
            ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS orders_last_30d,

        -- Monetary feature: rolling 90d spend
        SUM(amount) OVER (PARTITION BY user_id ORDER BY event_date
            ROWS BETWEEN 89 PRECEDING AND CURRENT ROW) AS spend_last_90d,

        -- Trend feature: change in spend vs previous period
        SUM(amount) OVER (PARTITION BY user_id ORDER BY event_date
            ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)
        - SUM(amount) OVER (PARTITION BY user_id ORDER BY event_date
            ROWS BETWEEN 59 PRECEDING AND 30 PRECEDING) AS spend_trend,

        -- Rank within user (is this their biggest order?)
        RANK() OVER (PARTITION BY user_id ORDER BY amount DESC)
            AS order_rank_by_amount,

        -- Quartile of this order globally
        NTILE(4) OVER (ORDER BY amount) AS amount_quartile

    FROM orders
)
SELECT * FROM user_features
WHERE event_date = (SELECT MAX(event_date) FROM orders); -- latest snapshot
```

---

## SQL for Feature Engineering

### Encoding Categorical Variables

```sql
-- Label encoding (ordinal)
SELECT
    product_id,
    CASE category
        WHEN 'Electronics' THEN 1
        WHEN 'Clothing'    THEN 2
        WHEN 'Books'       THEN 3
        ELSE 0
    END AS category_encoded
FROM products;

-- One-hot encoding in SQL (pivot)
SELECT
    user_id,
    MAX(CASE WHEN channel = 'email'   THEN 1 ELSE 0 END) AS channel_email,
    MAX(CASE WHEN channel = 'mobile'  THEN 1 ELSE 0 END) AS channel_mobile,
    MAX(CASE WHEN channel = 'desktop' THEN 1 ELSE 0 END) AS channel_desktop
FROM user_events
GROUP BY user_id;
```

### Numerical Feature Engineering

```sql
-- Binning (discretization)
SELECT
    user_id,
    age,
    CASE
        WHEN age < 18 THEN 'under_18'
        WHEN age < 25 THEN '18_24'
        WHEN age < 35 THEN '25_34'
        WHEN age < 50 THEN '35_49'
        ELSE '50_plus'
    END AS age_bin,

    -- Quantile-based binning using NTILE
    NTILE(10) OVER (ORDER BY lifetime_value) AS ltv_decile,
    NTILE(4)  OVER (ORDER BY recency_days)   AS recency_quartile
FROM users;

-- Min-max normalization (SQL version)
SELECT
    user_id,
    feature,
    (feature - MIN(feature) OVER ()) /
    NULLIF(MAX(feature) OVER () - MIN(feature) OVER (), 0) AS feature_normalized
FROM ml_data;

-- Z-score standardization
SELECT
    user_id,
    feature,
    (feature - AVG(feature) OVER ()) /
    NULLIF(STDDEV(feature) OVER (), 0) AS feature_standardized
FROM ml_data;
```

### Time-Based Features for ML

```sql
SELECT
    user_id,
    event_timestamp,

    -- Time components (potential cyclical features)
    HOUR(event_timestamp)        AS hour_of_day,
    DAYOFWEEK(event_timestamp)   AS day_of_week,   -- 1=Sun, 7=Sat
    DAY(event_timestamp)         AS day_of_month,
    MONTH(event_timestamp)       AS month,
    QUARTER(event_timestamp)     AS quarter,
    YEAR(event_timestamp)        AS year,

    -- Is it a weekend?
    CASE WHEN DAYOFWEEK(event_timestamp) IN (1, 7) THEN 1 ELSE 0 END
        AS is_weekend,

    -- Is it business hours?
    CASE WHEN HOUR(event_timestamp) BETWEEN 9 AND 17 THEN 1 ELSE 0 END
        AS is_business_hours,

    -- Time since first event (user age in days)
    DATEDIFF(event_timestamp,
        MIN(event_timestamp) OVER (PARTITION BY user_id)) AS user_age_days,

    -- Time delta between events
    TIMESTAMPDIFF(MINUTE, event_timestamp,
        LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp)
    ) AS minutes_since_prev_event

FROM user_events;
```

### RFM Analysis (Classic ML Feature Set)

```sql
-- RFM = Recency, Frequency, Monetary — classic customer segmentation
WITH rfm_raw AS (
    SELECT
        customer_id,
        DATEDIFF(CURRENT_DATE, MAX(order_date)) AS recency,
        COUNT(*) AS frequency,
        SUM(amount) AS monetary
    FROM orders
    WHERE order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 1 YEAR)
    GROUP BY customer_id
),
rfm_scored AS (
    SELECT
        customer_id,
        recency,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY recency ASC)   AS r_score,  -- lower=better
        NTILE(5) OVER (ORDER BY frequency DESC) AS f_score,  -- higher=better
        NTILE(5) OVER (ORDER BY monetary DESC)  AS m_score   -- higher=better
    FROM rfm_raw
)
SELECT
    customer_id,
    recency,
    frequency,
    monetary,
    r_score,
    f_score,
    m_score,
    r_score + f_score + m_score AS rfm_total,
    CASE
        WHEN r_score >= 4 AND f_score >= 4 THEN 'Champions'
        WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 4 THEN 'Recent Customers'
        WHEN f_score >= 4 THEN 'Frequent Buyers'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'At Risk'
        ELSE 'Needs Attention'
    END AS segment
FROM rfm_scored;
```

---

## SQL Pipelines for ML

### Building a Complete ML Dataset in SQL

```sql
-- Complete end-to-end ML feature pipeline
-- Use Case: Customer churn prediction

WITH
-- Base population: customers eligible for prediction
base_population AS (
    SELECT
        customer_id,
        signup_date,
        plan_type,
        country
    FROM customers
    WHERE signup_date BETWEEN '2023-01-01' AND '2023-12-31'
      AND account_status != 'suspended'
),

-- Purchase history features (90-day lookback)
purchase_features AS (
    SELECT
        o.customer_id,
        COUNT(*)                                AS orders_90d,
        SUM(o.amount)                           AS revenue_90d,
        AVG(o.amount)                           AS avg_order_value_90d,
        MAX(o.order_date)                       AS last_order_date,
        DATEDIFF(CURRENT_DATE, MAX(o.order_date)) AS days_since_last_order,
        COUNT(DISTINCT o.product_category)      AS categories_purchased
    FROM orders o
    WHERE o.order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
    GROUP BY o.customer_id
),

-- Support ticket features
support_features AS (
    SELECT
        customer_id,
        COUNT(*) AS tickets_90d,
        SUM(CASE WHEN severity = 'high' THEN 1 ELSE 0 END) AS high_severity_tickets,
        AVG(resolution_hours) AS avg_resolution_hours
    FROM support_tickets
    WHERE created_at >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
    GROUP BY customer_id
),

-- Engagement features
engagement_features AS (
    SELECT
        customer_id,
        COUNT(DISTINCT session_date) AS active_days_30d,
        SUM(page_views)              AS page_views_30d,
        AVG(session_duration_min)    AS avg_session_duration
    FROM user_sessions
    WHERE session_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
    GROUP BY customer_id
),

-- Label: did they churn in next 30 days?
churn_labels AS (
    SELECT customer_id, 1 AS churned
    FROM churned_customers
    WHERE churn_date BETWEEN CURRENT_DATE AND DATE_ADD(CURRENT_DATE, INTERVAL 30 DAY)
),

-- Final assembled dataset
final_dataset AS (
    SELECT
        bp.customer_id,
        bp.plan_type,
        bp.country,
        DATEDIFF(CURRENT_DATE, bp.signup_date) AS account_age_days,

        -- Purchase features (0 if no purchases)
        COALESCE(pf.orders_90d, 0)             AS orders_90d,
        COALESCE(pf.revenue_90d, 0)            AS revenue_90d,
        COALESCE(pf.avg_order_value_90d, 0)    AS avg_order_value_90d,
        COALESCE(pf.days_since_last_order, 999) AS days_since_last_order,
        COALESCE(pf.categories_purchased, 0)   AS categories_purchased,

        -- Support features
        COALESCE(sf.tickets_90d, 0)            AS tickets_90d,
        COALESCE(sf.high_severity_tickets, 0)  AS high_severity_tickets,

        -- Engagement features
        COALESCE(ef.active_days_30d, 0)        AS active_days_30d,
        COALESCE(ef.page_views_30d, 0)         AS page_views_30d,

        -- Label (0 = retained, 1 = churned)
        COALESCE(cl.churned, 0)                AS label

    FROM base_population bp
    LEFT JOIN purchase_features pf  ON bp.customer_id = pf.customer_id
    LEFT JOIN support_features sf   ON bp.customer_id = sf.customer_id
    LEFT JOIN engagement_features ef ON bp.customer_id = ef.customer_id
    LEFT JOIN churn_labels cl        ON bp.customer_id = cl.customer_id
)

SELECT * FROM final_dataset;
```

### Using SQL with Python (SQLAlchemy + Pandas)

```python
import pandas as pd
from sqlalchemy import create_engine, text

# Connect to database
engine = create_engine("postgresql://user:pass@host:5432/dbname")

# Execute SQL and load into Pandas DataFrame
query = """
    SELECT
        user_id,
        age,
        orders_90d,
        revenue_90d,
        label
    FROM ml_features
    WHERE split = 'train'
"""

df = pd.read_sql(query, engine)

# Parameterized queries (safe from SQL injection)
query = text("""
    SELECT * FROM ml_features
    WHERE split = :split
      AND created_at >= :start_date
""")

df = pd.read_sql(query, engine, params={"split": "train", "start_date": "2024-01-01"})

# Write model predictions back to database
predictions_df = pd.DataFrame({
    "user_id": test_df["user_id"],
    "predicted_churn_prob": model.predict_proba(X_test)[:, 1],
    "prediction_date": pd.Timestamp.now()
})

predictions_df.to_sql(
    "model_predictions",
    engine,
    if_exists="append",
    index=False,
    chunksize=1000
)
```

---

## Data Warehouse Concepts

### OLTP vs OLAP

```
OLTP (Online Transaction Processing)   OLAP (Online Analytical Processing)
= Production databases                  = Data warehouses

Purpose: Run the business              Purpose: Analyze the business
Queries: Many short read/writes        Queries: Few complex reads
Data size: GB                          Data size: TB to PB
Schema: Normalized (3NF)               Schema: Denormalized (star/snowflake)
Examples: MySQL, PostgreSQL            Examples: BigQuery, Snowflake, Redshift
ML Use: Source data                    ML Use: Feature engineering at scale
```

### Star Schema — The Data Warehouse Standard

```mermaid
graph TD
    FACT[Fact Table: sales\n- sale_id PK\n- date_id FK\n- product_id FK\n- customer_id FK\n- store_id FK\n- amount\n- quantity] --> DIM_DATE[Dim: date\n- date_id PK\n- year\n- month\n- day\n- quarter\n- is_weekend]
    FACT --> DIM_PRODUCT[Dim: product\n- product_id PK\n- name\n- category\n- brand\n- price]
    FACT --> DIM_CUSTOMER[Dim: customer\n- customer_id PK\n- name\n- age\n- city\n- segment]
    FACT --> DIM_STORE[Dim: store\n- store_id PK\n- name\n- city\n- region]

    style FACT fill:#ff6b6b,color:#fff
    style DIM_DATE fill:#4ecdc4,color:#fff
    style DIM_PRODUCT fill:#4ecdc4,color:#fff
    style DIM_CUSTOMER fill:#4ecdc4,color:#fff
    style DIM_STORE fill:#4ecdc4,color:#fff
```

**Fact tables**: Store measurable business events (sales, clicks, logins). Large, many rows.  
**Dimension tables**: Store context/attributes about the events. Small-medium, mostly read.

### Querying a Star Schema for ML

```sql
-- Assemble ML features from star schema
SELECT
    s.sale_id,

    -- Customer features (from dimension)
    c.age,
    c.city,
    c.customer_segment,

    -- Product features
    p.category,
    p.brand,
    p.price AS product_price,

    -- Time features
    d.year,
    d.month,
    d.day_of_week,
    d.is_holiday,
    d.is_weekend,

    -- Store features
    st.region,
    st.store_size,

    -- Target variable
    s.amount AS sale_amount

FROM fact_sales s
JOIN dim_date     d  ON s.date_id     = d.date_id
JOIN dim_customer c  ON s.customer_id = c.customer_id
JOIN dim_product  p  ON s.product_id  = p.product_id
JOIN dim_store    st ON s.store_id    = st.store_id

WHERE d.year = 2024
  AND s.amount > 0
  AND c.customer_segment IS NOT NULL;
```

### Partitioning for ML Data Scale

```sql
-- Table partitioned by date (critical for time-series ML)
-- BigQuery syntax:
CREATE TABLE ml_events
PARTITION BY DATE(event_date)
AS SELECT * FROM raw_events;

-- Always filter on partition column to avoid full scans
-- GOOD (partition pruning — only reads relevant partitions):
SELECT * FROM ml_events
WHERE event_date >= '2024-01-01'
  AND event_date <  '2024-02-01';

-- BAD (full table scan — reads ALL partitions):
SELECT * FROM ml_events
WHERE YEAR(event_date) = 2024;  -- function prevents pruning!
```

### Slowly Changing Dimensions (SCD) — Important for Training Data

```sql
-- Type 2 SCD: track historical customer state
-- Used when you need to know what a customer's attributes WERE
-- at the time of a transaction (not what they are NOW)

-- Customer dimension with history
CREATE TABLE dim_customer_scd2 (
    customer_key  INT PRIMARY KEY,
    customer_id   INT,
    city          VARCHAR(100),
    plan_type     VARCHAR(50),
    effective_date DATE,
    expiry_date    DATE,          -- NULL means current
    is_current     BOOLEAN
);

-- Join transaction to customer state AT TIME OF TRANSACTION
SELECT
    t.transaction_id,
    t.amount,
    c.city,        -- city when transaction happened, not current city
    c.plan_type    -- plan at time of transaction
FROM transactions t
JOIN dim_customer_scd2 c
    ON t.customer_id = c.customer_id
    AND t.transaction_date BETWEEN c.effective_date AND COALESCE(c.expiry_date, '9999-12-31')
    AND c.is_current = FALSE;  -- get historical record, not current
```

### Common Table Formats in Modern Data Warehouses

```
Delta Lake / Iceberg / Hudi: ACID transactions on data lakes
Parquet: Columnar storage format — best for analytical queries
ORC: Optimized Row Columnar — similar to Parquet

Why columnar matters for ML:
- ML queries aggregate columns (AVG, SUM, STDDEV)
- Columnar format reads only needed columns (vs row-based reads all)
- 10-100x faster for analytical ML workloads
```

---

## Performance Optimization

### Query Optimization Principles

```sql
-- 1. Filter early (push predicates down)
-- SLOW: compute everything, then filter
SELECT *
FROM (SELECT user_id, SUM(amount) AS total FROM orders GROUP BY user_id) t
WHERE total > 1000;

-- FAST: equivalent but filter pushdown happens automatically in good optimizers
-- Write it explicitly for clarity:
SELECT user_id, SUM(amount) AS total
FROM orders
WHERE status = 'completed'   -- filter before GROUP BY
GROUP BY user_id
HAVING SUM(amount) > 1000;

-- 2. Use indexes properly
-- Index on WHERE columns, JOIN columns, ORDER BY columns
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date); -- composite

-- 3. Avoid functions on indexed columns in WHERE
-- SLOW: function prevents index use
WHERE YEAR(created_at) = 2024

-- FAST: range query uses index
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- 4. SELECT only needed columns (avoid SELECT *)
-- SLOW: reads all columns
SELECT * FROM large_ml_table;

-- FAST: reads only needed columns (especially for columnar storage)
SELECT user_id, feature_1, feature_2, label FROM large_ml_table;
```

### EXPLAIN — Understanding Query Plans

```sql
-- MySQL/PostgreSQL: see query execution plan
EXPLAIN SELECT user_id, COUNT(*) FROM orders GROUP BY user_id;

-- Output shows:
-- type: ALL (full scan) vs range (index range) vs ref (index lookup) vs const (single row)
-- key: which index was used (NULL = no index)
-- rows: estimated rows examined
-- Extra: "Using filesort", "Using temporary" = slow operations to optimize

-- EXPLAIN ANALYZE (PostgreSQL): also shows actual execution time
EXPLAIN ANALYZE SELECT ...;
```

### Indexing Strategy for ML Pipelines

```sql
-- Indexes critical for ML feature queries:

-- 1. Primary key (always indexed)
-- 2. Foreign keys (always index these for JOINs)
CREATE INDEX idx_orders_user_id   ON orders(user_id);
CREATE INDEX idx_events_user_id   ON events(user_id);

-- 3. Time-range queries (partition + index)
CREATE INDEX idx_orders_date      ON orders(order_date);

-- 4. Composite index (column order matters!)
-- Use for: WHERE user_id = X AND order_date > Y
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
-- This index does NOT help: WHERE order_date > Y (missing leading column)

-- 5. Covering index (all query columns in index)
CREATE INDEX idx_orders_covering ON orders(user_id, order_date, amount);
-- Query hits index only, never touches the table!
```

---

## Interview Mastery

### Beginner Questions

---

**Q1: What is the difference between WHERE and HAVING?**

**Perfect Answer:**
> "WHERE filters rows **before** grouping and cannot use aggregate functions. HAVING filters groups **after** aggregation and is used with GROUP BY. For example, `WHERE salary > 50000` filters individual employees before counting, while `HAVING COUNT(*) > 5` keeps only departments with more than 5 employees after grouping."

**How to answer confidently:** Draw the SQL execution order on a whiteboard — WHERE runs at step 3, HAVING at step 5. Show a concrete example where using WHERE instead of HAVING gives a wrong answer.

**Common mistake:** Saying "HAVING is just WHERE for GROUP BY" — correct but incomplete. The key insight is that WHERE cannot reference aggregate functions because it executes before they're computed.

---

**Q2: What is a JOIN? Explain the difference between INNER, LEFT, and RIGHT JOIN.**

**Perfect Answer:**
> "A JOIN combines rows from two tables based on a related column. INNER JOIN returns only rows where the condition matches in BOTH tables. LEFT JOIN returns all rows from the left table plus matching rows from the right (NULLs where no match). RIGHT JOIN is the reverse. In ML, I use INNER JOIN when I need complete data for all records, and LEFT JOIN when I want to keep all users even if they have no events — using COALESCE to handle the NULLs."

**Interviewer expectation:** Show you understand the practical difference — not just definitions. Mention the fan-out problem (1-to-many joins create duplicate rows).

---

**Q3: What does NULL mean in SQL? How do you handle it?**

**Perfect Answer:**
> "NULL means the absence of a value — it's unknown, not zero or empty string. Three key rules: NULL compared to anything (including NULL itself) returns NULL/unknown, so you must use `IS NULL` not `= NULL`. Aggregate functions like COUNT(*) count NULLs, but SUM/AVG/MAX ignore NULLs. For ML feature engineering, I handle NULLs with COALESCE (return first non-NULL), NULLIF (return NULL if equal), or fill with 0/mean depending on domain knowledge."

---

### Intermediate Questions

---

**Q4: What are window functions and how are they different from GROUP BY?**

**Perfect Answer:**
> "Window functions perform calculations across related rows without collapsing them, unlike GROUP BY which reduces to one row per group. For example, `SUM(amount) OVER (PARTITION BY user_id ORDER BY date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)` computes a rolling 30-day sum for each row while keeping all rows intact. I use them heavily in ML for time-series features: LAG/LEAD for sequential patterns, NTILE for percentile features, RANK for ordering within groups. A key performance note: window functions require a sort, so they're heavier than simple aggregations."

**How to answer:** Demonstrate with a concrete example — show the input table, explain what GROUP BY would produce (one row per user) vs what a window function produces (all rows with the computed column).

---

**Q5: When would you use a CTE vs a subquery?**

**Perfect Answer:**
> "Both are logically equivalent in most databases, but CTEs have significant readability advantages for complex multi-step logic. I use CTEs when: (1) I need to reference the same subquery multiple times — CTEs avoid repetition; (2) the query has multiple transformation steps — each CTE is a readable stage; (3) I need recursive logic. I use subqueries for simple, one-off filters. Performance-wise, most modern query optimizers treat them identically, though in some databases CTEs may materialize (compute once and store) which can be faster or slower depending on context."

---

**Q6: Write a query to find the top 3 customers by revenue in each city.**

**Perfect Answer:**
```sql
WITH ranked_customers AS (
    SELECT
        c.customer_id,
        c.name,
        c.city,
        SUM(o.amount) AS total_revenue,
        ROW_NUMBER() OVER (PARTITION BY c.city ORDER BY SUM(o.amount) DESC) AS rank
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.name, c.city
)
SELECT customer_id, name, city, total_revenue
FROM ranked_customers
WHERE rank <= 3
ORDER BY city, rank;
```

**Interviewer expectation:** Use window function (ROW_NUMBER/RANK/DENSE_RANK). Explain the difference: ROW_NUMBER gives unique ranks even for ties, RANK skips ranks after ties (1,1,3), DENSE_RANK doesn't skip (1,1,2). For "top 3 unique customers" use ROW_NUMBER.

---

### Advanced Questions

---

**Q7: How do you build an ML training dataset from a production database? Walk me through your approach.**

**Perfect Answer:**
> "My approach has four stages: First, define the **observation window** — what time period does each training sample represent, and what's the **label window** (e.g., will this customer churn in the next 30 days?). Second, define the **feature lookback period** — I only use data available at prediction time (no data leakage). Third, join features from multiple tables using CTEs for readability — purchase history, behavioral data, demographics. Fourth, handle NULLs with COALESCE and verify class balance. The critical ML concern is temporal leakage: all features must be computed using data from BEFORE the observation date."

**SQL demonstration:**
```sql
WITH base AS (
    -- Each row is a user at a specific observation date
    SELECT user_id, '2024-06-01' AS obs_date FROM users
),
features AS (
    SELECT b.user_id,
           COUNT(o.order_id) AS orders_prior_30d
    FROM base b
    LEFT JOIN orders o ON b.user_id = o.user_id
        AND o.order_date >= DATE_SUB(b.obs_date, INTERVAL 30 DAY)
        AND o.order_date <  b.obs_date  -- STRICT: no data after obs_date
    GROUP BY b.user_id
)
SELECT * FROM features;
```

---

**Q8: What is the N+1 query problem and how do you fix it?**

**Perfect Answer:**
> "N+1 occurs when you execute 1 query to get N records, then N additional queries to get related data for each. For example: fetching 1000 users, then looping in Python to fetch each user's order count — that's 1001 queries. The fix is to JOIN or use a single aggregated query: `SELECT user_id, COUNT(*) FROM orders GROUP BY user_id` returns all users' order counts in one query. In ML pipelines this is critical — a 1000-row N+1 can be 1000x slower than a single JOIN."

---

**Q9: Write a SQL query to detect data drift between two time periods.**

**Perfect Answer:**
```sql
-- Compare feature distributions between training period and production
WITH training_stats AS (
    SELECT
        AVG(feature_1) AS mean_f1,
        STDDEV(feature_1) AS std_f1,
        MIN(feature_1) AS min_f1,
        MAX(feature_1) AS max_f1,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY feature_1) AS p25_f1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY feature_1) AS p75_f1
    FROM ml_features
    WHERE snapshot_date BETWEEN '2024-01-01' AND '2024-03-31'
),
production_stats AS (
    SELECT
        AVG(feature_1) AS mean_f1,
        STDDEV(feature_1) AS std_f1,
        MIN(feature_1) AS min_f1,
        MAX(feature_1) AS max_f1,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY feature_1) AS p25_f1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY feature_1) AS p75_f1
    FROM ml_features
    WHERE snapshot_date BETWEEN '2024-07-01' AND '2024-09-30'
)
SELECT
    t.mean_f1 AS train_mean,
    p.mean_f1 AS prod_mean,
    ABS(t.mean_f1 - p.mean_f1) / NULLIF(t.std_f1, 0) AS normalized_mean_shift,
    CASE
        WHEN ABS(t.mean_f1 - p.mean_f1) / NULLIF(t.std_f1, 0) > 2
        THEN 'DRIFT DETECTED'
        ELSE 'OK'
    END AS drift_status
FROM training_stats t, production_stats p;
```

---

**Q10: Scenario — Your ML pipeline is very slow due to a SQL query. How do you debug and fix it?**

**Perfect Answer:**
> "Step 1: Run `EXPLAIN ANALYZE` to get the actual query plan and identify bottlenecks — look for full table scans (type=ALL), missing index usage (key=NULL), high row estimates, or sort/temp table operations. Step 2: Check if the slow operation is a JOIN with no index on the join key — add index. Step 3: Check if WHERE clause uses functions on indexed columns — refactor to range queries. Step 4: Check for `SELECT *` — narrow to only needed columns, especially on columnar storage. Step 5: Check if data can be pre-aggregated — replace correlated subqueries with window functions. Step 6: Consider materialized views or pre-computed feature tables if the query runs repeatedly."

---

**Q11: Coding Question — Write SQL to compute the median of a column without PERCENTILE_CONT.**

**Perfect Answer:**
```sql
-- Works in MySQL which lacks PERCENTILE_CONT
WITH ordered AS (
    SELECT
        amount,
        ROW_NUMBER() OVER (ORDER BY amount) AS rn,
        COUNT(*) OVER () AS total
    FROM transactions
)
SELECT AVG(amount) AS median
FROM ordered
WHERE rn IN (
    FLOOR((total + 1) / 2.0),
    CEIL((total + 1) / 2.0)
);
-- For odd n: both FLOOR and CEIL point to the middle row
-- For even n: they point to the two middle rows, AVG gives the median
```

---

**Q12: ML System Design — How would you design a SQL-based feature store?**

**Perfect Answer:**
> "A feature store separates feature computation from model training/serving. My design:
>
> **Offline store (historical training):**  
> - Partitioned fact tables with point-in-time correct features  
> - Materialized views for expensive aggregations  
> - Snapshot tables with `as_of_date` for time-travel queries  
>
> **Online store (low-latency serving):**  
> - Key-value store (Redis/DynamoDB) for sub-millisecond lookup  
> - Populated by SQL pipeline writing latest features  
>
> **Point-in-time correctness:**  
> Always join features using `feature_timestamp <= prediction_date` to prevent leakage.
>
> **Schema:**"

```sql
CREATE TABLE feature_store (
    entity_id    BIGINT,
    feature_name VARCHAR(100),
    feature_value FLOAT,
    computed_at  TIMESTAMP,
    PRIMARY KEY (entity_id, feature_name, computed_at)
) PARTITION BY RANGE (computed_at);
```

---

### Quick Reference: Common Interview SQL Patterns

```sql
-- Running total
SUM(amount) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) AS running_total

-- Percentage of total
amount / SUM(amount) OVER () * 100 AS pct_of_total

-- Month-over-month growth
(current_month - prev_month) / prev_month * 100 AS mom_growth

-- Deduplicate (keep latest record)
ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) AS rn
-- then WHERE rn = 1

-- Find gaps in sequence
id - ROW_NUMBER() OVER (ORDER BY id) AS gap_group

-- Moving average
AVG(amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

-- First/last value in group
FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY date) AS first_purchase
LAST_VALUE(amount)  OVER (PARTITION BY user_id ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_purchase
```

---

## Summary: SQL Concepts Map for AI/ML

```mermaid
mindmap
  root((SQL for AI/ML))
    Data Extraction
      SELECT
      WHERE
      JOINS
      Subqueries
    Aggregation
      GROUP BY
      HAVING
      Aggregate Functions
    Advanced Queries
      CTEs
      Window Functions
      Recursive CTEs
    Feature Engineering
      CASE WHEN encoding
      Date/time features
      Binning & normalization
      RFM analysis
    ML Pipelines
      Training dataset assembly
      Point-in-time correctness
      Label creation
      Data drift detection
    Data Warehousing
      Star schema
      OLTP vs OLAP
      Partitioning
      SCD Type 2
    Performance
      Indexes
      EXPLAIN
      Query optimization
      Columnar storage
```

---

[⬇️ Download This File](#)

---

*Phase 6 Complete. Waiting for confirmation to proceed to Phase 7 — Data Visualization.*
