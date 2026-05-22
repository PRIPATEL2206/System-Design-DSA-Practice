# Phase 5 — Keys & Constraints

> **Goal**: Master every type of key and constraint in relational databases — the rules that keep your data valid, consistent, and trustworthy. Understand not just the syntax but *how* the DBMS enforces them internally, when to use each, and the performance implications.

---

## Table of Contents

1. [Why Constraints Matter](#1-why-constraints-matter)
2. [Primary Key (PK)](#2-primary-key-pk)
3. [Foreign Key (FK)](#3-foreign-key-fk)
4. [UNIQUE Constraint](#4-unique-constraint)
5. [NOT NULL Constraint](#5-not-null-constraint)
6. [CHECK Constraint](#6-check-constraint)
7. [DEFAULT Constraint](#7-default-constraint)
8. [Composite Keys](#8-composite-keys)
9. [Surrogate Keys vs. Natural Keys](#9-surrogate-keys-vs-natural-keys)
10. [Candidate Keys and Alternate Keys](#10-candidate-keys-and-alternate-keys)
11. [Referential Integrity — Deep Dive](#11-referential-integrity--deep-dive)
12. [Constraint Enforcement Internals](#12-constraint-enforcement-internals)
13. [Deferrable Constraints](#13-deferrable-constraints)
14. [EXCLUDE Constraints (PostgreSQL)](#14-exclude-constraints-postgresql)
15. [Naming and Managing Constraints](#15-naming-and-managing-constraints)
16. [Best Practices and Anti-Patterns](#16-best-practices-and-anti-patterns)
17. [Worked Examples — Beginner, Intermediate, Advanced](#17-worked-examples--beginner-intermediate-advanced)
18. [Common Interview Questions](#18-common-interview-questions)
19. [Key Takeaways](#19-key-takeaways)

---

## 1. Why Constraints Matter

### The Problem Without Constraints

Without constraints, your database accepts **anything** — garbage data, orphaned records, duplicates, invalid values. Every application that writes to the DB must independently validate data, and if even one app forgets, the entire system corrupts.

```
Without constraints:
  INSERT INTO students VALUES (NULL, NULL, NULL, 99.9);   ← null ID, GPA of 99.9?!
  INSERT INTO enrollments VALUES (999, 'XX99', 'A');      ← student 999 doesn't exist!
  INSERT INTO students VALUES (1, 'Alice', 'CS', 3.9);
  INSERT INTO students VALUES (1, 'Bob', 'Math', 3.5);   ← duplicate PK!
```

### Constraints = Rules at the Database Level

Constraints enforce rules **inside the database engine itself** — no application can bypass them. They are the database's immune system.

| Constraint | What It Prevents | Real-World Analogy |
|---|---|---|
| **PRIMARY KEY** | Duplicate/null identifiers | No two citizens share a passport number |
| **FOREIGN KEY** | Orphaned references | Can't reference a department that doesn't exist |
| **UNIQUE** | Duplicate values | No two students share an email address |
| **NOT NULL** | Missing required data | A birth certificate must have a name |
| **CHECK** | Invalid values | Age can't be negative, GPA can't exceed 4.0 |
| **DEFAULT** | Forgotten values | If no status given, default to 'pending' |

### SVG: Constraint Types Overview

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 310" font-family="Segoe UI, Arial, sans-serif">
  <rect width="760" height="310" fill="#f8f9fa" rx="12"/>
  <text x="380" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Database Constraints</text>

  <!-- PK -->
  <rect x="20" y="50" width="115" height="240" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="77" y="75" text-anchor="middle" font-size="12" font-weight="bold" fill="#2980b9">PRIMARY</text>
  <text x="77" y="90" text-anchor="middle" font-size="12" font-weight="bold" fill="#2980b9">KEY</text>
  <text x="77" y="115" text-anchor="middle" font-size="10" fill="#555">Unique + NOT</text>
  <text x="77" y="130" text-anchor="middle" font-size="10" fill="#555">NULL identity</text>
  <rect x="30" y="145" width="95" height="20" rx="3" fill="#d6eaf8"/>
  <text x="77" y="159" text-anchor="middle" font-size="9" fill="#1a5276">One per table</text>
  <rect x="30" y="170" width="95" height="20" rx="3" fill="#d6eaf8"/>
  <text x="77" y="184" text-anchor="middle" font-size="9" fill="#1a5276">Auto-indexed</text>
  <rect x="30" y="195" width="95" height="20" rx="3" fill="#d6eaf8"/>
  <text x="77" y="209" text-anchor="middle" font-size="9" fill="#1a5276">Can be composite</text>
  <text x="77" y="250" text-anchor="middle" font-size="9" fill="#2980b9" font-style="italic">student_id</text>
  <text x="77" y="265" text-anchor="middle" font-size="9" fill="#2980b9" font-style="italic">order_id</text>

  <!-- FK -->
  <rect x="145" y="50" width="115" height="240" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="202" y="75" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">FOREIGN</text>
  <text x="202" y="90" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">KEY</text>
  <text x="202" y="115" text-anchor="middle" font-size="10" fill="#555">References PK</text>
  <text x="202" y="130" text-anchor="middle" font-size="10" fill="#555">of another table</text>
  <rect x="155" y="145" width="95" height="20" rx="3" fill="#d5f5e3"/>
  <text x="202" y="159" text-anchor="middle" font-size="9" fill="#145a32">CASCADE rules</text>
  <rect x="155" y="170" width="95" height="20" rx="3" fill="#d5f5e3"/>
  <text x="202" y="184" text-anchor="middle" font-size="9" fill="#145a32">Many per table</text>
  <rect x="155" y="195" width="95" height="20" rx="3" fill="#d5f5e3"/>
  <text x="202" y="209" text-anchor="middle" font-size="9" fill="#145a32">NULL allowed</text>
  <text x="202" y="250" text-anchor="middle" font-size="9" fill="#27ae60" font-style="italic">dept_id → depts</text>
  <text x="202" y="265" text-anchor="middle" font-size="9" fill="#27ae60" font-style="italic">mgr_id → emps</text>

  <!-- UNIQUE -->
  <rect x="270" y="50" width="115" height="240" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="327" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#e67e22">UNIQUE</text>
  <text x="327" y="115" text-anchor="middle" font-size="10" fill="#555">No duplicate</text>
  <text x="327" y="130" text-anchor="middle" font-size="10" fill="#555">values</text>
  <rect x="280" y="145" width="95" height="20" rx="3" fill="#fdebd0"/>
  <text x="327" y="159" text-anchor="middle" font-size="9" fill="#784212">Allows ONE null</text>
  <rect x="280" y="170" width="95" height="20" rx="3" fill="#fdebd0"/>
  <text x="327" y="184" text-anchor="middle" font-size="9" fill="#784212">Many per table</text>
  <rect x="280" y="195" width="95" height="20" rx="3" fill="#fdebd0"/>
  <text x="327" y="209" text-anchor="middle" font-size="9" fill="#784212">Auto-indexed</text>
  <text x="327" y="250" text-anchor="middle" font-size="9" fill="#e67e22" font-style="italic">email</text>
  <text x="327" y="265" text-anchor="middle" font-size="9" fill="#e67e22" font-style="italic">ssn, license_no</text>

  <!-- NOT NULL -->
  <rect x="395" y="50" width="115" height="240" rx="8" fill="#f4ecf7" stroke="#8e44ad" stroke-width="2"/>
  <text x="452" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#8e44ad">NOT NULL</text>
  <text x="452" y="115" text-anchor="middle" font-size="10" fill="#555">Value must be</text>
  <text x="452" y="130" text-anchor="middle" font-size="10" fill="#555">provided</text>
  <rect x="405" y="145" width="95" height="20" rx="3" fill="#e8daef"/>
  <text x="452" y="159" text-anchor="middle" font-size="9" fill="#4a235a">Mandatory field</text>
  <rect x="405" y="170" width="95" height="20" rx="3" fill="#e8daef"/>
  <text x="452" y="184" text-anchor="middle" font-size="9" fill="#4a235a">Many per table</text>
  <rect x="405" y="195" width="95" height="20" rx="3" fill="#e8daef"/>
  <text x="452" y="209" text-anchor="middle" font-size="9" fill="#4a235a">Total participation</text>
  <text x="452" y="250" text-anchor="middle" font-size="9" fill="#8e44ad" font-style="italic">name NOT NULL</text>
  <text x="452" y="265" text-anchor="middle" font-size="9" fill="#8e44ad" font-style="italic">email NOT NULL</text>

  <!-- CHECK -->
  <rect x="520" y="50" width="115" height="240" rx="8" fill="#fce4ec" stroke="#c0392b" stroke-width="2"/>
  <text x="577" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#c0392b">CHECK</text>
  <text x="577" y="115" text-anchor="middle" font-size="10" fill="#555">Value must pass</text>
  <text x="577" y="130" text-anchor="middle" font-size="10" fill="#555">a boolean test</text>
  <rect x="530" y="145" width="95" height="20" rx="3" fill="#fadbd8"/>
  <text x="577" y="159" text-anchor="middle" font-size="9" fill="#922b21">Custom rules</text>
  <rect x="530" y="170" width="95" height="20" rx="3" fill="#fadbd8"/>
  <text x="577" y="184" text-anchor="middle" font-size="9" fill="#922b21">Many per table</text>
  <rect x="530" y="195" width="95" height="20" rx="3" fill="#fadbd8"/>
  <text x="577" y="209" text-anchor="middle" font-size="9" fill="#922b21">No subqueries</text>
  <text x="577" y="250" text-anchor="middle" font-size="9" fill="#c0392b" font-style="italic">gpa BETWEEN 0,4</text>
  <text x="577" y="265" text-anchor="middle" font-size="9" fill="#c0392b" font-style="italic">salary > 0</text>

  <!-- DEFAULT -->
  <rect x="645" y="50" width="100" height="240" rx="8" fill="#e8f8f5" stroke="#1abc9c" stroke-width="2"/>
  <text x="695" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#16a085">DEFAULT</text>
  <text x="695" y="115" text-anchor="middle" font-size="10" fill="#555">Fallback value</text>
  <text x="695" y="130" text-anchor="middle" font-size="10" fill="#555">if omitted</text>
  <rect x="652" y="145" width="86" height="20" rx="3" fill="#d1f2eb"/>
  <text x="695" y="159" text-anchor="middle" font-size="9" fill="#0e6655">Constant/func</text>
  <rect x="652" y="170" width="86" height="20" rx="3" fill="#d1f2eb"/>
  <text x="695" y="184" text-anchor="middle" font-size="9" fill="#0e6655">Many per table</text>
  <rect x="652" y="195" width="86" height="20" rx="3" fill="#d1f2eb"/>
  <text x="695" y="209" text-anchor="middle" font-size="9" fill="#0e6655">Not enforced</text>
  <text x="695" y="250" text-anchor="middle" font-size="9" fill="#16a085" font-style="italic">status='pending'</text>
  <text x="695" y="265" text-anchor="middle" font-size="9" fill="#16a085" font-style="italic">created=NOW()</text>
</svg>

---

## 2. Primary Key (PK)

### Definition

A **primary key** is a column (or set of columns) that **uniquely identifies** each row in a table. It is the main identifier for the table.

### Rules

| Rule | Description |
|---|---|
| **Unique** | No two rows can have the same PK value |
| **NOT NULL** | PK columns cannot contain NULL |
| **One per table** | Each table has exactly one primary key |
| **Immutable** (best practice) | PK values should never change |
| **Auto-indexed** | The DBMS automatically creates a unique index on the PK |

### Syntax

```sql
-- Column-level (single column PK)
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100)
);

-- Table-level (required for composite PKs)
CREATE TABLE enrollments (
    student_id  INT,
    course_id   CHAR(6),
    semester    CHAR(6),
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id, semester)
);

-- With explicit constraint name
CREATE TABLE products (
    product_id  INT,
    name        VARCHAR(200),
    CONSTRAINT pk_products PRIMARY KEY (product_id)
);

-- Auto-increment PK (PostgreSQL)
CREATE TABLE orders (
    order_id  SERIAL PRIMARY KEY,        -- 4-byte auto-increment
    -- or:
    -- order_id  BIGSERIAL PRIMARY KEY    -- 8-byte for large tables
    -- or (modern PostgreSQL 10+):
    -- order_id  INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
    total     DECIMAL(10,2)
);

-- UUID PK
CREATE TABLE sessions (
    session_id  UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id     INT,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

### What Happens Under the Hood?

When you create a PK, the DBMS:
1. Creates a **unique B-Tree index** on the PK column(s)
2. Adds an implicit **NOT NULL** constraint
3. Rejects any INSERT/UPDATE that would violate uniqueness or nullability

```sql
-- These FAIL:
INSERT INTO students (student_id, name) VALUES (1, 'Alice');
INSERT INTO students (student_id, name) VALUES (1, 'Bob');
-- ERROR: duplicate key value violates unique constraint "students_pkey"

INSERT INTO students (student_id, name) VALUES (NULL, 'Eve');
-- ERROR: null value in column "student_id" violates not-null constraint
```

### Single-Column vs. Composite PK

| Type | When to Use | Example |
|---|---|---|
| **Single-column** | The entity has a natural single identifier | `student_id`, `order_id`, `isbn` |
| **Composite** | No single column is unique, but a combination is | `(student_id, course_id, semester)` for enrollment |

---

## 3. Foreign Key (FK)

### Definition

A **foreign key** is a column (or set of columns) in one table that **references the primary key** (or a UNIQUE column) of another table. It creates a link between two tables and enforces **referential integrity**.

### Real-World Analogy

A foreign key is like an **address on a letter** — it must point to a real building. If the building doesn't exist, the letter can't be delivered. If the building is demolished, you need to decide what happens to all the letters addressed there.

### Syntax

```sql
-- Column-level FK
CREATE TABLE students (
    student_id  SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    dept_id     CHAR(4) REFERENCES departments(dept_id)  -- FK to departments
);

-- Table-level FK (more explicit)
CREATE TABLE enrollments (
    student_id  INT,
    course_id   CHAR(6),
    grade       CHAR(2),
    CONSTRAINT fk_student FOREIGN KEY (student_id) REFERENCES students(student_id),
    CONSTRAINT fk_course  FOREIGN KEY (course_id) REFERENCES courses(course_id)
);

-- FK with referential actions
CREATE TABLE enrollments (
    student_id  INT,
    course_id   CHAR(6),
    FOREIGN KEY (student_id) REFERENCES students(student_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);

-- Self-referencing FK (recursive)
CREATE TABLE employees (
    emp_id      SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    manager_id  INT REFERENCES employees(emp_id)  -- points to same table
);

-- Composite FK
CREATE TABLE grades (
    student_id  INT,
    course_id   CHAR(6),
    semester    CHAR(6),
    final_grade CHAR(2),
    FOREIGN KEY (student_id, course_id, semester)
        REFERENCES enrollments(student_id, course_id, semester)
        ON DELETE CASCADE
);
```

### Referential Actions — Complete Guide

When the **referenced** (parent) row is deleted or updated, what happens to the **referencing** (child) rows?

| Action | ON DELETE | ON UPDATE | Use When |
|---|---|---|---|
| **CASCADE** | Delete child rows too | Update child FKs too | Child is meaningless without parent (order items when order deleted) |
| **RESTRICT** | Block the delete | Block the update | Child must always reference a valid parent (can't delete a dept with employees) |
| **SET NULL** | Set child FK to NULL | Set child FK to NULL | Child can exist without parent (set manager_id to NULL when manager leaves) |
| **SET DEFAULT** | Set child FK to default | Set child FK to default | Child has a fallback reference (set dept to 'UNASSIGNED') |
| **NO ACTION** | Like RESTRICT (checked at end of statement) | Like RESTRICT | Default behavior — same as RESTRICT for most purposes |

```sql
-- Demonstration of each action:

-- CASCADE: delete order → auto-delete all its items
CREATE TABLE order_items (
    order_id   INT REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INT REFERENCES products(product_id) ON DELETE RESTRICT
);
DELETE FROM orders WHERE order_id = 100;  -- order_items for order 100 auto-deleted

-- RESTRICT: can't delete a product if any order references it
DELETE FROM products WHERE product_id = 42;
-- ERROR: update or delete on "products" violates foreign key constraint on "order_items"

-- SET NULL: when a manager leaves, employees' manager_id becomes NULL
CREATE TABLE employees (
    emp_id     SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    manager_id INT REFERENCES employees(emp_id) ON DELETE SET NULL
);
DELETE FROM employees WHERE emp_id = 5;  -- employees who reported to emp 5 now have manager_id = NULL

-- SET DEFAULT: when a department is removed, assign employees to default dept
CREATE TABLE employees (
    emp_id  SERIAL PRIMARY KEY,
    dept_id CHAR(4) DEFAULT 'GEN' REFERENCES departments(dept_id) ON DELETE SET DEFAULT
);
DELETE FROM departments WHERE dept_id = 'MKT';  -- employees in MKT now have dept_id = 'GEN'
```

### FK Constraints — What They Check

| Operation | What the DBMS Checks |
|---|---|
| INSERT into child | FK value must exist in parent PK (or be NULL) |
| UPDATE child FK | New FK value must exist in parent PK (or be NULL) |
| DELETE from parent | No child rows reference this parent (unless CASCADE/SET NULL) |
| UPDATE parent PK | No child rows reference old PK value (unless CASCADE/SET NULL) |

### SVG: Foreign Key Referential Actions

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 360" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="360" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Foreign Key Referential Actions</text>

  <!-- Parent table -->
  <rect x="270" y="50" width="200" height="55" rx="8" fill="#2980b9"/>
  <text x="370" y="72" text-anchor="middle" fill="white" font-size="14" font-weight="bold">Parent Table</text>
  <text x="370" y="92" text-anchor="middle" fill="#d6eaf8" font-size="11">departments (PK: dept_id)</text>

  <!-- Arrow down -->
  <line x1="370" y1="105" x2="370" y2="140" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,138 376,138 370,148" fill="#7f8c8d"/>
  <text x="390" y="130" font-size="10" fill="#888">FK references</text>

  <!-- Child table -->
  <rect x="270" y="150" width="200" height="55" rx="8" fill="#27ae60"/>
  <text x="370" y="172" text-anchor="middle" fill="white" font-size="14" font-weight="bold">Child Table</text>
  <text x="370" y="192" text-anchor="middle" fill="#d5f5e3" font-size="11">employees (FK: dept_id)</text>

  <!-- Actions -->
  <text x="370" y="235" text-anchor="middle" font-size="13" font-weight="bold" fill="#2c3e50">When parent row is DELETED:</text>

  <rect x="30" y="255" width="160" height="45" rx="6" fill="#e74c3c"/>
  <text x="110" y="275" text-anchor="middle" fill="white" font-size="12" font-weight="bold">CASCADE</text>
  <text x="110" y="292" text-anchor="middle" fill="#fadbd8" font-size="10">Delete children too</text>

  <rect x="205" y="255" width="160" height="45" rx="6" fill="#e67e22"/>
  <text x="285" y="275" text-anchor="middle" fill="white" font-size="12" font-weight="bold">RESTRICT</text>
  <text x="285" y="292" text-anchor="middle" fill="#fef9e7" font-size="10">Block the delete</text>

  <rect x="380" y="255" width="160" height="45" rx="6" fill="#8e44ad"/>
  <text x="460" y="275" text-anchor="middle" fill="white" font-size="12" font-weight="bold">SET NULL</text>
  <text x="460" y="292" text-anchor="middle" fill="#e8daef" font-size="10">Child FK → NULL</text>

  <rect x="555" y="255" width="160" height="45" rx="6" fill="#1abc9c"/>
  <text x="635" y="275" text-anchor="middle" fill="white" font-size="12" font-weight="bold">SET DEFAULT</text>
  <text x="635" y="292" text-anchor="middle" fill="#d1f2eb" font-size="10">Child FK → default</text>

  <text x="370" y="335" text-anchor="middle" font-size="11" fill="#888">Each FK can independently choose its ON DELETE and ON UPDATE action</text>
</svg>

---

## 4. UNIQUE Constraint

### Definition

Ensures that all values in a column (or combination of columns) are **distinct**. Unlike a PK, a UNIQUE column **can contain NULL** (and multiple NULLs in most databases, since NULL ≠ NULL).

### Syntax

```sql
-- Column-level
CREATE TABLE students (
    student_id  SERIAL PRIMARY KEY,
    email       VARCHAR(200) UNIQUE,           -- no two students share an email
    ssn         CHAR(11) UNIQUE                -- social security number
);

-- Table-level (composite unique)
CREATE TABLE course_schedule (
    room_id     INT,
    time_slot   CHAR(10),
    day_of_week CHAR(3),
    course_id   CHAR(6),
    UNIQUE (room_id, time_slot, day_of_week)   -- no double-booking
);

-- Named constraint
CREATE TABLE products (
    sku   VARCHAR(50),
    name  VARCHAR(200),
    CONSTRAINT uq_products_sku UNIQUE (sku)
);

-- Add UNIQUE to existing table
ALTER TABLE students ADD CONSTRAINT uq_email UNIQUE (email);
```

### PK vs. UNIQUE

| Feature | PRIMARY KEY | UNIQUE |
|---|---|---|
| How many per table? | Exactly ONE | Any number |
| Allows NULL? | No | Yes (one or more NULLs, varies by DBMS) |
| Creates index? | Yes (clustered in some DBMS) | Yes (non-clustered) |
| Purpose | Main row identifier | Enforce uniqueness of alternate keys |
| Can be FK target? | Yes | Yes (a FK can reference a UNIQUE column) |

### NULL Behavior in UNIQUE Constraints

```sql
-- PostgreSQL: multiple NULLs allowed in UNIQUE column
INSERT INTO students (student_id, email) VALUES (1, NULL);
INSERT INTO students (student_id, email) VALUES (2, NULL);
-- Both succeed! NULL ≠ NULL, so no uniqueness violation

-- SQL Server: only ONE null allowed in UNIQUE column (stricter)

-- PostgreSQL 15+: NULLS NOT DISTINCT option
CREATE TABLE t (
    col INT UNIQUE NULLS NOT DISTINCT  -- only one NULL allowed
);
```

---

## 5. NOT NULL Constraint

### Definition

Ensures that a column **cannot contain NULL** — a value must always be provided.

### Syntax

```sql
CREATE TABLE students (
    student_id  INT PRIMARY KEY,          -- implicitly NOT NULL (because PK)
    name        VARCHAR(100) NOT NULL,    -- explicitly NOT NULL
    email       VARCHAR(200) NOT NULL,
    dept_id     CHAR(4),                  -- nullable (default)
    gpa         DECIMAL(3,2)              -- nullable (default)
);

-- Add NOT NULL to existing column
ALTER TABLE students ALTER COLUMN dept_id SET NOT NULL;

-- Remove NOT NULL
ALTER TABLE students ALTER COLUMN dept_id DROP NOT NULL;
```

### When to Use NOT NULL

| Use NOT NULL When | Allow NULL When |
|---|---|
| The field is essential (name, email) | The field is optional (phone, middle_name) |
| Total participation in ER model | Partial participation in ER model |
| The column is part of a FK that must always reference something | The FK is optional (employee may or may not have a manager) |
| Business logic requires a value | The information might not be available yet |

### NOT NULL + DEFAULT Together

A common pattern: make a column NOT NULL but provide a DEFAULT so inserts don't need to specify it.

```sql
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    is_priority BOOLEAN NOT NULL DEFAULT FALSE
);

-- This INSERT works even without specifying status, created_at, or is_priority:
INSERT INTO orders DEFAULT VALUES;
-- Result: status='pending', created_at=NOW(), is_priority=FALSE
```

---

## 6. CHECK Constraint

### Definition

Ensures that values in a column satisfy a **boolean expression**. If the expression evaluates to FALSE, the insert/update is rejected. If it evaluates to NULL or TRUE, it passes.

### Syntax

```sql
-- Column-level CHECK
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    gpa         DECIMAL(3,2) CHECK (gpa >= 0.0 AND gpa <= 4.0),
    age         INT CHECK (age >= 16 AND age <= 100),
    status      VARCHAR(20) CHECK (status IN ('active', 'inactive', 'graduated', 'expelled'))
);

-- Table-level CHECK (can reference multiple columns)
CREATE TABLE events (
    event_id    SERIAL PRIMARY KEY,
    start_date  DATE NOT NULL,
    end_date    DATE NOT NULL,
    capacity    INT NOT NULL,
    registered  INT NOT NULL DEFAULT 0,
    CONSTRAINT chk_dates CHECK (end_date >= start_date),
    CONSTRAINT chk_capacity CHECK (registered <= capacity),
    CONSTRAINT chk_positive CHECK (capacity > 0 AND registered >= 0)
);

-- Add CHECK to existing table
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);
```

### Examples of CHECK Constraints

```sql
-- Percentage must be 0-100
CHECK (discount_pct BETWEEN 0 AND 100)

-- Email format (basic)
CHECK (email LIKE '%@%.%')

-- Phone number format
CHECK (phone ~ '^\+?[0-9]{10,15}$')  -- PostgreSQL regex

-- One of several allowed values
CHECK (gender IN ('M', 'F', 'O', 'N'))

-- Multi-column check
CHECK (min_salary < max_salary)

-- Conditional logic
CHECK (
    (account_type = 'savings' AND balance >= 0) OR
    (account_type = 'checking')  -- checking can go negative (overdraft)
)
```

### Limitations of CHECK

| Can Do | Cannot Do |
|---|---|
| Compare column values to constants | Reference other tables (no subqueries) |
| Compare columns within the same row | Call non-deterministic functions (in some DBMS) |
| Use AND, OR, NOT, IN, BETWEEN | Reference other rows in the same table |
| Use pattern matching (LIKE, regex) | Use aggregate functions |

> For cross-table or cross-row validation, use **triggers** or **application logic**.

---

## 7. DEFAULT Constraint

### Definition

Provides a **fallback value** when a column is not specified during INSERT.

### Syntax

```sql
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT NOW(),
    priority    INT DEFAULT 0,
    notes       TEXT DEFAULT '',
    is_active   BOOLEAN DEFAULT TRUE
);

-- Using DEFAULT in INSERT
INSERT INTO orders (status) VALUES ('shipped');
-- created_at = NOW(), priority = 0, notes = '', is_active = TRUE

-- Explicit DEFAULT keyword
INSERT INTO orders (order_id, status, created_at)
VALUES (DEFAULT, DEFAULT, DEFAULT);
-- Uses auto-increment for order_id, 'pending' for status, NOW() for created_at

-- Change a default
ALTER TABLE orders ALTER COLUMN priority SET DEFAULT 1;

-- Remove a default
ALTER TABLE orders ALTER COLUMN priority DROP DEFAULT;
```

### Types of DEFAULT Values

| Type | Example | Notes |
|---|---|---|
| Constant | `DEFAULT 0`, `DEFAULT 'pending'` | Most common |
| Current timestamp | `DEFAULT NOW()`, `DEFAULT CURRENT_TIMESTAMP` | For audit columns |
| Expression | `DEFAULT gen_random_uuid()` | UUID generation |
| Sequence | `DEFAULT NEXTVAL('order_seq')` | Auto-increment via sequence |
| Boolean | `DEFAULT TRUE`, `DEFAULT FALSE` | Feature flags |
| NULL | `DEFAULT NULL` | Explicit (same as no default) |

### Generated Columns (Computed Defaults)

PostgreSQL 12+ supports **generated columns** — their values are automatically computed and can't be manually set.

```sql
CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    base_price  DECIMAL(10,2) NOT NULL,
    tax_rate    DECIMAL(4,3) NOT NULL DEFAULT 0.08,
    total_price DECIMAL(10,2) GENERATED ALWAYS AS (base_price * (1 + tax_rate)) STORED
);

INSERT INTO products (base_price) VALUES (100.00);
-- total_price is automatically computed as 100.00 * 1.08 = 108.00
-- You CANNOT manually set total_price — it's always derived
```

---

## 8. Composite Keys

### Definition

A **composite key** is a primary key or unique key consisting of **two or more columns**. The combination must be unique, even if individual columns aren't.

### When to Use Composite Keys

| Scenario | Composite Key | Example |
|---|---|---|
| Junction table (M:N) | FK1 + FK2 | `(student_id, course_id)` |
| Multi-attribute identity | Natural columns | `(building_id, room_number)` |
| Time-series data | Entity + Timestamp | `(sensor_id, reading_time)` |
| Versioned records | Entity + Version | `(document_id, version_number)` |

### Example: Junction Table

```sql
CREATE TABLE enrollments (
    student_id  INT NOT NULL REFERENCES students(student_id),
    course_id   CHAR(6) NOT NULL REFERENCES courses(course_id),
    semester    CHAR(6) NOT NULL,
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id, semester)
);

-- These are BOTH valid (different semesters):
INSERT INTO enrollments VALUES (1, 'DB101', '2025F', 'B');
INSERT INTO enrollments VALUES (1, 'DB101', '2026S', 'A');  -- retake

-- This FAILS (same student, same course, same semester):
INSERT INTO enrollments VALUES (1, 'DB101', '2026S', 'A+');
-- ERROR: duplicate key value violates unique constraint
```

### Example: Weak Entity

```sql
CREATE TABLE rooms (
    building_id   INT NOT NULL REFERENCES buildings(building_id) ON DELETE CASCADE,
    room_number   VARCHAR(10) NOT NULL,
    capacity      INT,
    room_type     VARCHAR(50),
    PRIMARY KEY (building_id, room_number)
);

-- Room 101 in building 1 and room 101 in building 2 are DIFFERENT rooms
INSERT INTO rooms VALUES (1, '101', 30, 'lecture');
INSERT INTO rooms VALUES (2, '101', 50, 'lab');       -- different building, OK
INSERT INTO rooms VALUES (1, '101', 25, 'seminar');   -- FAILS: same (1, '101')
```

### Composite Keys — Pros and Cons

| Pros | Cons |
|---|---|
| Meaningful: key carries business meaning | Longer: more columns in joins and FK references |
| No extra surrogate column needed | Harder to reference from other tables |
| Enforces uniqueness naturally | More complex WHERE clauses |
| Smaller table (no extra ID column) | Can't change easily if business logic evolves |

---

## 9. Surrogate Keys vs. Natural Keys

This is one of the most debated topics in database design.

### Natural Key

A key formed from **real-world business data** that naturally identifies the entity.

```sql
-- Natural key examples:
CREATE TABLE countries (
    country_code  CHAR(2) PRIMARY KEY,    -- 'US', 'IN', 'GB' — ISO 3166
    country_name  VARCHAR(100)
);

CREATE TABLE books (
    isbn  CHAR(13) PRIMARY KEY,           -- ISBN is globally unique
    title VARCHAR(300)
);

CREATE TABLE us_citizens (
    ssn   CHAR(11) PRIMARY KEY,           -- Social Security Number
    name  VARCHAR(100)
);
```

### Surrogate Key

An **artificially generated** key with no business meaning — typically auto-increment or UUID.

```sql
-- Surrogate key examples:
CREATE TABLE students (
    student_id  SERIAL PRIMARY KEY,       -- auto-increment, no business meaning
    name        VARCHAR(100),
    email       VARCHAR(200) UNIQUE       -- natural candidate key preserved as UNIQUE
);

CREATE TABLE orders (
    order_id  BIGSERIAL PRIMARY KEY,      -- auto-increment
    order_date TIMESTAMP
);

CREATE TABLE distributed_events (
    event_id  UUID DEFAULT gen_random_uuid() PRIMARY KEY,  -- UUID for distributed systems
    event_type VARCHAR(50)
);
```

### Comparison

| Aspect | Natural Key | Surrogate Key |
|---|---|---|
| **Meaning** | Has business meaning | No business meaning |
| **Stability** | Can change (email, phone, name) | Never changes (auto-generated) |
| **Size** | Varies (could be wide composite) | Fixed (4 or 8 bytes for INT) |
| **Join performance** | Slower if wide | Faster (small INT) |
| **Readability** | Readable (`isbn='978-0-13-468599-1'`) | Opaque (`id=42`) |
| **Distributed systems** | May not be globally unique | UUID is globally unique |
| **Schema coupling** | Tightly coupled to business | Decoupled from business |
| **Extra column?** | No | Yes (uses space) |

### Recommendations

| Situation | Recommendation |
|---|---|
| Simple, stable, short natural key exists (country_code, currency_code) | Use natural key |
| Key might change (email, phone, name) | Use surrogate |
| Composite key would be wide (3+ columns) | Use surrogate |
| Junction table | Composite PK of both FKs (natural) |
| ORM-based application (Django, Rails, Hibernate) | Surrogate (ORMs expect single-column `id`) |
| Distributed system | UUID or ULID |
| Data warehouse fact table | Surrogate |

### The Pragmatic Middle Ground

Many experienced designers use **surrogate PK + natural UNIQUE**:

```sql
CREATE TABLE students (
    student_id  SERIAL PRIMARY KEY,           -- surrogate PK for joins/FKs
    email       VARCHAR(200) UNIQUE NOT NULL,  -- natural key preserved as UNIQUE
    name        VARCHAR(100) NOT NULL
);
```

This gives you: fast integer joins, stable FKs, AND business uniqueness enforcement.

---

## 10. Candidate Keys and Alternate Keys

### Definitions

- **Candidate Key**: Any minimal set of attributes that uniquely identifies a tuple (a minimal superkey)
- **Primary Key**: The candidate key chosen as the main identifier
- **Alternate Key**: Any candidate key that was NOT chosen as the PK

### Example

```sql
-- Students table: THREE candidate keys
CREATE TABLE students (
    student_id  SERIAL,           -- candidate key 1 → chosen as PK
    email       VARCHAR(200),     -- candidate key 2 → alternate key
    ssn         CHAR(11),         -- candidate key 3 → alternate key

    PRIMARY KEY (student_id),
    UNIQUE (email),               -- enforces CK2
    UNIQUE (ssn)                  -- enforces CK3
);
```

### How to Find Candidate Keys

Given: STUDENT(student_id, email, ssn, name, dept, gpa)

**Step 1**: Identify all possible superkeys:
```
{student_id}                          ← unique? YES → candidate (minimal)
{email}                               ← unique? YES → candidate (minimal)
{ssn}                                 ← unique? YES → candidate (minimal)
{name}                                ← unique? NO (names can repeat)
{dept}                                ← unique? NO
{gpa}                                 ← unique? NO
{student_id, email}                   ← unique? YES, but NOT minimal (student_id alone suffices)
{name, dept}                          ← unique? NO
{name, dept, gpa}                     ← unique? MAYBE — depends on data
```

**Step 2**: Candidate keys = minimal superkeys = `{student_id}`, `{email}`, `{ssn}`

**Step 3**: Choose one as PK → `student_id`. The others (`email`, `ssn`) become alternate keys.

---

## 11. Referential Integrity — Deep Dive

### What Referential Integrity Means

> Every foreign key value must either match an existing primary key value in the referenced table, or be NULL.

This prevents **orphaned records** — rows that reference non-existent parent rows.

### Violation Scenarios

```sql
-- Setup:
CREATE TABLE departments (
    dept_id    CHAR(4) PRIMARY KEY,
    dept_name  VARCHAR(100) NOT NULL
);
INSERT INTO departments VALUES ('CS', 'Computer Science'), ('MATH', 'Mathematics');

CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    dept_id    CHAR(4) REFERENCES departments(dept_id) ON DELETE RESTRICT
);
INSERT INTO students (name, dept_id) VALUES ('Alice', 'CS'), ('Bob', 'MATH');
```

**Scenario 1**: Insert a student referencing a non-existent department
```sql
INSERT INTO students (name, dept_id) VALUES ('Eve', 'BIO');
-- ERROR: insert or update on table "students" violates foreign key constraint
-- Key (dept_id)=(BIO) is not present in table "departments"
```

**Scenario 2**: Delete a department that has students
```sql
DELETE FROM departments WHERE dept_id = 'CS';
-- ERROR: update or delete on table "departments" violates foreign key constraint
-- Key (dept_id)=(CS) is still referenced from table "students"
```

**Scenario 3**: Update a PK that's referenced
```sql
UPDATE departments SET dept_id = 'CSCI' WHERE dept_id = 'CS';
-- ERROR (if ON UPDATE RESTRICT)
-- SUCCESS (if ON UPDATE CASCADE → students.dept_id also becomes 'CSCI')
```

### Circular References

Sometimes two tables reference each other. This creates a chicken-and-egg problem.

```sql
-- Department has a head professor; Professor belongs to a department
CREATE TABLE departments (
    dept_id       CHAR(4) PRIMARY KEY,
    dept_name     VARCHAR(100),
    head_prof_id  INT  -- FK to professors (added later)
);

CREATE TABLE professors (
    prof_id   SERIAL PRIMARY KEY,
    name      VARCHAR(100),
    dept_id   CHAR(4) REFERENCES departments(dept_id)
);

-- Now add the FK from departments to professors
ALTER TABLE departments
    ADD CONSTRAINT fk_head_prof
    FOREIGN KEY (head_prof_id) REFERENCES professors(prof_id)
    DEFERRABLE INITIALLY DEFERRED;  -- deferred so we can insert both first

-- Insert order matters, or use deferred constraints (see Section 13)
```

---

## 12. Constraint Enforcement Internals

### How Does the DBMS Enforce Constraints?

Understanding what happens "under the hood" helps you predict performance and design better schemas.

### PRIMARY KEY and UNIQUE — B-Tree Index

When you create a PK or UNIQUE constraint, the DBMS builds a **B-Tree index** on those columns.

```
B-Tree index on student_id:
          [  30  |  60  ]
         /       |       \
   [10|20]    [40|50]    [70|80]
    ↓   ↓      ↓   ↓      ↓   ↓
   rows       rows        rows

On INSERT: traverse B-Tree to check if value exists → O(log n)
On SELECT by PK: traverse B-Tree to find row → O(log n)
```

**Performance**:
- INSERT: O(log n) to check uniqueness + O(log n) to insert into index
- UPDATE PK: O(log n) to check new value + update index
- SELECT by PK: O(log n)

### FOREIGN KEY — Lookup in Referenced Table

On INSERT/UPDATE of child: The DBMS looks up the FK value in the parent's PK index.

On DELETE/UPDATE of parent: The DBMS scans the child table for referencing rows.

```
INSERT INTO students (dept_id = 'CS'):
  1. Look up 'CS' in departments PK index → O(log n)
  2. Found? → INSERT allowed
  3. Not found? → REJECT

DELETE FROM departments (dept_id = 'CS'):
  1. Scan students for any row with dept_id = 'CS'
  2. If found and RESTRICT → REJECT
  3. If found and CASCADE → delete those rows too
```

> **Performance tip**: Always create an **index on the FK column** in the child table. Without it, CASCADE operations and certain JOINs do a full table scan.

```sql
-- The PK index on departments.dept_id is automatic
-- But the FK column students.dept_id is NOT auto-indexed (in PostgreSQL)!
CREATE INDEX idx_students_dept ON students(dept_id);  -- add this manually
```

### CHECK — Expression Evaluation

CHECK constraints are evaluated for every INSERT and UPDATE. The DBMS evaluates the boolean expression on the new row's values.

```
INSERT INTO students (gpa = 3.9):
  1. Evaluate: 3.9 >= 0 AND 3.9 <= 4.0 → TRUE → allowed
  
INSERT INTO students (gpa = 5.0):
  1. Evaluate: 5.0 >= 0 AND 5.0 <= 4.0 → FALSE → REJECT
```

**Performance**: Trivial for simple comparisons. Can be noticeable for regex-heavy CHECKs on high-throughput tables.

### NOT NULL — Simple Presence Check

The cheapest constraint — just verifies the value is not the NULL sentinel. Negligible overhead.

### SVG: Constraint Enforcement Flow

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 380" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="380" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Constraint Enforcement on INSERT</text>

  <!-- INSERT -->
  <rect x="280" y="48" width="180" height="38" rx="8" fill="#2980b9"/>
  <text x="370" y="72" text-anchor="middle" fill="white" font-size="13" font-weight="bold">INSERT row</text>

  <line x1="370" y1="86" x2="370" y2="105" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,103 376,103 370,112" fill="#7f8c8d"/>

  <!-- NOT NULL check -->
  <rect x="260" y="113" width="220" height="34" rx="6" fill="#8e44ad"/>
  <text x="370" y="135" text-anchor="middle" fill="white" font-size="12">1. NOT NULL check</text>
  <text x="540" y="135" font-size="10" fill="#e74c3c">fail → ERROR</text>
  <line x1="370" y1="147" x2="370" y2="164" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,162 376,162 370,171" fill="#7f8c8d"/>

  <!-- CHECK constraint -->
  <rect x="260" y="172" width="220" height="34" rx="6" fill="#c0392b"/>
  <text x="370" y="194" text-anchor="middle" fill="white" font-size="12">2. CHECK constraints</text>
  <text x="540" y="194" font-size="10" fill="#e74c3c">fail → ERROR</text>
  <line x1="370" y1="206" x2="370" y2="223" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,221 376,221 370,230" fill="#7f8c8d"/>

  <!-- UNIQUE / PK check -->
  <rect x="260" y="231" width="220" height="34" rx="6" fill="#e67e22"/>
  <text x="370" y="253" text-anchor="middle" fill="white" font-size="12">3. PK / UNIQUE (B-Tree)</text>
  <text x="540" y="253" font-size="10" fill="#e74c3c">dup → ERROR</text>
  <line x1="370" y1="265" x2="370" y2="282" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,280 376,280 370,289" fill="#7f8c8d"/>

  <!-- FK check -->
  <rect x="260" y="290" width="220" height="34" rx="6" fill="#27ae60"/>
  <text x="370" y="312" text-anchor="middle" fill="white" font-size="12">4. FK (parent PK lookup)</text>
  <text x="540" y="312" font-size="10" fill="#e74c3c">missing → ERROR</text>
  <line x1="370" y1="324" x2="370" y2="341" stroke="#7f8c8d" stroke-width="2"/>
  <polygon points="364,339 376,339 370,348" fill="#7f8c8d"/>

  <!-- Success -->
  <rect x="290" y="350" width="160" height="24" rx="6" fill="#2ecc71"/>
  <text x="370" y="367" text-anchor="middle" fill="white" font-size="12" font-weight="bold">Row inserted ✓</text>
</svg>

---

## 13. Deferrable Constraints

### The Problem

Sometimes you need to temporarily violate a constraint within a transaction, then satisfy it by the end.

**Example**: Insert a department with a head professor, and a professor in that department, at the same time. If both FKs are checked immediately, neither INSERT can go first.

### Solution: DEFERRABLE Constraints

PostgreSQL allows constraints to be **deferred** — checked at COMMIT time instead of statement time.

```sql
-- Create a deferrable FK
CREATE TABLE departments (
    dept_id       CHAR(4) PRIMARY KEY,
    dept_name     VARCHAR(100),
    head_prof_id  INT,
    CONSTRAINT fk_head_prof FOREIGN KEY (head_prof_id)
        REFERENCES professors(prof_id)
        DEFERRABLE INITIALLY DEFERRED    -- checked at COMMIT, not at INSERT
);

CREATE TABLE professors (
    prof_id   SERIAL PRIMARY KEY,
    name      VARCHAR(100),
    dept_id   CHAR(4) REFERENCES departments(dept_id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Now we can do circular inserts:
BEGIN;
    INSERT INTO departments VALUES ('CS', 'Computer Science', 1);  -- prof 1 doesn't exist yet!
    INSERT INTO professors (prof_id, name, dept_id) VALUES (1, 'Dr. Smith', 'CS');
COMMIT;  -- both FKs checked HERE — both valid now
```

### Options

| Setting | Meaning |
|---|---|
| `NOT DEFERRABLE` (default) | Checked after each statement; cannot be deferred |
| `DEFERRABLE INITIALLY IMMEDIATE` | Checked after each statement by default, but can be deferred per-transaction |
| `DEFERRABLE INITIALLY DEFERRED` | Checked at COMMIT by default |

```sql
-- Change constraint timing within a transaction:
SET CONSTRAINTS fk_head_prof DEFERRED;
SET CONSTRAINTS fk_head_prof IMMEDIATE;
SET CONSTRAINTS ALL DEFERRED;
```

---

## 14. EXCLUDE Constraints (PostgreSQL)

A powerful PostgreSQL-specific constraint that prevents **overlapping** or **conflicting** values using operators.

### Use Case: Prevent Double-Booking

```sql
-- Requires the btree_gist extension
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    booking_id  SERIAL PRIMARY KEY,
    room_id     INT NOT NULL,
    time_range  TSRANGE NOT NULL,    -- timestamp range type
    booked_by   VARCHAR(100),

    -- No two bookings for the same room can overlap in time
    EXCLUDE USING GIST (
        room_id WITH =,           -- same room
        time_range WITH &&         -- overlapping time range
    )
);

-- This works:
INSERT INTO room_bookings (room_id, time_range, booked_by)
VALUES (1, '[2026-04-24 09:00, 2026-04-24 11:00)', 'Alice');

-- This FAILS (overlapping time for same room):
INSERT INTO room_bookings (room_id, time_range, booked_by)
VALUES (1, '[2026-04-24 10:00, 2026-04-24 12:00)', 'Bob');
-- ERROR: conflicting key value violates exclusion constraint

-- This works (different room):
INSERT INTO room_bookings (room_id, time_range, booked_by)
VALUES (2, '[2026-04-24 10:00, 2026-04-24 12:00)', 'Bob');
```

### Use Case: Prevent Overlapping Employee Assignments

```sql
CREATE TABLE employee_projects (
    emp_id      INT NOT NULL,
    project_id  INT NOT NULL,
    date_range  DATERANGE NOT NULL,

    EXCLUDE USING GIST (
        emp_id WITH =,
        date_range WITH &&
    )
);
-- An employee can't be assigned to two overlapping project periods
```

---

## 15. Naming and Managing Constraints

### Naming Conventions

Always name your constraints explicitly. Auto-generated names are cryptic.

```sql
-- Convention: {type}_{table}_{column(s)}
CONSTRAINT pk_students        PRIMARY KEY (student_id)
CONSTRAINT fk_students_dept   FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
CONSTRAINT uq_students_email  UNIQUE (email)
CONSTRAINT nn_students_name   CHECK (name IS NOT NULL)   -- alternative to column-level NOT NULL
CONSTRAINT chk_students_gpa   CHECK (gpa BETWEEN 0 AND 4.0)

-- Composite key naming:
CONSTRAINT pk_enrollments     PRIMARY KEY (student_id, course_id, semester)
CONSTRAINT uq_schedule_slot   UNIQUE (room_id, time_slot, day_of_week)
```

### Viewing Constraints

```sql
-- PostgreSQL: list all constraints for a table
SELECT conname, contype, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'students'::regclass;

-- contype key: p=PK, f=FK, u=UNIQUE, c=CHECK, x=EXCLUDE

-- Or use information_schema:
SELECT constraint_name, constraint_type
FROM information_schema.table_constraints
WHERE table_name = 'students';
```

### Modifying Constraints

```sql
-- Add a constraint
ALTER TABLE students ADD CONSTRAINT chk_gpa CHECK (gpa BETWEEN 0 AND 4.0);

-- Drop a constraint (by name)
ALTER TABLE students DROP CONSTRAINT chk_gpa;

-- Rename a constraint (PostgreSQL)
ALTER TABLE students RENAME CONSTRAINT chk_gpa TO chk_students_gpa_range;

-- Temporarily disable a FK check (dangerous — use with caution)
ALTER TABLE enrollments DISABLE TRIGGER ALL;  -- disables FK trigger
-- ... bulk load data ...
ALTER TABLE enrollments ENABLE TRIGGER ALL;   -- re-enable

-- Validate existing data against a new constraint
ALTER TABLE students ADD CONSTRAINT chk_gpa CHECK (gpa BETWEEN 0 AND 4.0) NOT VALID;
-- NOT VALID: constraint is enforced for new data but existing rows aren't checked
-- Later, validate existing data:
ALTER TABLE students VALIDATE CONSTRAINT chk_gpa;
```

---

## 16. Best Practices and Anti-Patterns

### Best Practices

| Practice | Why |
|---|---|
| **Always name constraints** | `fk_orders_customer` is debuggable; `orders_customer_id_fkey` is cryptic |
| **Always define PKs** | Every table needs a PK — no exceptions |
| **Index FK columns** | Without an index, CASCADE deletes and joins are slow (full scan) |
| **Use NOT NULL by default** | Only allow NULL when you have a specific reason |
| **Prefer RESTRICT over CASCADE** | CASCADE can delete thousands of rows silently — RESTRICT forces you to be explicit |
| **Use CHECK for domain rules** | `CHECK (status IN (...))` catches bugs at the DB level, not the app level |
| **Keep PKs immutable** | Never update PK values in production — use surrogate keys if the natural key might change |
| **Use DEFERRABLE for circular FKs** | Required for mutual-reference scenarios; don't hack around it with NULLs |

### Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| **No PK** | Can't uniquely identify rows; no automatic index | Always add a PK |
| **No FK constraints** | Orphaned records, broken references, data rot | Add FK constraints even if "the app handles it" |
| **Overly permissive NULLs** | NULLs spread through queries, cause unexpected results | Use NOT NULL + DEFAULT instead |
| **"God table" with no constraints** | One giant table with hundreds of nullable columns, no FKs | Normalize and add proper constraints |
| **Relying only on app-level validation** | One buggy app insert = corrupt data forever | DB constraints are the last line of defense |
| **CASCADE on everything** | One accidental delete cascades through the entire DB | Use RESTRICT by default; CASCADE only where appropriate |
| **Using VARCHAR for everything** | No domain constraints; "123abc" in an age column | Use proper types + CHECK constraints |
| **Not indexing FK columns** | Slow JOIN and CASCADE performance | `CREATE INDEX idx_child_fk ON child(parent_id)` |

---

## 17. Worked Examples — Beginner, Intermediate, Advanced

### Beginner: Student-Course Database

```sql
CREATE TABLE departments (
    dept_id    CHAR(4),
    dept_name  VARCHAR(100) NOT NULL,
    budget     DECIMAL(12,2) CHECK (budget > 0),
    CONSTRAINT pk_departments PRIMARY KEY (dept_id)
);

CREATE TABLE students (
    student_id  SERIAL,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(200) NOT NULL,
    dept_id     CHAR(4) NOT NULL,
    gpa         DECIMAL(3,2) DEFAULT 0.0,
    enrolled    DATE NOT NULL DEFAULT CURRENT_DATE,

    CONSTRAINT pk_students PRIMARY KEY (student_id),
    CONSTRAINT uq_students_email UNIQUE (email),
    CONSTRAINT fk_students_dept FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT chk_students_gpa CHECK (gpa >= 0 AND gpa <= 4.0)
);

CREATE TABLE courses (
    course_id  CHAR(6),
    title      VARCHAR(200) NOT NULL,
    credits    INT NOT NULL,
    dept_id    CHAR(4) NOT NULL,

    CONSTRAINT pk_courses PRIMARY KEY (course_id),
    CONSTRAINT fk_courses_dept FOREIGN KEY (dept_id) REFERENCES departments(dept_id),
    CONSTRAINT chk_courses_credits CHECK (credits BETWEEN 1 AND 6)
);

CREATE TABLE enrollments (
    student_id  INT NOT NULL,
    course_id   CHAR(6) NOT NULL,
    semester    CHAR(6) NOT NULL,
    grade       CHAR(2),

    CONSTRAINT pk_enrollments PRIMARY KEY (student_id, course_id, semester),
    CONSTRAINT fk_enroll_student FOREIGN KEY (student_id)
        REFERENCES students(student_id) ON DELETE CASCADE,
    CONSTRAINT fk_enroll_course FOREIGN KEY (course_id)
        REFERENCES courses(course_id) ON DELETE RESTRICT,
    CONSTRAINT chk_enroll_grade CHECK (grade IN ('A+','A','A-','B+','B','B-','C+','C','C-','D','F'))
);

-- Index FK columns for performance
CREATE INDEX idx_students_dept ON students(dept_id);
CREATE INDEX idx_courses_dept ON courses(dept_id);
CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
```

**What's enforced:**
- Every student must have a name, email, and department (NOT NULL + FK)
- No two students share an email (UNIQUE)
- GPA is always 0.0–4.0 (CHECK)
- Can't delete a department that has students (RESTRICT)
- Deleting a student auto-deletes their enrollments (CASCADE)
- Credits must be 1–6 (CHECK)
- Grades must be valid letter grades (CHECK)

---

### Intermediate: E-Commerce System

```sql
CREATE TABLE categories (
    category_id  SERIAL,
    name         VARCHAR(100) NOT NULL,
    parent_id    INT,

    CONSTRAINT pk_categories PRIMARY KEY (category_id),
    CONSTRAINT fk_cat_parent FOREIGN KEY (parent_id)
        REFERENCES categories(category_id) ON DELETE SET NULL,
    CONSTRAINT uq_cat_name UNIQUE (name)
);

CREATE TABLE products (
    product_id   SERIAL,
    sku          VARCHAR(50) NOT NULL,
    name         VARCHAR(200) NOT NULL,
    description  TEXT,
    price        DECIMAL(10,2) NOT NULL,
    cost         DECIMAL(10,2) NOT NULL,
    stock        INT NOT NULL DEFAULT 0,
    category_id  INT,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_products PRIMARY KEY (product_id),
    CONSTRAINT uq_products_sku UNIQUE (sku),
    CONSTRAINT fk_products_category FOREIGN KEY (category_id)
        REFERENCES categories(category_id) ON DELETE SET NULL,
    CONSTRAINT chk_products_price CHECK (price > 0),
    CONSTRAINT chk_products_cost CHECK (cost >= 0),
    CONSTRAINT chk_products_margin CHECK (price >= cost),
    CONSTRAINT chk_products_stock CHECK (stock >= 0)
);

CREATE TABLE customers (
    customer_id  SERIAL,
    email        VARCHAR(200) NOT NULL,
    name         VARCHAR(100) NOT NULL,
    phone        VARCHAR(20),
    tier         VARCHAR(20) NOT NULL DEFAULT 'standard',
    created_at   TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_customers PRIMARY KEY (customer_id),
    CONSTRAINT uq_customers_email UNIQUE (email),
    CONSTRAINT chk_cust_tier CHECK (tier IN ('standard', 'premium', 'vip'))
);

CREATE TABLE orders (
    order_id     SERIAL,
    customer_id  INT NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    order_date   TIMESTAMP NOT NULL DEFAULT NOW(),
    shipped_date TIMESTAMP,
    total_amount DECIMAL(10,2),

    CONSTRAINT pk_orders PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id) ON DELETE RESTRICT,
    CONSTRAINT chk_orders_status CHECK (status IN ('pending','confirmed','shipped','delivered','cancelled')),
    CONSTRAINT chk_orders_dates CHECK (shipped_date IS NULL OR shipped_date >= order_date),
    CONSTRAINT chk_orders_total CHECK (total_amount IS NULL OR total_amount >= 0)
);

CREATE TABLE order_items (
    order_id    INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,
    discount    DECIMAL(5,2) NOT NULL DEFAULT 0,

    CONSTRAINT pk_order_items PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_oi_order FOREIGN KEY (order_id)
        REFERENCES orders(order_id) ON DELETE CASCADE,
    CONSTRAINT fk_oi_product FOREIGN KEY (product_id)
        REFERENCES products(product_id) ON DELETE RESTRICT,
    CONSTRAINT chk_oi_quantity CHECK (quantity > 0),
    CONSTRAINT chk_oi_price CHECK (unit_price > 0),
    CONSTRAINT chk_oi_discount CHECK (discount >= 0 AND discount <= unit_price * quantity)
);

-- Performance indexes on FK columns
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_oi_order ON order_items(order_id);
CREATE INDEX idx_oi_product ON order_items(product_id);
```

**Highlights:**
- `chk_products_margin`: price must be ≥ cost (multi-column CHECK)
- `chk_orders_dates`: shipped_date must be after order_date (temporal CHECK)
- `chk_oi_discount`: discount can't exceed item total (cross-column CHECK)
- Self-referencing FK on categories (subcategories with SET NULL on delete)
- RESTRICT on customer delete (can't delete a customer who has orders)
- CASCADE on order delete (deleting an order removes its items)

---

### Advanced: Banking System with Audit Trail

```sql
CREATE TABLE branches (
    branch_id     SERIAL,
    branch_name   VARCHAR(100) NOT NULL,
    city          VARCHAR(100) NOT NULL,
    manager_id    INT,
    total_assets  DECIMAL(15,2) NOT NULL DEFAULT 0,

    CONSTRAINT pk_branches PRIMARY KEY (branch_id),
    CONSTRAINT uq_branch_name UNIQUE (branch_name),
    CONSTRAINT chk_branch_assets CHECK (total_assets >= 0)
);

CREATE TABLE accounts (
    account_id    SERIAL,
    branch_id     INT NOT NULL,
    holder_name   VARCHAR(100) NOT NULL,
    holder_ssn    CHAR(11) NOT NULL,
    account_type  VARCHAR(20) NOT NULL,
    balance       DECIMAL(15,2) NOT NULL DEFAULT 0,
    daily_limit   DECIMAL(10,2) NOT NULL DEFAULT 10000,
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    opened_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    closed_date   DATE,

    CONSTRAINT pk_accounts PRIMARY KEY (account_id),
    CONSTRAINT fk_acct_branch FOREIGN KEY (branch_id)
        REFERENCES branches(branch_id) ON DELETE RESTRICT,
    CONSTRAINT uq_acct_ssn_type UNIQUE (holder_ssn, account_type),
    CONSTRAINT chk_acct_type CHECK (account_type IN ('checking', 'savings', 'loan', 'credit')),
    CONSTRAINT chk_acct_balance CHECK (
        (account_type IN ('savings') AND balance >= 0) OR
        (account_type IN ('checking') AND balance >= -500) OR
        (account_type IN ('loan', 'credit'))
    ),
    CONSTRAINT chk_acct_limit CHECK (daily_limit > 0),
    CONSTRAINT chk_acct_dates CHECK (closed_date IS NULL OR closed_date >= opened_date)
);

CREATE TABLE transactions (
    txn_id        BIGSERIAL,
    from_account  INT,
    to_account    INT,
    amount        DECIMAL(15,2) NOT NULL,
    txn_type      VARCHAR(30) NOT NULL,
    txn_time      TIMESTAMP NOT NULL DEFAULT NOW(),
    status        VARCHAR(20) NOT NULL DEFAULT 'completed',
    description   VARCHAR(500),

    CONSTRAINT pk_transactions PRIMARY KEY (txn_id),
    CONSTRAINT fk_txn_from FOREIGN KEY (from_account)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT fk_txn_to FOREIGN KEY (to_account)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT chk_txn_amount CHECK (amount > 0),
    CONSTRAINT chk_txn_accounts CHECK (from_account != to_account OR from_account IS NULL),
    CONSTRAINT chk_txn_type CHECK (txn_type IN ('deposit','withdrawal','transfer','interest','fee','refund')),
    CONSTRAINT chk_txn_status CHECK (status IN ('pending','completed','failed','reversed')),
    CONSTRAINT chk_txn_has_account CHECK (from_account IS NOT NULL OR to_account IS NOT NULL)
);

CREATE TABLE audit_log (
    log_id       BIGSERIAL PRIMARY KEY,
    table_name   VARCHAR(50) NOT NULL,
    record_id    INT NOT NULL,
    action       VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_data     JSONB,
    new_data     JSONB,
    changed_by   VARCHAR(100) NOT NULL DEFAULT CURRENT_USER,
    changed_at   TIMESTAMP NOT NULL DEFAULT NOW()
);

-- FK indexes
CREATE INDEX idx_acct_branch ON accounts(branch_id);
CREATE INDEX idx_txn_from ON transactions(from_account);
CREATE INDEX idx_txn_to ON transactions(to_account);
CREATE INDEX idx_txn_time ON transactions(txn_time);
CREATE INDEX idx_audit_table ON audit_log(table_name, record_id);

-- Deferred FK for circular reference (branch manager = employee)
ALTER TABLE branches
    ADD CONSTRAINT fk_branch_manager
    FOREIGN KEY (manager_id) REFERENCES accounts(account_id)
    DEFERRABLE INITIALLY DEFERRED;
```

**Highlights:**
- `chk_acct_balance`: conditional balance rules — savings can't go negative, checking allows up to -$500 overdraft
- `uq_acct_ssn_type`: one person can't have two savings accounts (composite UNIQUE)
- `chk_txn_accounts`: can't transfer to yourself
- `chk_txn_has_account`: every transaction must involve at least one account
- Audit log uses JSONB to store old/new values of any table row
- BIGSERIAL for transactions (high-volume table)
- Deferrable FK for circular branch ↔ manager reference

---

## 18. Common Interview Questions

### Q1: What is the difference between PRIMARY KEY and UNIQUE?
> PK: one per table, no NULLs, creates clustered index (in some DBMS). UNIQUE: many per table, allows NULLs (typically multiple), creates non-clustered index. Both enforce uniqueness and auto-create indexes.

### Q2: Can a foreign key reference a UNIQUE column instead of a PK?
> Yes. A FK can reference any column or set of columns that has a UNIQUE or PRIMARY KEY constraint. The referenced column must be uniquely constrained.

### Q3: What is the difference between CASCADE and RESTRICT?
> CASCADE propagates the parent operation to child rows (delete parent → delete children). RESTRICT blocks the parent operation if child rows exist. Use RESTRICT by default for safety; use CASCADE only when children are meaningless without the parent.

### Q4: What is a composite key? Give an example.
> A key consisting of two or more columns whose combination is unique. Example: `(student_id, course_id, semester)` for enrollments — one student can take one course per semester, but can retake it in another semester.

### Q5: Surrogate key vs. natural key — when to use each?
> Natural key when it's stable, short, and guaranteed unique (country_code, ISBN). Surrogate key when the natural key might change (email), is too wide (composite of 4 columns), or when using an ORM. Many teams use surrogate PK + natural UNIQUE as a pragmatic middle ground.

### Q6: What happens if you don't index a foreign key column?
> The FK constraint still works, but operations become slow. CASCADE deletes, UPDATE cascades, and JOINs on the FK column will do full table scans of the child table instead of index lookups. Always index FK columns.

### Q7: What is a deferrable constraint? When would you use it?
> A constraint that's checked at COMMIT time instead of immediately after each statement. Use it for circular references (two tables that reference each other) or bulk data loads where temporary violations are resolved within the transaction.

### Q8: Can a CHECK constraint reference another table?
> No. Standard CHECK constraints can only reference columns within the same row. For cross-table validation, use triggers, functions, or application logic. Some systems (like PostgreSQL) also don't allow subqueries in CHECK.

### Q9: What is the difference between NOT NULL and CHECK (col IS NOT NULL)?
> Functionally identical — both prevent NULLs. But NOT NULL is a column-level constraint (simpler, standard), while `CHECK (col IS NOT NULL)` is a table-level constraint. Prefer `NOT NULL` for readability. Use `CHECK` when combining with other conditions.

### Q10: How do you enforce "at least one row" in a child table?
> SQL constraints can't directly enforce minimum cardinality (1:N total participation from the parent side). Options: (1) trigger that checks on parent insert/child delete, (2) application logic, (3) deferrable constraint + transaction that always inserts both. This is inherently difficult in SQL.

---

## 19. Key Takeaways

| # | Concept | One-Line Summary |
|---|---|---|
| 1 | **PRIMARY KEY** | Unique + NOT NULL + auto-indexed; one per table; main identifier |
| 2 | **FOREIGN KEY** | References parent PK; enforces referential integrity |
| 3 | **CASCADE** | Parent delete/update propagates to children |
| 4 | **RESTRICT** | Block parent delete/update if children exist (safer default) |
| 5 | **SET NULL** | Child FK becomes NULL when parent deleted |
| 6 | **UNIQUE** | No duplicate values; allows NULLs; many per table |
| 7 | **NOT NULL** | Value must be provided; use by default |
| 8 | **CHECK** | Custom boolean validation; same-row only |
| 9 | **DEFAULT** | Fallback value when column omitted in INSERT |
| 10 | **Composite Key** | PK of 2+ columns; for junction tables and weak entities |
| 11 | **Surrogate Key** | Auto-generated, no business meaning (SERIAL, UUID) |
| 12 | **Natural Key** | Real-world identifier (ISBN, country_code) |
| 13 | **Candidate Key** | Minimal superkey — there may be several |
| 14 | **Alternate Key** | Candidate key not chosen as PK — enforced with UNIQUE |
| 15 | **Index FK columns** | Always — without it, CASCADE and JOINs are slow |
| 16 | **Deferrable** | Constraint checked at COMMIT, not immediately — for circular refs |
| 17 | **EXCLUDE** | PostgreSQL: prevent overlapping ranges (scheduling, date ranges) |
| 18 | **Name your constraints** | `fk_orders_customer` is debuggable; auto-names are cryptic |
| 19 | **Constraints = DB immune system** | Never rely solely on app-level validation |
| 20 | **Surrogate PK + Natural UNIQUE** | The pragmatic middle ground for most tables |

---

<p style="text-align: center; margin-top: 40px;">
  <a href="05_keys_and_constraints.md" download="05_keys_and_constraints.md"
     style="display: inline-block; padding: 14px 36px; font-size: 16px; font-weight: bold;
            color: #fff; background-color: #c0392b; border-radius: 8px;
            text-decoration: none; box-shadow: 0 4px 6px rgba(0,0,0,0.15);">
    Download 05_keys_and_constraints.md
  </a>
</p>

---

*Ready? Say **"next"** to proceed to **Phase 6 — Normalization (1NF → 5NF)**.*
