# Phase 4 — SQL Fundamentals (DDL + DML + DQL)

> **Goal**: Master SQL from the ground up — the language that powers every relational database. Learn to define schemas (DDL), manipulate data (DML), and query it (DQL) with confidence. This phase contains 25+ fully worked query examples with sample data and output.

---

## Table of Contents

1. [What is SQL?](#1-what-is-sql)
2. [SQL Sub-Languages](#2-sql-sub-languages)
3. [Data Types — Deep Dive](#3-data-types--deep-dive)
4. [DDL — Data Definition Language](#4-ddl--data-definition-language)
5. [DML — Data Manipulation Language](#5-dml--data-manipulation-language)
6. [DQL — Data Query Language (SELECT)](#6-dql--data-query-language-select)
7. [Filtering with WHERE](#7-filtering-with-where)
8. [Sorting with ORDER BY](#8-sorting-with-order-by)
9. [Aggregation Functions](#9-aggregation-functions)
10. [GROUP BY and HAVING](#10-group-by-and-having)
11. [DISTINCT, LIMIT, OFFSET](#11-distinct-limit-offset)
12. [Aliases and Expressions](#12-aliases-and-expressions)
13. [NULL Handling](#13-null-handling)
14. [SQL Query Execution Order](#14-sql-query-execution-order)
15. [25+ Worked Query Examples](#15-25-worked-query-examples)
16. [Common Interview Questions](#16-common-interview-questions)
17. [Key Takeaways](#17-key-takeaways)

---

## 1. What is SQL?

**Structured Query Language (SQL)** is the standard language for communicating with relational databases. Every RDBMS speaks SQL (with minor dialect variations).

### Real-World Analogy

SQL is like ordering at a restaurant:
- **DDL**: Designing the menu (defining what dishes exist)
- **DML**: The kitchen preparing/modifying dishes (inserting, updating, deleting data)
- **DQL**: Ordering from the menu (querying — "give me all vegetarian dishes under $15, sorted by price")

### SQL is Declarative

You describe **what** you want, not **how** to get it. The database engine's query optimizer decides the most efficient execution plan.

```sql
-- You write THIS (what):
SELECT name, gpa FROM students WHERE dept = 'CS' ORDER BY gpa DESC;

-- The optimizer decides HOW:
-- "Use index on dept → filter → project → sort" (you never see this)
```

### SQL Standard vs. Dialects

| Standard | Year | Key Additions |
|---|---|---|
| SQL-86 | 1986 | First standard |
| SQL-92 | 1992 | JOIN syntax, CASE, CAST |
| SQL:1999 | 1999 | CTEs, triggers, recursive queries |
| SQL:2003 | 2003 | Window functions, MERGE, XML |
| SQL:2011 | 2011 | Temporal data |
| SQL:2016 | 2016 | JSON support |
| SQL:2023 | 2023 | Graph queries, property graphs |

Every vendor adds extensions: PostgreSQL has `JSONB`, `LATERAL`, `RETURNING`; MySQL has `LIMIT`; Oracle has `CONNECT BY`, `FLASHBACK`. The core SQL is the same everywhere.

---

## 2. SQL Sub-Languages

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 300" font-family="Segoe UI, Arial, sans-serif">
  <rect width="760" height="300" fill="#f8f9fa" rx="12"/>
  <text x="380" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">SQL Sub-Languages</text>

  <!-- DDL -->
  <rect x="20" y="50" width="170" height="230" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="105" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">DDL</text>
  <text x="105" y="92" text-anchor="middle" font-size="10" fill="#555">Data Definition</text>
  <rect x="32" y="105" width="146" height="22" rx="4" fill="#d6eaf8"/>
  <text x="105" y="120" text-anchor="middle" font-size="11" fill="#1a5276">CREATE</text>
  <rect x="32" y="132" width="146" height="22" rx="4" fill="#d6eaf8"/>
  <text x="105" y="147" text-anchor="middle" font-size="11" fill="#1a5276">ALTER</text>
  <rect x="32" y="159" width="146" height="22" rx="4" fill="#d6eaf8"/>
  <text x="105" y="174" text-anchor="middle" font-size="11" fill="#1a5276">DROP</text>
  <rect x="32" y="186" width="146" height="22" rx="4" fill="#d6eaf8"/>
  <text x="105" y="201" text-anchor="middle" font-size="11" fill="#1a5276">TRUNCATE</text>
  <text x="105" y="235" text-anchor="middle" font-size="10" fill="#2980b9" font-style="italic">Define structure</text>
  <text x="105" y="250" text-anchor="middle" font-size="10" fill="#2980b9" font-style="italic">Auto-commits</text>

  <!-- DML -->
  <rect x="205" y="50" width="170" height="230" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="290" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">DML</text>
  <text x="290" y="92" text-anchor="middle" font-size="10" fill="#555">Data Manipulation</text>
  <rect x="217" y="105" width="146" height="22" rx="4" fill="#d5f5e3"/>
  <text x="290" y="120" text-anchor="middle" font-size="11" fill="#145a32">INSERT</text>
  <rect x="217" y="132" width="146" height="22" rx="4" fill="#d5f5e3"/>
  <text x="290" y="147" text-anchor="middle" font-size="11" fill="#145a32">UPDATE</text>
  <rect x="217" y="159" width="146" height="22" rx="4" fill="#d5f5e3"/>
  <text x="290" y="174" text-anchor="middle" font-size="11" fill="#145a32">DELETE</text>
  <rect x="217" y="186" width="146" height="22" rx="4" fill="#d5f5e3"/>
  <text x="290" y="201" text-anchor="middle" font-size="11" fill="#145a32">MERGE (UPSERT)</text>
  <text x="290" y="235" text-anchor="middle" font-size="10" fill="#27ae60" font-style="italic">Modify data</text>
  <text x="290" y="250" text-anchor="middle" font-size="10" fill="#27ae60" font-style="italic">Can be rolled back</text>

  <!-- DQL -->
  <rect x="390" y="50" width="170" height="230" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="475" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#e67e22">DQL</text>
  <text x="475" y="92" text-anchor="middle" font-size="10" fill="#555">Data Query</text>
  <rect x="402" y="105" width="146" height="22" rx="4" fill="#fdebd0"/>
  <text x="475" y="120" text-anchor="middle" font-size="11" fill="#784212">SELECT</text>
  <rect x="402" y="132" width="146" height="22" rx="4" fill="#fdebd0"/>
  <text x="475" y="147" text-anchor="middle" font-size="11" fill="#784212">WHERE</text>
  <rect x="402" y="159" width="146" height="22" rx="4" fill="#fdebd0"/>
  <text x="475" y="174" text-anchor="middle" font-size="11" fill="#784212">GROUP BY / HAVING</text>
  <rect x="402" y="186" width="146" height="22" rx="4" fill="#fdebd0"/>
  <text x="475" y="201" text-anchor="middle" font-size="11" fill="#784212">ORDER BY / LIMIT</text>
  <text x="475" y="235" text-anchor="middle" font-size="10" fill="#e67e22" font-style="italic">Retrieve data</text>
  <text x="475" y="250" text-anchor="middle" font-size="10" fill="#e67e22" font-style="italic">Read-only</text>

  <!-- DCL / TCL -->
  <rect x="575" y="50" width="170" height="230" rx="8" fill="#f4ecf7" stroke="#8e44ad" stroke-width="2"/>
  <text x="660" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#8e44ad">DCL / TCL</text>
  <text x="660" y="92" text-anchor="middle" font-size="10" fill="#555">Control + Transaction</text>
  <rect x="587" y="105" width="146" height="22" rx="4" fill="#e8daef"/>
  <text x="660" y="120" text-anchor="middle" font-size="11" fill="#4a235a">GRANT</text>
  <rect x="587" y="132" width="146" height="22" rx="4" fill="#e8daef"/>
  <text x="660" y="147" text-anchor="middle" font-size="11" fill="#4a235a">REVOKE</text>
  <rect x="587" y="159" width="146" height="22" rx="4" fill="#e8daef"/>
  <text x="660" y="174" text-anchor="middle" font-size="11" fill="#4a235a">BEGIN / COMMIT</text>
  <rect x="587" y="186" width="146" height="22" rx="4" fill="#e8daef"/>
  <text x="660" y="201" text-anchor="middle" font-size="11" fill="#4a235a">ROLLBACK / SAVEPOINT</text>
  <text x="660" y="235" text-anchor="middle" font-size="10" fill="#8e44ad" font-style="italic">Security + transactions</text>
  <text x="660" y="250" text-anchor="middle" font-size="10" fill="#8e44ad" font-style="italic">(covered in later phases)</text>
</svg>

---

## 3. Data Types — Deep Dive

### Numeric Types

| Type | Range / Precision | Storage | Use Case |
|---|---|---|---|
| `SMALLINT` | -32,768 to 32,767 | 2 bytes | Status codes, small counts |
| `INTEGER` / `INT` | -2.1 billion to 2.1 billion | 4 bytes | General-purpose IDs, counts |
| `BIGINT` | -9.2 × 10¹⁸ to 9.2 × 10¹⁸ | 8 bytes | Large IDs (social media), timestamps |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | Exact: p total digits, s after decimal | Variable | **Money**, financial calculations |
| `REAL` / `FLOAT(24)` | ~7 decimal digits precision | 4 bytes | Scientific data (approx OK) |
| `DOUBLE PRECISION` / `FLOAT(53)` | ~15 decimal digits precision | 8 bytes | Scientific data (approx OK) |
| `SERIAL` (PostgreSQL) | Auto-increment INT | 4 bytes | Surrogate primary keys |
| `BIGSERIAL` (PostgreSQL) | Auto-increment BIGINT | 8 bytes | Large auto-increment keys |

```sql
-- CRITICAL: NEVER use FLOAT/DOUBLE for money!
SELECT 0.1 + 0.2;                 -- might return 0.30000000000000004 (floating point error)
SELECT 0.1::DECIMAL + 0.2::DECIMAL;  -- returns exactly 0.3

CREATE TABLE accounts (
    balance  DECIMAL(15,2) NOT NULL   -- exact! e.g., 1234567890123.99
);
```

### String Types

| Type | Description | Max Length | Use Case |
|---|---|---|---|
| `CHAR(n)` | Fixed-length, padded with spaces | n chars | Fixed codes: state='CA', country='US' |
| `VARCHAR(n)` | Variable-length | n chars | Names, emails, most text |
| `TEXT` | Variable-length, unlimited | Unlimited | Descriptions, comments, articles |

```sql
CREATE TABLE students (
    name       VARCHAR(100),      -- variable: "Alice" uses 5 bytes, not 100
    dept_code  CHAR(4),           -- fixed: always exactly 4 chars ("CS  " padded)
    bio        TEXT                -- unlimited length
);

-- CHAR vs VARCHAR gotcha:
-- CHAR('CS  ') = CHAR('CS') → TRUE (trailing spaces ignored in comparisons)
-- VARCHAR('CS  ') = VARCHAR('CS') → depends on DBMS (PostgreSQL: TRUE, MySQL: depends)
```

### Date and Time Types

| Type | Format | Example | Storage |
|---|---|---|---|
| `DATE` | YYYY-MM-DD | '2026-04-24' | 4 bytes |
| `TIME` | HH:MM:SS | '14:30:00' | 8 bytes |
| `TIMESTAMP` | YYYY-MM-DD HH:MM:SS | '2026-04-24 14:30:00' | 8 bytes |
| `TIMESTAMP WITH TIME ZONE` | With timezone | '2026-04-24 14:30:00+05:30' | 8 bytes |
| `INTERVAL` | Duration | '2 years 3 months' | 16 bytes |

```sql
CREATE TABLE events (
    event_date    DATE,
    start_time    TIME,
    created_at    TIMESTAMP DEFAULT NOW(),
    scheduled_at  TIMESTAMP WITH TIME ZONE
);

-- Date arithmetic:
SELECT NOW() - INTERVAL '30 days';                     -- 30 days ago
SELECT AGE('2026-04-24'::DATE, '1998-06-15'::DATE);   -- '27 years 10 mons 9 days'
SELECT EXTRACT(YEAR FROM NOW());                        -- 2026
SELECT DATE_TRUNC('month', NOW());                      -- first of current month
```

### Boolean Type

```sql
CREATE TABLE tasks (
    is_complete  BOOLEAN DEFAULT FALSE
);

-- Valid boolean values: TRUE, FALSE, NULL
-- PostgreSQL also accepts: 't', 'f', 'yes', 'no', '1', '0'

SELECT * FROM tasks WHERE is_complete;          -- shorthand for = TRUE
SELECT * FROM tasks WHERE NOT is_complete;      -- shorthand for = FALSE
SELECT * FROM tasks WHERE is_complete IS NULL;  -- explicitly check NULL
```

### JSON Types (PostgreSQL)

| Type | Description | Indexed? |
|---|---|---|
| `JSON` | Stores raw JSON text | No |
| `JSONB` | Binary JSON, decomposed | Yes (GIN index) |

```sql
CREATE TABLE products (
    id       SERIAL PRIMARY KEY,
    name     TEXT,
    details  JSONB
);

INSERT INTO products (name, details) VALUES
    ('Laptop', '{"brand": "Dell", "ram": 16, "ports": ["USB-C", "HDMI"]}');

-- Query JSON:
SELECT name FROM products WHERE details->>'brand' = 'Dell';
SELECT name FROM products WHERE (details->>'ram')::INT > 8;
SELECT name, jsonb_array_elements_text(details->'ports') AS port FROM products;
```

### UUID Type

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE sessions (
    session_id  UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id     INT,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

### Type Casting

```sql
-- PostgreSQL casting:
SELECT '42'::INTEGER;                     -- string to int
SELECT 3.14::TEXT;                        -- number to string
SELECT '2026-04-24'::DATE;               -- string to date
SELECT CAST('42' AS INTEGER);            -- SQL standard syntax
```

---

## 4. DDL — Data Definition Language

DDL defines the **structure** (schema) of the database. DDL statements are auto-committed in most databases (cannot be rolled back).

### 4.1 CREATE TABLE

```sql
CREATE TABLE table_name (
    column_name  data_type  [constraints],
    column_name  data_type  [constraints],
    ...
    [table-level constraints]
);
```

#### Complete Example with All Constraint Types

```sql
CREATE TABLE employees (
    -- Column definitions with inline constraints
    emp_id       SERIAL       PRIMARY KEY,                            -- PK + auto-increment
    first_name   VARCHAR(50)  NOT NULL,                               -- cannot be NULL
    last_name    VARCHAR(50)  NOT NULL,
    email        VARCHAR(200) UNIQUE NOT NULL,                        -- unique + not null
    phone        VARCHAR(20),                                          -- nullable (default)
    dept_id      INT          NOT NULL REFERENCES departments(dept_id), -- FK + not null
    salary       DECIMAL(10,2) DEFAULT 50000.00,                      -- default value
    hire_date    DATE         NOT NULL DEFAULT CURRENT_DATE,
    is_active    BOOLEAN      DEFAULT TRUE,
    gender       CHAR(1)      CHECK (gender IN ('M', 'F', 'O')),     -- CHECK constraint
    manager_id   INT          REFERENCES employees(emp_id),           -- self-referencing FK

    -- Table-level constraints (for composite constraints)
    CONSTRAINT salary_positive CHECK (salary > 0),
    CONSTRAINT unique_name_dept UNIQUE (first_name, last_name, dept_id)
);
```

#### CREATE TABLE with Foreign Key Actions

```sql
CREATE TABLE order_items (
    order_id    INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT NOT NULL CHECK (quantity > 0),
    unit_price  DECIMAL(10,2) NOT NULL,

    PRIMARY KEY (order_id, product_id),

    FOREIGN KEY (order_id) REFERENCES orders(order_id)
        ON DELETE CASCADE           -- delete order → delete items
        ON UPDATE CASCADE,

    FOREIGN KEY (product_id) REFERENCES products(product_id)
        ON DELETE RESTRICT          -- can't delete product if it's in an order
        ON UPDATE CASCADE
);
```

#### CREATE TABLE AS (create from a query)

```sql
-- Create a table from a query result
CREATE TABLE cs_students AS
SELECT student_id, name, gpa
FROM students
WHERE dept = 'CS';

-- Create an empty copy of a table's structure
CREATE TABLE students_backup (LIKE students INCLUDING ALL);
```

### 4.2 ALTER TABLE

Modify an existing table's structure.

```sql
-- ADD a column
ALTER TABLE employees ADD COLUMN middle_name VARCHAR(50);

-- DROP a column
ALTER TABLE employees DROP COLUMN middle_name;

-- RENAME a column
ALTER TABLE employees RENAME COLUMN phone TO phone_number;

-- CHANGE column type
ALTER TABLE employees ALTER COLUMN salary TYPE DECIMAL(12,2);

-- SET / DROP default
ALTER TABLE employees ALTER COLUMN salary SET DEFAULT 60000.00;
ALTER TABLE employees ALTER COLUMN salary DROP DEFAULT;

-- ADD / DROP NOT NULL
ALTER TABLE employees ALTER COLUMN phone SET NOT NULL;
ALTER TABLE employees ALTER COLUMN phone DROP NOT NULL;

-- ADD a constraint
ALTER TABLE employees ADD CONSTRAINT chk_salary CHECK (salary BETWEEN 20000 AND 500000);

-- DROP a constraint
ALTER TABLE employees DROP CONSTRAINT chk_salary;

-- ADD a foreign key
ALTER TABLE employees ADD CONSTRAINT fk_dept
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id);

-- RENAME table
ALTER TABLE employees RENAME TO staff;
```

### 4.3 DROP TABLE

Permanently removes a table and all its data.

```sql
-- Drop a table (error if doesn't exist)
DROP TABLE employees;

-- Drop only if it exists (safe)
DROP TABLE IF EXISTS employees;

-- Drop with CASCADE (also drops dependent objects: views, FKs, etc.)
DROP TABLE departments CASCADE;

-- Drop with RESTRICT (fail if anything depends on it — default behavior)
DROP TABLE departments RESTRICT;
```

### 4.4 TRUNCATE TABLE

Removes **all rows** from a table but keeps the structure. Faster than DELETE (doesn't log individual row deletions).

```sql
TRUNCATE TABLE enrollments;

-- Truncate with CASCADE (also truncates tables with FK references)
TRUNCATE TABLE students CASCADE;

-- TRUNCATE resets auto-increment counters:
TRUNCATE TABLE orders RESTART IDENTITY;  -- PostgreSQL

-- Key differences: TRUNCATE vs DELETE
-- TRUNCATE: DDL, auto-commits, faster, resets identity, can't filter (no WHERE)
-- DELETE:   DML, can rollback, slower, keeps identity, can filter with WHERE
```

### 4.5 CREATE INDEX, CREATE VIEW, CREATE SEQUENCE

```sql
-- Index (speeds up queries on that column)
CREATE INDEX idx_students_dept ON students(dept);
CREATE UNIQUE INDEX idx_students_email ON students(email);

-- View (virtual table from a query)
CREATE VIEW cs_honor_roll AS
SELECT name, gpa FROM students WHERE dept = 'CS' AND gpa > 3.5;

-- Sequence (auto-increment generator)
CREATE SEQUENCE order_seq START WITH 1000 INCREMENT BY 1;
SELECT NEXTVAL('order_seq');  -- returns 1000, then 1001, then 1002...
```

### DDL Summary

| Command | What It Does | Reversible? |
|---|---|---|
| `CREATE TABLE` | Creates a new table | DROP to undo |
| `ALTER TABLE` | Modifies table structure | ALTER to undo |
| `DROP TABLE` | Removes table entirely | No (data gone) |
| `TRUNCATE TABLE` | Removes all rows, keeps structure | No (auto-commits) |
| `CREATE INDEX` | Creates a search optimization structure | DROP INDEX |
| `CREATE VIEW` | Creates a virtual table | DROP VIEW |

---

## 5. DML — Data Manipulation Language

DML modifies the **data** inside tables. DML operations can be rolled back (inside a transaction).

### 5.1 INSERT

```sql
-- Insert a single row (explicit columns)
INSERT INTO students (student_id, name, dept, gpa)
VALUES (1, 'Alice', 'CS', 3.9);

-- Insert without specifying columns (must match table order exactly)
INSERT INTO students VALUES (2, 'Bob', 'Math', 3.5);

-- Insert multiple rows at once
INSERT INTO students (student_id, name, dept, gpa) VALUES
    (3, 'Carol', 'CS', 3.7),
    (4, 'Dave', 'EE', 3.2),
    (5, 'Eve', 'CS', 3.1);

-- Insert with DEFAULT values
INSERT INTO employees (first_name, last_name, email, dept_id)
VALUES ('John', 'Doe', 'john@example.com', 1);
-- salary uses DEFAULT 50000.00, hire_date uses DEFAULT CURRENT_DATE

-- Insert from a SELECT (copy data from another table/query)
INSERT INTO cs_students (student_id, name, gpa)
SELECT student_id, name, gpa FROM students WHERE dept = 'CS';

-- INSERT with RETURNING (PostgreSQL) — get back the inserted data
INSERT INTO students (name, dept, gpa)
VALUES ('Frank', 'Physics', 3.6)
RETURNING student_id, name;  -- returns the auto-generated ID
```

### 5.2 UPDATE

```sql
-- Update specific rows
UPDATE students SET gpa = 3.95 WHERE student_id = 1;

-- Update multiple columns
UPDATE students
SET dept = 'Physics', gpa = 3.8
WHERE student_id = 2;

-- Update all rows (no WHERE — dangerous!)
UPDATE products SET price = price * 1.10;  -- 10% price increase for ALL products

-- Update with a subquery
UPDATE students
SET gpa = gpa + 0.1
WHERE student_id IN (SELECT student_id FROM honor_list);

-- Update with FROM (PostgreSQL) — join-based update
UPDATE enrollments e
SET grade = 'A+'
FROM students s
WHERE e.student_id = s.student_id
  AND s.gpa > 3.9
  AND e.grade = 'A';

-- UPDATE with RETURNING
UPDATE accounts
SET balance = balance - 500
WHERE account_id = 1001
RETURNING account_id, balance;  -- see the new balance immediately
```

### 5.3 DELETE

```sql
-- Delete specific rows
DELETE FROM students WHERE student_id = 5;

-- Delete with a condition
DELETE FROM enrollments WHERE grade = 'F';

-- Delete all rows (keeps table structure — like TRUNCATE but slower, can rollback)
DELETE FROM temp_data;

-- Delete with a subquery
DELETE FROM enrollments
WHERE student_id IN (
    SELECT student_id FROM students WHERE is_active = FALSE
);

-- Delete with USING (PostgreSQL) — join-based delete
DELETE FROM enrollments e
USING students s
WHERE e.student_id = s.student_id AND s.dept = 'CS';

-- DELETE with RETURNING
DELETE FROM sessions WHERE expires_at < NOW()
RETURNING session_id, user_id;
```

### 5.4 MERGE (UPSERT)

`MERGE` performs INSERT, UPDATE, or DELETE in a single statement based on whether a match exists. SQL:2003 standard. Also called "UPSERT" (update or insert).

```sql
-- SQL Standard MERGE (Oracle, SQL Server, PostgreSQL 15+)
MERGE INTO inventory AS target
USING new_shipment AS source
ON target.product_id = source.product_id
WHEN MATCHED THEN
    UPDATE SET target.quantity = target.quantity + source.quantity,
               target.last_updated = NOW()
WHEN NOT MATCHED THEN
    INSERT (product_id, product_name, quantity, last_updated)
    VALUES (source.product_id, source.product_name, source.quantity, NOW());

-- PostgreSQL UPSERT (INSERT ... ON CONFLICT)
INSERT INTO students (student_id, name, dept, gpa)
VALUES (1, 'Alice', 'CS', 3.95)
ON CONFLICT (student_id)
DO UPDATE SET gpa = EXCLUDED.gpa, dept = EXCLUDED.dept;
-- If student_id 1 exists → update gpa and dept
-- If student_id 1 doesn't exist → insert new row

-- ON CONFLICT DO NOTHING (ignore duplicates silently)
INSERT INTO students (student_id, name, dept, gpa)
VALUES (1, 'Alice', 'CS', 3.95)
ON CONFLICT (student_id) DO NOTHING;
```

### DML Summary

| Command | What It Does | Needs WHERE? | Rollback? |
|---|---|---|---|
| `INSERT` | Add new rows | No | Yes |
| `UPDATE` | Modify existing rows | Usually yes! | Yes |
| `DELETE` | Remove rows | Usually yes! | Yes |
| `MERGE` | Upsert (insert or update) | Uses ON clause | Yes |

> **Golden Rule**: Always use `WHERE` with UPDATE and DELETE. Without it, you affect ALL rows!

---

## 6. DQL — Data Query Language (SELECT)

The `SELECT` statement is the most powerful and most-used SQL command. Everything from here on is about retrieving data.

### Basic Syntax

```sql
SELECT [DISTINCT] column_list       -- which columns
FROM table_name                     -- which table
[WHERE condition]                   -- filter rows
[GROUP BY columns]                  -- group rows
[HAVING condition]                  -- filter groups
[ORDER BY columns [ASC|DESC]]       -- sort results
[LIMIT n [OFFSET m]];              -- pagination
```

### The Simplest Queries

```sql
-- Select all columns, all rows
SELECT * FROM students;

-- Select specific columns
SELECT name, gpa FROM students;

-- Select with a calculation
SELECT name, gpa, gpa * 25 AS weighted_gpa FROM students;

-- Select a constant/expression
SELECT 1 + 1 AS result;           -- returns 2
SELECT NOW() AS current_time;     -- returns current timestamp
SELECT 'Hello, SQL!' AS greeting; -- returns a string
```

---

## 7. Filtering with WHERE

### Comparison Operators

| Operator | Meaning | Example |
|---|---|---|
| `=` | Equal | `WHERE dept = 'CS'` |
| `!=` or `<>` | Not equal | `WHERE dept != 'CS'` |
| `<` | Less than | `WHERE gpa < 3.5` |
| `>` | Greater than | `WHERE gpa > 3.0` |
| `<=` | Less or equal | `WHERE credits <= 4` |
| `>=` | Greater or equal | `WHERE salary >= 50000` |

### Logical Operators

```sql
-- AND: both conditions must be true
SELECT * FROM students WHERE dept = 'CS' AND gpa > 3.5;

-- OR: either condition can be true
SELECT * FROM students WHERE dept = 'CS' OR dept = 'Math';

-- NOT: negates a condition
SELECT * FROM students WHERE NOT dept = 'CS';

-- Combining (use parentheses for clarity!)
SELECT * FROM students
WHERE (dept = 'CS' OR dept = 'EE') AND gpa > 3.0;
```

### BETWEEN

```sql
-- Inclusive range
SELECT * FROM students WHERE gpa BETWEEN 3.0 AND 3.5;
-- Equivalent to: WHERE gpa >= 3.0 AND gpa <= 3.5

SELECT * FROM orders WHERE order_date BETWEEN '2026-01-01' AND '2026-03-31';

-- NOT BETWEEN
SELECT * FROM students WHERE gpa NOT BETWEEN 3.0 AND 3.5;
```

### IN

```sql
-- Match against a list of values
SELECT * FROM students WHERE dept IN ('CS', 'Math', 'EE');
-- Equivalent to: WHERE dept = 'CS' OR dept = 'Math' OR dept = 'EE'

-- NOT IN
SELECT * FROM students WHERE dept NOT IN ('CS', 'Math');

-- IN with subquery
SELECT * FROM students
WHERE student_id IN (SELECT student_id FROM enrollments WHERE grade = 'A');
```

### LIKE (Pattern Matching)

| Pattern | Meaning | Example | Matches |
|---|---|---|---|
| `%` | Any sequence of characters (0+) | `'A%'` | Alice, Ab, A |
| `_` | Any single character | `'_ob'` | Bob, Rob |
| `%word%` | Contains "word" anywhere | `'%data%'` | Database, Big data, data |

```sql
-- Names starting with 'A'
SELECT * FROM students WHERE name LIKE 'A%';

-- Names ending with 'e'
SELECT * FROM students WHERE name LIKE '%e';

-- Names with exactly 3 characters
SELECT * FROM students WHERE name LIKE '___';

-- Case-insensitive (PostgreSQL: ILIKE)
SELECT * FROM students WHERE name ILIKE '%alice%';

-- Escape special characters
SELECT * FROM products WHERE name LIKE '%10\%%' ESCAPE '\';  -- matches "10% off"
```

### IS NULL / IS NOT NULL

```sql
SELECT * FROM students WHERE phone IS NULL;
SELECT * FROM students WHERE phone IS NOT NULL;

-- WRONG (never use = NULL):
SELECT * FROM students WHERE phone = NULL;      -- returns NOTHING!
```

### EXISTS

```sql
-- Students who have at least one enrollment
SELECT * FROM students s
WHERE EXISTS (
    SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id
);

-- Students with no enrollments
SELECT * FROM students s
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id
);
```

### ANY / ALL

```sql
-- GPA greater than ANY CS student's GPA (i.e., > the minimum CS GPA)
SELECT * FROM students
WHERE gpa > ANY (SELECT gpa FROM students WHERE dept = 'CS');

-- GPA greater than ALL CS students' GPAs (i.e., > the maximum CS GPA)
SELECT * FROM students
WHERE gpa > ALL (SELECT gpa FROM students WHERE dept = 'CS');
```

---

## 8. Sorting with ORDER BY

```sql
-- Sort ascending (default)
SELECT * FROM students ORDER BY gpa;           -- lowest first
SELECT * FROM students ORDER BY gpa ASC;       -- same thing, explicit

-- Sort descending
SELECT * FROM students ORDER BY gpa DESC;      -- highest first

-- Multi-column sort
SELECT * FROM students ORDER BY dept ASC, gpa DESC;
-- First sort by dept alphabetically, then within each dept, sort by GPA descending

-- Sort by column position (not recommended for readability)
SELECT name, dept, gpa FROM students ORDER BY 2, 3 DESC;
-- 2 = dept (ASC), 3 = gpa (DESC)

-- Sort by expression
SELECT name, gpa FROM students ORDER BY gpa * 25 DESC;

-- NULLs in sorting
SELECT * FROM students ORDER BY gpa NULLS FIRST;   -- PostgreSQL
SELECT * FROM students ORDER BY gpa NULLS LAST;    -- PostgreSQL
```

---

## 9. Aggregation Functions

Aggregate functions compute a **single value** from a set of rows.

| Function | Description | Example | Result |
|---|---|---|---|
| `COUNT(*)` | Count all rows | `COUNT(*)` | 5 |
| `COUNT(col)` | Count non-NULL values | `COUNT(phone)` | 3 (if 2 are NULL) |
| `COUNT(DISTINCT col)` | Count distinct non-NULL values | `COUNT(DISTINCT dept)` | 3 |
| `SUM(col)` | Sum of values | `SUM(salary)` | 250000 |
| `AVG(col)` | Average (mean) | `AVG(gpa)` | 3.48 |
| `MIN(col)` | Minimum value | `MIN(gpa)` | 3.1 |
| `MAX(col)` | Maximum value | `MAX(gpa)` | 3.9 |
| `STRING_AGG(col, sep)` | Concatenate strings (PG) | `STRING_AGG(name, ', ')` | 'Alice, Bob' |

```sql
-- Basic aggregation (returns a single row)
SELECT
    COUNT(*) AS total_students,
    COUNT(DISTINCT dept) AS departments,
    AVG(gpa) AS avg_gpa,
    MIN(gpa) AS min_gpa,
    MAX(gpa) AS max_gpa,
    SUM(gpa) AS sum_gpa
FROM students;
```

**Result:**

| total_students | departments | avg_gpa | min_gpa | max_gpa | sum_gpa |
|---|---|---|---|---|---|
| 5 | 3 | 3.48 | 3.1 | 3.9 | 17.4 |

> **Aggregates ignore NULLs** (except `COUNT(*)`). If a column has NULLs, `AVG()` skips them. `COUNT(col)` counts only non-NULL values. `COUNT(*)` counts all rows regardless.

---

## 10. GROUP BY and HAVING

### GROUP BY

Divides rows into groups based on one or more columns. Aggregation functions then operate on each group independently.

```sql
-- Average GPA by department
SELECT dept, AVG(gpa) AS avg_gpa, COUNT(*) AS num_students
FROM students
GROUP BY dept;
```

**Result:**

| dept | avg_gpa | num_students |
|---|---|---|
| CS | 3.567 | 3 |
| Math | 3.500 | 1 |
| EE | 3.200 | 1 |

**Rule**: Every column in SELECT must either be in GROUP BY or inside an aggregate function.

```sql
-- WRONG (name is not in GROUP BY and not aggregated):
SELECT dept, name, AVG(gpa) FROM students GROUP BY dept;
-- ERROR: column "name" must appear in GROUP BY clause or be used in an aggregate

-- CORRECT:
SELECT dept, STRING_AGG(name, ', ') AS student_names, AVG(gpa) AS avg_gpa
FROM students
GROUP BY dept;
```

### GROUP BY with Multiple Columns

```sql
-- Count students by department AND GPA range
SELECT
    dept,
    CASE
        WHEN gpa >= 3.5 THEN 'High'
        WHEN gpa >= 3.0 THEN 'Medium'
        ELSE 'Low'
    END AS gpa_level,
    COUNT(*) AS count
FROM students
GROUP BY dept, gpa_level;
```

### HAVING — Filter Groups

`HAVING` filters **groups** (after GROUP BY), while `WHERE` filters **individual rows** (before GROUP BY).

```sql
-- Departments with average GPA > 3.3
SELECT dept, AVG(gpa) AS avg_gpa
FROM students
GROUP BY dept
HAVING AVG(gpa) > 3.3;
```

**Result:**

| dept | avg_gpa |
|---|---|
| CS | 3.567 |
| Math | 3.500 |

```sql
-- Courses with more than 2 enrolled students
SELECT e.course_id, c.title, COUNT(*) AS enrolled
FROM enrollments e
JOIN courses c ON e.course_id = c.course_id
GROUP BY e.course_id, c.title
HAVING COUNT(*) > 2;
```

### WHERE vs. HAVING

| | WHERE | HAVING |
|---|---|---|
| **Filters** | Individual rows | Groups (after GROUP BY) |
| **When** | Before grouping | After grouping |
| **Can use aggregates?** | No | Yes |
| **Example** | `WHERE gpa > 3.0` | `HAVING AVG(gpa) > 3.0` |

```sql
-- Both in one query:
SELECT dept, AVG(gpa) AS avg_gpa
FROM students
WHERE gpa > 2.0          -- first: filter out students with GPA ≤ 2.0
GROUP BY dept             -- then: group remaining students by dept
HAVING AVG(gpa) > 3.3;   -- finally: keep only groups with avg > 3.3
```

---

## 11. DISTINCT, LIMIT, OFFSET

### DISTINCT

Removes duplicate rows from the result.

```sql
-- Without DISTINCT:
SELECT dept FROM students;  -- returns: CS, Math, CS, EE, CS

-- With DISTINCT:
SELECT DISTINCT dept FROM students;  -- returns: CS, Math, EE

-- DISTINCT on multiple columns (unique combinations):
SELECT DISTINCT dept, grade FROM enrollments;
```

### LIMIT and OFFSET (Pagination)

```sql
-- Top 3 students by GPA
SELECT name, gpa FROM students ORDER BY gpa DESC LIMIT 3;

-- Skip first 2, get next 3 (page 2 of 3 results per page)
SELECT name, gpa FROM students ORDER BY gpa DESC LIMIT 3 OFFSET 2;

-- Pagination formula:
-- Page N with page_size rows:
-- LIMIT page_size OFFSET (N - 1) * page_size

-- Page 1: LIMIT 10 OFFSET 0
-- Page 2: LIMIT 10 OFFSET 10
-- Page 3: LIMIT 10 OFFSET 20
```

```sql
-- SQL Standard (FETCH FIRST ... ROWS):
SELECT name, gpa FROM students
ORDER BY gpa DESC
FETCH FIRST 3 ROWS ONLY;

-- With OFFSET:
SELECT name, gpa FROM students
ORDER BY gpa DESC
OFFSET 2 ROWS FETCH NEXT 3 ROWS ONLY;
```

---

## 12. Aliases and Expressions

### Column Aliases

```sql
SELECT
    name AS student_name,
    gpa AS grade_point_average,
    gpa * 25 AS weighted_score,
    CASE
        WHEN gpa >= 3.5 THEN 'Distinction'
        WHEN gpa >= 3.0 THEN 'First Class'
        ELSE 'Second Class'
    END AS classification
FROM students;
```

### Table Aliases

```sql
-- Short aliases make joins readable
SELECT s.name, e.grade, c.title
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id;
```

### CASE Expressions

The SQL equivalent of if-else.

```sql
-- Simple CASE (match against values)
SELECT name,
    CASE dept
        WHEN 'CS' THEN 'Computer Science'
        WHEN 'Math' THEN 'Mathematics'
        WHEN 'EE' THEN 'Electrical Engineering'
        ELSE 'Other'
    END AS full_dept_name
FROM students;

-- Searched CASE (arbitrary conditions)
SELECT name, gpa,
    CASE
        WHEN gpa >= 3.7 THEN 'A'
        WHEN gpa >= 3.3 THEN 'B+'
        WHEN gpa >= 3.0 THEN 'B'
        WHEN gpa >= 2.7 THEN 'C+'
        ELSE 'C or below'
    END AS letter_grade
FROM students;

-- CASE in aggregation (pivot-style)
SELECT dept,
    COUNT(CASE WHEN gpa >= 3.5 THEN 1 END) AS high_gpa,
    COUNT(CASE WHEN gpa < 3.5 THEN 1 END) AS low_gpa
FROM students
GROUP BY dept;
```

### COALESCE and NULLIF

```sql
-- COALESCE: return the first non-NULL value
SELECT name, COALESCE(phone, email, 'No contact') AS contact
FROM students;

-- NULLIF: return NULL if two values are equal (avoid division by zero)
SELECT department, total_revenue / NULLIF(total_orders, 0) AS avg_order_value
FROM dept_stats;
-- If total_orders = 0, NULLIF returns NULL, so division yields NULL instead of error
```

---

## 13. NULL Handling

NULL is the trickiest concept in SQL. Here's a comprehensive reference.

### NULL Behavior Summary

| Operation | Result | Why |
|---|---|---|
| `5 + NULL` | NULL | Arithmetic with NULL = NULL |
| `'hello' \|\| NULL` | NULL (PG) | String concat with NULL = NULL |
| `NULL = NULL` | UNKNOWN | Can't compare unknowns |
| `NULL != NULL` | UNKNOWN | Can't compare unknowns |
| `NULL > 5` | UNKNOWN | Can't compare unknown to value |
| `NULL AND TRUE` | UNKNOWN | Three-valued logic |
| `NULL OR TRUE` | TRUE | TRUE dominates in OR |
| `NULL AND FALSE` | FALSE | FALSE dominates in AND |
| `COUNT(*)` | Counts all rows | Includes rows where columns are NULL |
| `COUNT(col)` | Counts non-NULL | Skips NULLs |
| `SUM(col)` | Ignores NULLs | If all NULL → returns NULL |
| `AVG(col)` | Ignores NULLs | Average of non-NULL values only |

### Handling NULLs in Queries

```sql
-- Replace NULL with a default (display purposes)
SELECT name, COALESCE(phone, 'N/A') AS phone FROM students;

-- Filter NULLs
SELECT * FROM students WHERE phone IS NOT NULL;

-- Sort NULLs to the end
SELECT * FROM students ORDER BY gpa DESC NULLS LAST;

-- Count NULLs
SELECT COUNT(*) - COUNT(phone) AS null_phone_count FROM students;

-- Treat NULL as 0 in calculations
SELECT name, COALESCE(bonus, 0) + salary AS total_pay FROM employees;
```

---

## 14. SQL Query Execution Order

**This is one of the most important concepts in SQL.** The logical order of execution is NOT the order you write it.

### Written Order vs. Execution Order

```
Written Order:           Execution Order:
─────────────           ────────────────
1. SELECT               1. FROM / JOIN        ← which tables
2. FROM                 2. WHERE              ← filter rows
3. WHERE                3. GROUP BY           ← form groups
4. GROUP BY             4. HAVING             ← filter groups
5. HAVING               5. SELECT             ← compute columns
6. ORDER BY             6. DISTINCT           ← remove duplicates
7. LIMIT                7. ORDER BY           ← sort
                        8. LIMIT / OFFSET     ← paginate
```

### SVG: SQL Execution Pipeline

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 460" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="460" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">SQL Logical Execution Order</text>

  <!-- Step 1: FROM -->
  <rect x="210" y="50" width="320" height="40" rx="8" fill="#2980b9"/>
  <text x="240" y="75" fill="white" font-size="12" font-weight="bold">1. FROM / JOIN</text>
  <text x="440" y="75" fill="#d6eaf8" font-size="11">Identify tables, perform joins</text>
  <line x1="370" y1="90" x2="370" y2="108" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,106 376,106 370,114" fill="#7f8c8d"/>

  <!-- Step 2: WHERE -->
  <rect x="210" y="115" width="320" height="40" rx="8" fill="#27ae60"/>
  <text x="240" y="140" fill="white" font-size="12" font-weight="bold">2. WHERE</text>
  <text x="440" y="140" fill="#d5f5e3" font-size="11">Filter individual rows</text>
  <line x1="370" y1="155" x2="370" y2="173" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,171 376,171 370,179" fill="#7f8c8d"/>

  <!-- Step 3: GROUP BY -->
  <rect x="210" y="180" width="320" height="40" rx="8" fill="#e67e22"/>
  <text x="240" y="205" fill="white" font-size="12" font-weight="bold">3. GROUP BY</text>
  <text x="440" y="205" fill="#fef9e7" font-size="11">Form groups</text>
  <line x1="370" y1="220" x2="370" y2="238" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,236 376,236 370,244" fill="#7f8c8d"/>

  <!-- Step 4: HAVING -->
  <rect x="210" y="245" width="320" height="40" rx="8" fill="#e74c3c"/>
  <text x="240" y="270" fill="white" font-size="12" font-weight="bold">4. HAVING</text>
  <text x="440" y="270" fill="#fadbd8" font-size="11">Filter groups</text>
  <line x1="370" y1="285" x2="370" y2="303" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,301 376,301 370,309" fill="#7f8c8d"/>

  <!-- Step 5: SELECT -->
  <rect x="210" y="310" width="320" height="40" rx="8" fill="#8e44ad"/>
  <text x="240" y="335" fill="white" font-size="12" font-weight="bold">5. SELECT / DISTINCT</text>
  <text x="440" y="335" fill="#e8daef" font-size="11">Compute expressions, dedup</text>
  <line x1="370" y1="350" x2="370" y2="368" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,366 376,366 370,374" fill="#7f8c8d"/>

  <!-- Step 6: ORDER BY -->
  <rect x="210" y="375" width="320" height="40" rx="8" fill="#1abc9c"/>
  <text x="240" y="400" fill="white" font-size="12" font-weight="bold">6. ORDER BY</text>
  <text x="440" y="400" fill="#d1f2eb" font-size="11">Sort results</text>
  <line x1="370" y1="415" x2="370" y2="428" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,426 376,426 370,434" fill="#7f8c8d"/>

  <!-- Step 7: LIMIT -->
  <rect x="210" y="435" width="320" height="20" rx="5" fill="#34495e"/>
  <text x="370" y="450" text-anchor="middle" fill="white" font-size="11" font-weight="bold">7. LIMIT / OFFSET</text>

  <!-- Why this matters note -->
  <rect x="560" y="115" width="160" height="110" rx="6" fill="#fff" stroke="#e74c3c" stroke-width="1.5"/>
  <text x="640" y="135" text-anchor="middle" font-size="10" font-weight="bold" fill="#e74c3c">Why it matters:</text>
  <text x="575" y="155" font-size="9" fill="#555">- Can't use column</text>
  <text x="575" y="170" font-size="9" fill="#555">  alias in WHERE</text>
  <text x="575" y="185" font-size="9" fill="#555">- Can use alias in</text>
  <text x="575" y="200" font-size="9" fill="#555">  ORDER BY</text>
  <text x="575" y="215" font-size="9" fill="#555">- HAVING can use</text>
  <text x="575" y="230" font-size="9" fill="#555">  aggregates</text>
</svg>

### Why Execution Order Matters

```sql
-- This FAILS because WHERE runs before SELECT (alias not yet defined):
SELECT name, gpa * 25 AS score FROM students WHERE score > 80;
-- ERROR: column "score" does not exist

-- FIX: repeat the expression in WHERE:
SELECT name, gpa * 25 AS score FROM students WHERE gpa * 25 > 80;

-- Or use a subquery/CTE:
SELECT * FROM (
    SELECT name, gpa * 25 AS score FROM students
) sub WHERE score > 80;

-- ORDER BY runs AFTER SELECT, so aliases work there:
SELECT name, gpa * 25 AS score FROM students ORDER BY score DESC;  -- WORKS!
```

---

## 15. 25+ Worked Query Examples

### Setup: Sample Database

```sql
-- Create tables
CREATE TABLE departments (
    dept_id    CHAR(4) PRIMARY KEY,
    dept_name  VARCHAR(100) NOT NULL,
    budget     DECIMAL(12,2)
);

CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    dept_id    CHAR(4) REFERENCES departments(dept_id),
    gpa        DECIMAL(3,2),
    birth_date DATE,
    email      VARCHAR(200) UNIQUE,
    phone      VARCHAR(20),
    enrolled   DATE DEFAULT CURRENT_DATE
);

CREATE TABLE courses (
    course_id  CHAR(6) PRIMARY KEY,
    title      VARCHAR(200) NOT NULL,
    credits    INT CHECK (credits BETWEEN 1 AND 6),
    dept_id    CHAR(4) REFERENCES departments(dept_id)
);

CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id  CHAR(6) REFERENCES courses(course_id),
    semester   CHAR(6),
    grade      CHAR(2),
    PRIMARY KEY (student_id, course_id, semester)
);

-- Insert sample data
INSERT INTO departments VALUES
    ('CS', 'Computer Science', 500000),
    ('MATH', 'Mathematics', 300000),
    ('EE', 'Electrical Engineering', 450000),
    ('PHY', 'Physics', 350000);

INSERT INTO students (name, dept_id, gpa, birth_date, email, phone) VALUES
    ('Alice Johnson', 'CS', 3.90, '2002-03-15', 'alice@uni.edu', '555-0101'),
    ('Bob Smith', 'MATH', 3.50, '2001-07-22', 'bob@uni.edu', '555-0102'),
    ('Carol Davis', 'CS', 3.70, '2003-01-10', 'carol@uni.edu', NULL),
    ('Dave Wilson', 'EE', 3.20, '2002-11-05', 'dave@uni.edu', '555-0104'),
    ('Eve Brown', 'CS', 3.10, '2001-09-18', 'eve@uni.edu', '555-0105'),
    ('Frank Miller', 'PHY', 3.60, '2003-06-30', 'frank@uni.edu', NULL),
    ('Grace Lee', 'MATH', 3.80, '2002-04-12', 'grace@uni.edu', '555-0107'),
    ('Hank Taylor', 'EE', 2.90, '2001-12-01', 'hank@uni.edu', '555-0108');

INSERT INTO courses VALUES
    ('DB101', 'Databases', 4, 'CS'),
    ('OS202', 'Operating Systems', 3, 'CS'),
    ('ML303', 'Machine Learning', 4, 'CS'),
    ('CA101', 'Calculus I', 4, 'MATH'),
    ('LA201', 'Linear Algebra', 3, 'MATH'),
    ('CE101', 'Circuits', 4, 'EE'),
    ('QM201', 'Quantum Mechanics', 3, 'PHY');

INSERT INTO enrollments VALUES
    (1, 'DB101', '2026S1', 'A'),
    (1, 'OS202', '2026S1', 'B+'),
    (1, 'ML303', '2026S1', 'A-'),
    (2, 'DB101', '2026S1', 'A-'),
    (2, 'CA101', '2026S1', 'B'),
    (3, 'DB101', '2026S1', 'A'),
    (3, 'ML303', '2026S1', 'B+'),
    (4, 'CE101', '2026S1', 'B'),
    (5, 'OS202', '2026S1', 'C+'),
    (5, 'DB101', '2026S1', 'B'),
    (6, 'QM201', '2026S1', 'A'),
    (7, 'CA101', '2026S1', 'A'),
    (7, 'LA201', '2026S1', 'A-'),
    (8, 'CE101', '2026S1', 'C');
```

---

### Query 1: All students, all columns
```sql
SELECT * FROM students;
```

---

### Query 2: Names and GPAs of CS students
```sql
SELECT name, gpa FROM students WHERE dept_id = 'CS';
```
| name | gpa |
|---|---|
| Alice Johnson | 3.90 |
| Carol Davis | 3.70 |
| Eve Brown | 3.10 |

---

### Query 3: Students with GPA between 3.0 and 3.5
```sql
SELECT name, dept_id, gpa
FROM students
WHERE gpa BETWEEN 3.0 AND 3.5
ORDER BY gpa DESC;
```
| name | dept_id | gpa |
|---|---|---|
| Bob Smith | MATH | 3.50 |
| Dave Wilson | EE | 3.20 |
| Eve Brown | CS | 3.10 |

---

### Query 4: Students whose name starts with 'A' or 'B'
```sql
SELECT name, email
FROM students
WHERE name LIKE 'A%' OR name LIKE 'B%';
```
| name | email |
|---|---|
| Alice Johnson | alice@uni.edu |
| Bob Smith | bob@uni.edu |

---

### Query 5: Students without a phone number
```sql
SELECT name, email
FROM students
WHERE phone IS NULL;
```
| name | email |
|---|---|
| Carol Davis | carol@uni.edu |
| Frank Miller | frank@uni.edu |

---

### Query 6: Count of students per department
```sql
SELECT d.dept_name, COUNT(s.student_id) AS num_students
FROM departments d
LEFT JOIN students s ON d.dept_id = s.dept_id
GROUP BY d.dept_name
ORDER BY num_students DESC;
```
| dept_name | num_students |
|---|---|
| Computer Science | 3 |
| Mathematics | 2 |
| Electrical Engineering | 2 |
| Physics | 1 |

---

### Query 7: Average GPA by department, only depts with avg > 3.3
```sql
SELECT d.dept_name, ROUND(AVG(s.gpa), 2) AS avg_gpa
FROM students s
JOIN departments d ON s.dept_id = d.dept_id
GROUP BY d.dept_name
HAVING AVG(s.gpa) > 3.3
ORDER BY avg_gpa DESC;
```
| dept_name | avg_gpa |
|---|---|
| Mathematics | 3.65 |
| Computer Science | 3.57 |
| Physics | 3.60 |

---

### Query 8: Top 3 students by GPA
```sql
SELECT name, gpa
FROM students
ORDER BY gpa DESC
LIMIT 3;
```
| name | gpa |
|---|---|
| Alice Johnson | 3.90 |
| Grace Lee | 3.80 |
| Carol Davis | 3.70 |

---

### Query 9: Student with the highest GPA (tie-safe)
```sql
SELECT name, gpa
FROM students
WHERE gpa = (SELECT MAX(gpa) FROM students);
```
| name | gpa |
|---|---|
| Alice Johnson | 3.90 |

---

### Query 10: Students enrolled in 'Databases'
```sql
SELECT s.name, e.grade
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
WHERE c.title = 'Databases'
ORDER BY e.grade;
```
| name | grade |
|---|---|
| Alice Johnson | A |
| Carol Davis | A |
| Bob Smith | A- |
| Eve Brown | B |

---

### Query 11: Number of courses each student is taking
```sql
SELECT s.name, COUNT(e.course_id) AS courses_taken
FROM students s
LEFT JOIN enrollments e ON s.student_id = e.student_id
GROUP BY s.student_id, s.name
ORDER BY courses_taken DESC;
```
| name | courses_taken |
|---|---|
| Alice Johnson | 3 |
| Bob Smith | 2 |
| Carol Davis | 2 |
| Grace Lee | 2 |
| Eve Brown | 2 |
| Dave Wilson | 1 |
| Frank Miller | 1 |
| Hank Taylor | 1 |

---

### Query 12: Students enrolled in more than 2 courses
```sql
SELECT s.name, COUNT(*) AS course_count
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
GROUP BY s.student_id, s.name
HAVING COUNT(*) > 2;
```
| name | course_count |
|---|---|
| Alice Johnson | 3 |

---

### Query 13: Students NOT enrolled in any course
```sql
SELECT s.name, s.dept_id
FROM students s
LEFT JOIN enrollments e ON s.student_id = e.student_id
WHERE e.student_id IS NULL;
```
*(Empty result — all students are enrolled in at least one course)*

Alternative with NOT EXISTS:
```sql
SELECT name, dept_id FROM students s
WHERE NOT EXISTS (SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id);
```

---

### Query 14: Courses with their enrollment count
```sql
SELECT c.title, c.credits, COUNT(e.student_id) AS enrolled
FROM courses c
LEFT JOIN enrollments e ON c.course_id = e.course_id
GROUP BY c.course_id, c.title, c.credits
ORDER BY enrolled DESC;
```
| title | credits | enrolled |
|---|---|---|
| Databases | 4 | 4 |
| Operating Systems | 3 | 2 |
| Machine Learning | 4 | 2 |
| Circuits | 4 | 2 |
| Calculus I | 4 | 2 |
| Linear Algebra | 3 | 1 |
| Quantum Mechanics | 3 | 1 |

---

### Query 15: GPA classification with CASE
```sql
SELECT name, gpa,
    CASE
        WHEN gpa >= 3.7 THEN 'Distinction'
        WHEN gpa >= 3.3 THEN 'First Class'
        WHEN gpa >= 3.0 THEN 'Second Class'
        WHEN gpa >= 2.0 THEN 'Pass'
        ELSE 'Fail'
    END AS classification
FROM students
ORDER BY gpa DESC;
```
| name | gpa | classification |
|---|---|---|
| Alice Johnson | 3.90 | Distinction |
| Grace Lee | 3.80 | Distinction |
| Carol Davis | 3.70 | Distinction |
| Frank Miller | 3.60 | First Class |
| Bob Smith | 3.50 | First Class |
| Dave Wilson | 3.20 | Second Class |
| Eve Brown | 3.10 | Second Class |
| Hank Taylor | 2.90 | Pass |

---

### Query 16: Department with the highest budget
```sql
SELECT dept_name, budget
FROM departments
ORDER BY budget DESC
LIMIT 1;
```
| dept_name | budget |
|---|---|
| Computer Science | 500000.00 |

---

### Query 17: Students older than 23 (derived attribute)
```sql
SELECT name, birth_date,
    EXTRACT(YEAR FROM AGE(birth_date)) AS age
FROM students
WHERE EXTRACT(YEAR FROM AGE(birth_date)) > 23
ORDER BY birth_date;
```

---

### Query 18: Courses offered by CS department that nobody enrolled in
```sql
SELECT c.title
FROM courses c
WHERE c.dept_id = 'CS'
  AND c.course_id NOT IN (SELECT course_id FROM enrollments);
```
*(Empty — all CS courses have enrollments in our data)*

---

### Query 19: Students with above-average GPA in their department
```sql
SELECT s.name, s.dept_id, s.gpa, dept_avg.avg_gpa
FROM students s
JOIN (
    SELECT dept_id, AVG(gpa) AS avg_gpa
    FROM students
    GROUP BY dept_id
) dept_avg ON s.dept_id = dept_avg.dept_id
WHERE s.gpa > dept_avg.avg_gpa
ORDER BY s.dept_id, s.gpa DESC;
```
| name | dept_id | gpa | avg_gpa |
|---|---|---|---|
| Alice Johnson | CS | 3.90 | 3.567 |
| Carol Davis | CS | 3.70 | 3.567 |
| Dave Wilson | EE | 3.20 | 3.050 |
| Grace Lee | MATH | 3.80 | 3.650 |

---

### Query 20: Pivot — grade distribution for Databases
```sql
SELECT
    COUNT(CASE WHEN grade IN ('A', 'A+') THEN 1 END) AS grade_A,
    COUNT(CASE WHEN grade IN ('A-') THEN 1 END) AS grade_A_minus,
    COUNT(CASE WHEN grade IN ('B+', 'B') THEN 1 END) AS grade_B,
    COUNT(CASE WHEN grade IN ('B-', 'C+', 'C') THEN 1 END) AS grade_C_or_below,
    COUNT(*) AS total
FROM enrollments
WHERE course_id = 'DB101';
```
| grade_A | grade_A_minus | grade_B | grade_C_or_below | total |
|---|---|---|---|---|
| 2 | 1 | 1 | 0 | 4 |

---

### Query 21: Student contact info with NULL handling
```sql
SELECT
    name,
    COALESCE(phone, 'No phone') AS phone,
    email,
    COALESCE(phone, email) AS primary_contact
FROM students
ORDER BY name;
```

---

### Query 22: Departments and budget per student
```sql
SELECT
    d.dept_name,
    d.budget,
    COUNT(s.student_id) AS students,
    ROUND(d.budget / NULLIF(COUNT(s.student_id), 0), 2) AS budget_per_student
FROM departments d
LEFT JOIN students s ON d.dept_id = s.dept_id
GROUP BY d.dept_id, d.dept_name, d.budget
ORDER BY budget_per_student DESC;
```

---

### Query 23: Students and their complete course list (string aggregation)
```sql
SELECT
    s.name,
    STRING_AGG(c.title, ', ' ORDER BY c.title) AS courses
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
GROUP BY s.student_id, s.name
ORDER BY s.name;
```
| name | courses |
|---|---|
| Alice Johnson | Databases, Machine Learning, Operating Systems |
| Bob Smith | Calculus I, Databases |
| Carol Davis | Databases, Machine Learning |
| ... | ... |

---

### Query 24: Insert + Update + Delete in a transaction
```sql
BEGIN;

-- Insert a new student
INSERT INTO students (name, dept_id, gpa, birth_date, email)
VALUES ('Iris Zhang', 'CS', 3.85, '2003-08-20', 'iris@uni.edu');

-- Enroll them in Databases
INSERT INTO enrollments (student_id, course_id, semester)
VALUES (CURRVAL('students_student_id_seq'), 'DB101', '2026S1');

-- Update GPA of all CS students by 0.01 (curve)
UPDATE students SET gpa = LEAST(gpa + 0.01, 4.0) WHERE dept_id = 'CS';

-- Delete old enrollments
DELETE FROM enrollments WHERE semester < '2025S1';

COMMIT;
```

---

### Query 25: UPSERT — insert or update on conflict
```sql
INSERT INTO students (student_id, name, dept_id, gpa, email)
VALUES (1, 'Alice Johnson', 'CS', 3.95, 'alice@uni.edu')
ON CONFLICT (student_id)
DO UPDATE SET
    gpa = EXCLUDED.gpa,
    name = EXCLUDED.name;
```

---

### Query 26 (Bonus): Department report — multi-level aggregation
```sql
SELECT
    d.dept_name,
    COUNT(DISTINCT s.student_id) AS total_students,
    COUNT(DISTINCT e.course_id) AS courses_with_enrollment,
    ROUND(AVG(s.gpa), 2) AS avg_gpa,
    MIN(s.gpa) AS min_gpa,
    MAX(s.gpa) AS max_gpa,
    SUM(CASE WHEN s.gpa >= 3.5 THEN 1 ELSE 0 END) AS honor_students,
    ROUND(100.0 * SUM(CASE WHEN s.gpa >= 3.5 THEN 1 ELSE 0 END) / COUNT(DISTINCT s.student_id), 1)
        AS honor_pct
FROM departments d
LEFT JOIN students s ON d.dept_id = s.dept_id
LEFT JOIN enrollments e ON s.student_id = e.student_id
GROUP BY d.dept_id, d.dept_name
ORDER BY avg_gpa DESC NULLS LAST;
```

---

## 16. Common Interview Questions

### Q1: What is the difference between DDL and DML?
> **DDL** (CREATE, ALTER, DROP, TRUNCATE) defines structure and auto-commits. **DML** (INSERT, UPDATE, DELETE) modifies data and can be rolled back inside a transaction. DDL changes schema; DML changes data.

### Q2: What is the difference between DELETE and TRUNCATE?
> | Feature | DELETE | TRUNCATE |
> |---|---|---|
> | Type | DML | DDL |
> | WHERE clause? | Yes (selective) | No (all rows) |
> | Rollback? | Yes | Usually no (auto-commits) |
> | Speed | Slower (logs each row) | Faster (deallocates pages) |
> | Triggers fire? | Yes | No |
> | Resets identity? | No | Yes (with RESTART) |

### Q3: What is the difference between WHERE and HAVING?
> `WHERE` filters **individual rows** before grouping. `HAVING` filters **groups** after GROUP BY. `WHERE` cannot use aggregate functions; `HAVING` can. Example: `WHERE gpa > 3.0` filters students; `HAVING AVG(gpa) > 3.0` filters departments.

### Q4: Explain the SQL execution order.
> FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT. This is why you can't use a column alias in WHERE (SELECT hasn't run yet), but you can in ORDER BY (runs after SELECT).

### Q5: What is the difference between CHAR and VARCHAR?
> `CHAR(n)` stores fixed-length strings, padded with spaces (always uses n bytes). `VARCHAR(n)` stores variable-length strings (uses only as many bytes as needed). Use CHAR for fixed-length codes (state, country); use VARCHAR for everything else.

### Q6: Why should you never use FLOAT for money?
> Floating-point numbers use binary representation and can't exactly represent some decimal values. `0.1 + 0.2` might give `0.30000000000000004`. Use `DECIMAL(p,s)` or `NUMERIC(p,s)` for money — they store exact decimal values.

### Q7: What does COALESCE do? Give an example.
> `COALESCE(a, b, c, ...)` returns the first non-NULL value from its arguments. Example: `COALESCE(phone, email, 'No contact')` returns phone if not null, otherwise email, otherwise 'No contact'. It's essential for NULL handling.

### Q8: What is an UPSERT? How do you do it in PostgreSQL?
> UPSERT = INSERT or UPDATE. If the row exists, update it; if not, insert it. In PostgreSQL: `INSERT INTO t (...) VALUES (...) ON CONFLICT (pk) DO UPDATE SET col = EXCLUDED.col;`. The SQL standard uses `MERGE`.

### Q9: Can you use an alias in WHERE? Why or why not?
> No. WHERE executes before SELECT in the logical execution order, so aliases defined in SELECT don't exist yet during WHERE. Workaround: use a subquery or CTE, or repeat the expression.

### Q10: What is the difference between COUNT(*) and COUNT(column)?
> `COUNT(*)` counts all rows, including those with NULLs. `COUNT(column)` counts only rows where that column is NOT NULL. `COUNT(DISTINCT column)` counts unique non-NULL values.

### Q11: What is MERGE and when would you use it?
> `MERGE` (SQL:2003) performs conditional INSERT, UPDATE, or DELETE in one statement based on whether source rows match target rows. Use it for: syncing tables, loading data warehouses, upserting bulk data. It's more expressive than `INSERT ... ON CONFLICT`.

### Q12: Explain three-valued logic in SQL.
> Because of NULL, SQL comparisons can yield TRUE, FALSE, or UNKNOWN. `NULL = NULL` is UNKNOWN, not TRUE. WHERE returns only rows where the condition is TRUE — rows with UNKNOWN are excluded. This is why you must use `IS NULL` instead of `= NULL`.

---

## 17. Key Takeaways

| # | Concept | One-Line Summary |
|---|---|---|
| 1 | **SQL** | Declarative language — describe WHAT, not HOW |
| 2 | **DDL** | CREATE, ALTER, DROP, TRUNCATE — defines structure, auto-commits |
| 3 | **DML** | INSERT, UPDATE, DELETE, MERGE — modifies data, can rollback |
| 4 | **DQL** | SELECT — retrieves data (read-only) |
| 5 | **DECIMAL for money** | Never use FLOAT/DOUBLE for financial data |
| 6 | **VARCHAR vs CHAR** | VARCHAR for variable text, CHAR for fixed-length codes |
| 7 | **TIMESTAMP WITH TZ** | Always store timestamps with timezone for global apps |
| 8 | **Constraints** | PK, FK, UNIQUE, NOT NULL, CHECK, DEFAULT — enforce integrity |
| 9 | **ON CONFLICT** | PostgreSQL UPSERT — insert or update in one statement |
| 10 | **WHERE** | Filters rows BEFORE grouping (no aggregates allowed) |
| 11 | **HAVING** | Filters groups AFTER GROUP BY (aggregates allowed) |
| 12 | **Execution order** | FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT |
| 13 | **NULL** | Not a value; creates 3-valued logic; use IS NULL, not = NULL |
| 14 | **COALESCE** | First non-NULL value — essential NULL handler |
| 15 | **CASE** | SQL's if-else — works in SELECT, WHERE, ORDER BY, aggregates |
| 16 | **GROUP BY rule** | Every non-aggregated SELECT column must be in GROUP BY |
| 17 | **COUNT(*) vs COUNT(col)** | * counts all rows; column skips NULLs |
| 18 | **LEFT JOIN + IS NULL** | Find rows with no match (orphan detection) |
| 19 | **RETURNING** | PostgreSQL: get back affected rows from INSERT/UPDATE/DELETE |
| 20 | **DELETE vs TRUNCATE** | DELETE is DML (slow, rollback); TRUNCATE is DDL (fast, no rollback) |

---

<p style="text-align: center; margin-top: 40px;">
  <a href="04_sql_fundamentals.md" download="04_sql_fundamentals.md"
     style="display: inline-block; padding: 14px 36px; font-size: 16px; font-weight: bold;
            color: #fff; background-color: #27ae60; border-radius: 8px;
            text-decoration: none; box-shadow: 0 4px 6px rgba(0,0,0,0.15);">
    Download 04_sql_fundamentals.md
  </a>
</p>

---

*Ready? Say **"next"** to proceed to **Phase 5 — Keys & Constraints**.*
