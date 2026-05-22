# Phase 9 — Views

> **Curriculum:** DBMS A-to-Z | **File:** `09_views.md`  
> **Prerequisites:** Phases 1–8 (Foundations → Advanced Queries)

---

## Table of Contents

1. [What Is a View?](#1-what-is-a-view)
2. [Creating, Altering & Dropping Views](#2-creating-altering--dropping-views)
3. [Simple Views vs Complex Views](#3-simple-views-vs-complex-views)
4. [Updatable Views](#4-updatable-views)
5. [WITH CHECK OPTION](#5-with-check-option)
6. [Materialized Views](#6-materialized-views)
7. [Refresh Strategies for Materialized Views](#7-refresh-strategies-for-materialized-views)
8. [Views for Security & Access Control](#8-views-for-security--access-control)
9. [Views for Abstraction & Simplification](#9-views-for-abstraction--simplification)
10. [Performance: View vs CTE vs Subquery vs Temp Table](#10-performance-view-vs-cte-vs-subquery-vs-temp-table)
11. [Worked Examples](#11-worked-examples)
12. [Common Interview Questions](#12-common-interview-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. What Is a View?

A **view** is a **named, stored query** saved in the database catalog. When you query a view, the database executes the underlying query and returns the result as if it were a real table.

### Key Properties

| Property | Explanation |
|----------|-------------|
| **Stored query** | The view definition (SQL) is saved; data is not duplicated |
| **Virtual table** | Looks and behaves like a table to the caller |
| **Always fresh** | Each query against a regular view re-executes the underlying SQL |
| **No storage** | A regular view occupies no data storage (unlike a materialized view) |

### Real-World Analogy

A view is like a **window** cut into a wall. You see a live picture of whatever is outside — but you are not touching the outside world directly. The view does not store what you see; it just frames it. If the outside changes, the view instantly reflects it.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="200" viewBox="0 0 560 200">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <!-- Base tables -->
  <rect x="10" y="30" width="130" height="60" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="75" y="56" text-anchor="middle" font-weight="bold" fill="#2D6A2D">employees</text>
  <text x="75" y="76" text-anchor="middle" fill="#2D6A2D">(base table)</text>

  <rect x="10" y="120" width="130" height="60" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="75" y="146" text-anchor="middle" font-weight="bold" fill="#2D6A2D">departments</text>
  <text x="75" y="166" text-anchor="middle" fill="#2D6A2D">(base table)</text>

  <!-- Arrows to view -->
  <line x1="140" y1="60"  x2="210" y2="95" stroke="#555" stroke-width="1.5" stroke-dasharray="5" marker-end="url(#av)"/>
  <line x1="140" y1="150" x2="210" y2="115" stroke="#555" stroke-width="1.5" stroke-dasharray="5" marker-end="url(#av)"/>
  <defs><marker id="av" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- View box -->
  <rect x="210" y="65" width="150" height="70" fill="#DAE8FC" stroke="#2980B9" rx="6"/>
  <text x="285" y="90" text-anchor="middle" font-weight="bold" fill="#1A5276">VIEW</text>
  <text x="285" y="108" text-anchor="middle" fill="#1A5276">emp_dept_view</text>
  <text x="285" y="124" text-anchor="middle" fill="#7D3C98" font-size="10" font-style="italic">stored query only</text>

  <!-- Arrow to application -->
  <line x1="360" y1="100" x2="430" y2="100" stroke="#E74C3C" stroke-width="2" marker-end="url(#av2)"/>
  <defs><marker id="av2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/>
  </marker></defs>
  <text x="395" y="90" text-anchor="middle" fill="#C0392B" font-size="10">result set</text>

  <rect x="430" y="65" width="120" height="70" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="490" y="96" text-anchor="middle" font-weight="bold" fill="#784212">Application</text>
  <text x="490" y="116" text-anchor="middle" fill="#784212">/ Analyst</text>

  <text x="280" y="185" fill="#555" font-size="11" font-style="italic" text-anchor="middle">Data is never copied — query is re-executed on every SELECT from the view</text>
</svg>
```

### Sample Schema

```sql
CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(50),
    dept_id    INT,
    manager_id INT,
    salary     NUMERIC(10,2),
    hire_date  DATE,
    is_active  BOOLEAN DEFAULT TRUE
);

CREATE TABLE departments (
    dept_id   INT PRIMARY KEY,
    dept_name VARCHAR(50),
    budget    NUMERIC(12,2),
    region    VARCHAR(30)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    status      VARCHAR(20)
);

CREATE TABLE order_items (
    item_id    INT PRIMARY KEY,
    order_id   INT,
    product_id INT,
    qty        INT,
    unit_price NUMERIC(10,2)
);
```

---

## 2. Creating, Altering & Dropping Views

### CREATE VIEW

```sql
CREATE VIEW view_name AS
SELECT col1, col2, ...
FROM   base_table
WHERE  condition;
```

### CREATE OR REPLACE VIEW

Re-defines the view without dropping it (preserves existing permissions):

```sql
CREATE OR REPLACE VIEW active_employees AS
SELECT emp_id, name, dept_id, salary
FROM   employees
WHERE  is_active = TRUE;
```

### ALTER VIEW (SQL Server / Oracle syntax)

```sql
-- SQL Server
ALTER VIEW active_employees AS
SELECT emp_id, name, dept_id, salary, hire_date
FROM   employees
WHERE  is_active = TRUE;

-- PostgreSQL: use CREATE OR REPLACE VIEW
```

### DROP VIEW

```sql
DROP VIEW view_name;

-- Drop only if it exists (avoids error)
DROP VIEW IF EXISTS view_name;

-- Cascade: also drop views that depend on this one
DROP VIEW view_name CASCADE;    -- PostgreSQL
```

### Viewing the definition of an existing view

```sql
-- PostgreSQL
SELECT definition FROM pg_views WHERE viewname = 'active_employees';

-- MySQL
SHOW CREATE VIEW active_employees;

-- SQL Server
SELECT OBJECT_DEFINITION(OBJECT_ID('active_employees'));
```

---

## 3. Simple Views vs Complex Views

### Simple Views

A simple view queries **a single base table**, contains no:
- Aggregate functions (`SUM`, `AVG`, `COUNT`, …)
- `GROUP BY` / `HAVING`
- `DISTINCT`
- Subqueries
- Joins to other tables

Simple views are typically **updatable** (see Section 4).

```sql
-- Simple view: subset of columns from one table
CREATE VIEW active_employees AS
SELECT emp_id, name, dept_id, salary
FROM   employees
WHERE  is_active = TRUE;

-- Simple view: renamed columns
CREATE VIEW emp_summary AS
SELECT emp_id   AS id,
       name     AS full_name,
       salary   AS annual_salary
FROM   employees;
```

### Complex Views

A complex view involves any of: joins, aggregations, subqueries, DISTINCT, UNION, window functions. Complex views are generally **not updatable**.

```sql
-- Complex view: join across two tables
CREATE VIEW emp_with_dept AS
SELECT e.emp_id,
       e.name,
       e.salary,
       d.dept_name,
       d.region
FROM   employees   e
JOIN   departments d ON e.dept_id = d.dept_id
WHERE  e.is_active = TRUE;

-- Complex view: aggregation
CREATE VIEW dept_salary_stats AS
SELECT d.dept_name,
       COUNT(e.emp_id)       AS headcount,
       AVG(e.salary)         AS avg_salary,
       MAX(e.salary)         AS max_salary,
       SUM(e.salary)         AS total_payroll
FROM   departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;

-- Complex view: subquery + window function
CREATE VIEW ranked_employees AS
SELECT emp_id,
       name,
       salary,
       dept_id,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS salary_rank
FROM   employees
WHERE  is_active = TRUE;
```

### Side-by-Side Comparison

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="200" viewBox="0 0 620 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.h{font-weight:bold;font-size:12px}</style>

  <!-- Header -->
  <rect x="0" y="0" width="620" height="30" fill="#2C3E50" rx="4"/>
  <text x="155" y="20" text-anchor="middle" fill="white" class="h">Feature</text>
  <text x="355" y="20" text-anchor="middle" fill="white" class="h">Simple View</text>
  <text x="520" y="20" text-anchor="middle" fill="white" class="h">Complex View</text>

  <!-- Row 1 -->
  <rect x="0"   y="30" width="620" height="30" fill="#EBF5FB"/>
  <text x="155" y="50" text-anchor="middle" fill="#333">Tables involved</text>
  <text x="355" y="50" text-anchor="middle" fill="#27AE60">Single base table</text>
  <text x="520" y="50" text-anchor="middle" fill="#E67E22">Multiple / joined tables</text>

  <!-- Row 2 -->
  <rect x="0"   y="60" width="620" height="30" fill="#FDFEFE"/>
  <text x="155" y="80" text-anchor="middle" fill="#333">Aggregation allowed</text>
  <text x="355" y="80" text-anchor="middle" fill="#E74C3C">❌ No</text>
  <text x="520" y="80" text-anchor="middle" fill="#27AE60">✅ Yes</text>

  <!-- Row 3 -->
  <rect x="0"   y="90" width="620" height="30" fill="#EBF5FB"/>
  <text x="155" y="110" text-anchor="middle" fill="#333">Updatable (INSERT/UPDATE)</text>
  <text x="355" y="110" text-anchor="middle" fill="#27AE60">✅ Usually yes</text>
  <text x="520" y="110" text-anchor="middle" fill="#E74C3C">❌ Generally no</text>

  <!-- Row 4 -->
  <rect x="0"   y="120" width="620" height="30" fill="#FDFEFE"/>
  <text x="155" y="140" text-anchor="middle" fill="#333">WITH CHECK OPTION</text>
  <text x="355" y="140" text-anchor="middle" fill="#27AE60">✅ Supported</text>
  <text x="520" y="140" text-anchor="middle" fill="#E74C3C">❌ Not applicable</text>

  <!-- Row 5 -->
  <rect x="0"   y="150" width="620" height="30" fill="#EBF5FB"/>
  <text x="155" y="170" text-anchor="middle" fill="#333">Typical use</text>
  <text x="355" y="170" text-anchor="middle" fill="#555">Security filters, column masks</text>
  <text x="520" y="170" text-anchor="middle" fill="#555">Reporting, dashboards, analytics</text>
</svg>
```

---

## 4. Updatable Views

### Rules for a View to Be Updatable

A view is updatable when the database can map a DML statement on the view **directly back** to the base table. The rules (SQL standard + most RDBMS):

1. Derived from exactly **one base table** (no joins, no unions).
2. No **aggregate functions** (`SUM`, `AVG`, `COUNT`, …).
3. No **GROUP BY** or **HAVING**.
4. No **DISTINCT**.
5. No **subqueries in the SELECT list**.
6. All NOT NULL columns without defaults in the base table must be **included** in the view (for INSERT to work).

### Example — Updatable view

```sql
CREATE VIEW active_employees AS
SELECT emp_id, name, dept_id, salary
FROM   employees
WHERE  is_active = TRUE;

-- INSERT through the view (inserts into employees table)
INSERT INTO active_employees (emp_id, name, dept_id, salary)
VALUES (10, 'Frank', 10, 72000);
-- Row is inserted into employees with is_active defaulting to TRUE

-- UPDATE through the view
UPDATE active_employees
SET    salary = 75000
WHERE  emp_id = 10;
-- Translates directly to: UPDATE employees SET salary = 75000 WHERE emp_id = 10

-- DELETE through the view
DELETE FROM active_employees WHERE emp_id = 10;
-- Translates to: DELETE FROM employees WHERE emp_id = 10
```

### The Invisible Row Problem

Without `WITH CHECK OPTION`, you can insert a row that immediately disappears from the view:

```sql
-- Insert an INACTIVE employee through the active_employees view
INSERT INTO active_employees (emp_id, name, dept_id, salary)
VALUES (11, 'Ghost', 20, 65000);
-- This inserts with is_active = FALSE (or the default)
-- The row immediately vanishes from the view — but exists in the base table

SELECT * FROM active_employees WHERE emp_id = 11;   -- returns 0 rows!
SELECT * FROM employees      WHERE emp_id = 11;   -- returns the row
```

This is the exact problem `WITH CHECK OPTION` solves.

---

## 5. WITH CHECK OPTION

### Concept

`WITH CHECK OPTION` prevents any INSERT or UPDATE through the view that would cause the modified row to **fall outside the view's WHERE condition**. If the operation would make the row invisible through the view, it is rejected with an error.

### Syntax

```sql
CREATE VIEW view_name AS
SELECT ...
FROM   table
WHERE  condition
WITH CHECK OPTION;
```

### Example

```sql
CREATE VIEW active_employees AS
SELECT emp_id, name, dept_id, salary
FROM   employees
WHERE  is_active = TRUE
WITH CHECK OPTION;

-- This INSERT is REJECTED because is_active would not be TRUE:
INSERT INTO active_employees (emp_id, name, dept_id, salary)
VALUES (12, 'Phantom', 20, 65000);
-- Error: new row violates check option for view "active_employees"

-- This INSERT succeeds (is_active defaults to TRUE):
INSERT INTO active_employees (emp_id, name, dept_id, salary)
VALUES (12, 'Heidi', 20, 65000);

-- This UPDATE is REJECTED because it would move the row out of the view:
UPDATE active_employees SET is_active = FALSE WHERE emp_id = 12;
-- Error: check option violation
```

### CASCADED vs LOCAL (for stacked views)

When views are built on top of other views, `WITH CHECK OPTION` has two modes:

```sql
-- View 1: base filter
CREATE VIEW emp_active AS
SELECT * FROM employees WHERE is_active = TRUE
WITH CHECK OPTION;

-- View 2: built on View 1, adds salary filter
CREATE VIEW emp_active_senior AS
SELECT * FROM emp_active WHERE salary >= 80000
WITH LOCAL CHECK OPTION;     -- only checks THIS view's condition
-- vs
WITH CASCADED CHECK OPTION;  -- checks ALL view conditions in the chain (default)
```

| Mode | Behaviour |
|------|-----------|
| `CASCADED` (default) | Checks all views up the chain — strictest, safest |
| `LOCAL` | Checks only the current view's condition |

---

## 6. Materialized Views

### Concept

A **materialized view** is like a regular view, but its result set is **physically stored on disk**. Instead of re-executing the query on every SELECT, the database reads the stored snapshot. This makes reads extremely fast at the cost of storage and potential staleness.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="620" height="220" viewBox="0 0 620 220">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <!-- Regular View side -->
  <rect x="10" y="10" width="270" height="180" fill="#EBF5FB" stroke="#2980B9" rx="8"/>
  <text x="145" y="36" text-anchor="middle" font-weight="bold" fill="#1A5276" font-size="13">Regular View</text>
  <rect x="30" y="50" width="100" height="40" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="80" y="75" text-anchor="middle" fill="#2D6A2D">Base Tables</text>
  <line x1="130" y1="70" x2="175" y2="70" stroke="#555" stroke-width="1.5" marker-end="url(#amv)"/>
  <defs><marker id="amv" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>
  <rect x="175" y="50" width="90" height="40" fill="#DAE8FC" stroke="#6C8EBF" rx="4"/>
  <text x="220" y="75" text-anchor="middle" fill="#1A5276">Query</text>
  <text x="145" y="130" text-anchor="middle" fill="#555">Every SELECT re-executes</text>
  <text x="145" y="148" text-anchor="middle" fill="#555">the full query</text>
  <text x="145" y="168" text-anchor="middle" fill="#E74C3C">Slow for heavy aggregations</text>

  <!-- Materialized View side -->
  <rect x="340" y="10" width="270" height="180" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="475" y="36" text-anchor="middle" font-weight="bold" fill="#145A32" font-size="13">Materialized View</text>
  <rect x="360" y="50" width="100" height="40" fill="#D5E8D4" stroke="#82B366" rx="4"/>
  <text x="410" y="75" text-anchor="middle" fill="#2D6A2D">Base Tables</text>
  <line x1="460" y1="70" x2="505" y2="70" stroke="#555" stroke-width="1.5" marker-end="url(#amv)"/>
  <rect x="505" y="50" width="90" height="40" fill="#A9DFBF" stroke="#1E8449" rx="4"/>
  <text x="550" y="68" text-anchor="middle" fill="#145A32">Stored</text>
  <text x="550" y="84" text-anchor="middle" fill="#145A32">Snapshot</text>
  <text x="475" y="130" text-anchor="middle" fill="#555">SELECT reads stored data</text>
  <text x="475" y="148" text-anchor="middle" fill="#555">— no query re-execution</text>
  <text x="475" y="168" text-anchor="middle" fill="#27AE60">Fast reads, needs refresh</text>

  <text x="310" y="210" fill="#7D3C98" font-size="11" font-style="italic" text-anchor="middle">Trade-off: read speed vs data freshness</text>
</svg>
```

### Creating a Materialized View (PostgreSQL)

```sql
CREATE MATERIALIZED VIEW dept_revenue_summary AS
SELECT d.dept_name,
       COUNT(DISTINCT o.order_id)        AS order_count,
       SUM(oi.qty * oi.unit_price)       AS total_revenue,
       AVG(oi.qty * oi.unit_price)       AS avg_order_value
FROM   departments  d
JOIN   employees    e  ON d.dept_id     = e.dept_id
JOIN   orders       o  ON e.emp_id      = o.customer_id
JOIN   order_items  oi ON o.order_id    = oi.order_id
GROUP BY d.dept_name
WITH DATA;                    -- populate immediately
-- WITH NO DATA populates the structure but defers the initial query
```

### Creating a Materialized View (Oracle)

```sql
CREATE MATERIALIZED VIEW dept_revenue_summary
BUILD IMMEDIATE
REFRESH FAST ON COMMIT
AS
SELECT d.dept_name, SUM(oi.qty * oi.unit_price) AS total_revenue
FROM   departments d
JOIN   employees   e  ON d.dept_id = e.dept_id
JOIN   order_items oi ON e.emp_id  = oi.order_id
GROUP BY d.dept_name;
```

### Querying a Materialized View (same as a table)

```sql
SELECT * FROM dept_revenue_summary
WHERE  total_revenue > 100000
ORDER BY total_revenue DESC;
```

---

## 7. Refresh Strategies for Materialized Views

Since data is stored as a snapshot, it must be periodically refreshed when the base tables change.

### Refresh Methods

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="250" viewBox="0 0 680 250">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.h{font-size:12px;font-weight:bold}</style>

  <!-- FULL REFRESH -->
  <rect x="10" y="10" width="200" height="200" fill="#EBF5FB" stroke="#2980B9" rx="8"/>
  <text x="110" y="34" text-anchor="middle" class="h" fill="#1A5276">Full Refresh</text>
  <line x1="20" y1="42" x2="200" y2="42" stroke="#AED6F1"/>
  <text x="110" y="62" text-anchor="middle" fill="#333">Truncates and re-runs</text>
  <text x="110" y="78" text-anchor="middle" fill="#333">the entire query</text>
  <text x="110" y="106" text-anchor="middle" fill="#27AE60">✓ Always consistent</text>
  <text x="110" y="124" text-anchor="middle" fill="#E74C3C">✗ Slow for large data</text>
  <text x="110" y="142" text-anchor="middle" fill="#E74C3C">✗ View locked during refresh</text>
  <text x="110" y="175" text-anchor="middle" fill="#7D6608" font-style="italic">Best for: small views,</text>
  <text x="110" y="191" text-anchor="middle" fill="#7D6608" font-style="italic">nightly batch jobs</text>

  <!-- INCREMENTAL / FAST REFRESH -->
  <rect x="240" y="10" width="200" height="200" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="340" y="34" text-anchor="middle" class="h" fill="#145A32">Fast / Incremental</text>
  <line x1="250" y1="42" x2="430" y2="42" stroke="#A9DFBF"/>
  <text x="340" y="62" text-anchor="middle" fill="#333">Applies only the changes</text>
  <text x="340" y="78" text-anchor="middle" fill="#333">since last refresh</text>
  <text x="340" y="106" text-anchor="middle" fill="#27AE60">✓ Much faster</text>
  <text x="340" y="124" text-anchor="middle" fill="#27AE60">✓ Less I/O</text>
  <text x="340" y="142" text-anchor="middle" fill="#E74C3C">✗ Requires change logs</text>
  <text x="340" y="160" text-anchor="middle" fill="#E74C3C">✗ Not all queries qualify</text>
  <text x="340" y="182" text-anchor="middle" fill="#7D6608" font-style="italic">Best for: high-frequency</text>
  <text x="340" y="198" text-anchor="middle" fill="#7D6608" font-style="italic">updates, large tables</text>

  <!-- CONCURRENT REFRESH -->
  <rect x="470" y="10" width="200" height="200" fill="#FEF9E7" stroke="#F39C12" rx="8"/>
  <text x="570" y="34" text-anchor="middle" class="h" fill="#7D6608">Concurrent Refresh</text>
  <line x1="480" y1="42" x2="660" y2="42" stroke="#FAD7A0"/>
  <text x="570" y="62" text-anchor="middle" fill="#333">Full refresh without</text>
  <text x="570" y="78" text-anchor="middle" fill="#333">blocking readers</text>
  <text x="570" y="106" text-anchor="middle" fill="#27AE60">✓ No read lock</text>
  <text x="570" y="124" text-anchor="middle" fill="#27AE60">✓ Safe for production</text>
  <text x="570" y="142" text-anchor="middle" fill="#E74C3C">✗ Needs a UNIQUE index</text>
  <text x="570" y="160" text-anchor="middle" fill="#E74C3C">✗ Slightly slower than full</text>
  <text x="570" y="182" text-anchor="middle" fill="#7D6608" font-style="italic">Best for: production views</text>
  <text x="570" y="198" text-anchor="middle" fill="#7D6608" font-style="italic">queried constantly</text>
</svg>
```

### PostgreSQL Refresh Commands

```sql
-- Full refresh (blocks readers during refresh)
REFRESH MATERIALIZED VIEW dept_revenue_summary;

-- Concurrent refresh (no read lock — requires UNIQUE index)
CREATE UNIQUE INDEX ON dept_revenue_summary (dept_name);
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_revenue_summary;
```

### Oracle Refresh Modes

```sql
-- Manual full refresh
EXEC DBMS_MVIEW.REFRESH('dept_revenue_summary', 'C');   -- 'C' = complete

-- Manual fast (incremental) refresh
EXEC DBMS_MVIEW.REFRESH('dept_revenue_summary', 'F');   -- 'F' = fast

-- Automatic refresh on base table commit
CREATE MATERIALIZED VIEW ... REFRESH FAST ON COMMIT ...

-- Scheduled refresh
CREATE MATERIALIZED VIEW ... REFRESH COMPLETE START WITH SYSDATE
    NEXT SYSDATE + 1/24;   -- every hour
```

### Scheduling Refreshes (PostgreSQL with pg_cron)

```sql
-- Using pg_cron extension: refresh every hour
SELECT cron.schedule('0 * * * *', $$
    REFRESH MATERIALIZED VIEW CONCURRENTLY dept_revenue_summary;
$$);
```

### Staleness Check

```sql
-- PostgreSQL: when was the matview last refreshed?
SELECT schemaname, matviewname, last_refresh
FROM   pg_stat_user_tables          -- approximate; check pg_matviews
JOIN   pg_matviews USING (matviewname);

-- Or check the system catalog directly
SELECT matviewname,
       last_refresh
FROM   pg_matviews
WHERE  matviewname = 'dept_revenue_summary';
```

---

## 8. Views for Security & Access Control

One of the most powerful uses of views is **restricting what users can see** without changing the underlying tables.

### Pattern 1 — Column-level security: hide sensitive columns

```sql
-- Base table has salary and SSN — not everyone should see those
CREATE VIEW employees_public AS
SELECT emp_id, name, dept_id, hire_date
FROM   employees;

-- Grant the view, not the table
GRANT SELECT ON employees_public TO analyst_role;
REVOKE SELECT ON employees FROM analyst_role;
```

Analysts can now query `employees_public` but cannot access salary or SSN even indirectly.

### Pattern 2 — Row-level security: restrict by region

```sql
-- Finance team sees only their region's data
CREATE VIEW finance_emea AS
SELECT e.emp_id, e.name, e.salary, d.dept_name
FROM   employees   e
JOIN   departments d ON e.dept_id = d.dept_id
WHERE  d.region = 'EMEA';

GRANT SELECT ON finance_emea TO emea_finance_role;
```

### Pattern 3 — Obfuscation / data masking

```sql
-- HR analysts see masked salary bands, not exact figures
CREATE VIEW employees_masked AS
SELECT emp_id,
       name,
       dept_id,
       CASE
           WHEN salary >= 90000 THEN 'Band A'
           WHEN salary >= 70000 THEN 'Band B'
           ELSE                      'Band C'
       END AS salary_band,
       -- Mask SSN: show only last 4 digits
       '***-**-' || RIGHT(ssn, 4) AS ssn_masked
FROM   employees;
```

### Pattern 4 — Audit trail view

```sql
-- A read-only summary of who did what
CREATE VIEW audit_summary AS
SELECT u.username,
       a.action,
       a.table_name,
       a.record_id,
       a.changed_at
FROM   audit_log a
JOIN   users     u ON a.user_id = u.user_id
WHERE  a.changed_at >= CURRENT_DATE - INTERVAL '30 days';

GRANT SELECT ON audit_summary TO security_team;
```

### Security View Architecture

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="580" height="240" viewBox="0 0 580 240">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Base table -->
  <rect x="220" y="10" width="140" height="60" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="290" y="34" text-anchor="middle" font-weight="bold" fill="#2D6A2D">employees</text>
  <text x="290" y="54" text-anchor="middle" fill="#2D6A2D">ALL columns + ALL rows</text>

  <!-- Arrow down to views -->
  <line x1="250" y1="70" x2="120" y2="120" stroke="#555" stroke-width="1.5" stroke-dasharray="4" marker-end="url(#asec)"/>
  <line x1="290" y1="70" x2="290" y2="120" stroke="#555" stroke-width="1.5" stroke-dasharray="4" marker-end="url(#asec)"/>
  <line x1="330" y1="70" x2="460" y2="120" stroke="#555" stroke-width="1.5" stroke-dasharray="4" marker-end="url(#asec)"/>
  <defs><marker id="asec" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <!-- View 1 -->
  <rect x="30" y="120" width="160" height="55" fill="#DAE8FC" stroke="#2980B9" rx="6"/>
  <text x="110" y="143" text-anchor="middle" font-weight="bold" fill="#1A5276">employees_public</text>
  <text x="110" y="163" text-anchor="middle" fill="#555">No salary, no SSN</text>

  <!-- View 2 -->
  <rect x="210" y="120" width="160" height="55" fill="#DAE8FC" stroke="#2980B9" rx="6"/>
  <text x="290" y="143" text-anchor="middle" font-weight="bold" fill="#1A5276">employees_masked</text>
  <text x="290" y="163" text-anchor="middle" fill="#555">Salary → Band, SSN masked</text>

  <!-- View 3 -->
  <rect x="390" y="120" width="160" height="55" fill="#DAE8FC" stroke="#2980B9" rx="6"/>
  <text x="470" y="143" text-anchor="middle" font-weight="bold" fill="#1A5276">finance_emea</text>
  <text x="470" y="163" text-anchor="middle" fill="#555">EMEA region only</text>

  <!-- Roles -->
  <rect x="30"  y="205" width="160" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="110" y="224" text-anchor="middle" fill="#784212">analyst_role</text>
  <rect x="210" y="205" width="160" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="290" y="224" text-anchor="middle" fill="#784212">hr_role</text>
  <rect x="390" y="205" width="160" height="30" fill="#FAD7A0" stroke="#E67E22" rx="4"/>
  <text x="470" y="224" text-anchor="middle" fill="#784212">emea_finance_role</text>
</svg>
```

---

## 9. Views for Abstraction & Simplification

Beyond security, views are excellent **API contracts** between the database schema and its consumers.

### Pattern 1 — Hide join complexity from application developers

```sql
-- Instead of writing this 4-table join every time...
SELECT e.name, d.dept_name, d.region,
       m.name AS manager,
       e.salary, e.hire_date
FROM   employees   e
JOIN   departments d  ON e.dept_id    = d.dept_id
LEFT JOIN employees m ON e.manager_id = m.emp_id
WHERE  e.is_active = TRUE;

-- Define it once as a view:
CREATE VIEW employee_directory AS
SELECT e.emp_id,
       e.name,
       d.dept_name,
       d.region,
       m.name     AS manager_name,
       e.salary,
       e.hire_date
FROM   employees   e
JOIN   departments d  ON e.dept_id    = d.dept_id
LEFT JOIN employees m ON e.manager_id = m.emp_id
WHERE  e.is_active = TRUE;

-- Application code becomes simple:
SELECT * FROM employee_directory WHERE region = 'EMEA';
```

### Pattern 2 — Schema migration buffer (rename a table transparently)

```sql
-- Old name used by legacy code: employee_records
-- New name after refactor: employees
-- Keep old name as a view to avoid breaking existing queries

CREATE VIEW employee_records AS
SELECT * FROM employees;
```

### Pattern 3 — Provide a stable interface during schema evolution

```sql
-- v1 view (stable name used by reporting)
CREATE VIEW sales_report AS
SELECT o.order_id, o.order_date,
       c.customer_name,
       SUM(oi.qty * oi.unit_price) AS order_total
FROM   orders      o
JOIN   customers   c  ON o.customer_id  = c.customer_id
JOIN   order_items oi ON o.order_id     = oi.order_id
GROUP BY o.order_id, o.order_date, c.customer_name;

-- When schema changes (new tables, renamed columns), redefine the view.
-- Callers SELECT FROM sales_report with no changes to their code.
CREATE OR REPLACE VIEW sales_report AS
SELECT o.order_id, o.order_date,
       c.customer_name,
       c.tier,                        -- newly added column
       SUM(oi.qty * oi.unit_price) AS order_total
FROM   orders_v2    o                 -- renamed table
JOIN   customers    c  ON o.cust_id  = c.customer_id   -- renamed column
JOIN   order_items  oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.order_date, c.customer_name, c.tier;
```

---

## 10. Performance: View vs CTE vs Subquery vs Temp Table

### How Regular Views Are Executed

In most databases (PostgreSQL, MySQL, SQL Server), a regular view is **inlined** at query parse time. The database substitutes the view's definition wherever the view name appears — identical to writing the subquery inline. There is no performance benefit from a regular view over a subquery.

```sql
-- These produce identical query plans:
SELECT * FROM active_employees WHERE salary > 80000;

SELECT * FROM (
    SELECT emp_id, name, dept_id, salary
    FROM   employees WHERE is_active = TRUE
) t WHERE salary > 80000;
```

### Performance Comparison

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="700" height="250" viewBox="0 0 700 250">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <rect x="0" y="0" width="700" height="30" fill="#2C3E50" rx="4"/>
  <text x="80"  y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Approach</text>
  <text x="200" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Storage</text>
  <text x="320" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Read Speed</text>
  <text x="440" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Write Overhead</text>
  <text x="580" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Best For</text>

  <rect x="0" y="30" width="700" height="38" fill="#EBF5FB"/>
  <text x="80"  y="53" text-anchor="middle" fill="#1A5276" font-weight="bold">Regular View</text>
  <text x="200" y="53" text-anchor="middle" fill="#27AE60">None (virtual)</text>
  <text x="320" y="53" text-anchor="middle" fill="#E67E22">Same as inline SQL</text>
  <text x="440" y="53" text-anchor="middle" fill="#27AE60">None</text>
  <text x="580" y="53" text-anchor="middle" fill="#555">Reuse, security, abstraction</text>

  <rect x="0" y="68" width="700" height="38" fill="#FDFEFE"/>
  <text x="80"  y="91" text-anchor="middle" fill="#1A5276" font-weight="bold">Materialized View</text>
  <text x="200" y="91" text-anchor="middle" fill="#E74C3C">On disk (full result)</text>
  <text x="320" y="91" text-anchor="middle" fill="#27AE60">Very fast (table scan)</text>
  <text x="440" y="91" text-anchor="middle" fill="#E74C3C">Refresh cost</text>
  <text x="580" y="91" text-anchor="middle" fill="#555">Heavy aggregations, dashboards</text>

  <rect x="0" y="106" width="700" height="38" fill="#EBF5FB"/>
  <text x="80"  y="129" text-anchor="middle" fill="#1A5276" font-weight="bold">CTE</text>
  <text x="200" y="129" text-anchor="middle" fill="#27AE60">None (virtual)</text>
  <text x="320" y="129" text-anchor="middle" fill="#E67E22">Same as inline SQL*</text>
  <text x="440" y="129" text-anchor="middle" fill="#27AE60">None</text>
  <text x="580" y="129" text-anchor="middle" fill="#555">Complex in-query logic, recursion</text>

  <rect x="0" y="144" width="700" height="38" fill="#FDFEFE"/>
  <text x="80"  y="167" text-anchor="middle" fill="#1A5276" font-weight="bold">Subquery</text>
  <text x="200" y="167" text-anchor="middle" fill="#27AE60">None (virtual)</text>
  <text x="320" y="167" text-anchor="middle" fill="#E67E22">Same as inline SQL</text>
  <text x="440" y="167" text-anchor="middle" fill="#27AE60">None</text>
  <text x="580" y="167" text-anchor="middle" fill="#555">One-off inline logic</text>

  <rect x="0" y="182" width="700" height="38" fill="#EBF5FB"/>
  <text x="80"  y="205" text-anchor="middle" fill="#1A5276" font-weight="bold">Temp Table</text>
  <text x="200" y="205" text-anchor="middle" fill="#E74C3C">Temp segment on disk</text>
  <text x="320" y="205" text-anchor="middle" fill="#27AE60">Fast (can be indexed)</text>
  <text x="440" y="205" text-anchor="middle" fill="#E74C3C">Write + index cost</text>
  <text x="580" y="205" text-anchor="middle" fill="#555">Multi-step ETL, re-used mid-proc</text>

  <text x="350" y="240" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">* PostgreSQL CTEs may materialise by default — use AS NOT MATERIALIZED to allow inlining.</text>
</svg>
```

### When to choose what

| Need | Use |
|------|-----|
| Simplify queries, enforce naming | Regular view |
| Hide columns / rows for security | Regular view |
| Speed up heavy pre-aggregated reports | Materialized view |
| Complex multi-step logic within one query | CTE |
| Inline one-off filter or subselect | Subquery / derived table |
| Build a pipeline with indexable intermediate result | Temp table |

---

## 11. Worked Examples

### Example 1 — Security: public employee directory (no salary)

```sql
CREATE VIEW employee_directory AS
SELECT e.emp_id,
       e.name,
       d.dept_name,
       d.region,
       e.hire_date
FROM   employees   e
JOIN   departments d ON e.dept_id = d.dept_id
WHERE  e.is_active = TRUE;

GRANT SELECT ON employee_directory TO all_staff;
REVOKE SELECT ON employees         FROM all_staff;
```

### Example 2 — Updatable view with CHECK OPTION

```sql
CREATE VIEW engineering_employees AS
SELECT emp_id, name, salary, is_active
FROM   employees
WHERE  dept_id = 10
WITH CHECK OPTION;

-- Allowed: update salary of an Engineering employee
UPDATE engineering_employees SET salary = 91000 WHERE emp_id = 3;

-- Blocked: try to move emp to a different dept through this view
UPDATE engineering_employees SET dept_id = 20 WHERE emp_id = 3;
-- Error: check option violation (row would leave the view's WHERE scope)
```

### Example 3 — Reporting view with aggregation

```sql
CREATE VIEW monthly_sales_summary AS
SELECT DATE_TRUNC('month', o.order_date)   AS month,
       p.category,
       COUNT(DISTINCT o.order_id)          AS orders,
       SUM(oi.qty)                         AS units_sold,
       SUM(oi.qty * oi.unit_price)         AS revenue
FROM   orders      o
JOIN   order_items oi ON o.order_id    = oi.order_id
JOIN   products    p  ON oi.product_id = p.product_id
GROUP BY 1, 2;

-- Query the view like a table
SELECT * FROM monthly_sales_summary
WHERE  month  >= '2024-01-01'
  AND  revenue > 50000
ORDER BY month, revenue DESC;
```

### Example 4 — Materialized view for a dashboard

```sql
-- Create the materialized view
CREATE MATERIALIZED VIEW top_products_mv AS
SELECT p.product_name,
       p.category,
       SUM(oi.qty * oi.unit_price)  AS total_revenue,
       SUM(oi.qty)                  AS total_units
FROM   products    p
JOIN   order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_name, p.category
WITH DATA;

-- Index to support concurrent refresh
CREATE UNIQUE INDEX ON top_products_mv (product_name, category);

-- Dashboard query (instant — reads snapshot)
SELECT * FROM top_products_mv ORDER BY total_revenue DESC LIMIT 10;

-- Refresh nightly or on schedule
REFRESH MATERIALIZED VIEW CONCURRENTLY top_products_mv;
```

### Example 5 — Stacked views for layered access

```sql
-- Layer 1: all active employees
CREATE VIEW v_active AS
SELECT * FROM employees WHERE is_active = TRUE;

-- Layer 2: active employees with dept name
CREATE VIEW v_active_with_dept AS
SELECT e.*, d.dept_name
FROM   v_active     e
JOIN   departments  d ON e.dept_id = d.dept_id;

-- Layer 3: senior active employees with dept
CREATE VIEW v_senior_with_dept AS
SELECT * FROM v_active_with_dept
WHERE  salary >= 80000;

-- Each layer can be individually granted to different roles
GRANT SELECT ON v_active_with_dept TO hr_team;
GRANT SELECT ON v_senior_with_dept  TO executive_team;
```

### Example 6 — View as migration buffer

```sql
-- The app still uses the old table name "staff"
-- We've renamed the table to "employees"
CREATE VIEW staff AS
SELECT emp_id   AS staff_id,
       name     AS staff_name,
       dept_id,
       salary
FROM   employees;

-- Legacy queries keep working:
SELECT * FROM staff WHERE salary > 70000;
```

### Example 7 — View with CASE for business logic

```sql
CREATE VIEW order_status_dashboard AS
SELECT o.order_id,
       o.order_date,
       c.customer_name,
       SUM(oi.qty * oi.unit_price) AS order_total,
       o.status,
       CASE
           WHEN o.status = 'Pending'   AND o.order_date < CURRENT_DATE - 7
               THEN 'OVERDUE'
           WHEN o.status = 'Pending'
               THEN 'On Track'
           WHEN o.status = 'Completed'
               THEN 'Done'
           ELSE o.status
       END AS status_flag
FROM   orders      o
JOIN   customers   c  ON o.customer_id  = c.customer_id
JOIN   order_items oi ON o.order_id     = oi.order_id
GROUP BY o.order_id, o.order_date, c.customer_name, o.status;
```

### Example 8 — Materialized view + incremental update tracking

```sql
-- Track when data was last loaded
CREATE MATERIALIZED VIEW sales_cube AS
SELECT p.category,
       DATE_TRUNC('month', o.order_date) AS month,
       SUM(oi.qty * oi.unit_price)       AS revenue
FROM   orders o
JOIN   order_items oi ON o.order_id    = oi.order_id
JOIN   products    p  ON oi.product_id = p.product_id
GROUP BY 1, 2
WITH DATA;

-- Check freshness
SELECT pg_last_autovacuum(oid) AS last_refresh
FROM   pg_class
WHERE  relname = 'sales_cube';
```

---

## 12. Common Interview Questions

**Q1. What is a view and how does it differ from a table?**  
A view is a stored query — it does not hold data. A table physically stores rows. When you query a regular view, the database re-executes the stored query; when you query a table, it reads stored data directly.

**Q2. What is a materialized view and when would you use it?**  
A materialized view stores the query result physically on disk. Use it when the underlying query is expensive (heavy aggregations, complex joins) and reads must be fast. The trade-off is that you must refresh it to see current data.

**Q3. What is an updatable view?**  
A view that supports INSERT, UPDATE, and DELETE. For a view to be updatable it must query a single base table with no aggregations, DISTINCT, GROUP BY, or subqueries in the SELECT list.

**Q4. What does WITH CHECK OPTION do?**  
It prevents INSERT or UPDATE through a view from creating rows that fall outside the view's WHERE condition. Without it, you could insert a row that immediately becomes invisible through the view.

**Q5. Can you always SELECT from a view and get the latest data?**  
From a regular view, yes — it re-executes the query every time. From a materialized view, no — you see a snapshot that may be stale until the next refresh.

**Q6. What is the difference between CASCADED and LOCAL CHECK OPTION?**  
When views are stacked, `CASCADED` (default) enforces the WHERE conditions of all views in the chain. `LOCAL` enforces only the current view's condition, ignoring conditions from views it was built on.

**Q7. Does using a view improve query performance?**  
No — a regular view is inlined at parse time; the planner sees the same query as if you had written the subquery directly. Only materialized views improve read performance (at the cost of staleness and refresh overhead).

**Q8. How do you prevent users from accessing a base table and force them to use a view?**  
`REVOKE SELECT ON base_table FROM role;` then `GRANT SELECT ON view_name TO role;`. Users can only see the data the view exposes.

**Q9. What happens when you DROP a base table that a view depends on?**  
The view becomes invalid (broken). In PostgreSQL, `DROP TABLE ... CASCADE` also drops dependent views. Without CASCADE, the DROP is blocked if a view depends on the table.

**Q10. What is a stacked view and what are its risks?**  
A view built on top of another view. The risk is fragility: changes to the base view cascade to all dependent views. Deep stacking also makes EXPLAIN plans harder to read. Limit stacking to 2–3 levels.

**Q11. Write a view that shows only the columns and rows an HR analyst should see.**

```sql
CREATE VIEW hr_view AS
SELECT emp_id,
       name,
       dept_id,
       hire_date,
       CASE
           WHEN salary >= 90000 THEN 'Band A'
           WHEN salary >= 70000 THEN 'Band B'
           ELSE                      'Band C'
       END AS salary_band
FROM   employees
WHERE  is_active = TRUE;

GRANT SELECT ON hr_view TO hr_analyst_role;
```

**Q12. How do you refresh a materialized view without blocking reads in PostgreSQL?**  
Create a UNIQUE index on the materialized view, then use `REFRESH MATERIALIZED VIEW CONCURRENTLY view_name`. Without the UNIQUE index, concurrent refresh is not supported.

---

## 13. Key Takeaways

| Concept | One-line rule |
|---------|--------------|
| **Regular view** | Stored query — virtual, no storage, always current, inlined by planner |
| **Simple view** | Single-table, no aggregation — updatable |
| **Complex view** | Joins/aggregations — not updatable, great for reporting |
| **WITH CHECK OPTION** | Prevents INSERT/UPDATE from creating rows invisible through the view |
| **Materialized view** | Physical snapshot — fast reads, needs scheduled refresh |
| **REFRESH CONCURRENTLY** | PostgreSQL: refresh without blocking reads — requires UNIQUE index |
| **View for security** | Grant on view, revoke on table — hide columns and rows |
| **View for abstraction** | Stable interface — shield consumers from schema changes |
| **Performance** | Regular view ≈ subquery (no speedup); materialized view = real speedup |

### Decision Guide

```
Need to reuse a complex query?
  ├─ Within a single query only?       → CTE
  ├─ Across many queries / users?      → View

Need to restrict data access?
  ├─ Hide columns?                     → Regular view (column projection)
  ├─ Hide rows?                        → Regular view (WHERE filter)
  ├─ Prevent writes outside scope?     → WITH CHECK OPTION

Need fast reads on heavy aggregations?
  ├─ Data can be slightly stale?       → Materialized view
  ├─ Data must always be current?      → Regular view (accept query cost)

Need to guard against INSERT drift?
  ├─ View is updatable?                → WITH CHECK OPTION
  └─ View is not updatable?           → INSTEAD OF trigger (advanced)
```

---

*Phase 9 complete. Phase 10 covers Indexes & Query Optimization (`10_indexes_and_optimization.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="09_views.md" download="09_views.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 09_views.md
  </button>
</a>
