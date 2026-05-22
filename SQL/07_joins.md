# Phase 7 — Joins (All Types)

> **Curriculum:** DBMS A-to-Z | **File:** `07_joins.md`  
> **Prerequisites:** Phases 1–6 (Foundations → Normalization)

---

## Table of Contents

1. [What Is a Join?](#1-what-is-a-join)
2. [INNER JOIN](#2-inner-join)
3. [LEFT (OUTER) JOIN](#3-left-outer-join)
4. [RIGHT (OUTER) JOIN](#4-right-outer-join)
5. [FULL OUTER JOIN](#5-full-outer-join)
6. [CROSS JOIN](#6-cross-join)
7. [SELF JOIN](#7-self-join)
8. [NATURAL JOIN](#8-natural-join)
9. [LATERAL JOIN](#9-lateral-join)
10. [USING vs ON](#10-using-vs-on)
11. [SVG Venn Diagrams — Complete Reference](#11-svg-venn-diagrams--complete-reference)
12. [Performance Implications of Each Join](#12-performance-implications-of-each-join)
13. [15+ Join Query Examples with Output Tables](#13-15-join-query-examples-with-output-tables)
14. [Common Interview Questions](#14-common-interview-questions)
15. [Key Takeaways](#15-key-takeaways)

---

## 1. What Is a Join?

A **join** combines rows from two or more tables based on a related column between them. Joins are the backbone of relational databases — without them, normalised tables would be isolated islands of data.

### Real-World Analogy

Imagine a library with two filing cabinets:
- **Cabinet A** (Students): each folder has a student's name and their student ID.
- **Cabinet B** (Enrollments): each folder has a student ID and the course they enrolled in.

To find out which student enrolled in which course, you open a folder in Cabinet A, read the student ID, then walk to Cabinet B and find every folder with the same student ID. That matching process is a **join**.

### Sample Tables Used Throughout This Phase

We'll use these small tables so every example can be traced by hand.

```sql
-- Table: employees
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(50),
    dept_id     INT,
    manager_id  INT,
    salary      NUMERIC(10,2)
);

INSERT INTO employees VALUES
(1, 'Alice',   10, NULL, 95000),
(2, 'Bob',     20, 1,    78000),
(3, 'Carol',   10, 1,    85000),
(4, 'David',   30, 2,    62000),
(5, 'Eve',     NULL, 3,  71000);   -- no department assigned

-- Table: departments
CREATE TABLE departments (
    dept_id     INT PRIMARY KEY,
    dept_name   VARCHAR(50),
    budget      NUMERIC(12,2)
);

INSERT INTO departments VALUES
(10, 'Engineering',  500000),
(20, 'Marketing',    200000),
(40, 'Finance',      350000);      -- no employees in dept 40
-- Note: dept_id 30 (David's dept) does NOT exist in departments
```

**Data summary at a glance:**

| employees (5 rows) | departments (3 rows) |
|---------------------|----------------------|
| Alice → dept 10 ✅ match | 10 Engineering |
| Bob → dept 20 ✅ match   | 20 Marketing |
| Carol → dept 10 ✅ match | 40 Finance ❌ no employees |
| David → dept 30 ❌ no match | |
| Eve → dept NULL ❌ no match | |

---

## 2. INNER JOIN

### Concept

Returns **only** the rows where a match exists in **both** tables. Non-matching rows from either side are excluded.

### Syntax

```sql
SELECT columns
FROM   table_a  a
INNER JOIN table_b  b  ON a.key = b.key;

-- The keyword INNER is optional — JOIN alone defaults to INNER JOIN.
```

### Beginner Example — Students & Courses

```sql
SELECT s.name, c.course_name
FROM   students     s
JOIN   enrollments  e ON s.student_id = e.student_id
JOIN   courses      c ON e.course_id  = c.course_id;
```

A student with no enrollments? Excluded. A course nobody enrolled in? Excluded.

### Intermediate Example — Our employees table

```sql
SELECT e.emp_id,
       e.name,
       d.dept_name
FROM   employees   e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

| emp_id | name  | dept_name   |
|--------|-------|-------------|
| 1      | Alice | Engineering |
| 2      | Bob   | Marketing   |
| 3      | Carol | Engineering |

- **David** (dept_id = 30) → 30 not in departments → excluded.
- **Eve** (dept_id = NULL) → NULL never equals anything → excluded.
- **Finance** (dept_id = 40) → no employee has dept_id 40 → excluded.

### Advanced Example — Banking: transactions with account & branch

```sql
SELECT t.txn_id, a.account_number, b.branch_name, t.amount
FROM   transactions t
JOIN   accounts     a ON t.account_id = a.account_id
JOIN   branches     b ON a.branch_id  = b.branch_id
WHERE  t.amount > 10000;
```

---

## 3. LEFT (OUTER) JOIN

### Concept

Returns **all rows from the left table**, plus matching rows from the right table. Where there is no match, right-side columns are filled with `NULL`.

### Syntax

```sql
SELECT columns
FROM   table_a  a
LEFT JOIN table_b  b  ON a.key = b.key;

-- LEFT OUTER JOIN and LEFT JOIN are identical.
```

### Example — All employees, show department or NULL

```sql
SELECT e.emp_id,
       e.name,
       d.dept_name
FROM   employees   e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

| emp_id | name  | dept_name   |
|--------|-------|-------------|
| 1      | Alice | Engineering |
| 2      | Bob   | Marketing   |
| 3      | Carol | Engineering |
| 4      | David | NULL        |
| 5      | Eve   | NULL        |

David and Eve appear with NULLs on the right side.

### Anti-Join Pattern — find unmatched left rows

```sql
-- Employees who have NO matching department
SELECT e.name
FROM   employees   e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE  d.dept_id IS NULL;
```

| name  |
|-------|
| David |
| Eve   |

This pattern (LEFT JOIN + WHERE right-key IS NULL) is called a **left anti-join** — one of the most useful patterns in SQL.

### E-commerce Example — Customers who never ordered

```sql
SELECT c.customer_name, c.email
FROM   customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE  o.order_id IS NULL;
```

---

## 4. RIGHT (OUTER) JOIN

### Concept

The mirror image of LEFT JOIN. Returns **all rows from the right table**, plus matching rows from the left. Left-side columns are NULL where there is no match.

### Syntax

```sql
SELECT columns
FROM   table_a  a
RIGHT JOIN table_b  b  ON a.key = b.key;
```

### Example — All departments, even those with no employees

```sql
SELECT e.name,
       d.dept_name
FROM   employees   e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

| name  | dept_name   |
|-------|-------------|
| Alice | Engineering |
| Carol | Engineering |
| Bob   | Marketing   |
| NULL  | Finance     |

Finance has no employees → left side is NULL.

### Practical Note

Any RIGHT JOIN can be rewritten as a LEFT JOIN by swapping the table order. Most teams prefer LEFT JOIN exclusively for consistency and readability:

```sql
-- These are equivalent:
FROM employees e RIGHT JOIN departments d ON ...
FROM departments d LEFT JOIN employees e ON ...
```

---

## 5. FULL OUTER JOIN

### Concept

Returns **all rows from both tables**. Matches are combined; unmatched rows from either side get NULLs on the opposite side.

### Syntax

```sql
SELECT columns
FROM   table_a  a
FULL OUTER JOIN table_b  b  ON a.key = b.key;

-- MySQL does NOT support FULL OUTER JOIN natively.
```

### Example

```sql
SELECT e.name,
       d.dept_name
FROM   employees   e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```

| name  | dept_name   |
|-------|-------------|
| Alice | Engineering |
| Carol | Engineering |
| Bob   | Marketing   |
| David | NULL        |
| Eve   | NULL        |
| NULL  | Finance     |

Every row from both sides appears. David/Eve have no department → right-side NULL. Finance has no employees → left-side NULL.

### MySQL Workaround

```sql
SELECT e.name, d.dept_name
FROM   employees e LEFT JOIN departments d ON e.dept_id = d.dept_id
UNION
SELECT e.name, d.dept_name
FROM   employees e RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

### Full Outer Anti-Join — rows with no match on either side

```sql
SELECT COALESCE(e.name, '(no employee)') AS emp,
       COALESCE(d.dept_name, '(no dept)')  AS dept
FROM   employees   e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id
WHERE  e.emp_id IS NULL OR d.dept_id IS NULL;
```

| emp           | dept       |
|---------------|------------|
| David         | (no dept)  |
| Eve           | (no dept)  |
| (no employee) | Finance    |

---

## 6. CROSS JOIN

### Concept

Returns the **Cartesian product** — every row of the left table paired with every row of the right table. No join condition is used.

If left has **M** rows and right has **N** rows, the result has **M × N** rows.

### Syntax

```sql
-- Explicit syntax (preferred)
SELECT * FROM table_a CROSS JOIN table_b;

-- Implicit comma syntax (equivalent but less readable)
SELECT * FROM table_a, table_b;
```

### Example — Product variant generator

```sql
CREATE TABLE sizes  (size_label VARCHAR(5));
CREATE TABLE colors (color_name VARCHAR(20));

INSERT INTO sizes  VALUES ('S'), ('M'), ('L');
INSERT INTO colors VALUES ('Red'), ('Blue');

SELECT s.size_label, c.color_name,
       s.size_label || '-' || c.color_name AS sku
FROM   sizes s
CROSS JOIN colors c
ORDER BY s.size_label, c.color_name;
```

| size_label | color_name | sku    |
|------------|------------|--------|
| L          | Blue       | L-Blue |
| L          | Red        | L-Red  |
| M          | Blue       | M-Blue |
| M          | Red        | M-Red  |
| S          | Blue       | S-Blue |
| S          | Red        | S-Red  |

3 sizes × 2 colours = **6 rows**.

### Other use cases

- **Calendar grids**: CROSS JOIN a year-months table with day-numbers.
- **Test matrices**: every user × every feature flag combination.
- **Pivot scaffolding**: all categories × all time periods, then LEFT JOIN facts onto it.

> **Warning:** CROSS JOIN on large tables creates explosively large result sets. 10,000 × 10,000 = 100 million rows. Always be intentional.

---

## 7. SELF JOIN

### Concept

A table joined **to itself** using aliases. This is not a distinct SQL keyword — you use any join type (INNER, LEFT, etc.) but give the same table two different aliases. Self joins model relationships **within** a single table: hierarchies, comparisons, duplicate detection.

### Example 1 — Employee-Manager hierarchy

```sql
SELECT e.name  AS employee,
       m.name  AS manager
FROM   employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

| employee | manager |
|----------|---------|
| Alice    | NULL    |
| Bob      | Alice   |
| Carol    | Alice   |
| David    | Bob     |
| Eve      | Carol   |

Alice has no manager (top of hierarchy). Bob and Carol report to Alice.

### Example 2 — Employees earning more than their manager

```sql
SELECT e.name   AS employee,
       e.salary AS emp_salary,
       m.name   AS manager,
       m.salary AS mgr_salary
FROM   employees e
JOIN   employees m ON e.manager_id = m.emp_id
WHERE  e.salary > m.salary;
```

### Example 3 — All pairs of employees in the same department

```sql
SELECT a.name AS emp_1,
       b.name AS emp_2,
       a.dept_id
FROM   employees a
JOIN   employees b ON a.dept_id = b.dept_id
                  AND a.emp_id < b.emp_id;   -- avoids (Alice,Carol) AND (Carol,Alice)
```

| emp_1 | emp_2 | dept_id |
|-------|-------|---------|
| Alice | Carol | 10      |

### Example 4 — Detecting duplicate emails (real-world)

```sql
SELECT a.customer_id AS id_1,
       b.customer_id AS id_2,
       a.email
FROM   customers a
JOIN   customers b ON a.email = b.email
                  AND a.customer_id < b.customer_id;
```

---

## 8. NATURAL JOIN

### Concept

Automatically joins on **all columns with the same name** in both tables. No `ON` or `USING` clause.

```sql
SELECT * FROM employees NATURAL JOIN departments;
-- Implicitly matches on dept_id (the shared column name)
```

### Why NATURAL JOIN Is Dangerous

| Problem | What happens |
|---------|-------------|
| **Silent breakage** | Adding a column with the same name to either table silently changes the join condition — wrong results, no error. |
| **Ambiguity** | You cannot tell which columns are being used without inspecting both schemas. |
| **Maintenance risk** | A routine schema migration can break queries in production without anyone noticing. |
| **Wildcard behavior** | `SELECT *` returns the join column only once, which can confuse ORMs and application code. |

> **Best practice:** Never use NATURAL JOIN in production. Always use explicit `ON` or `USING`.

---

## 9. LATERAL JOIN

### Concept

A `LATERAL` join allows the right-side subquery to **reference columns from the left table** — like a correlated subquery that returns a table. The subquery re-executes for each row of the left table.

**Supported in:** PostgreSQL, MySQL 8.0+, Oracle 12c+. SQL Server uses `CROSS APPLY` / `OUTER APPLY` as equivalents.

### When to use LATERAL

- **Top-N per group** (far cleaner than window functions for this)
- **Unnesting arrays or JSON** (referencing the outer row's column)
- **Dependent subqueries that return multiple columns**

### Example 1 — Top 2 earners per department (PostgreSQL)

```sql
SELECT d.dept_name,
       top.name,
       top.salary
FROM   departments d
CROSS JOIN LATERAL (
    SELECT e.name, e.salary
    FROM   employees e
    WHERE  e.dept_id = d.dept_id     -- references left table
    ORDER BY e.salary DESC
    LIMIT 2
) AS top;
```

`CROSS JOIN LATERAL` excludes departments with no employees (like INNER). For departments with no employees to still appear (with NULLs), use `LEFT JOIN LATERAL ... ON TRUE`.

### Example 2 — LEFT JOIN LATERAL (include empty departments)

```sql
SELECT d.dept_name,
       top.name,
       top.salary
FROM   departments d
LEFT JOIN LATERAL (
    SELECT e.name, e.salary
    FROM   employees e
    WHERE  e.dept_id = d.dept_id
    ORDER BY e.salary DESC
    LIMIT 2
) AS top ON TRUE;
```

Finance (no employees) appears with NULLs for name and salary.

### SQL Server Equivalent — CROSS APPLY / OUTER APPLY

```sql
-- CROSS APPLY ≈ CROSS JOIN LATERAL (inner)
SELECT d.dept_name, t.name, t.salary
FROM   departments d
CROSS APPLY (
    SELECT TOP 2 e.name, e.salary
    FROM   employees e
    WHERE  e.dept_id = d.dept_id
    ORDER BY e.salary DESC
) t;

-- OUTER APPLY ≈ LEFT JOIN LATERAL (outer)
SELECT d.dept_name, t.name, t.salary
FROM   departments d
OUTER APPLY (
    SELECT TOP 2 e.name, e.salary
    FROM   employees e
    WHERE  e.dept_id = d.dept_id
    ORDER BY e.salary DESC
) t;
```

### LATERAL Execution Flow

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="480" height="200" viewBox="0 0 480 200">
  <style>text{font-family:Arial,sans-serif;font-size:12px}</style>
  <rect x="10" y="20" width="140" height="150" fill="#D5E8D4" stroke="#82B366" rx="8"/>
  <text x="80" y="48" text-anchor="middle" font-weight="bold" fill="#2D6A2D">Left Table</text>
  <text x="80" y="72" text-anchor="middle" fill="#2D6A2D">Engineering</text>
  <text x="80" y="92" text-anchor="middle" fill="#2D6A2D">Marketing</text>
  <text x="80" y="112" text-anchor="middle" fill="#2D6A2D">Finance</text>

  <line x1="150" y1="72" x2="210" y2="72" stroke="#E74C3C" stroke-width="2" marker-end="url(#lat1)"/>
  <line x1="150" y1="92" x2="210" y2="92" stroke="#E74C3C" stroke-width="2" marker-end="url(#lat1)"/>
  <line x1="150" y1="112" x2="210" y2="112" stroke="#E74C3C" stroke-width="2" marker-end="url(#lat1)"/>
  <defs><marker id="lat1" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3z" fill="#E74C3C"/></marker></defs>

  <text x="180" y="56" text-anchor="middle" fill="#C0392B" font-size="10">row by row</text>

  <rect x="210" y="20" width="250" height="150" fill="#DAE8FC" stroke="#6C8EBF" rx="8"/>
  <text x="335" y="48" text-anchor="middle" font-weight="bold" fill="#1A3A5C">LATERAL Subquery</text>
  <text x="335" y="72" text-anchor="middle" fill="#1A3A5C">WHERE e.dept_id = d.dept_id</text>
  <text x="335" y="96" text-anchor="middle" fill="#1A3A5C">ORDER BY e.salary DESC</text>
  <text x="335" y="120" text-anchor="middle" fill="#1A3A5C">LIMIT 2</text>
  <text x="335" y="148" text-anchor="middle" fill="#7D3C98" font-size="10" font-style="italic">Re-executes per left row</text>

  <text x="240" y="192" fill="#555" font-size="11" font-style="italic">SQL Server: CROSS APPLY / OUTER APPLY</text>
</svg>
```

---

## 10. USING vs ON

### ON — full flexibility

```sql
-- Any condition, different column names, multiple conditions
SELECT e.name, d.dept_name
FROM   employees   e
JOIN   departments d ON e.dept_id = d.dept_id
                     AND d.budget > 200000;
```

### USING — shorthand for same-named columns

```sql
-- Both tables share the column name dept_id
SELECT e.name, dept_name
FROM   employees e
JOIN   departments USING (dept_id);
```

`USING` requires the join column to have the **identical name** in both tables. In `SELECT *`, the column appears only once instead of twice.

### Comparison

| Feature | `ON` | `USING` | `NATURAL` |
|---------|------|---------|-----------|
| Arbitrary conditions | ✅ Yes | ❌ No | ❌ No |
| Different column names | ✅ Yes | ❌ No | ❌ No |
| Must name columns | ✅ Explicit | ✅ Explicit | ❌ Implicit |
| `SELECT *` behaviour | Both columns appear | Column appears once | Column appears once |
| Production-safe | ✅ Always | ✅ When names are clear | ⚠️ Risky |

**Recommendation:** Use `ON` by default. Use `USING` when both tables genuinely share a well-named key and you want cleaner `SELECT *` output.

---

## 11. SVG Venn Diagrams — Complete Reference

### All Join Types at a Glance

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="700" height="500" viewBox="0 0 700 500">
  <style>
    text { font-family: Arial, sans-serif; }
    .title { font-size: 13px; font-weight: bold; }
    .sub { font-size: 10px; fill: #555; }
  </style>

  <!-- ===== INNER JOIN ===== -->
  <circle cx="90"  cy="70" r="50" fill="#B3D9FF" fill-opacity="0.4" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="150" cy="70" r="50" fill="#FFD9B3" fill-opacity="0.4" stroke="#E67E22" stroke-width="1.5"/>
  <path d="M120,24 A50,50 0 0,1 120,116 A50,50 0 0,1 120,24" fill="#27AE60" fill-opacity="0.75"/>
  <text x="120" y="74" text-anchor="middle" fill="white" class="title">INNER</text>
  <text x="120" y="140" text-anchor="middle" class="sub">Only matching rows</text>

  <!-- ===== LEFT JOIN ===== -->
  <circle cx="310" cy="70" r="50" fill="#2980B9" fill-opacity="0.65" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="370" cy="70" r="50" fill="#FFD9B3" fill-opacity="0.2" stroke="#E67E22" stroke-width="1.5"/>
  <text x="340" y="74" text-anchor="middle" fill="white" class="title">LEFT</text>
  <text x="340" y="140" text-anchor="middle" class="sub">All left + matched right</text>

  <!-- ===== RIGHT JOIN ===== -->
  <circle cx="530" cy="70" r="50" fill="#B3D9FF" fill-opacity="0.2" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="590" cy="70" r="50" fill="#E67E22" fill-opacity="0.65" stroke="#E67E22" stroke-width="1.5"/>
  <text x="560" y="74" text-anchor="middle" fill="white" class="title">RIGHT</text>
  <text x="560" y="140" text-anchor="middle" class="sub">All right + matched left</text>

  <!-- ===== FULL OUTER JOIN ===== -->
  <circle cx="90"  cy="240" r="50" fill="#2980B9" fill-opacity="0.55" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="150" cy="240" r="50" fill="#E67E22" fill-opacity="0.55" stroke="#E67E22" stroke-width="1.5"/>
  <text x="120" y="244" text-anchor="middle" fill="white" class="title">FULL</text>
  <text x="120" y="310" text-anchor="middle" class="sub">All rows from both sides</text>

  <!-- ===== LEFT ANTI-JOIN ===== -->
  <circle cx="310" cy="240" r="50" fill="#2980B9" fill-opacity="0.65" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="370" cy="240" r="50" fill="white" fill-opacity="0.9" stroke="#E67E22" stroke-width="1.5"/>
  <path d="M340,194 A50,50 0 0,1 340,286 A50,50 0 0,1 340,194" fill="white" fill-opacity="0.95"/>
  <text x="295" y="244" text-anchor="middle" fill="white" class="title">LEFT</text>
  <text x="340" y="310" text-anchor="middle" class="sub">LEFT JOIN … WHERE right IS NULL</text>

  <!-- ===== FULL OUTER ANTI-JOIN ===== -->
  <circle cx="530" cy="240" r="50" fill="#2980B9" fill-opacity="0.55" stroke="#2980B9" stroke-width="1.5"/>
  <circle cx="590" cy="240" r="50" fill="#E67E22" fill-opacity="0.55" stroke="#E67E22" stroke-width="1.5"/>
  <path d="M560,194 A50,50 0 0,1 560,286 A50,50 0 0,1 560,194" fill="white" fill-opacity="0.95"/>
  <text x="560" y="244" text-anchor="middle" fill="#333" class="title">EXCL</text>
  <text x="560" y="310" text-anchor="middle" class="sub">FULL OUTER … WHERE either IS NULL</text>

  <!-- ===== CROSS JOIN ===== -->
  <rect x="40" y="370" width="80" height="80" fill="#AED6F1" stroke="#2980B9" rx="6"/>
  <text x="80" y="400" text-anchor="middle" fill="#1A5276" font-size="10">Table A</text>
  <text x="80" y="416" text-anchor="middle" fill="#1A5276" font-size="10">(M rows)</text>
  <text x="145" y="410" text-anchor="middle" fill="#333" font-size="16">×</text>
  <rect x="160" y="370" width="80" height="80" fill="#FAD7A0" stroke="#E67E22" rx="6"/>
  <text x="200" y="400" text-anchor="middle" fill="#784212" font-size="10">Table B</text>
  <text x="200" y="416" text-anchor="middle" fill="#784212" font-size="10">(N rows)</text>
  <text x="260" y="410" text-anchor="middle" fill="#333" font-size="14">=</text>
  <text x="310" y="410" text-anchor="middle" fill="#C0392B" font-weight="bold" font-size="12">M × N</text>
  <text x="160" y="475" text-anchor="middle" class="sub">CROSS JOIN: Cartesian product</text>

  <!-- ===== SELF JOIN ===== -->
  <circle cx="500" cy="410" r="50" fill="#A9DFBF" fill-opacity="0.5" stroke="#1E8449" stroke-width="1.5"/>
  <circle cx="540" cy="410" r="50" fill="#A9DFBF" fill-opacity="0.5" stroke="#1E8449" stroke-width="1.5" stroke-dasharray="5"/>
  <text x="520" y="414" text-anchor="middle" fill="#145A32" class="title">SELF</text>
  <text x="520" y="475" text-anchor="middle" class="sub">Same table, two aliases</text>
</svg>
```

---

## 12. Performance Implications of Each Join

### The Three Join Algorithms

Every database engine chooses among three physical algorithms to execute a join. Understanding them helps you write faster queries and read EXPLAIN plans.

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="660" height="260" viewBox="0 0 660 260">
  <style>text{font-family:Arial,sans-serif;font-size:11px}.head{font-size:13px;font-weight:bold}</style>

  <!-- Nested Loop -->
  <rect x="10" y="10" width="200" height="210" fill="#EBF5FB" stroke="#2E86C1" rx="8"/>
  <text x="110" y="34" text-anchor="middle" class="head" fill="#1A5276">Nested Loop Join</text>
  <line x1="20" y1="42" x2="200" y2="42" stroke="#AED6F1"/>
  <text x="110" y="62" text-anchor="middle" fill="#333">For each outer row,</text>
  <text x="110" y="78" text-anchor="middle" fill="#333">scan inner table</text>
  <text x="110" y="104" text-anchor="middle" fill="#27AE60">✓ Small outer table</text>
  <text x="110" y="120" text-anchor="middle" fill="#27AE60">✓ Index on inner key</text>
  <text x="110" y="142" text-anchor="middle" fill="#E74C3C">✗ O(M × N) w/o index</text>
  <text x="110" y="170" text-anchor="middle" fill="#7D6608" font-style="italic">Best for: indexed FK</text>
  <text x="110" y="186" text-anchor="middle" fill="#7D6608" font-style="italic">lookups, small tables</text>

  <!-- Hash Join -->
  <rect x="230" y="10" width="200" height="210" fill="#FDEDEC" stroke="#C0392B" rx="8"/>
  <text x="330" y="34" text-anchor="middle" class="head" fill="#7B241C">Hash Join</text>
  <line x1="240" y1="42" x2="420" y2="42" stroke="#F5B7B1"/>
  <text x="330" y="62" text-anchor="middle" fill="#333">Build hash on smaller table;</text>
  <text x="330" y="78" text-anchor="middle" fill="#333">probe with larger table</text>
  <text x="330" y="104" text-anchor="middle" fill="#27AE60">✓ Large unsorted equi-joins</text>
  <text x="330" y="120" text-anchor="middle" fill="#27AE60">✓ O(M + N) time</text>
  <text x="330" y="142" text-anchor="middle" fill="#E74C3C">✗ Needs memory for hash</text>
  <text x="330" y="170" text-anchor="middle" fill="#7D6608" font-style="italic">Best for: large table joins</text>
  <text x="330" y="186" text-anchor="middle" fill="#7D6608" font-style="italic">without useful indexes</text>

  <!-- Sort-Merge Join -->
  <rect x="450" y="10" width="200" height="210" fill="#EAFAF1" stroke="#1E8449" rx="8"/>
  <text x="550" y="34" text-anchor="middle" class="head" fill="#145A32">Sort-Merge Join</text>
  <line x1="460" y1="42" x2="640" y2="42" stroke="#A9DFBF"/>
  <text x="550" y="62" text-anchor="middle" fill="#333">Sort both sides, then</text>
  <text x="550" y="78" text-anchor="middle" fill="#333">merge like a zipper</text>
  <text x="550" y="104" text-anchor="middle" fill="#27AE60">✓ Both sides pre-sorted</text>
  <text x="550" y="120" text-anchor="middle" fill="#27AE60">✓ Great for range joins</text>
  <text x="550" y="142" text-anchor="middle" fill="#E74C3C">✗ Sort cost if unsorted</text>
  <text x="550" y="170" text-anchor="middle" fill="#7D6608" font-style="italic">Best for: indexed range</text>
  <text x="550" y="186" text-anchor="middle" fill="#7D6608" font-style="italic">joins, merge of sorted sets</text>

  <text x="330" y="245" text-anchor="middle" fill="#7D3C98" font-size="12" font-style="italic">The query planner picks automatically — use EXPLAIN ANALYZE to see which was chosen.</text>
</svg>
```

### Performance Rules of Thumb

| # | Rule | Why |
|---|------|-----|
| 1 | **Index the foreign key** | Turns nested-loop inner scan from O(N) to O(log N). Single biggest speedup. |
| 2 | **Filter before joining** | Smaller input → fewer comparisons. Push `WHERE` predicates as deep as possible. |
| 3 | **Avoid functions on join keys** | `ON YEAR(a.date) = b.year` prevents index use. Compute once and store, or use range predicates. |
| 4 | **Prefer EXISTS over IN with large subqueries** | EXISTS short-circuits at the first match. |
| 5 | **LEFT JOIN costs ≈ INNER JOIN** | The optimizer handles them similarly. Don't force INNER JOIN "for performance." |
| 6 | **CROSS JOIN is intentional danger** | 10K × 10K = 100M rows. Always sanity-check row counts. |
| 7 | **EXPLAIN ANALYZE your queries** | Trust the optimizer, but verify. Look for sequential scans on large tables. |

### Reading EXPLAIN output (PostgreSQL example)

```sql
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM   employees e
JOIN   departments d ON e.dept_id = d.dept_id;

-- Output:
-- Hash Join  (cost=1.09..2.23 rows=3 width=88) (actual time=0.04..0.06 rows=3 loops=1)
--   Hash Cond: (e.dept_id = d.dept_id)
--   -> Seq Scan on employees e   (cost=0.00..1.05 rows=5 width=54)
--   -> Hash  (cost=1.03..1.03 rows=3 width=42)
--         -> Seq Scan on departments d  (cost=0.00..1.03 rows=3 width=42)
-- Planning Time: 0.15 ms
-- Execution Time: 0.08 ms
```

Key things to look for: **join type** (Hash/Nested Loop/Merge), **Seq Scan vs Index Scan** on large tables, and **actual rows** vs **estimated rows** (a big mismatch means stale statistics).

---

## 13. 15+ Join Query Examples with Output Tables

### Setup Reminder

All queries use the `employees` and `departments` tables from Section 1, plus additional schemas where noted.

---

### Q1 — INNER JOIN: employees with department name

```sql
SELECT e.emp_id, e.name, d.dept_name
FROM   employees e
JOIN   departments d ON e.dept_id = d.dept_id;
```

| emp_id | name  | dept_name   |
|--------|-------|-------------|
| 1      | Alice | Engineering |
| 2      | Bob   | Marketing   |
| 3      | Carol | Engineering |

---

### Q2 — LEFT JOIN: all employees, department or placeholder

```sql
SELECT e.name,
       COALESCE(d.dept_name, '— unassigned —') AS department
FROM   employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

| name  | department      |
|-------|-----------------|
| Alice | Engineering     |
| Bob   | Marketing       |
| Carol | Engineering     |
| David | — unassigned —  |
| Eve   | — unassigned —  |

---

### Q3 — LEFT anti-join: employees with no department

```sql
SELECT e.name
FROM   employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE  d.dept_id IS NULL;
```

| name  |
|-------|
| David |
| Eve   |

---

### Q4 — RIGHT JOIN: department headcount (include empty departments)

```sql
SELECT d.dept_name,
       COUNT(e.emp_id) AS headcount
FROM   employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name
ORDER BY headcount DESC;
```

| dept_name   | headcount |
|-------------|-----------|
| Engineering | 2         |
| Marketing   | 1         |
| Finance     | 0         |

---

### Q5 — FULL OUTER JOIN: complete picture

```sql
SELECT COALESCE(e.name, '(none)')     AS employee,
       COALESCE(d.dept_name, '(none)') AS department
FROM   employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```

| employee | department  |
|----------|-------------|
| Alice    | Engineering |
| Carol    | Engineering |
| Bob      | Marketing   |
| David    | (none)      |
| Eve      | (none)      |
| (none)   | Finance     |

---

### Q6 — SELF JOIN: employee-manager hierarchy

```sql
SELECT e.name AS employee,
       COALESCE(m.name, '— CEO —') AS manager
FROM   employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY m.name NULLS FIRST;
```

| employee | manager   |
|----------|-----------|
| Alice    | — CEO —   |
| Bob      | Alice     |
| Carol    | Alice     |
| David    | Bob       |
| Eve      | Carol     |

---

### Q7 — SELF JOIN: employees earning more than their manager

```sql
SELECT e.name   AS employee,  e.salary,
       m.name   AS manager,   m.salary AS mgr_salary
FROM   employees e
JOIN   employees m ON e.manager_id = m.emp_id
WHERE  e.salary > m.salary;
```

| employee | salary | manager | mgr_salary |
|----------|--------|---------|------------|
| Carol    | 85000  | Alice   | 95000      |

_(Carol does NOT earn more. If salaries were different, matches would appear.)_

---

### Q8 — CROSS JOIN: size × colour product variants

```sql
SELECT s.size_label, c.color_name
FROM   sizes s CROSS JOIN colors c
ORDER BY s.size_label, c.color_name;
```

| size_label | color_name |
|------------|------------|
| L          | Blue       |
| L          | Red        |
| M          | Blue       |
| M          | Red        |
| S          | Blue       |
| S          | Red        |

---

### Q9 — Three-table JOIN: order line items with product names

```sql
-- Schema: orders(order_id, customer_id, order_date),
--         order_items(order_id, product_id, qty, unit_price),
--         products(product_id, product_name, category)

SELECT o.order_id,
       o.order_date,
       p.product_name,
       oi.qty,
       oi.unit_price,
       (oi.qty * oi.unit_price) AS line_total
FROM   orders      o
JOIN   order_items oi ON o.order_id    = oi.order_id
JOIN   products    p  ON oi.product_id = p.product_id
ORDER BY o.order_id;
```

---

### Q10 — Non-equi JOIN: salary grade lookup

```sql
-- salary_grades(grade, min_salary, max_salary)
-- ('A', 90000, 999999), ('B', 70000, 89999), ('C', 0, 69999)

SELECT e.name, e.salary, g.grade
FROM   employees     e
JOIN   salary_grades g ON e.salary BETWEEN g.min_salary AND g.max_salary;
```

| name  | salary | grade |
|-------|--------|-------|
| Alice | 95000  | A     |
| Carol | 85000  | B     |
| Bob   | 78000  | B     |
| Eve   | 71000  | B     |
| David | 62000  | C     |

---

### Q11 — Non-equi JOIN: overlapping hotel reservations

```sql
SELECT a.booking_id, b.booking_id AS conflict_id, a.room_id
FROM   bookings a
JOIN   bookings b ON a.room_id    = b.room_id
                 AND a.booking_id < b.booking_id   -- avoid self-match & duplicates
                 AND a.check_in  < b.check_out
                 AND a.check_out > b.check_in;
```

---

### Q12 — LATERAL: top 2 earners per department (PostgreSQL)

```sql
SELECT d.dept_name, t.name, t.salary
FROM   departments d
CROSS JOIN LATERAL (
    SELECT e.name, e.salary
    FROM   employees e
    WHERE  e.dept_id = d.dept_id
    ORDER BY e.salary DESC
    LIMIT 2
) t;
```

| dept_name   | name  | salary |
|-------------|-------|--------|
| Engineering | Alice | 95000  |
| Engineering | Carol | 85000  |
| Marketing   | Bob   | 78000  |

---

### Q13 — JOIN + aggregate: total salary budget per department vs actual budget

```sql
SELECT d.dept_name,
       d.budget,
       SUM(e.salary)                AS total_salaries,
       d.budget - SUM(e.salary)     AS remaining_budget
FROM   departments d
JOIN   employees   e ON d.dept_id = e.dept_id
GROUP BY d.dept_name, d.budget
ORDER BY remaining_budget;
```

---

### Q14 — LEFT JOIN + aggregate: customers with total spend (including $0 customers)

```sql
SELECT c.customer_name,
       COALESCE(SUM(oi.qty * oi.unit_price), 0) AS total_spend
FROM   customers   c
LEFT JOIN orders      o  ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id    = oi.order_id
GROUP BY c.customer_name
ORDER BY total_spend DESC;
```

---

### Q15 — FULL OUTER JOIN: inventory reconciliation between two systems

```sql
SELECT COALESCE(a.sku, b.sku) AS sku,
       COALESCE(a.qty, 0)     AS warehouse_qty,
       COALESCE(b.qty, 0)     AS store_qty,
       COALESCE(a.qty, 0) - COALESCE(b.qty, 0) AS discrepancy
FROM   warehouse_inventory a
FULL OUTER JOIN store_inventory b ON a.sku = b.sku
WHERE  COALESCE(a.qty, 0) <> COALESCE(b.qty, 0)
ORDER BY ABS(COALESCE(a.qty, 0) - COALESCE(b.qty, 0)) DESC;
```

---

### Q16 — Multi-join with window function: rank products by revenue per category

```sql
SELECT p.category,
       p.product_name,
       SUM(oi.qty * oi.unit_price) AS revenue,
       RANK() OVER (
           PARTITION BY p.category
           ORDER BY SUM(oi.qty * oi.unit_price) DESC
       ) AS rank_in_category
FROM   products    p
JOIN   order_items oi ON p.product_id = oi.product_id
GROUP BY p.category, p.product_name
ORDER BY p.category, rank_in_category;
```

---

### Q17 — Semi-join via EXISTS: departments that have at least one employee

```sql
SELECT d.dept_name
FROM   departments d
WHERE  EXISTS (
    SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id
);
```

| dept_name   |
|-------------|
| Engineering |
| Marketing   |

---

## 14. Common Interview Questions

**Q1. What is the difference between INNER JOIN and LEFT JOIN?**  
INNER JOIN returns only rows with matches in both tables. LEFT JOIN returns all left-table rows; unmatched right-side columns are NULL.

**Q2. Can a LEFT JOIN produce more rows than the left table?**  
Yes. If the right table has multiple matches for a single left row, that left row is duplicated. Example: one customer with 5 orders → 5 result rows for that customer.

**Q3. What is a self join? Give a use case.**  
Joining a table to itself with two aliases. Used for: employee-manager hierarchies, finding duplicate records, comparing rows within the same table (e.g., find students in the same city).

**Q4. What is the difference between ON and USING?**  
`ON` accepts any expression (`ON a.x = b.y AND a.z > 10`). `USING(col)` requires identically named columns and returns the column once in `SELECT *`. Use `ON` for flexibility, `USING` for convenience.

**Q5. How do you find rows in A that have no match in B?**  
Three approaches, all equivalent:
```sql
-- 1. LEFT anti-join
SELECT a.* FROM a LEFT JOIN b ON a.id = b.id WHERE b.id IS NULL;
-- 2. NOT EXISTS
SELECT a.* FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.id = a.id);
-- 3. NOT IN (beware NULLs)
SELECT a.* FROM a WHERE a.id NOT IN (SELECT id FROM b WHERE id IS NOT NULL);
```

**Q6. What is a CROSS JOIN and when would you use it?**  
Cartesian product — every combination of rows from both tables (M × N rows). Used to generate all permutations (product variants, calendar grids, test matrices). Never use accidentally.

**Q7. What is a LATERAL JOIN?**  
A join where the right-side subquery can reference columns from the left table (like a correlated subquery that returns a table). Used for top-N-per-group and unnesting. SQL Server equivalent: CROSS APPLY / OUTER APPLY.

**Q8. Why is NATURAL JOIN dangerous?**  
It silently joins on ALL columns with the same name. Adding or renaming a column can change the join condition without any error — just wrong results. Always use explicit `ON` or `USING`.

**Q9. What are the three join algorithms? When does each apply?**  
- **Nested Loop:** small outer table, indexed inner table — O(M × log N).  
- **Hash Join:** large equi-joins without useful indexes — O(M + N).  
- **Sort-Merge Join:** both sides already sorted or indexed — O(M log M + N log N).  
The optimizer chooses automatically; EXPLAIN reveals the choice.

**Q10. How does FULL OUTER JOIN work in MySQL?**  
MySQL doesn't support it natively. Emulate with LEFT JOIN + UNION + RIGHT JOIN:
```sql
SELECT * FROM a LEFT JOIN b ON a.id = b.id
UNION
SELECT * FROM a RIGHT JOIN b ON a.id = b.id;
```

**Q11. Write a query to find the second highest salary per department.**  
```sql
WITH ranked AS (
    SELECT name, dept_id, salary,
           DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT name, dept_id, salary FROM ranked WHERE rnk = 2;
```

**Q12. What is an anti-join and how do you write one?**  
An anti-join returns rows from one table that have NO match in another. Use `LEFT JOIN … WHERE right.key IS NULL` or `NOT EXISTS`. There is no explicit `ANTI JOIN` keyword in standard SQL.

---

## 15. Key Takeaways

| Join Type | What it returns | NULL side | Typical use case |
|-----------|----------------|-----------|-----------------|
| **INNER JOIN** | Matching rows only | Neither | Default: FK relationships |
| **LEFT JOIN** | All left + matched right | Right | Keep all left, optional right |
| **RIGHT JOIN** | All right + matched left | Left | Rarely used — rewrite as LEFT |
| **FULL OUTER JOIN** | Everything | Either/both | Reconciliation, auditing |
| **CROSS JOIN** | Cartesian product (M×N) | N/A | Combinations, grids |
| **SELF JOIN** | Same table, two aliases | Depends on type | Hierarchies, duplicates |
| **NATURAL JOIN** | Auto-equi on shared names | Either | Avoid in production |
| **LATERAL JOIN** | Per-row correlated subquery table | Depends on CROSS/LEFT | Top-N per group, unnesting |

### Five Golden Rules

1. **Be explicit** — always use `ON` or `USING`, never rely on NATURAL in production.
2. **Index the FK column** — the single biggest performance improvement for any join.
3. **Filter before joining** — push WHERE predicates as early as possible.
4. **Understand NULL behaviour** — unmatched rows in outer joins produce NULLs, not missing rows. NULL = NULL is false.
5. **EXPLAIN your joins** — trust the optimizer but verify it chose the right algorithm.

---

*Phase 7 complete. Phase 8 covers Subqueries, CTEs & Advanced Queries (`08_advanced_queries.md`).*

---

<!-- DOWNLOAD BUTTON -->
<a href="07_joins.md" download="07_joins.md">
  <button style="background:#2E86C1;color:white;padding:12px 28px;border:none;border-radius:6px;font-size:15px;font-weight:bold;cursor:pointer;margin-top:16px;">
    ⬇ Download 07_joins.md
  </button>
</a>
