# Phase 8 — Subqueries, CTEs & Advanced Queries

> **Curriculum:** DBMS A-to-Z | **File:** `08_advanced_queries.md`  
> **Prerequisites:** Phases 1–7 (Foundations → Joins)

---

## Table of Contents

1. [What Is a Subquery?](#1-what-is-a-subquery)
2. [Non-Correlated Subqueries](#2-non-correlated-subqueries)
3. [Correlated Subqueries](#3-correlated-subqueries)
4. [Subquery Positions (WHERE / SELECT / FROM)](#4-subquery-positions)
5. [EXISTS and NOT EXISTS](#5-exists-and-not-exists)
6. [IN, NOT IN, ANY, ALL](#6-in-not-in-any-all)
7. [CTEs — WITH Clause](#7-ctes--with-clause)
8. [Recursive CTEs](#8-recursive-ctes)
9. [CASE Expressions](#9-case-expressions)
10. [COALESCE, NULLIF, NVL](#10-coalesce-nullif-nvl)
11. [Set Operations: UNION, INTERSECT, EXCEPT](#11-set-operations-union-intersect-except)
12. [Scalar Subqueries](#12-scalar-subqueries)
13. [Derived Tables (Inline Views)](#13-derived-tables-inline-views)
14. [Performance: Subquery vs CTE vs JOIN vs Temp Table](#14-performance-subquery-vs-cte-vs-join-vs-temp-table)
15. [20+ Worked Query Examples](#15-20-worked-query-examples)
16. [Common Interview Questions](#16-common-interview-questions)
17. [Key Takeaways](#17-key-takeaways)

---

## 1. What Is a Subquery?

A **subquery** (also called an *inner query* or *nested query*) is a complete `SELECT` statement embedded inside another SQL statement. The outer query uses the subquery's result as if it were a table, a list of values, or a single value.

### Real-World Analogy

> *"Give me all employees who earn more than the company average."*

You cannot answer this in a single pass because you don't know the average until you've read every row. So you:
1. Run the inner query → compute the average (e.g., 78 200).
2. Run the outer query → return employees whose salary > 78 200.

That two-step process is a subquery.

### Sample Schema Used Throughout This Phase

```sql
-- employees
CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(50),
    dept_id    INT,
    manager_id INT,
    salary     NUMERIC(10,2),
    hire_date  DATE
);

-- departments
CREATE TABLE departments (
    dept_id   INT PRIMARY KEY,
    dept_name VARCHAR(50),
    budget    NUMERIC(12,2)
);

-- customers
CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    customer_name VARCHAR(100),
    country       VARCHAR(50),
    email         VARCHAR(100)
);

-- orders
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    status      VARCHAR(20)
);

-- order_items
CREATE TABLE order_items (
    item_id    INT PRIMARY KEY,
    order_id   INT,
    product_id INT,
    qty        INT,
    unit_price NUMERIC(10,2)
);

-- products
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100),
    category     VARCHAR(50),
    list_price   NUMERIC(10,2)
);
```

---

## 2. Non-Correlated Subqueries

### Concept

A **non-correlated subquery** is completely independent of the outer query. It executes **once**, produces a result, and passes that result to the outer query as a constant.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="520" height="150" viewBox="0 0 520 150">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <rect x="10" y="30" width="170" height="70" fill="#D5E8D4" stroke="#82B366" rx="8"/>
  <text x="95" y="58" text-anchor="middle" font-weight="bold" fill="#2D6A2D">Inner Query</text>
  <text x="95" y="78" text-anchor="middle" fill="#2D6A2D">Runs ONCE independently</text>
  <text x="95" y="95" text-anchor="middle" fill="#2D6A2D">→ returns a value/set</text>

  <line x1="180" y1="65" x2="250" y2="65" stroke="#333" stroke-width="2" marker-end="url(#a1)"/>
  <defs><marker id="a1" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#333"/>
  </marker></defs>
  <text x="215" y="58" text-anchor="middle" fill="#C0392B" font-size="10">result</text>

  <rect x="250" y="20" width="250" height="90" fill="#DAE8FC" stroke="#6C8EBF" rx="8"/>
  <text x="375" y="48" text-anchor="middle" font-weight="bold" fill="#1A3A5C">Outer Query</text>
  <text x="375" y="68" text-anchor="middle" fill="#1A3A5C">Uses inner result as a constant</text>
  <text x="375" y="88" text-anchor="middle" fill="#1A3A5C">No reference back to inner</text>

  <text x="260" y="135" fill="#7D3C98" font-size="11" font-style="italic">The inner query has NO knowledge of the outer query's current row.</text>
</svg>
```

### Example 1 — Beginner: employees above company average salary

```sql
SELECT name, salary
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees);
```

The inner `SELECT AVG(salary) FROM employees` executes once and returns, say, `78200`. The outer query then becomes:

```sql
SELECT name, salary FROM employees WHERE salary > 78200;
```

### Example 2 — Intermediate: employees in the highest-budget department

```sql
SELECT name, dept_id
FROM   employees
WHERE  dept_id = (
    SELECT dept_id
    FROM   departments
    ORDER BY budget DESC
    LIMIT 1
);
```

### Example 3 — Advanced (e-commerce): products priced above category average

```sql
-- Non-correlated version using a derived table (runs per category, not per row)
SELECT p.product_name, p.category, p.list_price
FROM   products p
JOIN  (SELECT category, AVG(list_price) AS cat_avg
       FROM   products
       GROUP BY category) ca ON p.category = ca.category
WHERE  p.list_price > ca.cat_avg
ORDER BY p.category, p.list_price DESC;
```

---

## 3. Correlated Subqueries

### Concept

A **correlated subquery** references a column from the **outer query**. It cannot run independently — it re-executes **once for every row** the outer query processes.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="520" height="210" viewBox="0 0 520 210">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <rect x="10" y="20" width="190" height="160" fill="#DAE8FC" stroke="#6C8EBF" rx="8"/>
  <text x="105" y="46" text-anchor="middle" font-weight="bold" fill="#1A3A5C">Outer Query</text>
  <line x1="20" y1="54" x2="190" y2="54" stroke="#AED6F1"/>
  <text x="105" y="74" text-anchor="middle" fill="#1A3A5C">Row 1 → passes to inner</text>
  <text x="105" y="92" text-anchor="middle" fill="#1A3A5C">Row 2 → passes to inner</text>
  <text x="105" y="110" text-anchor="middle" fill="#1A3A5C">Row 3 → passes to inner</text>
  <text x="105" y="128" text-anchor="middle" fill="#1A3A5C">... N rows → N inner runs</text>

  <!-- Forward arrow -->
  <line x1="200" y1="85" x2="270" y2="85" stroke="#E74C3C" stroke-width="2" marker-end="url(#a2)"/>
  <defs><marker id="a2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/>
  </marker></defs>
  <text x="235" y="75" text-anchor="middle" fill="#C0392B" font-size="10">outer row</text>

  <!-- Return arrow -->
  <line x1="270" y1="105" x2="200" y2="105" stroke="#27AE60" stroke-width="2" marker-end="url(#a3)"/>
  <defs><marker id="a3" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#27AE60"/>
  </marker></defs>
  <text x="235" y="125" text-anchor="middle" fill="#1E8449" font-size="10">result</text>

  <rect x="270" y="50" width="230" height="90" fill="#D5E8D4" stroke="#82B366" rx="8"/>
  <text x="385" y="76" text-anchor="middle" font-weight="bold" fill="#2D6A2D">Inner Query</text>
  <text x="385" y="96" text-anchor="middle" fill="#2D6A2D">References outer.column</text>
  <text x="385" y="116" text-anchor="middle" fill="#2D6A2D">Re-executes PER outer row</text>

  <text x="260" y="195" fill="#7D3C98" font-size="11" font-style="italic" text-anchor="middle">Slow on large tables — consider rewriting as a JOIN + GROUP BY.</text>
</svg>
```

### Example 1 — Each employee vs their department average

```sql
SELECT e.name,
       e.salary,
       (SELECT ROUND(AVG(e2.salary), 2)
        FROM   employees e2
        WHERE  e2.dept_id = e.dept_id) AS dept_avg   -- references outer e.dept_id
FROM   employees e;
```

### Example 2 — Employees earning more than their department average

```sql
SELECT e.name, e.salary, e.dept_id
FROM   employees e
WHERE  e.salary > (
    SELECT AVG(e2.salary)
    FROM   employees e2
    WHERE  e2.dept_id = e.dept_id     -- correlated reference
);
```

### Rewriting as a JOIN (faster — aggregation runs once per dept)

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, e.dept_id
FROM   employees e
JOIN   dept_avg  d ON e.dept_id = d.dept_id
WHERE  e.salary > d.avg_sal;
```

---

## 4. Subquery Positions

Subqueries can appear in four distinct positions, each with its own rules:

| Position | Name | Must return | Example usage |
|----------|------|-------------|---------------|
| `WHERE col = (...)` | Scalar in WHERE | 1 row, 1 col | Equality, comparison |
| `WHERE col IN (...)` | List in WHERE | Many rows, 1 col | Membership test |
| `WHERE EXISTS (...)` | Existence in WHERE | Any row(s) | Boolean check |
| `FROM (...)` AS t | Derived table | Many rows, many cols | Acts as a temporary table |
| `SELECT (...)` | Scalar in SELECT | 1 row, 1 col | Computed column per row |

### Position examples at a glance

```sql
-- 1. Scalar in WHERE
WHERE salary = (SELECT MAX(salary) FROM employees)

-- 2. List in WHERE
WHERE dept_id IN (SELECT dept_id FROM departments WHERE budget > 300000)

-- 3. EXISTS in WHERE
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)

-- 4. Derived table in FROM
FROM (SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id) t

-- 5. Scalar in SELECT
SELECT name, (SELECT dept_name FROM departments d WHERE d.dept_id = e.dept_id)
FROM employees e
```

---

## 5. EXISTS and NOT EXISTS

### EXISTS

Returns `TRUE` if the subquery produces **at least one row**. As soon as the first row is found, the scan stops (short-circuit). The value returned by the subquery doesn't matter — by convention, `SELECT 1` is used.

```sql
-- Departments that have at least one employee
SELECT dept_name
FROM   departments d
WHERE  EXISTS (
    SELECT 1
    FROM   employees e
    WHERE  e.dept_id = d.dept_id
);
```

### NOT EXISTS

Returns `TRUE` if the subquery produces **zero rows**.

```sql
-- Departments with no employees
SELECT dept_name
FROM   departments d
WHERE  NOT EXISTS (
    SELECT 1
    FROM   employees e
    WHERE  e.dept_id = d.dept_id
);
```

### EXISTS vs IN — Which to Use When?

| Scenario | Prefer | Why |
|----------|--------|-----|
| Checking existence only | `EXISTS` | Short-circuits at first match |
| Right side may contain NULLs | `EXISTS` | NULL-safe; `NOT IN` with NULLs is a trap |
| Correlated check | `EXISTS` | Natural fit |
| Small hard-coded list | `IN ('A','B','C')` | Simple and readable |
| Large subquery, no NULLs | Either | Planner usually optimises both equally |

### The NULL Trap in NOT IN

```sql
-- departments table has dept_id: 10, 20, 40
-- employees.dept_id values include NULL (Eve's row)

-- DANGER: returns 0 rows if any dept_id in the subquery is NULL
SELECT name FROM employees
WHERE  dept_id NOT IN (SELECT dept_id FROM departments);
-- Because: NULL comparison → UNKNOWN, never TRUE → no rows pass

-- SAFE alternatives:
-- Option A: filter NULLs in subquery
WHERE  dept_id NOT IN (SELECT dept_id FROM departments WHERE dept_id IS NOT NULL);

-- Option B: use NOT EXISTS (always safe)
WHERE  NOT EXISTS (SELECT 1 FROM departments d WHERE d.dept_id = e.dept_id);
```

---

## 6. IN, NOT IN, ANY, ALL

### IN / NOT IN

```sql
-- Employees in Engineering or Marketing
SELECT name
FROM   employees
WHERE  dept_id IN (
    SELECT dept_id FROM departments
    WHERE  dept_name IN ('Engineering', 'Marketing')
);
```

### ANY (synonym: SOME)

`col op ANY (subquery)` — TRUE if the condition holds for **at least one** value in the subquery result. Equivalent to `col op MIN(...)` for `>`, `col op MAX(...)` for `<`.

```sql
-- Employees earning more than at least one person in Marketing
SELECT name, salary
FROM   employees
WHERE  salary > ANY (
    SELECT salary FROM employees
    WHERE  dept_id = (SELECT dept_id FROM departments WHERE dept_name = 'Marketing')
);
-- Equivalent to: salary > MIN(marketing salaries)
```

### ALL

`col op ALL (subquery)` — TRUE only if the condition holds for **every** value in the subquery result.

```sql
-- Employees earning more than EVERY person in Marketing
SELECT name, salary
FROM   employees
WHERE  salary > ALL (
    SELECT salary FROM employees
    WHERE  dept_id = (SELECT dept_id FROM departments WHERE dept_name = 'Marketing')
);
-- Equivalent to: salary > MAX(marketing salaries)
```

### Operator Equivalence Reference

| Expression | Equivalent |
|------------|-----------|
| `= ANY(...)` | `IN (...)` |
| `<> ALL(...)` | `NOT IN (...)` |
| `> ANY(...)` | `> MIN(...)` |
| `> ALL(...)` | `> MAX(...)` |
| `< ANY(...)` | `< MAX(...)` |
| `< ALL(...)` | `< MIN(...)` |

---

## 7. CTEs — WITH Clause

### Concept

A **Common Table Expression (CTE)** is a named, temporary result set defined at the top of a query using `WITH`. It makes complex queries readable by breaking them into clearly named steps, and can be referenced **multiple times** within the same statement.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="560" height="200" viewBox="0 0 560 200">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <rect x="10" y="20" width="150" height="55" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="85" y="44" text-anchor="middle" font-weight="bold" fill="#2D6A2D">CTE 1: dept_avg</text>
  <text x="85" y="64" text-anchor="middle" fill="#2D6A2D">AVG salary per dept</text>

  <rect x="10" y="100" width="150" height="55" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="85" y="124" text-anchor="middle" font-weight="bold" fill="#784212">CTE 2: top_earners</text>
  <text x="85" y="144" text-anchor="middle" fill="#784212">salary > dept_avg</text>

  <line x1="160" y1="47" x2="230" y2="95" stroke="#555" stroke-width="1.5" stroke-dasharray="5" marker-end="url(#ac)"/>
  <line x1="160" y1="127" x2="230" y2="107" stroke="#555" stroke-width="1.5" stroke-dasharray="5" marker-end="url(#ac)"/>
  <defs><marker id="ac" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#555"/>
  </marker></defs>

  <rect x="230" y="65" width="310" height="60" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="385" y="90" text-anchor="middle" font-weight="bold" fill="#1A3A5C">Final SELECT</text>
  <text x="385" y="110" text-anchor="middle" fill="#1A3A5C">References CTE 1 and CTE 2</text>

  <text x="280" y="185" fill="#7D3C98" font-size="11" font-style="italic">CTEs are not persisted — they exist only for the duration of the query.</text>
</svg>
```

### Syntax

```sql
WITH cte_name AS (
    SELECT ...          -- define the CTE
),
second_cte AS (
    SELECT ...
    FROM   cte_name     -- can reference an earlier CTE
    ...
)
SELECT *
FROM   second_cte
WHERE  ...;
```

### Example 1 — Beginner: employees above company average

```sql
WITH company_avg AS (
    SELECT AVG(salary) AS avg_sal FROM employees
)
SELECT e.name, e.salary
FROM   employees    e
CROSS JOIN company_avg c
WHERE  e.salary > c.avg_sal;
```

### Example 2 — Intermediate: month-over-month revenue

```sql
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', o.order_date) AS month,
           SUM(oi.qty * oi.unit_price)       AS revenue
    FROM   orders      o
    JOIN   order_items oi ON o.order_id = oi.order_id
    GROUP BY 1
),
with_lag AS (
    SELECT month,
           revenue,
           LAG(revenue) OVER (ORDER BY month) AS prev_revenue
    FROM   monthly_revenue
)
SELECT month,
       revenue,
       prev_revenue,
       ROUND(100.0 * (revenue - prev_revenue)
             / NULLIF(prev_revenue, 0), 2) AS pct_change
FROM   with_lag
ORDER BY month;
```

### Example 3 — Advanced: multi-step fraud detection pipeline (banking)

```sql
WITH
high_value AS (
    SELECT account_id, txn_date, amount
    FROM   transactions
    WHERE  amount > 10000
),
flagged AS (
    SELECT DISTINCT account_id
    FROM   high_value
    WHERE  txn_date >= CURRENT_DATE - INTERVAL '7 days'
),
account_info AS (
    SELECT a.account_id, c.full_name, c.email
    FROM   accounts  a
    JOIN   customers c ON a.customer_id = c.customer_id
    WHERE  a.account_id IN (SELECT account_id FROM flagged)
)
SELECT * FROM account_info ORDER BY full_name;
```

### CTE Materialisation in PostgreSQL

In PostgreSQL, CTEs are **materialisation fences** by default — the planner cannot push predicates into them or inline them. This protects complex CTEs from being re-evaluated, but can block optimisations.

```sql
-- Allow planner to inline (PostgreSQL 12+)
WITH dept_avg AS NOT MATERIALIZED (
    SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
)
SELECT e.name FROM employees e
JOIN   dept_avg d ON e.dept_id = d.dept_id
WHERE  e.salary > d.avg_sal;
```

---

## 8. Recursive CTEs

### Concept

A **recursive CTE** references itself. It has two mandatory parts joined by `UNION ALL`:

- **Anchor member** — the base case; runs once, seeds the initial rows.
- **Recursive member** — references the CTE name; runs repeatedly, adding rows each iteration.

Execution stops when the recursive member returns **zero rows**.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="460" height="260" viewBox="0 0 460 260">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>

  <!-- Anchor box -->
  <rect x="10" y="10" width="200" height="50" fill="#D5E8D4" stroke="#82B366" rx="6"/>
  <text x="110" y="32" text-anchor="middle" font-weight="bold" fill="#2D6A2D">① Anchor Member</text>
  <text x="110" y="52" text-anchor="middle" fill="#2D6A2D">Base case — seeds the result</text>

  <!-- Down arrow -->
  <line x1="110" y1="60" x2="110" y2="90" stroke="#333" stroke-width="2" marker-end="url(#ar)"/>
  <defs><marker id="ar" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3z" fill="#333"/>
  </marker></defs>

  <!-- Working table -->
  <rect x="10" y="90" width="200" height="50" fill="#DAE8FC" stroke="#6C8EBF" rx="6"/>
  <text x="110" y="112" text-anchor="middle" font-weight="bold" fill="#1A3A5C">Working Table</text>
  <text x="110" y="132" text-anchor="middle" fill="#1A3A5C">UNION ALL accumulates rows</text>

  <!-- Right arrow to recursive -->
  <line x1="210" y1="115" x2="270" y2="115" stroke="#E74C3C" stroke-width="2"/>
  <line x1="270" y1="115" x2="270" y2="175" stroke="#E74C3C" stroke-width="2"/>

  <!-- Recursive box -->
  <rect x="220" y="155" width="220" height="60" fill="#FDEDEC" stroke="#C0392B" rx="6"/>
  <text x="330" y="178" text-anchor="middle" font-weight="bold" fill="#7B241C">② Recursive Member</text>
  <text x="330" y="198" text-anchor="middle" fill="#7B241C">JOIN CTE to itself + stop cond.</text>

  <!-- Back arrow -->
  <line x1="220" y1="185" x2="110" y2="185" stroke="#E74C3C" stroke-width="2"/>
  <line x1="110" y1="185" x2="110" y2="140" stroke="#E74C3C" stroke-width="2" marker-end="url(#ar)"/>

  <text x="230" y="245" fill="#7D3C98" font-size="11" font-style="italic" text-anchor="middle">Stops when recursive member returns 0 rows</text>
</svg>
```

### Syntax

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor
    SELECT ...

    UNION ALL

    -- Recursive member — references cte_name
    SELECT ...
    FROM   cte_name
    WHERE  <termination condition>
)
SELECT * FROM cte_name;
```

### Example 1 — Generate numbers 1 to 10

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n

    UNION ALL

    SELECT n + 1
    FROM   nums
    WHERE  n < 10
)
SELECT n FROM nums;
```

**Result:** 1 · 2 · 3 · 4 · 5 · 6 · 7 · 8 · 9 · 10

### Example 2 — Org-chart traversal with depth and path

```sql
WITH RECURSIVE org AS (
    -- Anchor: top-level (no manager)
    SELECT emp_id, name, manager_id,
           1            AS depth,
           name::TEXT   AS path
    FROM   employees
    WHERE  manager_id IS NULL

    UNION ALL

    -- Recursive: employees whose manager is already in org
    SELECT e.emp_id, e.name, e.manager_id,
           o.depth + 1,
           (o.path || ' → ' || e.name)::TEXT
    FROM   employees e
    JOIN   org       o ON e.manager_id = o.emp_id
)
SELECT depth, path
FROM   org
ORDER BY path;
```

**Sample result:**

| depth | path |
|-------|------|
| 1 | Alice |
| 2 | Alice → Bob |
| 2 | Alice → Carol |
| 3 | Alice → Bob → David |
| 3 | Alice → Carol → Eve |

### Example 3 — Generate a date series (reporting scaffolding)

```sql
WITH RECURSIVE calendar AS (
    SELECT '2024-01-01'::DATE AS dt

    UNION ALL

    SELECT dt + INTERVAL '1 day'
    FROM   calendar
    WHERE  dt < '2024-01-31'
)
SELECT dt FROM calendar;
```

### Example 4 — Bill of Materials (parts explosion)

```sql
-- components(part_id, part_name, parent_id, qty_needed)
WITH RECURSIVE bom AS (
    SELECT part_id, part_name, parent_id, qty_needed, 0 AS depth
    FROM   components
    WHERE  parent_id IS NULL   -- top-level assembly

    UNION ALL

    SELECT c.part_id, c.part_name, c.parent_id,
           b.qty_needed * c.qty_needed AS qty_needed,
           b.depth + 1
    FROM   components c
    JOIN   bom        b ON c.parent_id = b.part_id
)
SELECT REPEAT('  ', depth) || part_name AS part,
       qty_needed
FROM   bom
ORDER BY depth, part_name;
```

### Preventing Infinite Loops

```sql
-- Add a depth guard
WHERE  o.depth < 10          -- stop at 10 levels regardless

-- PostgreSQL: set session-level recursion limit
SET max_recursive_iterations = 100;

-- SQL Server: at end of query
OPTION (MAXRECURSION 50);
```

---

## 9. CASE Expressions

### Concept

`CASE` is SQL's conditional expression. It can appear anywhere an expression is valid: `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, inside aggregate functions.

### Two Forms

**Searched CASE** (most flexible — any condition):

```sql
CASE
    WHEN condition_1 THEN result_1
    WHEN condition_2 THEN result_2
    ...
    ELSE default_result
END
```

**Simple CASE** (equality comparison only):

```sql
CASE expression
    WHEN value_1 THEN result_1
    WHEN value_2 THEN result_2
    ELSE default_result
END
```

### Example 1 — Salary banding

```sql
SELECT name, salary,
       CASE
           WHEN salary >= 90000 THEN 'Senior'
           WHEN salary >= 70000 THEN 'Mid-level'
           ELSE                      'Junior'
       END AS band
FROM   employees;
```

| name  | salary | band      |
|-------|--------|-----------|
| Alice | 95000  | Senior    |
| Carol | 85000  | Mid-level |
| Bob   | 78000  | Mid-level |
| Eve   | 71000  | Mid-level |
| David | 62000  | Junior    |

### Example 2 — Pivot: count employees by band per department

```sql
SELECT dept_id,
       COUNT(CASE WHEN salary >= 90000 THEN 1 END)              AS senior,
       COUNT(CASE WHEN salary BETWEEN 70000 AND 89999 THEN 1 END) AS mid_level,
       COUNT(CASE WHEN salary < 70000  THEN 1 END)              AS junior
FROM   employees
GROUP BY dept_id;
```

### Example 3 — Conditional ORDER BY

```sql
-- Show 'Critical' orders first, then by date
SELECT order_id, status, order_date
FROM   orders
ORDER BY
    CASE status
        WHEN 'Critical' THEN 1
        WHEN 'Pending'  THEN 2
        ELSE                 3
    END,
    order_date;
```

### Example 4 — Conditional aggregation: year-over-year comparison

```sql
SELECT category,
       SUM(CASE WHEN EXTRACT(YEAR FROM order_date) = 2023 THEN qty * unit_price ELSE 0 END) AS rev_2023,
       SUM(CASE WHEN EXTRACT(YEAR FROM order_date) = 2024 THEN qty * unit_price ELSE 0 END) AS rev_2024
FROM   order_items oi
JOIN   orders       o ON oi.order_id    = o.order_id
JOIN   products     p ON oi.product_id  = p.product_id
GROUP BY category;
```

---

## 10. COALESCE, NULLIF, NVL

### COALESCE — first non-NULL wins

```sql
COALESCE(expr1, expr2, ..., exprN)
-- Returns the first argument that is NOT NULL.
```

```sql
-- Display nickname if available, else first name, else 'Anonymous'
SELECT COALESCE(nickname, first_name, 'Anonymous') AS display_name
FROM   users;

-- Treat missing salary as 0 for SUM
SELECT dept_id, SUM(COALESCE(salary, 0)) AS total_payroll
FROM   employees
GROUP BY dept_id;
```

### NULLIF — convert a sentinel value to NULL

```sql
NULLIF(expr, value_to_treat_as_null)
-- Returns NULL if expr = value_to_treat_as_null; otherwise returns expr.
```

```sql
-- Safe division: avoid divide-by-zero
SELECT product_name,
       total_revenue / NULLIF(units_sold, 0) AS avg_price
FROM   sales;

-- Treat blank strings as NULL
SELECT NULLIF(TRIM(comments), '') AS comments
FROM   feedback;
```

### NVL — Oracle / DB2 only

```sql
NVL(expr, fallback)        -- exactly 2 args; equivalent to COALESCE(expr, fallback)
SELECT NVL(commission, 0) FROM sales_reps;
-- Use COALESCE for portability
```

### IFNULL / ISNULL

```sql
IFNULL(expr, fallback)     -- MySQL
ISNULL(expr, fallback)     -- SQL Server
-- Both behave like 2-arg COALESCE; prefer COALESCE for portability
```

### Combined pattern — safe average price

```sql
SELECT product_name,
       COALESCE(
           total_revenue / NULLIF(units_sold, 0),
           list_price       -- fallback when no sales data
       ) AS effective_price
FROM   products;
```

---

## 11. Set Operations: UNION, INTERSECT, EXCEPT

Set operations combine the result sets of two SELECT statements. **Rules:**
- Both sides must have the same number of columns.
- Corresponding columns must have compatible data types.
- Column names come from the **first** SELECT.

### Visual Reference

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="640" height="200" viewBox="0 0 640 200">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.lbl{font-size:13px;font-weight:bold}</style>

  <!-- UNION -->
  <circle cx="55"  cy="80" r="48" fill="#AED6F1" fill-opacity="0.7" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="105" cy="80" r="48" fill="#AED6F1" fill-opacity="0.7" stroke="#2980B9" stroke-width="1.5"/>
  <text x="80"  y="84" text-anchor="middle" class="lbl" fill="#1A5276">UNION</text>
  <text x="80"  y="160" text-anchor="middle" fill="#555">All distinct rows</text>
  <text x="80"  y="175" text-anchor="middle" fill="#555">from both sides</text>

  <!-- UNION ALL -->
  <circle cx="215" cy="80" r="48" fill="#A9DFBF" fill-opacity="0.7" stroke="#1E8449" stroke-width="1.5"/>
  <circle cx="265" cy="80" r="48" fill="#A9DFBF" fill-opacity="0.7" stroke="#1E8449" stroke-width="1.5"/>
  <text x="240" y="78" text-anchor="middle" class="lbl" fill="#145A32">UNION</text>
  <text x="240" y="94" text-anchor="middle" class="lbl" fill="#145A32">ALL</text>
  <text x="240" y="160" text-anchor="middle" fill="#555">All rows including</text>
  <text x="240" y="175" text-anchor="middle" fill="#555">duplicates (faster)</text>

  <!-- INTERSECT -->
  <circle cx="375" cy="80" r="48" fill="#F8C471" fill-opacity="0.3" stroke="#E67E22" stroke-width="1.5"/>
  <circle cx="425" cy="80" r="48" fill="#F8C471" fill-opacity="0.3" stroke="#E67E22" stroke-width="1.5"/>
  <path d="M400,34 A48,48 0 0,1 400,126 A48,48 0 0,1 400,34" fill="#F39C12" fill-opacity="0.85"/>
  <text x="400" y="84" text-anchor="middle" class="lbl" fill="white">INTER-</text>
  <text x="400" y="100" text-anchor="middle" class="lbl" fill="white">SECT</text>
  <text x="400" y="160" text-anchor="middle" fill="#555">Rows in</text>
  <text x="400" y="175" text-anchor="middle" fill="#555">both queries</text>

  <!-- EXCEPT -->
  <circle cx="535" cy="80" r="48" fill="#EC7063" fill-opacity="0.7" stroke="#C0392B" stroke-width="1.5"/>
  <circle cx="585" cy="80" r="48" fill="white" fill-opacity="0.9" stroke="#C0392B" stroke-width="1.5"/>
  <path d="M560,34 A48,48 0 0,1 560,126 A48,48 0 0,1 560,34" fill="white" fill-opacity="0.95"/>
  <text x="522" y="84" text-anchor="middle" class="lbl" fill="white">EX-</text>
  <text x="522" y="100" text-anchor="middle" class="lbl" fill="white">CEPT</text>
  <text x="560" y="160" text-anchor="middle" fill="#555">Left rows</text>
  <text x="560" y="175" text-anchor="middle" fill="#555">not in right</text>
</svg>
```

### UNION — combine and deduplicate

```sql
SELECT email FROM employees
UNION
SELECT email FROM contractors;
-- Duplicate emails appear only once
```

### UNION ALL — combine including duplicates (always prefer unless you need dedup)

```sql
SELECT txn_id, amount FROM transactions_2023
UNION ALL
SELECT txn_id, amount FROM transactions_2024
ORDER BY txn_id;
```

> `UNION` performs an implicit `DISTINCT`, which requires a sort. `UNION ALL` skips that step and is significantly faster for large result sets. Always use `UNION ALL` unless deduplication is specifically required.

### INTERSECT — rows in both sets

```sql
-- Customers who ordered in BOTH 2023 and 2024 (returning customers)
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2023
INTERSECT
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2024;
```

### EXCEPT / MINUS — rows in the first set but not the second

```sql
-- Customers who ordered in 2023 but NOT in 2024 (lapsed customers)
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2023
EXCEPT
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2024;
```

> Oracle uses `MINUS` instead of `EXCEPT`.

### Operator Precedence

`INTERSECT` has higher precedence than `UNION`/`EXCEPT`. Use parentheses when mixing:

```sql
(SELECT ... UNION ALL SELECT ...)
INTERSECT
(SELECT ... UNION ALL SELECT ...);
```

---

## 12. Scalar Subqueries

A **scalar subquery** in the `SELECT` list returns exactly one value per row. It behaves like a computed column.

```sql
SELECT e.name,
       e.salary,
       (SELECT AVG(e2.salary)
        FROM   employees e2
        WHERE  e2.dept_id = e.dept_id) AS dept_avg
FROM   employees e;
```

If the subquery returns more than one row, the query fails at runtime with an error. Always use aggregation (`MAX`, `MIN`, `AVG`, `COUNT`) or a LIMIT 1 to guarantee a single value.

**Performance note:** Scalar subqueries in SELECT are correlated — they execute per row. For large tables, rewrite as a JOIN to a CTE/derived table.

---

## 13. Derived Tables (Inline Views)

A **derived table** is a subquery in the `FROM` clause. It acts like a named temporary table that exists only for the query.

```sql
-- Alias is required in most databases
SELECT dept_stats.dept_name, dept_stats.avg_sal
FROM  (
    SELECT d.dept_name,
           AVG(e.salary) AS avg_sal
    FROM   employees   e
    JOIN   departments d ON e.dept_id = d.dept_id
    GROUP BY d.dept_name
) AS dept_stats
WHERE dept_stats.avg_sal > 75000;
```

### Derived table vs CTE — same logic, different style

```sql
-- Derived table (nested, harder to read)
SELECT * FROM (SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id) t
WHERE t.avg_sal > 75000;

-- CTE (flat, easier to read)
WITH dept_avgs AS (
    SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
)
SELECT * FROM dept_avgs WHERE avg_sal > 75000;
```

Prefer CTEs for readability when the derived table is more than one level deep or referenced more than once.

---

## 14. Performance: Subquery vs CTE vs JOIN vs Temp Table

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="220" viewBox="0 0 680 220">
  <style>text{font-family:Arial,sans-serif;font-size:11px}</style>

  <!-- Header row -->
  <rect x="0" y="0" width="680" height="30" fill="#2C3E50" rx="4"/>
  <text x="80"  y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Approach</text>
  <text x="220" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Readability</text>
  <text x="360" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Performance</text>
  <text x="490" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Reusable?</text>
  <text x="610" y="20" text-anchor="middle" fill="white" font-weight="bold" font-size="12">Recursive?</text>

  <!-- Subquery row -->
  <rect x="0" y="30" width="680" height="40" fill="#EBF5FB"/>
  <text x="80"  y="55" text-anchor="middle" fill="#1A5276" font-weight="bold">Subquery</text>
  <text x="220" y="55" text-anchor="middle" fill="#555">⚠ Poor when deeply nested</text>
  <text x="360" y="55" text-anchor="middle" fill="#555">Same as JOIN (planner inlines)</text>
  <text x="490" y="55" text-anchor="middle" fill="#C0392B">❌ No</text>
  <text x="610" y="55" text-anchor="middle" fill="#C0392B">❌ No</text>

  <!-- CTE row -->
  <rect x="0" y="70" width="680" height="40" fill="#FDFEFE"/>
  <text x="80"  y="95" text-anchor="middle" fill="#1A5276" font-weight="bold">CTE</text>
  <text x="220" y="95" text-anchor="middle" fill="#27AE60">✅ Excellent</text>
  <text x="360" y="95" text-anchor="middle" fill="#555">Good (may materialise in PG)</text>
  <text x="490" y="95" text-anchor="middle" fill="#27AE60">✅ In same query</text>
  <text x="610" y="95" text-anchor="middle" fill="#27AE60">✅ Yes</text>

  <!-- JOIN row -->
  <rect x="0" y="110" width="680" height="40" fill="#EBF5FB"/>
  <text x="80"  y="135" text-anchor="middle" fill="#1A5276" font-weight="bold">JOIN</text>
  <text x="220" y="135" text-anchor="middle" fill="#555">✅ Clear for 2–3 tables</text>
  <text x="360" y="135" text-anchor="middle" fill="#27AE60">✅ Best — optimizer-friendly</text>
  <text x="490" y="135" text-anchor="middle" fill="#C0392B">❌ No</text>
  <text x="610" y="135" text-anchor="middle" fill="#C0392B">❌ No</text>

  <!-- Temp Table row -->
  <rect x="0" y="150" width="680" height="40" fill="#FDFEFE"/>
  <text x="80"  y="175" text-anchor="middle" fill="#1A5276" font-weight="bold">Temp Table</text>
  <text x="220" y="175" text-anchor="middle" fill="#27AE60">✅ Clear</text>
  <text x="360" y="175" text-anchor="middle" fill="#27AE60">✅ Can be indexed</text>
  <text x="490" y="175" text-anchor="middle" fill="#27AE60">✅ Across queries</text>
  <text x="610" y="175" text-anchor="middle" fill="#C0392B">❌ No</text>

  <text x="340" y="212" text-anchor="middle" fill="#7D3C98" font-size="11" font-style="italic">Rule: JOIN first, CTE for readability/recursion, Temp Table when you need a mid-pipeline index.</text>
</svg>
```

---

## 15. 20+ Worked Query Examples

### Q1 — Non-correlated: above-average salary

```sql
SELECT name, salary
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees)
ORDER BY salary DESC;
```

### Q2 — Non-correlated: employees in departments with budget > 300 000

```sql
SELECT name, dept_id
FROM   employees
WHERE  dept_id IN (
    SELECT dept_id FROM departments WHERE budget > 300000
);
```

### Q3 — Correlated: each employee vs their department average

```sql
SELECT e.name, e.salary,
       ROUND((SELECT AVG(e2.salary) FROM employees e2
              WHERE e2.dept_id = e.dept_id), 2) AS dept_avg
FROM   employees e;
```

### Q4 — Correlated rewritten as CTE (faster)

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, ROUND(d.avg_sal, 2) AS dept_avg
FROM   employees e
JOIN   dept_avg  d ON e.dept_id = d.dept_id;
```

### Q5 — EXISTS: departments with at least one senior employee

```sql
SELECT dept_name
FROM   departments d
WHERE  EXISTS (
    SELECT 1 FROM employees e
    WHERE  e.dept_id = d.dept_id
    AND    e.salary >= 90000
);
```

### Q6 — NOT EXISTS: customers who have never ordered

```sql
SELECT customer_name, email
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE  o.customer_id = c.customer_id
);
```

### Q7 — ANY: employees earning more than at least one Marketing employee

```sql
SELECT name, salary
FROM   employees
WHERE  salary > ANY (
    SELECT salary FROM employees
    WHERE  dept_id = (SELECT dept_id FROM departments WHERE dept_name = 'Marketing')
);
```

### Q8 — ALL: employees earning more than every Marketing employee

```sql
SELECT name, salary
FROM   employees
WHERE  salary > ALL (
    SELECT salary FROM employees
    WHERE  dept_id = (SELECT dept_id FROM departments WHERE dept_name = 'Marketing')
);
```

### Q9 — CTE: top 3 revenue products

```sql
WITH product_revenue AS (
    SELECT p.product_name,
           SUM(oi.qty * oi.unit_price) AS revenue
    FROM   products    p
    JOIN   order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_name
)
SELECT product_name, revenue
FROM   product_revenue
ORDER BY revenue DESC
LIMIT 3;
```

### Q10 — Multi-CTE: full sales funnel summary

```sql
WITH
orders_summary AS (
    SELECT customer_id, COUNT(*) AS order_count,
           SUM(oi.qty * oi.unit_price) AS total_spend
    FROM   orders o
    JOIN   order_items oi ON o.order_id = oi.order_id
    GROUP BY customer_id
),
ranked AS (
    SELECT customer_id, order_count, total_spend,
           NTILE(4) OVER (ORDER BY total_spend DESC) AS quartile
    FROM   orders_summary
)
SELECT c.customer_name, r.order_count, r.total_spend,
       CASE r.quartile WHEN 1 THEN 'Platinum'
                       WHEN 2 THEN 'Gold'
                       WHEN 3 THEN 'Silver'
                       ELSE        'Bronze' END AS tier
FROM   customers c
JOIN   ranked    r ON c.customer_id = r.customer_id
ORDER BY r.total_spend DESC;
```

### Q11 — Recursive CTE: org-chart with depth guard

```sql
WITH RECURSIVE hierarchy AS (
    SELECT emp_id, name, manager_id, 0 AS depth
    FROM   employees WHERE manager_id IS NULL

    UNION ALL

    SELECT e.emp_id, e.name, e.manager_id, h.depth + 1
    FROM   employees e
    JOIN   hierarchy h ON e.manager_id = h.emp_id
    WHERE  h.depth < 10   -- prevent infinite loop
)
SELECT REPEAT('  ', depth) || name AS org_tree, depth
FROM   hierarchy
ORDER BY depth, name;
```

### Q12 — Recursive CTE: date series with left join for zero-sales days

```sql
WITH RECURSIVE dates AS (
    SELECT '2024-01-01'::DATE AS dt
    UNION ALL
    SELECT dt + 1 FROM dates WHERE dt < '2024-01-31'
),
daily AS (
    SELECT DATE(o.order_date) AS day,
           SUM(oi.qty * oi.unit_price) AS revenue
    FROM   orders o
    JOIN   order_items oi ON o.order_id = oi.order_id
    GROUP BY 1
)
SELECT d.dt, COALESCE(dy.revenue, 0) AS revenue
FROM   dates d
LEFT JOIN daily dy ON d.dt = dy.day
ORDER BY d.dt;
```

### Q13 — CASE pivot: orders by status per month

```sql
SELECT TO_CHAR(order_date, 'YYYY-MM') AS month,
       COUNT(CASE WHEN status = 'Completed' THEN 1 END) AS completed,
       COUNT(CASE WHEN status = 'Pending'   THEN 1 END) AS pending,
       COUNT(CASE WHEN status = 'Cancelled' THEN 1 END) AS cancelled
FROM   orders
GROUP BY 1
ORDER BY 1;
```

### Q14 — COALESCE + NULLIF: safe price calculation

```sql
SELECT product_name,
       COALESCE(
           total_revenue / NULLIF(units_sold, 0),
           list_price
       ) AS effective_price
FROM   products;
```

### Q15 — UNION ALL: combine employee and contractor audit log

```sql
SELECT 'employee'  AS entity_type, emp_id  AS id, action, action_date
FROM   employee_audit_log
UNION ALL
SELECT 'contractor', contractor_id, action, action_date
FROM   contractor_audit_log
ORDER BY action_date DESC
LIMIT 100;
```

### Q16 — INTERSECT: customers loyal across three years

```sql
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2022
INTERSECT
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2023
INTERSECT
SELECT customer_id FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2024;
```

### Q17 — EXCEPT: products never sold

```sql
SELECT product_id, product_name FROM products
EXCEPT
SELECT DISTINCT p.product_id, p.product_name
FROM   products p
JOIN   order_items oi ON p.product_id = oi.product_id;
```

### Q18 — Derived table: second-highest salary

```sql
SELECT MAX(salary) AS second_highest
FROM   employees
WHERE  salary < (SELECT MAX(salary) FROM employees);
```

### Q19 — Scalar subquery in SELECT + CASE: performance label

```sql
SELECT e.name, e.salary,
       CASE
           WHEN e.salary > 1.2 * (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id)
               THEN 'High Performer'
           WHEN e.salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id)
               THEN 'Above Average'
           ELSE 'Average / Below'
       END AS performance
FROM   employees e;
```

### Q20 — CTE + NOT EXISTS: at-risk customers (90-day no purchase)

```sql
WITH last_order AS (
    SELECT customer_id, MAX(order_date) AS last_purchase
    FROM   orders
    GROUP BY customer_id
)
SELECT c.customer_name, c.email, l.last_purchase,
       CURRENT_DATE - l.last_purchase AS days_inactive
FROM   customers c
JOIN   last_order l ON c.customer_id = l.customer_id
WHERE  l.last_purchase < CURRENT_DATE - INTERVAL '90 days'
ORDER BY days_inactive DESC;
```

### Q21 — CTE + window: running revenue total by month

```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', o.order_date) AS month,
           SUM(oi.qty * oi.unit_price)        AS revenue
    FROM   orders o
    JOIN   order_items oi ON o.order_id = oi.order_id
    GROUP BY 1
)
SELECT month, revenue,
       SUM(revenue) OVER (ORDER BY month) AS running_total
FROM   monthly
ORDER BY month;
```

### Q22 — NOT IN (safe): employees with no active project

```sql
SELECT emp_id, name
FROM   employees
WHERE  emp_id NOT IN (
    SELECT DISTINCT emp_id
    FROM   project_assignments
    WHERE  emp_id IS NOT NULL    -- always guard NOT IN against NULLs
      AND  status = 'Active'
);
```

---

## 16. Common Interview Questions

**Q1. What is the difference between a correlated and a non-correlated subquery?**  
A non-correlated subquery runs once, independently of the outer query, and its result is used as a constant. A correlated subquery references a column from the outer query and re-executes once per outer row — making it equivalent to a nested loop and potentially slow on large tables.

**Q2. Why does NOT IN return zero rows when the subquery contains a NULL?**  
`NOT IN` translates to a series of `<>` comparisons. Comparing any value to `NULL` yields `UNKNOWN` (not `TRUE`), so no row ever satisfies the condition. Solution: use `NOT EXISTS`, or add `WHERE col IS NOT NULL` inside the subquery.

**Q3. What is a CTE and how is it different from a subquery?**  
A CTE (Common Table Expression) is a named, temporary result set defined with `WITH`. Unlike a subquery, it can be referenced multiple times within the same statement and supports recursion. It does not persist beyond the query.

**Q4. When would you use a recursive CTE?**  
For hierarchical or iterative data: org charts, category trees, bill of materials, generating date/number sequences, and graph traversal.

**Q5. What is the difference between UNION and UNION ALL?**  
`UNION` deduplicates the combined result set (implicit `DISTINCT`), which requires sorting. `UNION ALL` keeps all rows including duplicates and is faster. Always use `UNION ALL` unless you specifically need deduplication.

**Q6. What does EXISTS return and how does it differ from IN?**  
`EXISTS` returns a boolean — `TRUE` the moment the subquery finds one matching row (short-circuit). `IN` materialises the entire subquery result and then checks membership. `EXISTS` is generally preferred for correlated checks and handles NULLs safely.

**Q7. How do you find the Nth highest salary?**

```sql
-- Window function approach (portable, recommended)
WITH ranked AS (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM   employees
)
SELECT DISTINCT salary FROM ranked WHERE rnk = 3;  -- 3rd highest
```

**Q8. What is COALESCE and how is it different from NULLIF?**  
`COALESCE(a,b,c)` returns the first non-NULL argument — used to substitute defaults for NULLs. `NULLIF(a,b)` converts `a` to `NULL` when `a = b`, typically used to make sentinel values (0, empty string) behave as proper NULLs for downstream functions.

**Q9. When does a scalar subquery fail?**  
When it returns more than one row. Guard it with an aggregate (`MAX`, `MIN`, `SUM`) or `LIMIT 1`. A zero-row result returns `NULL`, not an error.

**Q10. How do you write a query to find duplicate email addresses?**

```sql
SELECT email, COUNT(*) AS occurrences
FROM   customers
GROUP BY email
HAVING COUNT(*) > 1;
```

**Q11. What is INTERSECT and how is it different from INNER JOIN?**  
`INTERSECT` compares complete rows across two result sets and returns only rows that appear in both, deduplicated. `INNER JOIN` matches rows based on a specified key condition and can return multiple result rows per match. `INTERSECT` is equivalent to a self-join with equality on all columns plus a `DISTINCT`.

**Q12. What is the difference between WHERE and HAVING when using subqueries?**  
`WHERE` filters individual rows **before** grouping — aggregate functions are not allowed there. `HAVING` filters **groups** after `GROUP BY` — aggregate functions are allowed. Subqueries in `HAVING` can reference aggregates; subqueries in `WHERE` cannot reference the current query's aggregate results.

---

## 17. Key Takeaways

| Concept | One-line rule |
|---------|--------------|
| **Non-correlated subquery** | Runs once; treat result as a constant |
| **Correlated subquery** | Runs per outer row — rewrite as JOIN for large tables |
| **EXISTS** | Boolean, short-circuits, NULL-safe — prefer over IN for existence checks |
| **NOT IN** | Dangerous with NULLs — use NOT EXISTS instead |
| **CTE** | Named step, flat structure, reusable in same query |
| **Recursive CTE** | Hierarchies & sequences — always include a termination condition |
| **CASE** | Conditional expression usable anywhere — the pivot workhorse |
| **COALESCE** | First non-NULL wins — your default-for-NULL tool |
| **NULLIF** | Value → NULL converter — prevents divide-by-zero and sentinel pollution |
| **UNION ALL** | Default for combining result sets; use UNION only when dedup is needed |
| **INTERSECT** | Rows common to both queries |
| **EXCEPT** | Rows in first query absent from second |

### Decision Flowchart

```
Combining rows from two queries?
  ├─ Need all rows (keep dupes)?     → UNION ALL
  ├─ Need distinct rows?             → UNION
  ├─ Rows appearing in both?         → INTERSECT
  └─ Rows in first, not second?      → EXCEPT

Filtering based on another query?
  ├─ Checking existence only?        → EXISTS / NOT EXISTS
  ├─ Small hard-coded list?          → IN ('a','b','c')
  ├─ Column might be NULL?           → NOT EXISTS (not NOT IN)
  └─ Greater/less than all values?   → ALL / ANY

Structuring complex logic?
  ├─ One-time inline use?            → Derived table / subquery
  ├─ Referenced multiple times?      → CTE
  ├─ Hierarchical traversal?         → Recursive CTE
  └─ Needs index mid-pipeline?       → Temp Table
```

---

*Phase 8 complete. Phase 9 covers Views (`09_views.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="08_advanced_queries.md" download="08_advanced_queries.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 08_advanced_queries.md
  </button>
</a>
