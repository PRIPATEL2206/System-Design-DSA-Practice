# Phase 3 — Relational Model & Schema Design

> **Goal**: Master the mathematical foundation that underlies every SQL database. Understand relations, tuples, domains, keys, integrity constraints, relational algebra (the procedural query language), and relational calculus (the declarative query language). These concepts are what SQL is built on — knowing them makes you fluent in both theory and practice.

---

## Table of Contents

1. [The Relational Model — Origin and Core Idea](#1-the-relational-model--origin-and-core-idea)
2. [Formal Terminology](#2-formal-terminology)
3. [Domains and Data Types](#3-domains-and-data-types)
4. [Relational Schema Notation](#4-relational-schema-notation)
5. [Keys — All Types](#5-keys--all-types)
6. [Integrity Constraints](#6-integrity-constraints)
7. [Relational Algebra — Complete Guide](#7-relational-algebra--complete-guide)
8. [Relational Algebra — Extended Operations](#8-relational-algebra--extended-operations)
9. [Relational Calculus — Tuple Relational Calculus (TRC)](#9-relational-calculus--tuple-relational-calculus-trc)
10. [Relational Calculus — Domain Relational Calculus (DRC)](#10-relational-calculus--domain-relational-calculus-drc)
11. [Algebra vs. Calculus vs. SQL — Comparison](#11-algebra-vs-calculus-vs-sql--comparison)
12. [Worked Examples — Beginner, Intermediate, Advanced](#12-worked-examples--beginner-intermediate-advanced)
13. [Common Interview Questions](#13-common-interview-questions)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. The Relational Model — Origin and Core Idea

### History

In 1970, **Edgar F. Codd** (IBM Research) published the landmark paper *"A Relational Model of Data for Large Shared Data Banks"*. Before this, databases were navigational — you had to write code to traverse pointers between records (hierarchical and network models). Codd's breakthrough was:

> **Represent data as mathematical relations (tables), and let set-based operations retrieve it.**

This meant:
- Users describe **what** they want, not **how** to navigate to it
- The DBMS figures out the optimal access path (query optimization)
- The logical structure is independent of physical storage (data independence)

### The Core Idea in One Sentence

> A relational database is a collection of **relations** (tables), where each relation is a set of **tuples** (rows), and each tuple is an ordered list of **attribute values** drawn from specific **domains** (data types).

### Real-World Analogy

Think of a **spreadsheet**:
- The spreadsheet = a **relation** (table)
- Each row = a **tuple** (record)
- Each column header = an **attribute** (field)
- The type of data allowed in a column (text, number, date) = the **domain**
- The rule that "no two rows are identical" = the set property of relations

But unlike spreadsheets:
- Relations have **no duplicate rows** (it's a set, not a multiset)
- Relations have **no inherent row ordering** (rows are unordered)
- Each cell holds an **atomic (indivisible) value** (1st Normal Form)

---

## 2. Formal Terminology

### The Mapping: Informal ↔ Formal

| Informal (everyday) | Formal (theory) | SQL |
|---|---|---|
| Table | Relation | TABLE |
| Row | Tuple | ROW / RECORD |
| Column | Attribute | COLUMN / FIELD |
| Column data type | Domain | DATA TYPE |
| Table definition | Relation Schema | CREATE TABLE |
| Table with data | Relation Instance | Table contents at a point in time |
| Number of columns | Degree (arity) | — |
| Number of rows | Cardinality | COUNT(*) |

### Formal Definitions

#### Relation Schema

A **relation schema** R(A₁, A₂, ..., Aₙ) is a named relation R with a set of attributes {A₁, A₂, ..., Aₙ}, where each attribute Aᵢ has a domain dom(Aᵢ).

```
Schema:   STUDENT(student_id: INT, name: VARCHAR, dept: VARCHAR, gpa: DECIMAL)
          ───────                                                              
          Relation name    Attributes with their domains
```

#### Relation Instance

A **relation instance** r(R) is a set of tuples {t₁, t₂, ..., tₘ} where each tuple tᵢ is an ordered list of values (v₁, v₂, ..., vₙ), one value per attribute, each drawn from its respective domain.

```
Instance of STUDENT:

 student_id │  name  │ dept │ gpa
────────────┼────────┼──────┼─────
      1     │ Alice  │  CS  │ 3.9
      2     │ Bob    │ Math │ 3.5
      3     │ Carol  │  CS  │ 3.7

Degree (arity) = 4 (four attributes)
Cardinality    = 3 (three tuples)
```

#### Key Properties of a Relation

| Property | Meaning |
|---|---|
| **No duplicate tuples** | A relation is a SET — every row is unique |
| **Tuples are unordered** | There is no "first row" or "last row" |
| **Attributes are unordered** | Column order doesn't matter (identified by name) |
| **Attribute values are atomic** | No lists, sets, or nested structures in a single cell (1NF) |
| **Each attribute draws from a domain** | Values must be of the correct type |

### SVG: Relation Anatomy

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 340" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="340" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Anatomy of a Relation</text>

  <!-- Relation name -->
  <text x="80" y="65" font-size="16" font-weight="bold" fill="#2980b9">STUDENT</text>
  <text x="200" y="65" font-size="12" fill="#888">← Relation name</text>

  <!-- Header row (attributes) -->
  <rect x="50" y="80" width="580" height="35" rx="0" fill="#2980b9"/>
  <text x="120" y="103" text-anchor="middle" fill="white" font-size="13" font-weight="bold">student_id</text>
  <text x="260" y="103" text-anchor="middle" fill="white" font-size="13" font-weight="bold">name</text>
  <text x="390" y="103" text-anchor="middle" fill="white" font-size="13" font-weight="bold">dept</text>
  <text x="520" y="103" text-anchor="middle" fill="white" font-size="13" font-weight="bold">gpa</text>
  <line x1="190" y1="80" x2="190" y2="245" stroke="#aed6f1" stroke-width="1"/>
  <line x1="325" y1="80" x2="325" y2="245" stroke="#aed6f1" stroke-width="1"/>
  <line x1="455" y1="80" x2="455" y2="245" stroke="#aed6f1" stroke-width="1"/>

  <!-- Label: Attributes -->
  <line x1="640" y1="97" x2="670" y2="97" stroke="#e74c3c" stroke-width="1.5"/>
  <text x="680" y="101" font-size="11" fill="#e74c3c">← Attributes (columns)</text>

  <!-- Data rows (tuples) -->
  <rect x="50" y="115" width="580" height="32" fill="#ebf5fb"/>
  <text x="120" y="136" text-anchor="middle" font-size="12" fill="#333">1</text>
  <text x="260" y="136" text-anchor="middle" font-size="12" fill="#333">Alice</text>
  <text x="390" y="136" text-anchor="middle" font-size="12" fill="#333">CS</text>
  <text x="520" y="136" text-anchor="middle" font-size="12" fill="#333">3.9</text>

  <rect x="50" y="147" width="580" height="32" fill="#fff"/>
  <text x="120" y="168" text-anchor="middle" font-size="12" fill="#333">2</text>
  <text x="260" y="168" text-anchor="middle" font-size="12" fill="#333">Bob</text>
  <text x="390" y="168" text-anchor="middle" font-size="12" fill="#333">Math</text>
  <text x="520" y="168" text-anchor="middle" font-size="12" fill="#333">3.5</text>

  <rect x="50" y="179" width="580" height="32" fill="#ebf5fb"/>
  <text x="120" y="200" text-anchor="middle" font-size="12" fill="#333">3</text>
  <text x="260" y="200" text-anchor="middle" font-size="12" fill="#333">Carol</text>
  <text x="390" y="200" text-anchor="middle" font-size="12" fill="#333">CS</text>
  <text x="520" y="200" text-anchor="middle" font-size="12" fill="#333">3.7</text>

  <rect x="50" y="211" width="580" height="32" fill="#fff"/>
  <text x="120" y="232" text-anchor="middle" font-size="12" fill="#333">4</text>
  <text x="260" y="232" text-anchor="middle" font-size="12" fill="#333">Dave</text>
  <text x="390" y="232" text-anchor="middle" font-size="12" fill="#333">EE</text>
  <text x="520" y="232" text-anchor="middle" font-size="12" fill="#333">3.2</text>

  <!-- Borders -->
  <rect x="50" y="80" width="580" height="165" fill="none" stroke="#2980b9" stroke-width="2" rx="0"/>

  <!-- Labels -->
  <line x1="640" y1="131" x2="670" y2="131" stroke="#27ae60" stroke-width="1.5"/>
  <text x="680" y="135" font-size="11" fill="#27ae60">← Tuple (row)</text>

  <line x1="520" y1="248" x2="520" y2="270" stroke="#8e44ad" stroke-width="1.5"/>
  <text x="520" y="285" text-anchor="middle" font-size="11" fill="#8e44ad">Attribute value</text>
  <text x="520" y="300" text-anchor="middle" font-size="10" fill="#888">(from domain DECIMAL)</text>

  <!-- Degree / Cardinality -->
  <rect x="50" y="260" width="200" height="28" rx="5" fill="#eafaf1" stroke="#27ae60" stroke-width="1"/>
  <text x="150" y="279" text-anchor="middle" font-size="11" fill="#145a32">Degree = 4 attributes</text>
  <rect x="260" y="260" width="200" height="28" rx="5" fill="#fef9e7" stroke="#f39c12" stroke-width="1"/>
  <text x="360" y="279" text-anchor="middle" font-size="11" fill="#784212">Cardinality = 4 tuples</text>

  <!-- Domain annotation -->
  <text x="120" y="320" text-anchor="middle" font-size="10" fill="#888">dom: INT</text>
  <text x="260" y="320" text-anchor="middle" font-size="10" fill="#888">dom: VARCHAR</text>
  <text x="390" y="320" text-anchor="middle" font-size="10" fill="#888">dom: VARCHAR</text>
  <text x="520" y="320" text-anchor="middle" font-size="10" fill="#888">dom: DECIMAL</text>
</svg>

---

## 3. Domains and Data Types

A **domain** is the set of all permissible values for an attribute. It's the theoretical concept behind SQL data types.

### Formal Definition

> dom(Aᵢ) = the set of atomic values that attribute Aᵢ can take.

### Examples

| Attribute | Domain | Description | SQL Type |
|---|---|---|---|
| student_id | Positive integers | {1, 2, 3, ...} | INT |
| name | Strings up to 100 chars | All strings ≤ 100 characters | VARCHAR(100) |
| gpa | Decimals 0.00–4.00 | {0.00, 0.01, ..., 4.00} | DECIMAL(3,2) |
| dept | {'CS', 'Math', 'EE', ...} | Finite set of department codes | VARCHAR(50) |
| enrolled | Valid dates | All dates in the Gregorian calendar | DATE |
| is_active | {TRUE, FALSE} | Boolean domain | BOOLEAN |

### NULL: The Special Value

**NULL** is not a value — it represents the **absence** of a value. It means:
- **Unknown**: We don't know Alice's phone number
- **Inapplicable**: An unmarried person has no spouse name
- **Withheld**: The user chose not to provide their email

**NULL is NOT the same as 0, empty string, or FALSE.**

```sql
-- NULL comparisons are tricky:
SELECT * FROM students WHERE gpa = NULL;      -- WRONG! Returns nothing
SELECT * FROM students WHERE gpa IS NULL;     -- CORRECT

-- NULL in arithmetic:
5 + NULL = NULL    -- anything + NULL = NULL
NULL = NULL        -- FALSE (even NULL ≠ NULL in SQL!)
NULL AND TRUE      -- NULL (three-valued logic)
```

### Three-Valued Logic (because of NULL)

SQL uses **three-valued logic**: TRUE, FALSE, UNKNOWN.

| A | B | A AND B | A OR B | NOT A |
|---|---|---|---|---|
| T | T | T | T | F |
| T | F | F | T | F |
| T | U | U | T | F |
| F | F | F | F | T |
| F | U | F | U | T |
| U | U | U | U | U |

> **Key rule**: `WHERE` only returns rows where the condition is **TRUE** (not UNKNOWN). This is why `WHERE gpa = NULL` returns nothing — the comparison yields UNKNOWN, not TRUE.

---

## 4. Relational Schema Notation

### Schema Notation

A database schema is written as a set of relation schemas:

```
DEPARTMENT(dept_id, dept_name, building, budget)
STUDENT(student_id, name, dept_id, gpa, enrolled)
COURSE(course_id, title, credits, dept_id)
ENROLLMENT(student_id, course_id, semester, grade)
```

**Conventions**:
- **Underlined** attributes form the **primary key**: STUDENT(<u>student_id</u>, name, dept_id, gpa)
- **Foreign keys** shown with arrows or annotation: STUDENT(student_id, name, **dept_id** → DEPARTMENT)
- **NOT NULL** marked with an asterisk or bold: name*

### Full Schema Diagram

```
DEPARTMENT(dept_id, dept_name, building, budget)
    PK: dept_id

STUDENT(student_id, name, dept_id, gpa, enrolled)
    PK: student_id
    FK: dept_id → DEPARTMENT(dept_id)

COURSE(course_id, title, credits, dept_id, prof_id)
    PK: course_id
    FK: dept_id → DEPARTMENT(dept_id)
    FK: prof_id → PROFESSOR(prof_id)

ENROLLMENT(student_id, course_id, semester, grade)
    PK: (student_id, course_id, semester)
    FK: student_id → STUDENT(student_id)
    FK: course_id → COURSE(course_id)
```

---

## 5. Keys — All Types

### Definitions

#### Superkey
Any set of attributes that **uniquely identifies** a tuple. Can have extra (unnecessary) attributes.

```
STUDENT: {student_id}                     ← superkey
         {student_id, name}               ← superkey (includes extra 'name')
         {student_id, name, dept, gpa}    ← superkey (the whole tuple)
         {name}                           ← NOT a superkey (names can repeat)
```

#### Candidate Key
A **minimal** superkey — remove any attribute and it's no longer a superkey.

```
STUDENT:
  {student_id}    ← candidate key (minimal)
  {email}         ← candidate key (if emails are unique)
```

#### Primary Key
The candidate key **chosen** by the designer to be the main identifier. Every table must have exactly one.

```sql
CREATE TABLE students (
    student_id  INT PRIMARY KEY,    -- chosen as primary key
    email       VARCHAR(200) UNIQUE -- this is also a candidate key (alternate key)
);
```

#### Alternate Key
Any candidate key that was **not chosen** as the primary key. Enforced with `UNIQUE`.

#### Foreign Key
An attribute (or set of attributes) in one relation that **references the primary key** of another relation. Establishes relationships between tables.

```sql
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    dept_id     CHAR(4) REFERENCES departments(dept_id)  -- foreign key
);
```

#### Composite Key
A key that consists of **two or more** attributes.

```sql
CREATE TABLE enrollments (
    student_id  INT,
    course_id   CHAR(8),
    semester    CHAR(6),
    PRIMARY KEY (student_id, course_id, semester)  -- composite key
);
```

#### Surrogate Key
An artificial key (auto-increment, UUID) with no business meaning. Used when no natural candidate key exists or when composite keys are unwieldy.

```sql
CREATE TABLE orders (
    order_id  SERIAL PRIMARY KEY,  -- surrogate key (auto-generated)
    -- vs. natural key would be something like (customer_id, order_date, sequence_num)
);
```

### Key Hierarchy Diagram

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 280" font-family="Segoe UI, Arial, sans-serif">
  <rect width="720" height="280" fill="#f8f9fa" rx="12"/>
  <text x="360" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Key Hierarchy</text>

  <!-- Superkey (outermost) -->
  <rect x="40" y="50" width="640" height="210" rx="12" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="360" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">SUPERKEYS</text>
  <text x="360" y="88" text-anchor="middle" font-size="10" fill="#555">Any set of attributes that uniquely identifies a tuple (may have extras)</text>

  <!-- Candidate keys -->
  <rect x="80" y="100" width="560" height="140" rx="10" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="360" y="120" text-anchor="middle" font-size="13" font-weight="bold" fill="#27ae60">CANDIDATE KEYS</text>
  <text x="360" y="136" text-anchor="middle" font-size="10" fill="#555">Minimal superkeys — remove any attribute and uniqueness is lost</text>

  <!-- Primary Key -->
  <rect x="120" y="150" width="230" height="72" rx="8" fill="#27ae60"/>
  <text x="235" y="175" text-anchor="middle" font-size="13" font-weight="bold" fill="white">PRIMARY KEY</text>
  <text x="235" y="195" text-anchor="middle" font-size="10" fill="#d5f5e3">The chosen candidate key</text>
  <text x="235" y="210" text-anchor="middle" font-size="10" fill="#d5f5e3">{student_id}</text>

  <!-- Alternate Keys -->
  <rect x="370" y="150" width="250" height="72" rx="8" fill="#f39c12"/>
  <text x="495" y="175" text-anchor="middle" font-size="13" font-weight="bold" fill="white">ALTERNATE KEYS</text>
  <text x="495" y="195" text-anchor="middle" font-size="10" fill="#fef9e7">Other candidate keys (UNIQUE)</text>
  <text x="495" y="210" text-anchor="middle" font-size="10" fill="#fef9e7">{email}, {ssn}</text>

  <!-- Examples on the right -->
  <text x="650" y="238" font-size="9" fill="#888">{sid, name} = superkey</text>
  <text x="650" y="252" font-size="9" fill="#888">{sid} = candidate = primary</text>
</svg>

### Quick Summary Table

| Key Type | Minimal? | Chosen? | SQL Construct | Example |
|---|---|---|---|---|
| Superkey | No | — | — | {student_id, name} |
| Candidate Key | Yes | — | UNIQUE | {student_id}, {email} |
| Primary Key | Yes | Yes | PRIMARY KEY | {student_id} |
| Alternate Key | Yes | No (other CK) | UNIQUE | {email} |
| Foreign Key | — | — | REFERENCES | dept_id → departments |
| Composite Key | — | — | PRIMARY KEY(a, b) | (student_id, course_id) |
| Surrogate Key | — | — | SERIAL / UUID | auto-increment id |

---

## 6. Integrity Constraints

Integrity constraints are **rules** that the DBMS enforces to keep data valid and consistent. They are the backbone of data quality.

### 6.1 Domain Constraints

Every attribute value must come from its defined domain.

```sql
CREATE TABLE students (
    gpa  DECIMAL(3,2) CHECK (gpa >= 0 AND gpa <= 4.0),  -- domain constraint
    dept VARCHAR(50)                                       -- domain: strings ≤ 50 chars
);

-- Violation:
INSERT INTO students (gpa) VALUES (5.0);   -- ERROR: fails CHECK constraint
INSERT INTO students (gpa) VALUES ('abc'); -- ERROR: not a DECIMAL
```

### 6.2 Key Constraints (Entity Integrity)

**Rule**: The primary key must be **unique** and **NOT NULL** for every tuple.

> No two tuples can have the same PK value. No PK attribute can be NULL.

```sql
-- These will FAIL:
INSERT INTO students (student_id, name) VALUES (1, 'Alice');
INSERT INTO students (student_id, name) VALUES (1, 'Bob');    -- ERROR: duplicate PK

INSERT INTO students (student_id, name) VALUES (NULL, 'Eve'); -- ERROR: PK cannot be NULL
```

### 6.3 Referential Integrity (Foreign Key Constraints)

**Rule**: A foreign key value must either:
1. Match an existing primary key value in the referenced table, OR
2. Be NULL (if the FK column allows NULL)

```sql
-- departments has: dept_id = 'CS', 'Math', 'EE'

INSERT INTO students (student_id, name, dept_id)
VALUES (10, 'Frank', 'CS');     -- OK: 'CS' exists in departments

INSERT INTO students (student_id, name, dept_id)
VALUES (11, 'Grace', 'BIO');    -- ERROR: 'BIO' not in departments

INSERT INTO students (student_id, name, dept_id)
VALUES (12, 'Hank', NULL);     -- OK if dept_id allows NULL (partial participation)
```

#### Referential Actions (what happens when the referenced row is deleted/updated?)

```sql
CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id)
        ON DELETE CASCADE       -- delete student → delete their enrollments
        ON UPDATE CASCADE,      -- update student_id → update in enrollments too
    course_id  CHAR(8) REFERENCES courses(course_id)
        ON DELETE RESTRICT      -- can't delete a course if students are enrolled
        ON UPDATE SET NULL      -- update course_id → set enrollment's course_id to NULL
);
```

| Action | Meaning |
|---|---|
| **CASCADE** | Propagate the delete/update to referencing rows |
| **RESTRICT** | Prevent the delete/update if referencing rows exist |
| **SET NULL** | Set the FK to NULL in referencing rows |
| **SET DEFAULT** | Set the FK to its default value |
| **NO ACTION** | Like RESTRICT but checked at end of transaction |

### 6.4 Semantic Constraints (Business Rules)

Rules specific to the business domain, beyond structural integrity.

```sql
-- "A student can enroll in at most 6 courses per semester"
-- Can't be expressed with simple constraints — need a trigger or application logic

CREATE OR REPLACE FUNCTION check_enrollment_limit()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM enrollments
        WHERE student_id = NEW.student_id AND semester = NEW.semester) >= 6
    THEN
        RAISE EXCEPTION 'Student cannot enroll in more than 6 courses per semester';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enrollment_limit_trigger
BEFORE INSERT ON enrollments
FOR EACH ROW EXECUTE FUNCTION check_enrollment_limit();
```

### SVG: Integrity Constraints Overview

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 300" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="300" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Integrity Constraints</text>

  <!-- Domain -->
  <rect x="20" y="50" width="165" height="230" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="102" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#2980b9">Domain</text>
  <text x="102" y="100" text-anchor="middle" font-size="10" fill="#555">Values must be from</text>
  <text x="102" y="115" text-anchor="middle" font-size="10" fill="#555">the attribute's domain</text>
  <rect x="32" y="130" width="140" height="22" rx="4" fill="#d6eaf8"/>
  <text x="102" y="145" text-anchor="middle" font-size="10" fill="#1a5276">Data type checks</text>
  <rect x="32" y="158" width="140" height="22" rx="4" fill="#d6eaf8"/>
  <text x="102" y="173" text-anchor="middle" font-size="10" fill="#1a5276">CHECK constraints</text>
  <rect x="32" y="186" width="140" height="22" rx="4" fill="#d6eaf8"/>
  <text x="102" y="201" text-anchor="middle" font-size="10" fill="#1a5276">NOT NULL</text>
  <text x="102" y="240" text-anchor="middle" font-size="10" fill="#2980b9" font-style="italic">gpa DECIMAL(3,2)</text>
  <text x="102" y="255" text-anchor="middle" font-size="10" fill="#2980b9" font-style="italic">CHECK(gpa &lt;= 4.0)</text>

  <!-- Key / Entity -->
  <rect x="200" y="50" width="165" height="230" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="282" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#27ae60">Key (Entity)</text>
  <text x="282" y="100" text-anchor="middle" font-size="10" fill="#555">PK must be unique</text>
  <text x="282" y="115" text-anchor="middle" font-size="10" fill="#555">and NOT NULL</text>
  <rect x="212" y="130" width="140" height="22" rx="4" fill="#d5f5e3"/>
  <text x="282" y="145" text-anchor="middle" font-size="10" fill="#145a32">PRIMARY KEY</text>
  <rect x="212" y="158" width="140" height="22" rx="4" fill="#d5f5e3"/>
  <text x="282" y="173" text-anchor="middle" font-size="10" fill="#145a32">UNIQUE</text>
  <rect x="212" y="186" width="140" height="22" rx="4" fill="#d5f5e3"/>
  <text x="282" y="201" text-anchor="middle" font-size="10" fill="#145a32">No duplicate tuples</text>
  <text x="282" y="240" text-anchor="middle" font-size="10" fill="#27ae60" font-style="italic">student_id PK</text>
  <text x="282" y="255" text-anchor="middle" font-size="10" fill="#27ae60" font-style="italic">email UNIQUE</text>

  <!-- Referential -->
  <rect x="380" y="50" width="165" height="230" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="462" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#e67e22">Referential</text>
  <text x="462" y="100" text-anchor="middle" font-size="10" fill="#555">FK must match a PK</text>
  <text x="462" y="115" text-anchor="middle" font-size="10" fill="#555">in referenced table</text>
  <rect x="392" y="130" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="462" y="145" text-anchor="middle" font-size="10" fill="#784212">REFERENCES</text>
  <rect x="392" y="158" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="462" y="173" text-anchor="middle" font-size="10" fill="#784212">ON DELETE CASCADE</text>
  <rect x="392" y="186" width="140" height="22" rx="4" fill="#fdebd0"/>
  <text x="462" y="201" text-anchor="middle" font-size="10" fill="#784212">ON UPDATE SET NULL</text>
  <text x="462" y="240" text-anchor="middle" font-size="10" fill="#e67e22" font-style="italic">dept_id FK →</text>
  <text x="462" y="255" text-anchor="middle" font-size="10" fill="#e67e22" font-style="italic">departments(dept_id)</text>

  <!-- Semantic -->
  <rect x="560" y="50" width="165" height="230" rx="8" fill="#f4ecf7" stroke="#8e44ad" stroke-width="2"/>
  <text x="642" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#8e44ad">Semantic</text>
  <text x="642" y="100" text-anchor="middle" font-size="10" fill="#555">Business rules beyond</text>
  <text x="642" y="115" text-anchor="middle" font-size="10" fill="#555">structural constraints</text>
  <rect x="572" y="130" width="140" height="22" rx="4" fill="#e8daef"/>
  <text x="642" y="145" text-anchor="middle" font-size="10" fill="#4a235a">Triggers</text>
  <rect x="572" y="158" width="140" height="22" rx="4" fill="#e8daef"/>
  <text x="642" y="173" text-anchor="middle" font-size="10" fill="#4a235a">Assertions</text>
  <rect x="572" y="186" width="140" height="22" rx="4" fill="#e8daef"/>
  <text x="642" y="201" text-anchor="middle" font-size="10" fill="#4a235a">Application logic</text>
  <text x="642" y="240" text-anchor="middle" font-size="10" fill="#8e44ad" font-style="italic">"Max 6 courses</text>
  <text x="642" y="255" text-anchor="middle" font-size="10" fill="#8e44ad" font-style="italic">per semester"</text>
</svg>

---

## 7. Relational Algebra — Complete Guide

**Relational algebra** is a procedural query language — you specify **how** to compute the result step by step. It operates on relations and produces relations (closed under the algebra). SQL is based on relational algebra.

### The Six Fundamental Operations

Every possible query can be expressed using these six operations:

| # | Operation | Symbol | Type | Description |
|---|---|---|---|---|
| 1 | **Selection** | σ (sigma) | Unary | Filter rows by condition |
| 2 | **Projection** | π (pi) | Unary | Choose specific columns |
| 3 | **Union** | ∪ | Binary | Combine rows from two relations |
| 4 | **Set Difference** | − | Binary | Rows in R₁ but not in R₂ |
| 5 | **Cartesian Product** | × | Binary | Every combination of rows |
| 6 | **Rename** | ρ (rho) | Unary | Rename relation or attributes |

### Sample Data for All Examples

```
STUDENT                                   COURSE
┌────┬───────┬──────┬─────┐              ┌──────┬────────────┬─────┐
│ id │ name  │ dept │ gpa │              │ cid  │ title      │ cr  │
├────┼───────┼──────┼─────┤              ├──────┼────────────┼─────┤
│  1 │ Alice │ CS   │ 3.9 │              │ DB01 │ Databases  │  4  │
│  2 │ Bob   │ Math │ 3.5 │              │ OS02 │ Op Systems │  3  │
│  3 │ Carol │ CS   │ 3.7 │              │ ML03 │ Machine L  │  4  │
│  4 │ Dave  │ EE   │ 3.2 │              └──────┴────────────┴─────┘
│  5 │ Eve   │ CS   │ 3.1 │
└────┴───────┴──────┴─────┘

ENROLLMENT
┌────┬──────┬────────┬───────┐
│ sid│ cid  │semester│ grade │
├────┼──────┼────────┼───────┤
│  1 │ DB01 │ 2026S1 │  A    │
│  1 │ OS02 │ 2026S1 │  B+   │
│  2 │ DB01 │ 2026S1 │  A-   │
│  3 │ ML03 │ 2026S1 │  B    │
│  3 │ DB01 │ 2026S1 │  A    │
│  5 │ OS02 │ 2026S1 │  C+   │
└────┴──────┴────────┴───────┘
```

---

### 7.1 Selection (σ) — Filter Rows

**Syntax**: σ<sub>condition</sub>(R)

Returns all tuples from R that satisfy the condition. Like SQL `WHERE`.

```
σ_{dept='CS'}(STUDENT)

Result:
┌────┬───────┬──────┬─────┐
│ id │ name  │ dept │ gpa │
├────┼───────┼──────┼─────┤
│  1 │ Alice │ CS   │ 3.9 │
│  3 │ Carol │ CS   │ 3.7 │
│  5 │ Eve   │ CS   │ 3.1 │
└────┴───────┴──────┴─────┘
```

```
σ_{dept='CS' AND gpa > 3.5}(STUDENT)

Result:
┌────┬───────┬──────┬─────┐
│  1 │ Alice │ CS   │ 3.9 │
│  3 │ Carol │ CS   │ 3.7 │
└────┴───────┴──────┴─────┘
```

**SQL equivalent**:
```sql
SELECT * FROM student WHERE dept = 'CS' AND gpa > 3.5;
```

**Properties**:
- σ is commutative: σ<sub>c1</sub>(σ<sub>c2</sub>(R)) = σ<sub>c2</sub>(σ<sub>c1</sub>(R))
- σ is idempotent: σ<sub>c</sub>(σ<sub>c</sub>(R)) = σ<sub>c</sub>(R)
- Cascading selections: σ<sub>c1 AND c2</sub>(R) = σ<sub>c1</sub>(σ<sub>c2</sub>(R))

---

### 7.2 Projection (π) — Choose Columns

**Syntax**: π<sub>A1, A2, ...</sub>(R)

Returns a relation with only the specified attributes. **Eliminates duplicates** (because the result is a set). Like SQL `SELECT DISTINCT col1, col2`.

```
π_{name, dept}(STUDENT)

Result:
┌───────┬──────┐
│ name  │ dept │
├───────┼──────┤
│ Alice │ CS   │
│ Bob   │ Math │
│ Carol │ CS   │
│ Dave  │ EE   │
│ Eve   │ CS   │
└───────┴──────┘
```

```
π_{dept}(STUDENT)

Result:
┌──────┐
│ dept │     ← only 3 rows! duplicates removed
├──────┤
│ CS   │
│ Math │
│ EE   │
└──────┘
```

**SQL equivalent**:
```sql
SELECT DISTINCT dept FROM student;
```

### Combining Selection and Projection

"Names of CS students with GPA > 3.5":

```
π_{name}(σ_{dept='CS' AND gpa>3.5}(STUDENT))

Step 1: σ_{dept='CS' AND gpa>3.5}(STUDENT) → {Alice, Carol}
Step 2: π_{name}(...) → {Alice, Carol}

Result:
┌───────┐
│ name  │
├───────┤
│ Alice │
│ Carol │
└───────┘
```

**SQL**:
```sql
SELECT DISTINCT name FROM student WHERE dept = 'CS' AND gpa > 3.5;
```

---

### 7.3 Union (∪) — Combine Two Relations

**Syntax**: R₁ ∪ R₂

Returns all tuples that are in R₁ OR R₂ (or both). Duplicates removed.

**Requirement**: R₁ and R₂ must be **union-compatible** — same number of attributes with matching domains.

```
CS_STUDENTS = π_{name}(σ_{dept='CS'}(STUDENT))     = {Alice, Carol, Eve}
MATH_STUDENTS = π_{name}(σ_{dept='Math'}(STUDENT))  = {Bob}

CS_STUDENTS ∪ MATH_STUDENTS:
┌───────┐
│ name  │
├───────┤
│ Alice │
│ Carol │
│ Eve   │
│ Bob   │
└───────┘
```

**SQL**:
```sql
SELECT name FROM student WHERE dept = 'CS'
UNION
SELECT name FROM student WHERE dept = 'Math';
```

---

### 7.4 Set Difference (−) — Rows in R₁ but not R₂

**Syntax**: R₁ − R₂

Returns tuples that are in R₁ but NOT in R₂. Also requires union-compatibility.

```
ALL_STUDENTS = π_{name}(STUDENT)    = {Alice, Bob, Carol, Dave, Eve}
CS_STUDENTS = π_{name}(σ_{dept='CS'}(STUDENT)) = {Alice, Carol, Eve}

ALL_STUDENTS − CS_STUDENTS:
┌───────┐
│ name  │
├───────┤
│ Bob   │
│ Dave  │
└───────┘
```

**SQL**:
```sql
SELECT name FROM student
EXCEPT
SELECT name FROM student WHERE dept = 'CS';
```

---

### 7.5 Cartesian Product (×) — All Combinations

**Syntax**: R₁ × R₂

Combines **every** tuple from R₁ with **every** tuple from R₂. If R₁ has m tuples and R₂ has n tuples, the result has m × n tuples.

```
Example with small relations:

A:                B:
┌────┬───────┐    ┌──────┬─────┐
│ id │ name  │    │ cid  │ cr  │
├────┼───────┤    ├──────┼─────┤
│  1 │ Alice │    │ DB01 │  4  │
│  2 │ Bob   │    │ OS02 │  3  │
└────┴───────┘    └──────┴─────┘

A × B:
┌────┬───────┬──────┬─────┐
│ id │ name  │ cid  │ cr  │
├────┼───────┼──────┼─────┤
│  1 │ Alice │ DB01 │  4  │    ← 2 × 2 = 4 tuples
│  1 │ Alice │ OS02 │  3  │
│  2 │ Bob   │ DB01 │  4  │
│  2 │ Bob   │ OS02 │  3  │
└────┴───────┴──────┴─────┘
```

On its own, Cartesian product is rarely useful (it produces too many rows). But combined with selection, it becomes a **join**:

```
σ_{STUDENT.id = ENROLLMENT.sid}(STUDENT × ENROLLMENT)

This is equivalent to:  STUDENT ⋈_{id=sid} ENROLLMENT  (a join!)
```

**SQL**:
```sql
-- Cartesian product:
SELECT * FROM student CROSS JOIN course;

-- With selection (= join):
SELECT * FROM student, enrollment WHERE student.id = enrollment.sid;
```

---

### 7.6 Rename (ρ) — Rename Relations/Attributes

**Syntax**: ρ<sub>S(B1, B2, ...)</sub>(R)

Renames relation R to S and/or renames its attributes. Essential for self-joins and disambiguating attributes.

```
ρ_{S(sid, sname, sdept, sgpa)}(STUDENT)

Now we can refer to it as S with attributes sid, sname, sdept, sgpa
```

**Use case — self-join**: "Find pairs of students in the same department"

```
S1 = ρ_{S1}(STUDENT)
S2 = ρ_{S2}(STUDENT)

π_{S1.name, S2.name}(σ_{S1.dept = S2.dept AND S1.id < S2.id}(S1 × S2))

Result:
┌─────────┬─────────┐
│ S1.name │ S2.name │
├─────────┼─────────┤
│ Alice   │ Carol   │
│ Alice   │ Eve     │
│ Carol   │ Eve     │
└─────────┴─────────┘
```

**SQL**:
```sql
SELECT s1.name, s2.name
FROM student s1, student s2
WHERE s1.dept = s2.dept AND s1.id < s2.id;
```

---

## 8. Relational Algebra — Extended Operations

These are **derived** operations — expressible using the six fundamentals but used so frequently they get their own notation.

### 8.1 Natural Join (⋈)

Combines two relations by matching on **all common attributes** (same name, same domain). Automatically performs equality on common attributes and eliminates duplicate columns.

```
STUDENT(id, name, dept, gpa)
ENROLLMENT(sid, cid, semester, grade)

-- These don't share column names, so natural join = Cartesian product.
-- Let's use a version where ENROLLMENT has 'id' instead of 'sid':

ENROLLMENT2(id, cid, semester, grade)

STUDENT ⋈ ENROLLMENT2:
┌────┬───────┬──────┬─────┬──────┬────────┬───────┐
│ id │ name  │ dept │ gpa │ cid  │semester│ grade │
├────┼───────┼──────┼─────┼──────┼────────┼───────┤
│  1 │ Alice │ CS   │ 3.9 │ DB01 │ 2026S1 │  A    │
│  1 │ Alice │ CS   │ 3.9 │ OS02 │ 2026S1 │  B+   │
│  2 │ Bob   │ Math │ 3.5 │ DB01 │ 2026S1 │  A-   │
│  3 │ Carol │ CS   │ 3.7 │ ML03 │ 2026S1 │  B    │
│  3 │ Carol │ CS   │ 3.7 │ DB01 │ 2026S1 │  A    │
│  5 │ Eve   │ CS   │ 3.1 │ OS02 │ 2026S1 │  C+   │
└────┴───────┴──────┴─────┴──────┴────────┴───────┘
```

Formally: R ⋈ S = π<sub>all unique attrs</sub>(σ<sub>R.common = S.common</sub>(R × S))

**SQL**:
```sql
SELECT * FROM student NATURAL JOIN enrollment2;
-- or:
SELECT * FROM student s JOIN enrollment e ON s.id = e.sid;
```

### 8.2 Theta Join (⋈<sub>θ</sub>) and Equi-Join

**Theta Join**: Cartesian product followed by a general condition θ.

```
R ⋈_{θ} S = σ_{θ}(R × S)
```

**Equi-Join**: Theta join where θ uses only = (equality).

```
STUDENT ⋈_{id=sid} ENROLLMENT

= σ_{STUDENT.id = ENROLLMENT.sid}(STUDENT × ENROLLMENT)
```

**SQL**:
```sql
SELECT * FROM student JOIN enrollment ON student.id = enrollment.sid;
```

### 8.3 Left/Right/Full Outer Join (⟕, ⟖, ⟗)

Regular joins lose tuples that don't have a match. Outer joins **preserve** unmatched tuples by filling missing attributes with NULL.

```
STUDENT ⟕ ENROLLMENT  (LEFT OUTER JOIN)

Keeps ALL students, even Dave (id=4) who has no enrollment:
┌────┬───────┬──────┬─────┬──────┬────────┬───────┐
│ id │ name  │ dept │ gpa │ cid  │semester│ grade │
├────┼───────┼──────┼─────┼──────┼────────┼───────┤
│  1 │ Alice │ CS   │ 3.9 │ DB01 │ 2026S1 │  A    │
│  1 │ Alice │ CS   │ 3.9 │ OS02 │ 2026S1 │  B+   │
│  2 │ Bob   │ Math │ 3.5 │ DB01 │ 2026S1 │  A-   │
│  3 │ Carol │ CS   │ 3.7 │ ML03 │ 2026S1 │  B    │
│  3 │ Carol │ CS   │ 3.7 │ DB01 │ 2026S1 │  A    │
│  4 │ Dave  │ EE   │ 3.2 │ NULL │  NULL  │ NULL  │  ← preserved!
│  5 │ Eve   │ CS   │ 3.1 │ OS02 │ 2026S1 │  C+   │
└────┴───────┴──────┴─────┴──────┴────────┴───────┘
```

| Type | Symbol | Preserves |
|---|---|---|
| Left outer join | ⟕ | All tuples from left relation |
| Right outer join | ⟖ | All tuples from right relation |
| Full outer join | ⟗ | All tuples from both relations |

### 8.4 Semijoin (⋉) and Antijoin (▷)

**Semijoin** R ⋉ S: Tuples from R that have a match in S (result only has R's attributes).

```
STUDENT ⋉_{id=sid} ENROLLMENT

= Students who have at least one enrollment:
┌────┬───────┬──────┬─────┐
│  1 │ Alice │ CS   │ 3.9 │
│  2 │ Bob   │ Math │ 3.5 │
│  3 │ Carol │ CS   │ 3.7 │
│  5 │ Eve   │ CS   │ 3.1 │
└────┴───────┴──────┴─────┘
```

**SQL**: `SELECT DISTINCT s.* FROM student s WHERE EXISTS (SELECT 1 FROM enrollment e WHERE e.sid = s.id);`

**Antijoin** R ▷ S: Tuples from R that have **no** match in S (the opposite of semijoin).

```
STUDENT ▷_{id=sid} ENROLLMENT

= Students with NO enrollments:
┌────┬──────┬──────┬─────┐
│  4 │ Dave │ EE   │ 3.2 │
└────┴──────┴──────┴─────┘
```

**SQL**: `SELECT * FROM student s WHERE NOT EXISTS (SELECT 1 FROM enrollment e WHERE e.sid = s.id);`

### 8.5 Division (÷)

**R ÷ S**: Returns tuples in R that are associated with **all** tuples in S. This is the relational algebra equivalent of "for all" queries.

**Example**: "Which students are enrolled in ALL courses?"

```
A = π_{sid, cid}(ENROLLMENT):       B = π_{cid}(COURSE):
┌────┬──────┐                       ┌──────┐
│ sid│ cid  │                       │ cid  │
├────┼──────┤                       ├──────┤
│  1 │ DB01 │                       │ DB01 │
│  1 │ OS02 │                       │ OS02 │
│  2 │ DB01 │                       │ ML03 │
│  3 │ ML03 │                       └──────┘
│  3 │ DB01 │
│  5 │ OS02 │
└────┴──────┘

A ÷ B = students enrolled in ALL of {DB01, OS02, ML03}

Result: EMPTY SET (no student is enrolled in all 3 courses)

-- If we change B to just {DB01}:
A ÷ π_{cid}(σ_{cid='DB01'}(COURSE))
= {1, 2, 3}  -- students enrolled in DB01
```

**Formal definition**: R ÷ S = π<sub>R−S</sub>(R) − π<sub>R−S</sub>((π<sub>R−S</sub>(R) × S) − R)

**SQL** (no direct DIVISION operator — use double NOT EXISTS):
```sql
-- Students enrolled in ALL courses
SELECT s.id
FROM student s
WHERE NOT EXISTS (
    SELECT c.cid FROM course c
    WHERE NOT EXISTS (
        SELECT 1 FROM enrollment e
        WHERE e.sid = s.id AND e.cid = c.cid
    )
);
```

### 8.6 Aggregation and Grouping (𝒢)

Not part of the original relational algebra, but essential in practice.

**Syntax**: <sub>G</sub>𝒢<sub>F(A)</sub>(R)

Where G = grouping attributes, F = aggregate function (SUM, AVG, COUNT, MIN, MAX), A = attribute to aggregate.

```
dept 𝒢 AVG(gpa), COUNT(*) (STUDENT)

Result:
┌──────┬──────────┬───────┐
│ dept │ avg_gpa  │ count │
├──────┼──────────┼───────┤
│ CS   │   3.567  │   3   │
│ Math │   3.500  │   1   │
│ EE   │   3.200  │   1   │
└──────┴──────────┴───────┘
```

**SQL**:
```sql
SELECT dept, AVG(gpa) AS avg_gpa, COUNT(*) AS count
FROM student
GROUP BY dept;
```

### SVG: Relational Algebra Operations

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 420" font-family="Segoe UI, Arial, sans-serif">
  <rect width="760" height="420" fill="#f8f9fa" rx="12"/>
  <text x="380" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Relational Algebra Operations</text>

  <!-- Fundamental -->
  <rect x="20" y="50" width="350" height="350" rx="10" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="195" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">6 Fundamental Operations</text>

  <rect x="35" y="90" width="320" height="38" rx="5" fill="#d6eaf8"/>
  <text x="55" y="114" font-size="13" fill="#1a5276" font-weight="bold">σ  Selection</text>
  <text x="220" y="114" font-size="11" fill="#555">Filter rows by condition</text>

  <rect x="35" y="135" width="320" height="38" rx="5" fill="#d6eaf8"/>
  <text x="55" y="159" font-size="13" fill="#1a5276" font-weight="bold">π  Projection</text>
  <text x="220" y="159" font-size="11" fill="#555">Choose columns (dedup)</text>

  <rect x="35" y="180" width="320" height="38" rx="5" fill="#d6eaf8"/>
  <text x="55" y="204" font-size="13" fill="#1a5276" font-weight="bold">∪  Union</text>
  <text x="220" y="204" font-size="11" fill="#555">Combine two relations</text>

  <rect x="35" y="225" width="320" height="38" rx="5" fill="#d6eaf8"/>
  <text x="55" y="249" font-size="13" fill="#1a5276" font-weight="bold">−  Set Difference</text>
  <text x="220" y="249" font-size="11" fill="#555">In R₁ but not R₂</text>

  <rect x="35" y="270" width="320" height="38" rx="5" fill="#d6eaf8"/>
  <text x="55" y="294" font-size="13" fill="#1a5276" font-weight="bold">×  Cartesian Product</text>
  <text x="220" y="294" font-size="11" fill="#555">All row combinations</text>

  <rect x="35" y="315" width="320" height="38" rx="5" fill="#d6eaf8"/>
  <text x="55" y="339" font-size="13" fill="#1a5276" font-weight="bold">ρ  Rename</text>
  <text x="220" y="339" font-size="11" fill="#555">Rename relation/attrs</text>

  <text x="195" y="382" text-anchor="middle" font-size="10" fill="#2980b9" font-style="italic">All queries can be expressed with just these 6</text>

  <!-- Extended -->
  <rect x="390" y="50" width="350" height="350" rx="10" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="565" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">Extended (Derived) Operations</text>

  <rect x="405" y="90" width="320" height="38" rx="5" fill="#d5f5e3"/>
  <text x="425" y="114" font-size="13" fill="#145a32" font-weight="bold">⋈  Natural Join</text>
  <text x="560" y="114" font-size="11" fill="#555">Match on common attrs</text>

  <rect x="405" y="135" width="320" height="38" rx="5" fill="#d5f5e3"/>
  <text x="425" y="159" font-size="13" fill="#145a32" font-weight="bold">⋈θ Theta / Equi Join</text>
  <text x="580" y="159" font-size="11" fill="#555">Join with condition</text>

  <rect x="405" y="180" width="320" height="38" rx="5" fill="#d5f5e3"/>
  <text x="425" y="204" font-size="13" fill="#145a32" font-weight="bold">⟕⟖⟗ Outer Joins</text>
  <text x="580" y="204" font-size="11" fill="#555">Preserve unmatched</text>

  <rect x="405" y="225" width="320" height="38" rx="5" fill="#d5f5e3"/>
  <text x="425" y="249" font-size="13" fill="#145a32" font-weight="bold">⋉▷ Semi / Anti Join</text>
  <text x="580" y="249" font-size="11" fill="#555">Exists / not exists</text>

  <rect x="405" y="270" width="320" height="38" rx="5" fill="#d5f5e3"/>
  <text x="425" y="294" font-size="13" fill="#145a32" font-weight="bold">÷  Division</text>
  <text x="580" y="294" font-size="11" fill="#555">"For all" queries</text>

  <rect x="405" y="315" width="320" height="38" rx="5" fill="#d5f5e3"/>
  <text x="425" y="339" font-size="13" fill="#145a32" font-weight="bold">𝒢  Aggregation</text>
  <text x="580" y="339" font-size="11" fill="#555">SUM, AVG, COUNT, ...</text>

  <text x="565" y="382" text-anchor="middle" font-size="10" fill="#27ae60" font-style="italic">Derivable from the 6 fundamentals (except 𝒢)</text>
</svg>

---

## 9. Relational Calculus — Tuple Relational Calculus (TRC)

While relational algebra is **procedural** (how to compute), relational calculus is **declarative** (what you want). SQL is more closely based on calculus.

### TRC Basics

In TRC, a query has the form:

```
{ t | P(t) }
```

Read as: "the set of all tuples t such that predicate P(t) is true."

Where `t` is a **tuple variable** that ranges over tuples of a relation.

### Notation

| Symbol | Meaning |
|---|---|
| t ∈ R | Tuple variable t ranges over relation R |
| t[A] or t.A | The value of attribute A in tuple t |
| ∃ | "there exists" (existential quantifier) |
| ∀ | "for all" (universal quantifier) |
| ∧ | AND |
| ∨ | OR |
| ¬ | NOT |

### Examples

#### Example 1: All CS students
```
{ t | t ∈ STUDENT ∧ t[dept] = 'CS' }

English: "All tuples t from STUDENT where dept is CS"

SQL:     SELECT * FROM student WHERE dept = 'CS';
Algebra: σ_{dept='CS'}(STUDENT)
```

#### Example 2: Names of CS students with GPA > 3.5
```
{ t[name] | t ∈ STUDENT ∧ t[dept] = 'CS' ∧ t[gpa] > 3.5 }

English: "The name of every STUDENT tuple where dept = CS and gpa > 3.5"

SQL:     SELECT name FROM student WHERE dept = 'CS' AND gpa > 3.5;
Algebra: π_{name}(σ_{dept='CS' ∧ gpa>3.5}(STUDENT))
```

#### Example 3: Students enrolled in 'DB01' (uses ∃)
```
{ t[name] | t ∈ STUDENT ∧ ∃e (e ∈ ENROLLMENT ∧ e[sid] = t[id] ∧ e[cid] = 'DB01') }

English: "Names of students for whom THERE EXISTS an enrollment in DB01"

SQL:
SELECT s.name FROM student s
WHERE EXISTS (SELECT 1 FROM enrollment e WHERE e.sid = s.id AND e.cid = 'DB01');
```

#### Example 4: Students enrolled in ALL courses (uses ∀)
```
{ t[name] | t ∈ STUDENT ∧ ∀c (c ∈ COURSE → ∃e (e ∈ ENROLLMENT ∧ e[sid] = t[id] ∧ e[cid] = c[cid])) }

English: "Names of students where FOR ALL courses, THERE EXISTS an enrollment linking them"

SQL:
SELECT s.name FROM student s
WHERE NOT EXISTS (
    SELECT 1 FROM course c
    WHERE NOT EXISTS (
        SELECT 1 FROM enrollment e WHERE e.sid = s.id AND e.cid = c.cid
    )
);
```

> **Note**: ∀x P(x) is logically equivalent to ¬∃x ¬P(x) — "for all" = "there does not exist a counterexample."

### Safety in TRC

A TRC expression must be **safe** — it must produce a **finite** result. The expression `{ t | ¬(t ∈ STUDENT) }` is unsafe because it returns all tuples in the universe that are NOT in STUDENT (infinite!).

**Safety rule**: Every variable in the result must be bound to a finite relation.

---

## 10. Relational Calculus — Domain Relational Calculus (DRC)

DRC uses **domain variables** (individual attribute values) instead of tuple variables.

### DRC Basics

A query has the form:
```
{ <x₁, x₂, ..., xₙ> | P(x₁, x₂, ..., xₙ) }
```

Where x₁, x₂, etc. are domain variables (representing individual column values, not whole tuples).

### Examples

#### Example 1: All CS students
```
{ <i, n, d, g> | <i, n, d, g> ∈ STUDENT ∧ d = 'CS' }

English: "All combinations of (id, name, dept, gpa) from STUDENT where dept is CS"
```

#### Example 2: Names of students in 'DB01'
```
{ <n> | ∃i ∃d ∃g (<i, n, d, g> ∈ STUDENT ∧ ∃c ∃s ∃gr (<i, c, s, gr> ∈ ENROLLMENT ∧ c = 'DB01')) }

English: "Names n where there exist values i, d, g such that (i, n, d, g) is in STUDENT
          and there exist c, s, gr such that (i, c, s, gr) is in ENROLLMENT with c = DB01"
```

### TRC vs. DRC

| Aspect | TRC | DRC |
|---|---|---|
| Variables | Tuple variables (whole rows) | Domain variables (individual values) |
| Notation | t[name] | n (standalone domain variable) |
| Closer to | SQL (works with rows) | QBE (Query By Example) |
| Power | Equivalent | Equivalent |

> **Fundamental Theorem of Relational Completeness**: Relational algebra, TRC (safe), and DRC (safe) are **equivalent in expressive power**. Any query expressible in one can be expressed in the others.

---

## 11. Algebra vs. Calculus vs. SQL — Comparison

Let's express the same queries in all three languages.

### Query 1: "Names of CS students with GPA > 3.5"

| Language | Expression |
|---|---|
| **Algebra** | π<sub>name</sub>(σ<sub>dept='CS' ∧ gpa>3.5</sub>(STUDENT)) |
| **TRC** | { t[name] \| t ∈ STUDENT ∧ t[dept]='CS' ∧ t[gpa]>3.5 } |
| **DRC** | { \<n\> \| ∃i ∃d ∃g (\<i,n,d,g\> ∈ STUDENT ∧ d='CS' ∧ g>3.5) } |
| **SQL** | `SELECT name FROM student WHERE dept='CS' AND gpa > 3.5;` |

### Query 2: "Students enrolled in the Databases course"

| Language | Expression |
|---|---|
| **Algebra** | π<sub>name</sub>(STUDENT ⋈<sub>id=sid</sub> (σ<sub>cid='DB01'</sub>(ENROLLMENT))) |
| **TRC** | { t[name] \| t ∈ STUDENT ∧ ∃e(e ∈ ENROLLMENT ∧ e[sid]=t[id] ∧ e[cid]='DB01') } |
| **SQL** | `SELECT s.name FROM student s JOIN enrollment e ON s.id=e.sid WHERE e.cid='DB01';` |

### Query 3: "Departments that have no students"

| Language | Expression |
|---|---|
| **Algebra** | π<sub>dept_name</sub>(DEPARTMENT) − π<sub>dept_name</sub>(DEPARTMENT ⋈ STUDENT) |
| **TRC** | { t[dept_name] \| t ∈ DEPARTMENT ∧ ¬∃s(s ∈ STUDENT ∧ s[dept_id]=t[dept_id]) } |
| **SQL** | `SELECT dept_name FROM department d WHERE NOT EXISTS (SELECT 1 FROM student s WHERE s.dept_id = d.dept_id);` |

### Query 4: "Average GPA by department"

| Language | Expression |
|---|---|
| **Algebra** | <sub>dept</sub>𝒢<sub>AVG(gpa)</sub>(STUDENT) |
| **TRC/DRC** | Cannot express aggregation (calculus has no aggregate functions) |
| **SQL** | `SELECT dept, AVG(gpa) FROM student GROUP BY dept;` |

### Summary: Procedural vs. Declarative

```
Relational Algebra (Procedural)          Relational Calculus (Declarative)
────────────────────────────────          ──────────────────────────────────
"HOW to get the data"                    "WHAT data I want"
Step-by-step operations                  Describe the result's properties
σ, π, ⋈, ∪, −, ×                        { t | P(t) }
Like an algorithm                        Like a specification
Maps to query execution plan             SQL is based on this
```

---

## 12. Worked Examples — Beginner, Intermediate, Advanced

### Beginner: Student-Course Database

**Schema**:
```
STUDENT(sid, sname, dept, gpa)
COURSE(cid, title, credits)
ENROLLMENT(sid, cid, grade)
```

**Q1**: Names of all students (projection)
```
Algebra: π_{sname}(STUDENT)
TRC:     { t[sname] | t ∈ STUDENT }
SQL:     SELECT DISTINCT sname FROM student;
```

**Q2**: Students with GPA > 3.5 (selection + projection)
```
Algebra: π_{sname, gpa}(σ_{gpa > 3.5}(STUDENT))
TRC:     { t[sname], t[gpa] | t ∈ STUDENT ∧ t[gpa] > 3.5 }
SQL:     SELECT sname, gpa FROM student WHERE gpa > 3.5;
```

**Q3**: Students enrolled in 'Databases' (join + selection + projection)
```
Algebra: π_{sname}(STUDENT ⋈_{sid} ENROLLMENT ⋈_{cid} σ_{title='Databases'}(COURSE))
TRC:     { t[sname] | t ∈ STUDENT ∧ ∃e(e ∈ ENROLLMENT ∧ e[sid]=t[sid] ∧
           ∃c(c ∈ COURSE ∧ c[cid]=e[cid] ∧ c[title]='Databases')) }
SQL:     SELECT s.sname FROM student s
         JOIN enrollment e ON s.sid = e.sid
         JOIN course c ON e.cid = c.cid
         WHERE c.title = 'Databases';
```

---

### Intermediate: E-Commerce Database

**Schema**:
```
CUSTOMER(cid, name, city)
PRODUCT(pid, pname, price, category)
ORDERS(oid, cid, order_date, total)
ORDER_ITEM(oid, pid, qty, unit_price)
```

**Q4**: Customers from Mumbai who ordered products over $100
```
Algebra:
π_{name}(
    σ_{city='Mumbai'}(CUSTOMER) ⋈_{cid}
    ORDERS ⋈_{oid}
    σ_{unit_price > 100}(ORDER_ITEM)
)

SQL:
SELECT DISTINCT c.name
FROM customer c
JOIN orders o ON c.cid = o.cid
JOIN order_item oi ON o.oid = oi.oid
WHERE c.city = 'Mumbai' AND oi.unit_price > 100;
```

**Q5**: Products that no one has ordered (antijoin)
```
Algebra: PRODUCT ▷_{pid} ORDER_ITEM
       = π_{pid, pname, price, category}(PRODUCT) − π_{pid, pname, price, category}(PRODUCT ⋈ ORDER_ITEM)

SQL:
SELECT * FROM product p
WHERE NOT EXISTS (SELECT 1 FROM order_item oi WHERE oi.pid = p.pid);
```

**Q6**: Total revenue by category
```
Algebra: category 𝒢 SUM(qty * unit_price) (ORDER_ITEM ⋈_{pid} PRODUCT)

SQL:
SELECT p.category, SUM(oi.qty * oi.unit_price) AS revenue
FROM order_item oi
JOIN product p ON oi.pid = p.pid
GROUP BY p.category;
```

---

### Advanced: Banking Database

**Schema**:
```
BRANCH(bid, bname, city, assets)
ACCOUNT(aid, bid, holder, balance, type)
TRANSACTION(tid, from_aid, to_aid, amount, txn_date)
```

**Q7**: Customers who have accounts at ALL branches in Mumbai (division!)
```
Algebra:
Let MUMBAI_BRANCHES = π_{bid}(σ_{city='Mumbai'}(BRANCH))
Let CUST_BRANCHES = π_{holder, bid}(ACCOUNT)

CUST_BRANCHES ÷ MUMBAI_BRANCHES

SQL:
SELECT a.holder
FROM account a
JOIN branch b ON a.bid = b.bid
WHERE b.city = 'Mumbai'
GROUP BY a.holder
HAVING COUNT(DISTINCT a.bid) = (SELECT COUNT(*) FROM branch WHERE city = 'Mumbai');
```

**Q8**: Branches where every account has balance > 10000 (universal quantifier)
```
TRC: { t[bname] | t ∈ BRANCH ∧ ∀a(a ∈ ACCOUNT ∧ a[bid] = t[bid] → a[balance] > 10000) }

SQL:
SELECT b.bname
FROM branch b
WHERE NOT EXISTS (
    SELECT 1 FROM account a
    WHERE a.bid = b.bid AND a.balance <= 10000
);
```

**Q9**: Accounts that sent money to themselves (through intermediary)
```
Algebra:
π_{T1.from_aid}(
    σ_{T1.to_aid = T2.from_aid ∧ T2.to_aid = T1.from_aid ∧ T1.tid ≠ T2.tid}(
        ρ_{T1}(TRANSACTION) × ρ_{T2}(TRANSACTION)
    )
)

SQL:
SELECT DISTINCT t1.from_aid
FROM transaction t1
JOIN transaction t2 ON t1.to_aid = t2.from_aid AND t2.to_aid = t1.from_aid
WHERE t1.tid != t2.tid;
```

---

## 13. Common Interview Questions

### Q1: What is a relation in the relational model? How is it different from a table?
> A **relation** is a mathematical concept — a set of tuples with no duplicates and no ordering. A **table** is the SQL implementation, which may allow duplicates (without DISTINCT) and may have an implicit order. Strictly, a table is a multiset (bag) of rows; a relation is a set.

### Q2: What is the difference between a superkey, candidate key, and primary key?
> A **superkey** is any set of attributes that uniquely identifies tuples (may include extras). A **candidate key** is a minimal superkey — remove any attribute and it loses uniqueness. A **primary key** is the candidate key chosen as the official identifier. Others become alternate keys.

### Q3: Why can't a primary key be NULL?
> Because the primary key uniquely identifies each tuple. NULL represents an unknown value, and you can't compare unknowns — if two PKs were NULL, we couldn't determine if they're the same tuple or different. This would violate entity integrity.

### Q4: What is referential integrity? What happens when it's violated?
> Referential integrity ensures that every foreign key value either matches an existing primary key in the referenced table or is NULL. When violated (e.g., deleting a department that still has students), the DBMS enforces the chosen action: RESTRICT (block), CASCADE (propagate), SET NULL, or SET DEFAULT.

### Q5: Explain the six fundamental operations of relational algebra.
> 1. **σ (Selection)**: Filter rows by condition  
> 2. **π (Projection)**: Choose columns, remove duplicates  
> 3. **∪ (Union)**: Combine compatible relations  
> 4. **− (Difference)**: Rows in R₁ but not R₂  
> 5. **× (Cartesian Product)**: All row combinations  
> 6. **ρ (Rename)**: Rename relation or attributes  
> Together they are **relationally complete** — they can express any computable query on relations.

### Q6: What is the difference between relational algebra and relational calculus?
> **Algebra** is procedural: you specify a sequence of operations to produce the result. **Calculus** is declarative: you describe what the result should look like. They are equivalent in expressive power (Codd's theorem). SQL is closer to calculus (declarative), while the query optimizer converts it to an algebra-like execution plan.

### Q7: How do you express "for all" in SQL? Give an example.
> SQL has no direct FORALL. Use double negation: **NOT EXISTS ... NOT EXISTS**. Example: "Students enrolled in all courses":
> ```sql
> SELECT s.name FROM student s
> WHERE NOT EXISTS (
>     SELECT 1 FROM course c WHERE NOT EXISTS (
>         SELECT 1 FROM enrollment e WHERE e.sid = s.id AND e.cid = c.cid
>     ));
> ```
> Read as: "students for whom there does NOT EXIST a course for which there does NOT EXIST an enrollment."

### Q8: What is the division operation? When do you use it?
> Division R ÷ S returns tuples from R associated with ALL tuples in S. It answers "for all" queries: "which students took all courses?", "which suppliers supply all parts?" In SQL, it's implemented with double NOT EXISTS or GROUP BY + HAVING COUNT.

### Q9: What is three-valued logic? Why does SQL need it?
> Because of NULL, SQL uses three truth values: TRUE, FALSE, UNKNOWN. Any comparison with NULL yields UNKNOWN (even NULL = NULL). WHERE clauses only return rows where the condition is TRUE (not UNKNOWN). This is why `WHERE x = NULL` returns nothing — you must use `WHERE x IS NULL`.

### Q10: What is union compatibility? Why is it required?
> Two relations are union-compatible if they have the **same number of attributes** and corresponding attributes have **compatible domains**. Union (∪), Difference (−), and Intersection (∩) all require this because the result must be a valid relation with a consistent schema.

### Q11: What is the difference between a natural join and an equi-join?
> A **natural join** automatically matches on ALL attributes with the same name in both relations and eliminates duplicate columns. An **equi-join** lets you explicitly specify which attributes to match (even if names differ) and may keep both columns. Natural joins are dangerous if relations share column names unintentionally.

### Q12: What is Codd's theorem?
> **Codd's theorem** (the Fundamental Theorem of Relational Completeness) states that relational algebra and safe relational calculus (both TRC and DRC) have the **same expressive power**. Any query expressible in one can be expressed in the others. A query language is "relationally complete" if it can express everything relational algebra can.

---

## 14. Key Takeaways

| # | Concept | One-Line Summary |
|---|---|---|
| 1 | **Relational Model** | Data as mathematical relations (tables of tuples); proposed by Codd 1970 |
| 2 | **Relation** | A set of tuples — no duplicates, no ordering, atomic values |
| 3 | **Degree** | Number of attributes (columns) in a relation |
| 4 | **Cardinality** | Number of tuples (rows) in a relation instance |
| 5 | **Domain** | Set of permissible values for an attribute (maps to SQL data types) |
| 6 | **NULL** | Not a value — means unknown/inapplicable; creates 3-valued logic |
| 7 | **Superkey** | Any unique identifier (may have extras) |
| 8 | **Candidate Key** | Minimal superkey — no redundant attributes |
| 9 | **Primary Key** | Chosen candidate key — unique + NOT NULL |
| 10 | **Foreign Key** | References a PK in another table — enforces referential integrity |
| 11 | **Entity Integrity** | PK must be unique and NOT NULL |
| 12 | **Referential Integrity** | FK must match an existing PK (or be NULL) |
| 13 | **σ Selection** | Filter rows by condition (= WHERE) |
| 14 | **π Projection** | Choose columns, deduplicate (= SELECT DISTINCT) |
| 15 | **⋈ Join** | Combine related tuples from two relations (= JOIN) |
| 16 | **÷ Division** | "For all" queries — tuples associated with all in S |
| 17 | **TRC** | Declarative calculus with tuple variables: { t \| P(t) } |
| 18 | **DRC** | Declarative calculus with domain variables: { \<x,y\> \| P(x,y) } |
| 19 | **Codd's Theorem** | Algebra = safe TRC = safe DRC (equal expressive power) |
| 20 | **SQL is calculus-based** | Declarative (WHAT), but optimizer converts to algebra (HOW) |

---

<p style="text-align: center; margin-top: 40px;">
  <a href="03_relational_model.md" download="03_relational_model.md"
     style="display: inline-block; padding: 14px 36px; font-size: 16px; font-weight: bold;
            color: #fff; background-color: #e67e22; border-radius: 8px;
            text-decoration: none; box-shadow: 0 4px 6px rgba(0,0,0,0.15);">
    Download 03_relational_model.md
  </a>
</p>

---

*Ready? Say **"next"** to proceed to **Phase 4 — SQL Fundamentals (DDL + DML + DQL)**.*