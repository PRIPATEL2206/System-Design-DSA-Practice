# Phase 15 — Window Functions

> **Window functions** let you perform calculations across a set of rows related to the current row — without collapsing them into a single output row like `GROUP BY` does. They are the Swiss Army knife of analytical SQL.

---

## Table of Contents

1. [What Are Window Functions?](#1-what-are-window-functions)
2. [Anatomy of a Window Function](#2-anatomy-of-a-window-function)
3. [OVER() — The Window Clause](#3-over-the-window-clause)
4. [PARTITION BY — Grouping Without Collapsing](#4-partition-by)
5. [ORDER BY in Window Context](#5-order-by-in-window-context)
6. [Frame Specification (ROWS / RANGE / GROUPS)](#6-frame-specification)
7. [Ranking Functions](#7-ranking-functions)
8. [Offset Functions — LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE](#8-offset-functions)
9. [Aggregate Window Functions](#9-aggregate-window-functions)
10. [Running Totals & Moving Averages](#10-running-totals-and-moving-averages)
11. [NTILE — Bucket Distribution](#11-ntile)
12. [Named Windows (WINDOW Clause)](#12-named-windows)
13. [Window Functions vs GROUP BY](#13-window-functions-vs-group-by)
14. [Execution Order & Performance](#14-execution-order-and-performance)
15. [Worked Examples (15+)](#15-worked-examples)
16. [Common Interview Questions](#16-interview-questions)
17. [Key Takeaways](#17-key-takeaways)
18. [Download](#18-download)

---

## 1. What Are Window Functions?

### The Problem

Suppose you want to show each employee's salary **and** the average salary of their department — in the same row. With `GROUP BY`, you collapse rows and lose individual detail. With a correlated subquery, you re-execute for every row.

```sql
-- GROUP BY loses individual rows
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- Correlated subquery is slow
SELECT name, salary,
       (SELECT AVG(salary) FROM employees e2 WHERE e2.dept = e1.dept)
  FROM employees e1;
```

### The Solution: Window Functions

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
  FROM employees;
```

Every row is preserved. Every row gets the aggregate. No self-join, no subquery.

### Real-World Analogy

Imagine a classroom where the teacher asks everyone to stand up. A `GROUP BY` would merge all students into one line per group and report just the group average. A **window function** lets every student remain standing at their desk while a banner above each group shows the group average — individual detail plus aggregate context, side by side.

<svg viewBox="0 0 750 300" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">GROUP BY vs Window Function</text>

  <!-- GROUP BY side -->
  <rect x="20" y="45" width="330" height="240" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="185" y="70" text-anchor="middle" font-size="14" font-weight="bold" fill="#e65100">GROUP BY — Collapses Rows</text>

  <!-- Input rows -->
  <rect x="40" y="85" width="130" height="25" rx="4" fill="#ffe0b2" stroke="#e65100"/>
  <text x="105" y="103" text-anchor="middle" font-size="10" fill="#333">Alice | Sales | 60k</text>
  <rect x="40" y="112" width="130" height="25" rx="4" fill="#ffe0b2" stroke="#e65100"/>
  <text x="105" y="130" text-anchor="middle" font-size="10" fill="#333">Bob | Sales | 70k</text>
  <rect x="40" y="139" width="130" height="25" rx="4" fill="#ffccbc" stroke="#e65100"/>
  <text x="105" y="157" text-anchor="middle" font-size="10" fill="#333">Carol | Eng | 90k</text>
  <rect x="40" y="166" width="130" height="25" rx="4" fill="#ffccbc" stroke="#e65100"/>
  <text x="105" y="184" text-anchor="middle" font-size="10" fill="#333">Dave | Eng | 80k</text>

  <text x="195" y="135" font-size="18" fill="#e65100">→</text>

  <!-- Output rows (collapsed) -->
  <rect x="215" y="100" width="120" height="30" rx="4" fill="#ff9800" stroke="#e65100"/>
  <text x="275" y="120" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Sales | AVG 65k</text>
  <rect x="215" y="145" width="120" height="30" rx="4" fill="#ff9800" stroke="#e65100"/>
  <text x="275" y="165" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold">Eng | AVG 85k</text>

  <text x="185" y="260" text-anchor="middle" font-size="11" fill="#bf360c" font-weight="bold">4 rows → 2 rows (detail lost)</text>

  <!-- Window function side -->
  <rect x="400" y="45" width="330" height="240" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="565" y="70" text-anchor="middle" font-size="14" font-weight="bold" fill="#2e7d32">Window — Preserves Rows</text>

  <rect x="420" y="85" width="290" height="25" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="565" y="103" text-anchor="middle" font-size="10" fill="#333">Alice | Sales | 60k | dept_avg=65k</text>
  <rect x="420" y="112" width="290" height="25" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="565" y="130" text-anchor="middle" font-size="10" fill="#333">Bob | Sales | 70k | dept_avg=65k</text>
  <rect x="420" y="139" width="290" height="25" rx="4" fill="#a5d6a7" stroke="#2e7d32"/>
  <text x="565" y="157" text-anchor="middle" font-size="10" fill="#333">Carol | Eng | 90k | dept_avg=85k</text>
  <rect x="420" y="166" width="290" height="25" rx="4" fill="#a5d6a7" stroke="#2e7d32"/>
  <text x="565" y="184" text-anchor="middle" font-size="10" fill="#333">Dave | Eng | 80k | dept_avg=85k</text>

  <text x="565" y="260" text-anchor="middle" font-size="11" fill="#1b5e20" font-weight="bold">4 rows → 4 rows (detail + aggregate)</text>
</svg>

---

## 2. Anatomy of a Window Function

```
function_name ( arguments )  OVER  (
    [ PARTITION BY expr, ... ]
    [ ORDER BY expr [ ASC | DESC ] [ NULLS { FIRST | LAST } ], ... ]
    [ frame_clause ]
)
```

| Component | Purpose | Optional? |
|---|---|---|
| `function_name` | The calculation (SUM, RANK, LAG, …) | Required |
| `OVER (…)` | Declares this is a window function | Required |
| `PARTITION BY` | Divides rows into groups (like GROUP BY, but rows are kept) | Yes — omit = entire table is one partition |
| `ORDER BY` | Orders rows within each partition | Yes — required for ranking/offset functions |
| `frame_clause` | Defines which rows in the partition the function sees | Yes — defaults vary by function |

---

## 3. OVER() — The Window Clause

### Empty OVER() — The Entire Table Is the Window

```sql
SELECT name, salary,
       SUM(salary) OVER () AS total_payroll,
       ROUND(salary * 100.0 / SUM(salary) OVER (), 2) AS pct_of_payroll
  FROM employees;
```

| name | salary | total_payroll | pct_of_payroll |
|---|---|---|---|
| Alice | 60000 | 300000 | 20.00 |
| Bob | 70000 | 300000 | 23.33 |
| Carol | 90000 | 300000 | 30.00 |
| Dave | 80000 | 300000 | 26.67 |

Every row sees the **same** total because the window is the entire result set.

---

## 4. PARTITION BY

`PARTITION BY` splits the window into independent groups. The function restarts for each partition.

```sql
SELECT name, department, salary,
       SUM(salary)   OVER (PARTITION BY department) AS dept_total,
       COUNT(*)      OVER (PARTITION BY department) AS dept_headcount,
       ROUND(AVG(salary) OVER (PARTITION BY department), 0) AS dept_avg
  FROM employees
 ORDER BY department, salary DESC;
```

| name | department | salary | dept_total | dept_headcount | dept_avg |
|---|---|---|---|---|---|
| Carol | Engineering | 90000 | 170000 | 2 | 85000 |
| Dave | Engineering | 80000 | 170000 | 2 | 85000 |
| Bob | Sales | 70000 | 130000 | 2 | 65000 |
| Alice | Sales | 60000 | 130000 | 2 | 65000 |

<svg viewBox="0 0 700 220" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <text x="350" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">PARTITION BY — Independent Windows</text>

  <!-- Partition 1 -->
  <rect x="30" y="45" width="280" height="150" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2" stroke-dasharray="6,3"/>
  <text x="170" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Partition: Engineering</text>
  <rect x="50" y="80" width="240" height="25" rx="4" fill="#bbdefb"/>
  <text x="170" y="97" text-anchor="middle" font-size="11" fill="#333">Carol | 90k → SUM=170k, AVG=85k</text>
  <rect x="50" y="110" width="240" height="25" rx="4" fill="#bbdefb"/>
  <text x="170" y="127" text-anchor="middle" font-size="11" fill="#333">Dave  | 80k → SUM=170k, AVG=85k</text>
  <text x="170" y="170" text-anchor="middle" font-size="10" fill="#1565c0">Window scoped to this partition only</text>

  <!-- Partition 2 -->
  <rect x="380" y="45" width="280" height="150" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2" stroke-dasharray="6,3"/>
  <text x="520" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Partition: Sales</text>
  <rect x="400" y="80" width="240" height="25" rx="4" fill="#c8e6c9"/>
  <text x="520" y="97" text-anchor="middle" font-size="11" fill="#333">Bob   | 70k → SUM=130k, AVG=65k</text>
  <rect x="400" y="110" width="240" height="25" rx="4" fill="#c8e6c9"/>
  <text x="520" y="127" text-anchor="middle" font-size="11" fill="#333">Alice | 60k → SUM=130k, AVG=65k</text>
  <text x="520" y="170" text-anchor="middle" font-size="10" fill="#2e7d32">Window scoped to this partition only</text>
</svg>

---

## 5. ORDER BY in Window Context

Adding `ORDER BY` inside `OVER()` does two things:

1. **Defines the row order** within each partition
2. **Changes the default frame** from "entire partition" to "RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW" — i.e., a running calculation up to the current row

```sql
SELECT name, department, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY salary) AS running_total
  FROM employees;
```

| name | department | salary | running_total |
|---|---|---|---|
| Alice | Sales | 60000 | 60000 |
| Bob | Sales | 70000 | 130000 |
| Dave | Engineering | 80000 | 80000 |
| Carol | Engineering | 90000 | 170000 |

Notice how `running_total` accumulates within each partition as rows are processed in salary order.

---

## 6. Frame Specification (ROWS / RANGE / GROUPS)

The **frame** defines exactly which rows, relative to the current row, the window function can "see."

### Syntax

```
{ ROWS | RANGE | GROUPS } BETWEEN frame_start AND frame_end
```

### Frame Boundaries

| Boundary | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | First row of the partition |
| `N PRECEDING` | N rows/value-units before current |
| `CURRENT ROW` | The current row |
| `N FOLLOWING` | N rows/value-units after current |
| `UNBOUNDED FOLLOWING` | Last row of the partition |

### ROWS vs RANGE vs GROUPS

| Mode | Unit of Measurement | Peer Handling |
|---|---|---|
| **ROWS** | Physical row positions | Each row is distinct |
| **RANGE** | Logical value distance from ORDER BY column | Peers (equal values) treated as one unit |
| **GROUPS** | Peer groups (distinct ORDER BY values) | Groups of tied values |

<svg viewBox="0 0 750 380" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Window Frame — Visual Guide</text>

  <text x="375" y="50" text-anchor="middle" font-size="12" fill="#555">SUM(salary) OVER (ORDER BY hire_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)</text>

  <!-- Row boxes -->
  <rect x="80" y="75" width="590" height="30" rx="4" fill="#f5f5f5" stroke="#999"/>
  <text x="90" y="95" font-size="10" fill="#666">Row 1</text>
  <text x="200" y="95" font-size="11" fill="#333">Jan 5 | Alice | 60k</text>

  <rect x="80" y="110" width="590" height="30" rx="4" fill="#f5f5f5" stroke="#999"/>
  <text x="90" y="130" font-size="10" fill="#666">Row 2</text>
  <text x="200" y="130" font-size="11" fill="#333">Feb 12 | Bob | 70k</text>

  <rect x="80" y="145" width="590" height="30" rx="4" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="90" y="165" font-size="10" fill="#666">Row 3</text>
  <text x="200" y="165" font-size="11" fill="#333" font-weight="bold">Mar 20 | Carol | 90k  ← CURRENT ROW</text>

  <rect x="80" y="180" width="590" height="30" rx="4" fill="#f5f5f5" stroke="#999"/>
  <text x="90" y="200" font-size="10" fill="#666">Row 4</text>
  <text x="200" y="200" font-size="11" fill="#333">Apr 8 | Dave | 80k</text>

  <rect x="80" y="215" width="590" height="30" rx="4" fill="#f5f5f5" stroke="#999"/>
  <text x="90" y="235" font-size="10" fill="#666">Row 5</text>
  <text x="200" y="235" font-size="11" fill="#333">May 15 | Eve | 75k</text>

  <!-- Frame bracket -->
  <rect x="60" y="108" width="625" height="107" rx="8" fill="none" stroke="#e65100" stroke-width="2.5" stroke-dasharray="6,3"/>
  <text x="60" y="265" font-size="11" fill="#e65100" font-weight="bold">Frame: rows 2,3,4 → SUM = 70k + 90k + 80k = 240k</text>

  <!-- Labels -->
  <line x1="50" y1="125" x2="75" y2="125" stroke="#e65100" stroke-width="1.5"/>
  <text x="30" y="129" text-anchor="middle" font-size="9" fill="#e65100">1 PREC</text>

  <line x1="50" y1="160" x2="75" y2="160" stroke="#f9a825" stroke-width="1.5"/>
  <text x="30" y="164" text-anchor="middle" font-size="9" fill="#f9a825">CURR</text>

  <line x1="50" y1="195" x2="75" y2="195" stroke="#e65100" stroke-width="1.5"/>
  <text x="30" y="199" text-anchor="middle" font-size="9" fill="#e65100">1 FOLL</text>

  <!-- Common frames cheat sheet -->
  <rect x="80" y="285" width="590" height="80" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="375" y="308" text-anchor="middle" font-size="13" font-weight="bold" fill="#6a1b9a">Common Frame Shortcuts</text>
  <text x="100" y="330" font-size="10" fill="#333">ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW → running total</text>
  <text x="100" y="348" font-size="10" fill="#333">ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING → 5-row moving average</text>
  <text x="100" y="366" font-size="10" fill="#333">ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING → entire partition</text>
</svg>

### Frame Examples

```sql
-- Running total (default when ORDER BY is present)
SUM(salary) OVER (ORDER BY hire_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- 3-day moving average
AVG(revenue) OVER (ORDER BY sale_date
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)

-- Entire partition (ignore ORDER BY for aggregate)
MAX(salary) OVER (PARTITION BY department
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
```

---

## 7. Ranking Functions

### The Big Four

<svg viewBox="0 0 750 360" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Ranking Functions — Handling Ties</text>

  <text x="375" y="48" text-anchor="middle" font-size="11" fill="#555">ORDER BY score DESC — scores: 95, 90, 90, 85, 80</text>

  <!-- Header -->
  <rect x="30" y="65" width="90" height="30" rx="4" fill="#37474f"/>
  <text x="75" y="85" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">Score</text>
  <rect x="125" y="65" width="135" height="30" rx="4" fill="#1565c0"/>
  <text x="192" y="85" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">ROW_NUMBER()</text>
  <rect x="265" y="65" width="135" height="30" rx="4" fill="#00695c"/>
  <text x="332" y="85" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">RANK()</text>
  <rect x="405" y="65" width="135" height="30" rx="4" fill="#6a1b9a"/>
  <text x="472" y="85" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">DENSE_RANK()</text>
  <rect x="545" y="65" width="175" height="30" rx="4" fill="#e65100"/>
  <text x="632" y="85" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">NTILE(3)</text>

  <!-- Row 1: 95 -->
  <rect x="30" y="100" width="90" height="30" fill="#f5f5f5" stroke="#ddd"/>
  <text x="75" y="120" text-anchor="middle" font-size="12" fill="#333" font-weight="bold">95</text>
  <rect x="125" y="100" width="135" height="30" fill="#e3f2fd" stroke="#ddd"/>
  <text x="192" y="120" text-anchor="middle" font-size="12" fill="#1565c0" font-weight="bold">1</text>
  <rect x="265" y="100" width="135" height="30" fill="#e0f2f1" stroke="#ddd"/>
  <text x="332" y="120" text-anchor="middle" font-size="12" fill="#00695c" font-weight="bold">1</text>
  <rect x="405" y="100" width="135" height="30" fill="#f3e5f5" stroke="#ddd"/>
  <text x="472" y="120" text-anchor="middle" font-size="12" fill="#6a1b9a" font-weight="bold">1</text>
  <rect x="545" y="100" width="175" height="30" fill="#fff3e0" stroke="#ddd"/>
  <text x="632" y="120" text-anchor="middle" font-size="12" fill="#e65100" font-weight="bold">1</text>

  <!-- Row 2: 90 (tie) -->
  <rect x="30" y="135" width="90" height="30" fill="#fff9c4" stroke="#f9a825" stroke-width="1.5"/>
  <text x="75" y="155" text-anchor="middle" font-size="12" fill="#333" font-weight="bold">90</text>
  <rect x="125" y="135" width="135" height="30" fill="#e3f2fd" stroke="#ddd"/>
  <text x="192" y="155" text-anchor="middle" font-size="12" fill="#1565c0" font-weight="bold">2</text>
  <rect x="265" y="135" width="135" height="30" fill="#b2dfdb" stroke="#00695c" stroke-width="1.5"/>
  <text x="332" y="155" text-anchor="middle" font-size="12" fill="#00695c" font-weight="bold">2</text>
  <rect x="405" y="135" width="135" height="30" fill="#e1bee7" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="472" y="155" text-anchor="middle" font-size="12" fill="#6a1b9a" font-weight="bold">2</text>
  <rect x="545" y="135" width="175" height="30" fill="#fff3e0" stroke="#ddd"/>
  <text x="632" y="155" text-anchor="middle" font-size="12" fill="#e65100" font-weight="bold">1</text>

  <!-- Row 3: 90 (tie) -->
  <rect x="30" y="170" width="90" height="30" fill="#fff9c4" stroke="#f9a825" stroke-width="1.5"/>
  <text x="75" y="190" text-anchor="middle" font-size="12" fill="#333" font-weight="bold">90</text>
  <rect x="125" y="170" width="135" height="30" fill="#e3f2fd" stroke="#ddd"/>
  <text x="192" y="190" text-anchor="middle" font-size="12" fill="#1565c0" font-weight="bold">3</text>
  <rect x="265" y="170" width="135" height="30" fill="#b2dfdb" stroke="#00695c" stroke-width="1.5"/>
  <text x="332" y="190" text-anchor="middle" font-size="12" fill="#00695c" font-weight="bold">2</text>
  <rect x="405" y="170" width="135" height="30" fill="#e1bee7" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="472" y="190" text-anchor="middle" font-size="12" fill="#6a1b9a" font-weight="bold">2</text>
  <rect x="545" y="170" width="175" height="30" fill="#fff3e0" stroke="#ddd"/>
  <text x="632" y="190" text-anchor="middle" font-size="12" fill="#e65100" font-weight="bold">2</text>

  <!-- Row 4: 85 -->
  <rect x="30" y="205" width="90" height="30" fill="#f5f5f5" stroke="#ddd"/>
  <text x="75" y="225" text-anchor="middle" font-size="12" fill="#333" font-weight="bold">85</text>
  <rect x="125" y="205" width="135" height="30" fill="#e3f2fd" stroke="#ddd"/>
  <text x="192" y="225" text-anchor="middle" font-size="12" fill="#1565c0" font-weight="bold">4</text>
  <rect x="265" y="205" width="135" height="30" fill="#e0f2f1" stroke="#ddd"/>
  <text x="332" y="225" text-anchor="middle" font-size="12" fill="#00695c" font-weight="bold">4</text>
  <rect x="405" y="205" width="135" height="30" fill="#f3e5f5" stroke="#ddd"/>
  <text x="472" y="225" text-anchor="middle" font-size="12" fill="#6a1b9a" font-weight="bold">3</text>
  <rect x="545" y="205" width="175" height="30" fill="#fff3e0" stroke="#ddd"/>
  <text x="632" y="225" text-anchor="middle" font-size="12" fill="#e65100" font-weight="bold">2</text>

  <!-- Row 5: 80 -->
  <rect x="30" y="240" width="90" height="30" fill="#f5f5f5" stroke="#ddd"/>
  <text x="75" y="260" text-anchor="middle" font-size="12" fill="#333" font-weight="bold">80</text>
  <rect x="125" y="240" width="135" height="30" fill="#e3f2fd" stroke="#ddd"/>
  <text x="192" y="260" text-anchor="middle" font-size="12" fill="#1565c0" font-weight="bold">5</text>
  <rect x="265" y="240" width="135" height="30" fill="#e0f2f1" stroke="#ddd"/>
  <text x="332" y="260" text-anchor="middle" font-size="12" fill="#00695c" font-weight="bold">5</text>
  <rect x="405" y="240" width="135" height="30" fill="#f3e5f5" stroke="#ddd"/>
  <text x="472" y="260" text-anchor="middle" font-size="12" fill="#6a1b9a" font-weight="bold">4</text>
  <rect x="545" y="240" width="175" height="30" fill="#fff3e0" stroke="#ddd"/>
  <text x="632" y="260" text-anchor="middle" font-size="12" fill="#e65100" font-weight="bold">3</text>

  <!-- Explanations -->
  <text x="192" y="300" text-anchor="middle" font-size="10" fill="#1565c0">Always unique</text>
  <text x="192" y="315" text-anchor="middle" font-size="10" fill="#1565c0">Arbitrary tie-break</text>
  <text x="332" y="300" text-anchor="middle" font-size="10" fill="#00695c">Ties share rank</text>
  <text x="332" y="315" text-anchor="middle" font-size="10" fill="#00695c">Gaps after ties</text>
  <text x="472" y="300" text-anchor="middle" font-size="10" fill="#6a1b9a">Ties share rank</text>
  <text x="472" y="315" text-anchor="middle" font-size="10" fill="#6a1b9a">No gaps</text>
  <text x="632" y="300" text-anchor="middle" font-size="10" fill="#e65100">Splits into N</text>
  <text x="632" y="315" text-anchor="middle" font-size="10" fill="#e65100">equal-ish buckets</text>

  <!-- Highlight tie region -->
  <rect x="25" y="133" width="700" height="70" rx="4" fill="none" stroke="#f9a825" stroke-width="2" stroke-dasharray="5,3"/>
  <text x="20" y="348" font-size="10" fill="#f9a825" font-weight="bold">Yellow highlight = tied values (score 90). Observe how each function handles the tie differently.</text>
</svg>

### SQL

```sql
SELECT name, score,
       ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,
       RANK()       OVER (ORDER BY score DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rnk,
       NTILE(3)     OVER (ORDER BY score DESC) AS bucket
  FROM exam_results;
```

### When to Use Which

| Function | Use When |
|---|---|
| `ROW_NUMBER()` | You need a unique sequential number — pagination, deduplication |
| `RANK()` | Ties matter and you want gaps (e.g., Olympic medals: gold, silver, silver, 4th) |
| `DENSE_RANK()` | Ties matter but you don't want gaps (e.g., top-3 salary bands) |
| `NTILE(N)` | You need to split rows into N equal groups (quartiles, deciles) |

---

## 8. Offset Functions — LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE

### LAG and LEAD

Access a value from a **previous** (`LAG`) or **next** (`LEAD`) row within the partition.

```sql
LAG(column, offset, default)  OVER (PARTITION BY … ORDER BY …)
LEAD(column, offset, default) OVER (PARTITION BY … ORDER BY …)
```

```sql
SELECT sale_date, revenue,
       LAG(revenue, 1, 0)  OVER (ORDER BY sale_date) AS prev_day_rev,
       LEAD(revenue, 1, 0) OVER (ORDER BY sale_date) AS next_day_rev,
       revenue - LAG(revenue, 1) OVER (ORDER BY sale_date) AS day_over_day
  FROM daily_sales;
```

| sale_date | revenue | prev_day_rev | next_day_rev | day_over_day |
|---|---|---|---|---|
| 2026-04-01 | 1200 | 0 | 1500 | NULL |
| 2026-04-02 | 1500 | 1200 | 1100 | 300 |
| 2026-04-03 | 1100 | 1500 | 1800 | -400 |
| 2026-04-04 | 1800 | 1100 | 1600 | 700 |
| 2026-04-05 | 1600 | 1800 | 0 | -200 |

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
SELECT name, department, salary,
       FIRST_VALUE(name) OVER w AS highest_paid,
       LAST_VALUE(name)  OVER w AS lowest_paid,
       NTH_VALUE(name, 2) OVER w AS second_highest
  FROM employees
WINDOW w AS (
    PARTITION BY department
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

| name | department | salary | highest_paid | lowest_paid | second_highest |
|---|---|---|---|---|---|
| Carol | Engineering | 90000 | Carol | Dave | Dave |
| Dave | Engineering | 80000 | Carol | Dave | Dave |
| Bob | Sales | 70000 | Bob | Alice | Alice |
| Alice | Sales | 60000 | Bob | Alice | Alice |

> **LAST_VALUE gotcha:** Without an explicit frame extending to `UNBOUNDED FOLLOWING`, `LAST_VALUE` uses the default frame (up to CURRENT ROW), making it just return the current row's value. Always specify the full frame.

<svg viewBox="0 0 700 200" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <text x="350" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Offset Functions — Visual Reference</text>

  <!-- Row representations -->
  <rect x="50" y="55" width="80" height="35" rx="4" fill="#e0e0e0" stroke="#999"/>
  <text x="90" y="77" text-anchor="middle" font-size="10" fill="#555">Row N-2</text>

  <rect x="145" y="55" width="80" height="35" rx="4" fill="#bbdefb" stroke="#1565c0"/>
  <text x="185" y="72" text-anchor="middle" font-size="10" fill="#1565c0" font-weight="bold">LAG(…,1)</text>
  <text x="185" y="84" text-anchor="middle" font-size="9" fill="#555">Row N-1</text>

  <rect x="240" y="48" width="90" height="48" rx="4" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="285" y="68" text-anchor="middle" font-size="11" fill="#f9a825" font-weight="bold">CURRENT</text>
  <text x="285" y="84" text-anchor="middle" font-size="10" fill="#333">Row N</text>

  <rect x="345" y="55" width="80" height="35" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="385" y="72" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">LEAD(…,1)</text>
  <text x="385" y="84" text-anchor="middle" font-size="9" fill="#555">Row N+1</text>

  <rect x="440" y="55" width="80" height="35" rx="4" fill="#e0e0e0" stroke="#999"/>
  <text x="480" y="77" text-anchor="middle" font-size="10" fill="#555">Row N+2</text>

  <!-- First/Last arrows -->
  <line x1="90" y1="110" x2="90" y2="135" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="90" y="150" text-anchor="middle" font-size="10" fill="#6a1b9a" font-weight="bold">FIRST_VALUE</text>
  <text x="90" y="165" text-anchor="middle" font-size="9" fill="#6a1b9a">(if partition start)</text>

  <line x1="480" y1="110" x2="480" y2="135" stroke="#c62828" stroke-width="1.5"/>
  <text x="480" y="150" text-anchor="middle" font-size="10" fill="#c62828" font-weight="bold">LAST_VALUE</text>
  <text x="480" y="165" text-anchor="middle" font-size="9" fill="#c62828">(needs full frame!)</text>

  <line x1="185" y1="110" x2="185" y2="135" stroke="#e65100" stroke-width="1.5"/>
  <text x="185" y="150" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">NTH_VALUE(…,2)</text>
  <text x="185" y="165" text-anchor="middle" font-size="9" fill="#e65100">(2nd in partition)</text>
</svg>

---

## 9. Aggregate Window Functions

All standard aggregate functions (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) can be used as window functions by adding `OVER()`.

```sql
SELECT order_date, product, amount,
       SUM(amount)   OVER () AS grand_total,
       SUM(amount)   OVER (PARTITION BY product) AS product_total,
       COUNT(*)      OVER (PARTITION BY product) AS product_orders,
       MIN(amount)   OVER (PARTITION BY product) AS product_min,
       MAX(amount)   OVER (PARTITION BY product) AS product_max,
       ROUND(AVG(amount) OVER (PARTITION BY product), 2) AS product_avg
  FROM orders;
```

### Percentage of Total Pattern

```sql
SELECT product, amount,
       ROUND(amount * 100.0 / SUM(amount) OVER (), 2) AS pct_of_grand_total,
       ROUND(amount * 100.0 / SUM(amount) OVER (PARTITION BY category), 2) AS pct_of_category
  FROM orders;
```

### Difference from Partition Average

```sql
SELECT name, department, salary,
       ROUND(AVG(salary) OVER (PARTITION BY department), 0) AS dept_avg,
       salary - ROUND(AVG(salary) OVER (PARTITION BY department), 0) AS diff_from_avg
  FROM employees;
```

| name | department | salary | dept_avg | diff_from_avg |
|---|---|---|---|---|
| Carol | Engineering | 90000 | 85000 | 5000 |
| Dave | Engineering | 80000 | 85000 | -5000 |
| Bob | Sales | 70000 | 65000 | 5000 |
| Alice | Sales | 60000 | 65000 | -5000 |

---

## 10. Running Totals and Moving Averages

### Running Total

```sql
SELECT order_date, amount,
       SUM(amount) OVER (
           ORDER BY order_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
  FROM daily_revenue;
```

| order_date | amount | running_total |
|---|---|---|
| 2026-04-01 | 1200 | 1200 |
| 2026-04-02 | 1500 | 2700 |
| 2026-04-03 | 1100 | 3800 |
| 2026-04-04 | 1800 | 5600 |
| 2026-04-05 | 1600 | 7200 |

### 7-Day Moving Average

```sql
SELECT sale_date, revenue,
       ROUND(AVG(revenue) OVER (
           ORDER BY sale_date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ), 2) AS moving_avg_7d
  FROM daily_sales;
```

| sale_date | revenue | moving_avg_7d |
|---|---|---|
| 2026-04-01 | 1200 | 1200.00 |
| 2026-04-02 | 1500 | 1350.00 |
| 2026-04-03 | 1100 | 1266.67 |
| 2026-04-04 | 1800 | 1400.00 |
| 2026-04-05 | 1600 | 1440.00 |
| 2026-04-06 | 1300 | 1416.67 |
| 2026-04-07 | 1700 | 1457.14 |
| 2026-04-08 | 2000 | 1571.43 |

### Cumulative Distribution

```sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY salary) AS cumulative,
       ROUND(SUM(salary) OVER (ORDER BY salary) * 100.0 /
             SUM(salary) OVER (), 2) AS cumulative_pct
  FROM employees;
```

<svg viewBox="0 0 700 260" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <text x="350" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Running Total — Step by Step</text>

  <!-- Rows -->
  <rect x="60" y="50" width="120" height="30" rx="4" fill="#e3f2fd" stroke="#1565c0"/>
  <text x="120" y="70" text-anchor="middle" font-size="11" fill="#333">Apr 1: $1200</text>

  <rect x="60" y="90" width="120" height="30" rx="4" fill="#e3f2fd" stroke="#1565c0"/>
  <text x="120" y="110" text-anchor="middle" font-size="11" fill="#333">Apr 2: $1500</text>

  <rect x="60" y="130" width="120" height="30" rx="4" fill="#e3f2fd" stroke="#1565c0"/>
  <text x="120" y="150" text-anchor="middle" font-size="11" fill="#333">Apr 3: $1100</text>

  <rect x="60" y="170" width="120" height="30" rx="4" fill="#e3f2fd" stroke="#1565c0"/>
  <text x="120" y="190" text-anchor="middle" font-size="11" fill="#333">Apr 4: $1800</text>

  <rect x="60" y="210" width="120" height="30" rx="4" fill="#e3f2fd" stroke="#1565c0"/>
  <text x="120" y="230" text-anchor="middle" font-size="11" fill="#333">Apr 5: $1600</text>

  <!-- Accumulation arrows and totals -->
  <text x="220" y="70" font-size="14" fill="#1565c0">→</text>
  <rect x="250" y="50" width="100" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="300" y="70" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">$1,200</text>

  <text x="220" y="110" font-size="14" fill="#1565c0">→</text>
  <rect x="250" y="90" width="140" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="320" y="110" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">$2,700</text>

  <text x="220" y="150" font-size="14" fill="#1565c0">→</text>
  <rect x="250" y="130" width="175" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="337" y="150" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">$3,800</text>

  <text x="220" y="190" font-size="14" fill="#1565c0">→</text>
  <rect x="250" y="170" width="230" height="30" rx="4" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="365" y="190" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">$5,600</text>

  <text x="220" y="230" font-size="14" fill="#1565c0">→</text>
  <rect x="250" y="210" width="285" height="30" rx="4" fill="#a5d6a7" stroke="#2e7d32" stroke-width="2"/>
  <text x="392" y="230" text-anchor="middle" font-size="11" fill="#1b5e20" font-weight="bold">$7,200</text>

  <!-- Formula -->
  <text x="570" y="130" font-size="11" fill="#555">Each bar = SUM of</text>
  <text x="570" y="148" font-size="11" fill="#555">all preceding rows</text>
  <text x="570" y="166" font-size="11" fill="#555">+ current row</text>
</svg>

---

## 11. NTILE — Bucket Distribution

`NTILE(N)` divides the ordered partition into **N approximately equal groups** and assigns each row a bucket number (1 to N).

```sql
SELECT name, salary,
       NTILE(4) OVER (ORDER BY salary DESC) AS quartile,
       NTILE(10) OVER (ORDER BY salary DESC) AS decile,
       NTILE(100) OVER (ORDER BY salary DESC) AS percentile_bucket
  FROM employees;
```

### Practical Use — Salary Bands

```sql
SELECT quartile, 
       MIN(salary) AS band_min,
       MAX(salary) AS band_max,
       COUNT(*) AS headcount
  FROM (
      SELECT salary,
             NTILE(4) OVER (ORDER BY salary) AS quartile
        FROM employees
  ) t
 GROUP BY quartile
 ORDER BY quartile;
```

| quartile | band_min | band_max | headcount |
|---|---|---|---|
| 1 | 35000 | 52000 | 25 |
| 2 | 53000 | 68000 | 25 |
| 3 | 69000 | 85000 | 25 |
| 4 | 86000 | 150000 | 25 |

### PERCENT_RANK and CUME_DIST

```sql
SELECT name, salary,
       ROUND(PERCENT_RANK() OVER (ORDER BY salary), 3) AS pct_rank,
       ROUND(CUME_DIST()    OVER (ORDER BY salary), 3) AS cume_dist
  FROM employees;
```

| name | salary | pct_rank | cume_dist |
|---|---|---|---|
| Alice | 60000 | 0.000 | 0.250 |
| Bob | 70000 | 0.333 | 0.500 |
| Dave | 80000 | 0.667 | 0.750 |
| Carol | 90000 | 1.000 | 1.000 |

- **PERCENT_RANK**: `(rank - 1) / (total_rows - 1)` — ranges from 0 to 1
- **CUME_DIST**: fraction of rows ≤ current row's value — ranges from > 0 to 1

---

## 12. Named Windows (WINDOW Clause)

When multiple columns use the same window specification, define it once with the `WINDOW` clause:

```sql
SELECT name, department, salary,
       ROW_NUMBER() OVER w AS row_num,
       RANK()       OVER w AS rnk,
       DENSE_RANK() OVER w AS dense_rnk,
       SUM(salary)  OVER w AS running_total,
       LAG(salary)  OVER w AS prev_salary
  FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

You can also **extend** a named window:

```sql
SELECT name, department, salary,
       SUM(salary) OVER (w ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running,
       AVG(salary) OVER (w ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS moving_avg
  FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

---

## 13. Window Functions vs GROUP BY

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">GROUP BY vs Window Functions</text>

  <!-- Headers -->
  <rect x="20" y="50" width="180" height="30" rx="4" fill="#37474f"/>
  <text x="110" y="70" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">Aspect</text>
  <rect x="205" y="50" width="250" height="30" rx="4" fill="#e65100"/>
  <text x="330" y="70" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">GROUP BY</text>
  <rect x="460" y="50" width="270" height="30" rx="4" fill="#1565c0"/>
  <text x="595" y="70" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">Window Functions</text>

  <!-- Row 1 -->
  <rect x="20" y="85" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="107" text-anchor="middle" font-size="11" fill="#333">Row output</text>
  <rect x="205" y="85" width="250" height="35" fill="#fff3e0" stroke="#ddd"/>
  <text x="330" y="107" text-anchor="middle" font-size="11" fill="#333">One row per group</text>
  <rect x="460" y="85" width="270" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="595" y="107" text-anchor="middle" font-size="11" fill="#333">All original rows preserved</text>

  <!-- Row 2 -->
  <rect x="20" y="120" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="142" text-anchor="middle" font-size="11" fill="#333">Detail access</text>
  <rect x="205" y="120" width="250" height="35" fill="#fff3e0" stroke="#ddd"/>
  <text x="330" y="142" text-anchor="middle" font-size="11" fill="#c62828">Lost — only grouped cols</text>
  <rect x="460" y="120" width="270" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="595" y="142" text-anchor="middle" font-size="11" fill="#2e7d32">Full row detail retained</text>

  <!-- Row 3 -->
  <rect x="20" y="155" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="177" text-anchor="middle" font-size="11" fill="#333">Ranking</text>
  <rect x="205" y="155" width="250" height="35" fill="#fff3e0" stroke="#ddd"/>
  <text x="330" y="177" text-anchor="middle" font-size="11" fill="#c62828">Not possible</text>
  <rect x="460" y="155" width="270" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="595" y="177" text-anchor="middle" font-size="11" fill="#2e7d32">ROW_NUMBER, RANK, etc.</text>

  <!-- Row 4 -->
  <rect x="20" y="190" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="212" text-anchor="middle" font-size="11" fill="#333">Running calculations</text>
  <rect x="205" y="190" width="250" height="35" fill="#fff3e0" stroke="#ddd"/>
  <text x="330" y="212" text-anchor="middle" font-size="11" fill="#c62828">Not possible</text>
  <rect x="460" y="190" width="270" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="595" y="212" text-anchor="middle" font-size="11" fill="#2e7d32">Running totals, moving avgs</text>

  <!-- Row 5 -->
  <rect x="20" y="225" width="180" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="110" y="247" text-anchor="middle" font-size="11" fill="#333">Row comparison</text>
  <rect x="205" y="225" width="250" height="35" fill="#fff3e0" stroke="#ddd"/>
  <text x="330" y="247" text-anchor="middle" font-size="11" fill="#c62828">Needs self-join</text>
  <rect x="460" y="225" width="270" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="595" y="247" text-anchor="middle" font-size="11" fill="#2e7d32">LAG / LEAD — direct access</text>

  <!-- Row 6 -->
  <rect x="20" y="260" width="180" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="110" y="282" text-anchor="middle" font-size="11" fill="#333">HAVING filter</text>
  <rect x="205" y="260" width="250" height="35" fill="#fff3e0" stroke="#ddd"/>
  <text x="330" y="282" text-anchor="middle" font-size="11" fill="#2e7d32">Yes — HAVING clause</text>
  <rect x="460" y="260" width="270" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="595" y="282" text-anchor="middle" font-size="11" fill="#333">Use subquery / CTE to filter</text>

  <!-- Decision -->
  <rect x="100" y="305" width="550" height="28" rx="6" fill="#fff8e1" stroke="#f9a825" stroke-width="1.5"/>
  <text x="375" y="324" text-anchor="middle" font-size="11" fill="#e65100" font-weight="bold">Need aggregates only? → GROUP BY. Need aggregates + row detail? → Window function.</text>
</svg>

---

## 14. Execution Order & Performance

### SQL Logical Processing Order

```
1. FROM / JOIN
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT        ← window functions evaluate HERE
6. DISTINCT
7. ORDER BY
8. LIMIT / OFFSET
```

**Implication:** Window functions **cannot** be used in `WHERE` or `HAVING` clauses. To filter on a window function result, wrap it in a subquery or CTE:

```sql
-- ❌ This won't work
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
 WHERE rnk <= 3;

-- ✅ Use a CTE
WITH ranked AS (
    SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS rnk
      FROM employees
)
SELECT * FROM ranked WHERE rnk <= 3;
```

### Performance Considerations

| Concern | Detail |
|---|---|
| **Sorting** | Each distinct `OVER(…ORDER BY…)` requires a sort. Matching ORDER BY across windows avoids re-sorts |
| **Memory** | Frames like `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` must buffer the partition |
| **Indexes** | An index on `(partition_col, order_col)` can help avoid sorts |
| **Multiple windows** | Named windows (`WINDOW w AS …`) help the optimizer share sorts |
| **PARTITION BY** | Large partitions = more memory. Small, selective partitions are faster |

### EXPLAIN ANALYZE Example

```sql
EXPLAIN ANALYZE
SELECT department, name, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC)
  FROM employees;
```

```
WindowAgg  (cost=12.45..14.95 rows=100 width=52) (actual time=0.085..0.112 rows=100 loops=1)
  ->  Sort  (cost=12.45..12.70 rows=100 width=44) (actual time=0.065..0.072 rows=100 loops=1)
        Sort Key: department, salary DESC
        Sort Method: quicksort  Memory: 32kB
        ->  Seq Scan on employees  (cost=0.00..9.00 rows=100 width=44) (...)
```

The `Sort` node sorts by `(department, salary DESC)` — matching the `PARTITION BY` + `ORDER BY`. The `WindowAgg` node applies the ranking function over the sorted stream.

---

## 15. Worked Examples

### Example 1 — Top-N per Group (Deduplication Pattern)

**Scenario:** Find the most recent order per customer.

```sql
WITH ranked AS (
    SELECT customer_id, order_id, order_date, total,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC
           ) AS rn
      FROM orders
)
SELECT customer_id, order_id, order_date, total
  FROM ranked
 WHERE rn = 1;
```

| customer_id | order_id | order_date | total |
|---|---|---|---|
| 101 | 5892 | 2026-04-28 | 259.99 |
| 102 | 5901 | 2026-04-29 | 89.50 |
| 103 | 5887 | 2026-04-27 | 450.00 |

### Example 2 — Running Revenue by Product Category

```sql
SELECT category, order_date, revenue,
       SUM(revenue) OVER (
           PARTITION BY category
           ORDER BY order_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS cumulative_revenue
  FROM daily_category_sales
 ORDER BY category, order_date;
```

| category | order_date | revenue | cumulative_revenue |
|---|---|---|---|
| Electronics | 2026-04-01 | 5200 | 5200 |
| Electronics | 2026-04-02 | 4800 | 10000 |
| Electronics | 2026-04-03 | 6100 | 16100 |
| Clothing | 2026-04-01 | 2100 | 2100 |
| Clothing | 2026-04-02 | 1800 | 3900 |
| Clothing | 2026-04-03 | 2500 | 6400 |

### Example 3 — Month-over-Month Growth Rate

```sql
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       ROUND(
           (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 /
           NULLIF(LAG(revenue) OVER (ORDER BY month), 0)
       , 1) AS growth_pct
  FROM monthly_revenue;
```

| month | revenue | prev_month | growth_pct |
|---|---|---|---|
| 2026-01 | 120000 | NULL | NULL |
| 2026-02 | 135000 | 120000 | 12.5 |
| 2026-03 | 128000 | 135000 | -5.2 |
| 2026-04 | 150000 | 128000 | 17.2 |

### Example 4 — Percentile-Based Salary Analysis

```sql
SELECT name, department, salary,
       NTILE(4)       OVER (ORDER BY salary) AS quartile,
       PERCENT_RANK() OVER (ORDER BY salary) AS percentile,
       CASE
           WHEN NTILE(4) OVER (ORDER BY salary) = 1 THEN 'Below Average'
           WHEN NTILE(4) OVER (ORDER BY salary) IN (2, 3) THEN 'Average'
           WHEN NTILE(4) OVER (ORDER BY salary) = 4 THEN 'Top Performer'
       END AS salary_band
  FROM employees;
```

### Example 5 — Gap Detection in Sequences

```sql
SELECT id,
       LEAD(id) OVER (ORDER BY id) AS next_id,
       LEAD(id) OVER (ORDER BY id) - id AS gap
  FROM orders
 WHERE LEAD(id) OVER (ORDER BY id) - id > 1;

-- Correct version (can't use window in WHERE):
WITH gaps AS (
    SELECT id,
           LEAD(id) OVER (ORDER BY id) AS next_id
      FROM orders
)
SELECT id, next_id, next_id - id AS gap_size
  FROM gaps
 WHERE next_id - id > 1;
```

| id | next_id | gap_size |
|---|---|---|
| 1042 | 1045 | 3 |
| 1050 | 1055 | 5 |

### Example 6 — Median Calculation

```sql
SELECT department,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
  FROM employees
 GROUP BY department;

-- Alternative using window functions:
WITH ranked AS (
    SELECT department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
           COUNT(*)     OVER (PARTITION BY department) AS cnt
      FROM employees
)
SELECT department,
       AVG(salary) AS median_salary
  FROM ranked
 WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
 GROUP BY department;
```

### Example 7 — Detecting Consecutive Events

**Scenario:** Find users with 3+ consecutive days of login.

```sql
WITH daily_logins AS (
    SELECT DISTINCT user_id, login_date::DATE AS login_day
      FROM user_sessions
),
grouped AS (
    SELECT user_id, login_day,
           login_day - ROW_NUMBER() OVER (
               PARTITION BY user_id ORDER BY login_day
           )::INT AS grp
      FROM daily_logins
),
streaks AS (
    SELECT user_id, grp,
           MIN(login_day) AS streak_start,
           MAX(login_day) AS streak_end,
           COUNT(*)       AS streak_length
      FROM grouped
     GROUP BY user_id, grp
)
SELECT user_id, streak_start, streak_end, streak_length
  FROM streaks
 WHERE streak_length >= 3
 ORDER BY streak_length DESC;
```

| user_id | streak_start | streak_end | streak_length |
|---|---|---|---|
| 42 | 2026-04-10 | 2026-04-17 | 8 |
| 17 | 2026-04-22 | 2026-04-25 | 4 |
| 42 | 2026-04-01 | 2026-04-03 | 3 |

### Example 8 — First and Last Purchase per Customer

```sql
SELECT DISTINCT customer_id,
       FIRST_VALUE(product_name) OVER w AS first_purchase,
       FIRST_VALUE(order_date)   OVER w AS first_purchase_date,
       LAST_VALUE(product_name)  OVER w AS latest_purchase,
       LAST_VALUE(order_date)    OVER w AS latest_purchase_date,
       COUNT(*) OVER (PARTITION BY customer_id) AS total_orders
  FROM orders o
  JOIN products p ON p.product_id = o.product_id
WINDOW w AS (
    PARTITION BY customer_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

### Example 9 — Inventory Days-of-Supply

```sql
SELECT product_id, stock_date, quantity_on_hand, daily_usage,
       SUM(daily_usage) OVER (
           PARTITION BY product_id
           ORDER BY stock_date
           ROWS BETWEEN CURRENT ROW AND 30 FOLLOWING
       ) AS next_30d_usage,
       ROUND(quantity_on_hand::NUMERIC /
             NULLIF(AVG(daily_usage) OVER (
                 PARTITION BY product_id
                 ORDER BY stock_date
                 ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
             ), 0), 1) AS days_of_supply
  FROM inventory_daily;
```

### Example 10 — Sessionization (Web Analytics)

**Scenario:** Group page views into sessions — a new session starts after 30 minutes of inactivity.

```sql
WITH events AS (
    SELECT user_id, page, event_time,
           LAG(event_time) OVER (
               PARTITION BY user_id ORDER BY event_time
           ) AS prev_event_time
      FROM page_views
),
flagged AS (
    SELECT *,
           CASE WHEN event_time - prev_event_time > INTERVAL '30 minutes'
                  OR prev_event_time IS NULL
                THEN 1 ELSE 0
           END AS new_session_flag
      FROM events
),
sessioned AS (
    SELECT *,
           SUM(new_session_flag) OVER (
               PARTITION BY user_id ORDER BY event_time
           ) AS session_id
      FROM flagged
)
SELECT user_id, session_id,
       MIN(event_time) AS session_start,
       MAX(event_time) AS session_end,
       COUNT(*)        AS page_views,
       MAX(event_time) - MIN(event_time) AS duration
  FROM sessioned
 GROUP BY user_id, session_id;
```

### Example 11 — Year-over-Year Comparison

```sql
SELECT month_name,
       yr_2025, yr_2026,
       yr_2026 - yr_2025 AS abs_change,
       ROUND((yr_2026 - yr_2025) * 100.0 / NULLIF(yr_2025, 0), 1) AS yoy_pct
  FROM (
      SELECT TO_CHAR(sale_date, 'Mon') AS month_name,
             EXTRACT(MONTH FROM sale_date) AS month_num,
             SUM(CASE WHEN EXTRACT(YEAR FROM sale_date) = 2025 THEN revenue END) AS yr_2025,
             SUM(CASE WHEN EXTRACT(YEAR FROM sale_date) = 2026 THEN revenue END) AS yr_2026
        FROM sales
       GROUP BY TO_CHAR(sale_date, 'Mon'), EXTRACT(MONTH FROM sale_date)
  ) t
 ORDER BY month_num;
```

### Example 12 — Top-3 Products per Category by Revenue

```sql
WITH product_revenue AS (
    SELECT p.category, p.name, SUM(o.total) AS total_revenue,
           DENSE_RANK() OVER (
               PARTITION BY p.category
               ORDER BY SUM(o.total) DESC
           ) AS revenue_rank
      FROM orders o
      JOIN products p ON p.product_id = o.product_id
     GROUP BY p.category, p.name
)
SELECT category, name, total_revenue, revenue_rank
  FROM product_revenue
 WHERE revenue_rank <= 3
 ORDER BY category, revenue_rank;
```

| category | name | total_revenue | revenue_rank |
|---|---|---|---|
| Electronics | Laptop Pro | 284000 | 1 |
| Electronics | Wireless Headphones | 38900 | 2 |
| Electronics | USB-C Hub | 13350 | 3 |
| Clothing | Winter Jacket | 22400 | 1 |
| Clothing | Running Shoes | 18500 | 2 |
| Clothing | Cotton T-Shirt | 9200 | 3 |

### Example 13 — Distribution Analysis with CUME_DIST

```sql
SELECT name, department, salary,
       ROUND(CUME_DIST() OVER (
           PARTITION BY department ORDER BY salary
       ), 3) AS cume_dist,
       CASE
           WHEN CUME_DIST() OVER (PARTITION BY department ORDER BY salary) <= 0.25
               THEN 'Bottom 25%'
           WHEN CUME_DIST() OVER (PARTITION BY department ORDER BY salary) <= 0.75
               THEN 'Middle 50%'
           ELSE 'Top 25%'
       END AS salary_tier
  FROM employees
 ORDER BY department, salary;
```

### Example 14 — Moving Standard Deviation (Volatility)

```sql
SELECT trade_date, close_price,
       ROUND(STDDEV(close_price) OVER (
           ORDER BY trade_date
           ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
       ), 4) AS volatility_20d,
       ROUND(AVG(close_price) OVER (
           ORDER BY trade_date
           ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
       ), 2) AS sma_20d
  FROM stock_prices
 WHERE symbol = 'AAPL';
```

### Example 15 — Island Detection (Contiguous Status Periods)

**Scenario:** Find contiguous periods where a server was in "degraded" status.

```sql
WITH status_changes AS (
    SELECT server_id, status, check_time,
           LAG(status) OVER (
               PARTITION BY server_id ORDER BY check_time
           ) AS prev_status
      FROM server_health_checks
),
boundaries AS (
    SELECT *,
           SUM(CASE WHEN status <> prev_status OR prev_status IS NULL THEN 1 ELSE 0 END)
               OVER (PARTITION BY server_id ORDER BY check_time) AS island_id
      FROM status_changes
)
SELECT server_id, status,
       MIN(check_time) AS period_start,
       MAX(check_time) AS period_end,
       COUNT(*)        AS check_count
  FROM boundaries
 WHERE status = 'degraded'
 GROUP BY server_id, status, island_id
 ORDER BY period_start;
```

| server_id | status | period_start | period_end | check_count |
|---|---|---|---|---|
| web-01 | degraded | 2026-04-25 02:15 | 2026-04-25 03:45 | 7 |
| web-03 | degraded | 2026-04-27 18:30 | 2026-04-27 19:00 | 3 |

### Example 16 — Comparing Each Row to Group Extremes

```sql
SELECT name, department, salary,
       salary - MIN(salary) OVER (PARTITION BY department) AS above_dept_min,
       MAX(salary) OVER (PARTITION BY department) - salary AS below_dept_max,
       ROUND(salary * 100.0 / MAX(salary) OVER (PARTITION BY department), 1)
           AS pct_of_dept_max
  FROM employees
 ORDER BY department, salary DESC;
```

| name | department | salary | above_dept_min | below_dept_max | pct_of_dept_max |
|---|---|---|---|---|---|
| Carol | Engineering | 90000 | 10000 | 0 | 100.0 |
| Dave | Engineering | 80000 | 0 | 10000 | 88.9 |
| Bob | Sales | 70000 | 10000 | 0 | 100.0 |
| Alice | Sales | 60000 | 0 | 10000 | 85.7 |

---

## 16. Common Interview Questions

### Q1: What is a window function and how does it differ from GROUP BY?

**Answer:** A window function performs a calculation across a set of rows related to the current row, without collapsing them. `GROUP BY` aggregates rows into groups and returns one row per group — losing individual row detail. Window functions preserve all rows while adding the computed aggregate, rank, or offset value as an additional column.

### Q2: What is the difference between ROW_NUMBER(), RANK(), and DENSE_RANK()?

**Answer:**
- **ROW_NUMBER()**: Assigns a unique sequential integer. Ties get arbitrary (non-deterministic) numbering.
- **RANK()**: Tied rows get the same rank, but the next rank skips. With scores 90, 90, 80 → ranks 1, 1, 3.
- **DENSE_RANK()**: Tied rows get the same rank, and the next rank doesn't skip. Same scores → 1, 1, 2.

Use `ROW_NUMBER` for deduplication/pagination, `RANK` when gaps after ties are acceptable (like Olympic rankings), and `DENSE_RANK` for continuous numbering without gaps (like salary bands).

### Q3: What is a window frame and when does it matter?

**Answer:** A window frame defines which rows within a partition the function can "see," relative to the current row. It matters for aggregate window functions (`SUM`, `AVG`, etc.) but not for ranking functions (`RANK`, `ROW_NUMBER`). The frame is specified with `ROWS`, `RANGE`, or `GROUPS` and boundaries like `UNBOUNDED PRECEDING`, `N PRECEDING`, `CURRENT ROW`, `N FOLLOWING`, `UNBOUNDED FOLLOWING`. The default frame when `ORDER BY` is present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which gives a running total. Without `ORDER BY`, the default is the entire partition.

### Q4: Explain LAG() and LEAD() with an example.

**Answer:** `LAG(column, offset, default)` accesses a value from a **previous** row (by offset, default 1). `LEAD()` accesses a **subsequent** row. They require `ORDER BY` in the `OVER()` clause.

Example: calculate day-over-day revenue change:
```sql
SELECT sale_date, revenue,
       revenue - LAG(revenue) OVER (ORDER BY sale_date) AS daily_change
  FROM daily_sales;
```
LAG looks back one row (previous day); the difference gives the change. First row returns NULL since there's no predecessor.

### Q5: Can you use a window function in a WHERE clause? Why or why not?

**Answer:** No. Window functions are evaluated at the `SELECT` stage of query processing — after `WHERE`, `GROUP BY`, and `HAVING`. To filter on a window function result, wrap the query in a CTE or subquery:

```sql
WITH ranked AS (
    SELECT *, RANK() OVER (ORDER BY salary DESC) AS rnk FROM employees
)
SELECT * FROM ranked WHERE rnk <= 5;
```

### Q6: What is the LAST_VALUE() gotcha?

**Answer:** When `ORDER BY` is specified, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This means `LAST_VALUE()` returns the **current row's** value — not the last row of the partition. To get the actual last value, you must explicitly extend the frame:

```sql
LAST_VALUE(col) OVER (
    ORDER BY col2
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

### Q7: How do you find the top-N rows per group?

**Answer:** Use `ROW_NUMBER()` with `PARTITION BY` in a CTE, then filter:

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
      FROM employees
)
SELECT * FROM ranked WHERE rn <= 3;
```

Use `DENSE_RANK` instead if ties should all be included (possibly returning more than N rows).

### Q8: What is the NTILE() function used for?

**Answer:** `NTILE(N)` divides an ordered partition into N approximately equal buckets. Row count per bucket differs by at most 1. Common uses:
- **Quartiles** (`NTILE(4)`): divide into four salary bands
- **Deciles** (`NTILE(10)`): divide into ten performance tiers
- **Percentiles** (`NTILE(100)`): approximate percentile buckets

### Q9: How can you calculate a running total?

**Answer:**
```sql
SUM(amount) OVER (ORDER BY transaction_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```
The `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame tells `SUM` to include all rows from the start of the partition through the current row. This is actually the default frame when `ORDER BY` is present, but it's best practice to state it explicitly for clarity.

### Q10: What is the difference between ROWS and RANGE in a frame clause?

**Answer:**
- **ROWS**: Counts physical row positions. `3 PRECEDING` means exactly 3 rows back.
- **RANGE**: Uses logical value distance from the `ORDER BY` column. `3 PRECEDING` means all rows whose `ORDER BY` value is within 3 units of the current row's value. Peers (rows with equal `ORDER BY` values) are all included or excluded together.

For example, with dates: `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` includes all rows within the last 7 calendar days — even if there are gaps or duplicates.

### Q11: How do window functions perform compared to correlated subqueries?

**Answer:** Window functions are typically **much faster** because:
1. The query planner sorts the data once and streams through it
2. Correlated subqueries re-execute for every row in the outer query
3. Multiple window functions sharing the same `OVER()` specification share a single sort

For a 100k-row table with a department aggregate, a correlated subquery might execute 100k inner queries; a window function does one sort + one pass.

### Q12: How would you detect gaps in a date series using window functions?

**Answer:**
```sql
WITH dates AS (
    SELECT event_date,
           LEAD(event_date) OVER (ORDER BY event_date) AS next_date
      FROM events
)
SELECT event_date, next_date, next_date - event_date AS gap_days
  FROM dates
 WHERE next_date - event_date > 1;
```

`LEAD()` looks at the next row's date. If the difference is more than 1, there's a gap. This pattern also works for sequence gaps (IDs), login streaks, or any ordered series.

---

## 17. Key Takeaways

<svg viewBox="0 0 750 520" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">Phase 15 — Key Takeaways</text>

  <!-- Card 1 -->
  <rect x="20" y="50" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="40" y="75" font-size="13" font-weight="bold" fill="#1565c0">Aggregates Without Collapse</text>
  <text x="40" y="95" font-size="11" fill="#333">Window functions add aggregate context</text>
  <text x="40" y="112" font-size="11" fill="#333">while keeping every row intact.</text>

  <!-- Card 2 -->
  <rect x="390" y="50" width="340" height="80" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="410" y="75" font-size="13" font-weight="bold" fill="#00695c">PARTITION BY = GROUP BY for Windows</text>
  <text x="410" y="95" font-size="11" fill="#333">Splits rows into independent windows.</text>
  <text x="410" y="112" font-size="11" fill="#333">Omit it to use the entire result set.</text>

  <!-- Card 3 -->
  <rect x="20" y="150" width="340" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="40" y="175" font-size="13" font-weight="bold" fill="#e65100">ORDER BY Changes the Frame</text>
  <text x="40" y="195" font-size="11" fill="#333">With ORDER BY, default frame becomes</text>
  <text x="40" y="212" font-size="11" fill="#333">UNBOUNDED PRECEDING → CURRENT ROW.</text>

  <!-- Card 4 -->
  <rect x="390" y="150" width="340" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="410" y="175" font-size="13" font-weight="bold" fill="#6a1b9a">Ranking ≠ Aggregation</text>
  <text x="410" y="195" font-size="11" fill="#333">ROW_NUMBER / RANK / DENSE_RANK</text>
  <text x="410" y="212" font-size="11" fill="#333">assign positions, not compute values.</text>

  <!-- Card 5 -->
  <rect x="20" y="250" width="340" height="80" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="40" y="275" font-size="13" font-weight="bold" fill="#c62828">LAG / LEAD Are Your Time Machine</text>
  <text x="40" y="295" font-size="11" fill="#333">Look backward or forward without</text>
  <text x="40" y="312" font-size="11" fill="#333">self-joins. Perfect for change detection.</text>

  <!-- Card 6 -->
  <rect x="390" y="250" width="340" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="410" y="275" font-size="13" font-weight="bold" fill="#2e7d32">LAST_VALUE Needs Full Frame</text>
  <text x="410" y="295" font-size="11" fill="#333">Always add ROWS BETWEEN UNBOUNDED</text>
  <text x="410" y="312" font-size="11" fill="#333">PRECEDING AND UNBOUNDED FOLLOWING.</text>

  <!-- Card 7 -->
  <rect x="20" y="350" width="340" height="80" rx="10" fill="#fff8e1" stroke="#f9a825" stroke-width="2"/>
  <text x="40" y="375" font-size="13" font-weight="bold" fill="#f9a825">Can't Filter in WHERE</text>
  <text x="40" y="395" font-size="11" fill="#333">Window functions evaluate at SELECT.</text>
  <text x="40" y="412" font-size="11" fill="#333">Use CTE/subquery to filter results.</text>

  <!-- Card 8 -->
  <rect x="390" y="350" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="410" y="375" font-size="13" font-weight="bold" fill="#1565c0">Name Your Windows</text>
  <text x="410" y="395" font-size="11" fill="#333">WINDOW w AS (…) avoids duplication</text>
  <text x="410" y="412" font-size="11" fill="#333">and helps the optimizer share sorts.</text>

  <!-- Decision Guide -->
  <rect x="80" y="455" width="590" height="50" rx="10" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="375" y="478" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Window Function Cheat Sheet</text>
  <text x="375" y="496" text-anchor="middle" font-size="10" fill="#555">Unique # → ROW_NUMBER | Rank with gaps → RANK | Rank no gaps → DENSE_RANK | Buckets → NTILE | Prev/Next → LAG/LEAD</text>
</svg>

---

## 18. Download

<a href="15_window_functions.md" download="15_window_functions.md" style="display:inline-block;padding:14px 28px;background:linear-gradient(135deg,#1565c0,#0d47a1);color:#fff;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;box-shadow:0 4px 15px rgba(21,101,192,0.3);margin:10px 0;">Download 15_window_functions.md</a>
