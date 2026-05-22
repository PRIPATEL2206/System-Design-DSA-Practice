# Phase 2 — Data Modeling & ER Diagrams

> **Goal**: Master the Entity-Relationship (ER) model — the primary tool for designing databases before writing a single line of SQL. Understand entities, attributes, relationships, cardinality, participation, weak entities, ISA hierarchies, and the systematic rules for converting an ER diagram into a relational schema.

---

## Table of Contents

1. [What is Data Modeling?](#1-what-is-data-modeling)
2. [The ER Model — Overview](#2-the-er-model--overview)
3. [Entities and Entity Sets](#3-entities-and-entity-sets)
4. [Attributes — All Types](#4-attributes--all-types)
5. [Relationships and Relationship Sets](#5-relationships-and-relationship-sets)
6. [Cardinality Constraints](#6-cardinality-constraints)
7. [Participation Constraints](#7-participation-constraints)
8. [Weak Entities](#8-weak-entities)
9. [ISA (Generalization/Specialization) Hierarchies](#9-isa-generalizationspecialization-hierarchies)
10. [ER Diagram Notation — Complete Symbol Reference](#10-er-diagram-notation--complete-symbol-reference)
11. [Full ER Diagram — University System (SVG)](#11-full-er-diagram--university-system-svg)
12. [ER → Relational Mapping Rules](#12-er--relational-mapping-rules)
13. [Mapping the University ER Diagram to SQL](#13-mapping-the-university-er-diagram-to-sql)
14. [Worked Examples — Beginner, Intermediate, Advanced](#14-worked-examples--beginner-intermediate-advanced)
15. [Common Mistakes in ER Modeling](#15-common-mistakes-in-er-modeling)
16. [Extended ER (EER) Concepts](#16-extended-er-eer-concepts)
17. [Common Interview Questions](#17-common-interview-questions)
18. [Key Takeaways](#18-key-takeaways)

---

## 1. What is Data Modeling?

### Definition

**Data modeling** is the process of creating a visual representation of a system's data — its structure, relationships, and constraints — before implementing it in a database.

> **Analogy — Blueprint before building:**  
> You wouldn't build a house by stacking bricks randomly. You'd draw a blueprint first — rooms, doors, plumbing, electrical. Data modeling is the **blueprint** for your database. The ER diagram is the most common blueprint format.

### Why Model Before Coding?

| Without Data Modeling | With Data Modeling |
|---|---|
| Tables designed ad-hoc → redundancy, missing relationships | Clean schema from the start |
| Requirements discovered mid-development | Requirements captured early |
| Expensive refactoring later | Changes cheap in the diagram phase |
| Miscommunication between stakeholders | Shared visual language |

### Levels of Data Modeling

| Level | Purpose | Output | Audience |
|---|---|---|---|
| **Conceptual** | High-level business view | ER Diagram (entities, relationships) | Business stakeholders, analysts |
| **Logical** | Detailed structure, no physical details | Tables, columns, keys, constraints | Database designers, developers |
| **Physical** | Implementation-specific | DDL (CREATE TABLE), indexes, partitions | DBA, developers |

```
Conceptual:  Student ──enrolls in──▶ Course
                 ↓
Logical:     students(id, name, dept) ←FK→ enrollments(sid, cid) ←FK→ courses(id, title)
                 ↓
Physical:    CREATE TABLE students (...) ENGINE=InnoDB PARTITION BY ...
```

---

## 2. The ER Model — Overview

The **Entity-Relationship Model** was proposed by **Peter Chen** in 1976. It provides a graphical notation for describing the conceptual schema of a database.

### Core Components

| Component | Symbol | Description |
|---|---|---|
| **Entity** | Rectangle | A "thing" in the real world (Student, Course, Product) |
| **Attribute** | Oval / Ellipse | A property of an entity (name, GPA, price) |
| **Relationship** | Diamond | An association between entities (enrolls_in, works_for) |
| **Key Attribute** | Underlined oval | Uniquely identifies each entity instance (student_id) |

### The ER Modeling Process

```
1. Identify ENTITIES        →  What "things" exist? (Student, Course, Department)
2. Identify ATTRIBUTES      →  What properties do they have? (name, GPA, credits)
3. Identify RELATIONSHIPS   →  How are entities connected? (enrolls_in, teaches)
4. Determine CARDINALITY    →  1:1, 1:N, or M:N?
5. Determine PARTICIPATION  →  Total or partial?
6. Identify KEYS            →  What uniquely identifies each entity?
7. Handle SPECIAL CASES     →  Weak entities, ISA hierarchies, multi-valued attributes
```

---

## 3. Entities and Entity Sets

### Definition

An **entity** is a distinguishable real-world object or concept about which data is stored.

An **entity set** (or entity type) is the collection of all entities of the same type. Think of it as a "class" in OOP.

```
Entity Set:    STUDENT
Entity:        Alice (student_id=1, name="Alice", dept="CS", gpa=3.9)
Entity:        Bob   (student_id=2, name="Bob", dept="Math", gpa=3.5)
```

### Strong Entities vs. Weak Entities

| | Strong Entity | Weak Entity |
|---|---|---|
| **Has its own primary key?** | Yes | No — depends on owner entity |
| **Can exist independently?** | Yes | No — existence depends on owner |
| **ER Symbol** | Single-line rectangle | Double-line rectangle |
| **Example** | Student, Course, Employee | Dependent (of Employee), Room (of Building) |

**Strong Entity Example**: A `Student` exists independently. It has `student_id` as its own primary key.

**Weak Entity Example**: A `Dependent` (child of an employee for insurance) can't be uniquely identified by itself. It needs the `employee_id` of its parent employee + its own partial key (like `dependent_name`).

```
Employee (strong)             Dependent (weak)
┌──────────────────┐          ╔═══════════════════════════╗
│ employee_id (PK) │──has──▶  ║ dependent_name (partial)  ║
│ name             │          ║ birth_date                ║
│ salary           │          ║ relationship              ║
└──────────────────┘          ╚═══════════════════════════╝
                              PK = (employee_id, dependent_name)
```

---

## 4. Attributes — All Types

Attributes describe the properties of entities or relationships.

### 4.1 Simple (Atomic) Attribute

Cannot be divided further.

```
Examples:  student_id, gpa, price, age, email
```

### 4.2 Composite Attribute

Can be broken down into sub-attributes.

```
full_name
├── first_name
├── middle_name
└── last_name

address
├── street
├── city
├── state
└── zip_code
```

**ER Representation**: The composite attribute has child ovals branching from it.

**SQL Mapping**: Typically stored as separate columns:
```sql
CREATE TABLE students (
    student_id   INT PRIMARY KEY,
    first_name   VARCHAR(50),
    middle_name  VARCHAR(50),
    last_name    VARCHAR(50),
    street       VARCHAR(200),
    city         VARCHAR(100),
    state        CHAR(2),
    zip_code     CHAR(10)
);
```

### 4.3 Multi-Valued Attribute

An attribute that can have **multiple values** for a single entity.

```
A student can have multiple phone numbers:
  Alice → {555-1234, 555-5678, 555-9999}

A product can have multiple colors:
  T-Shirt → {Red, Blue, Black}
```

**ER Representation**: Double-line oval.

**SQL Mapping**: Create a **separate table** (because a cell can't hold multiple values in 1NF):
```sql
-- Separate table for multi-valued attribute
CREATE TABLE student_phones (
    student_id   INT REFERENCES students(student_id),
    phone_number VARCHAR(20),
    PRIMARY KEY (student_id, phone_number)
);
```

### 4.4 Derived Attribute

A value that can be **computed** from other stored attributes. Not stored — calculated on the fly.

```
age        → derived from birth_date and current date
total_cost → derived from unit_price × quantity
years_employed → derived from hire_date and current date
```

**ER Representation**: Dashed-line oval.

**SQL Mapping**: Typically NOT stored — computed in queries or as generated columns:
```sql
-- Computed in a query:
SELECT name, EXTRACT(YEAR FROM AGE(birth_date)) AS age FROM students;

-- Or as a generated column (PostgreSQL 12+):
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    birth_date  DATE,
    age         INT GENERATED ALWAYS AS (EXTRACT(YEAR FROM AGE(birth_date))) STORED
);
```

### 4.5 Key Attribute

The attribute (or set of attributes) that **uniquely identifies** each entity in the entity set.

```
Student entity:      student_id (key)
Course entity:       course_id (key)
Composite key:       (student_id, course_id) for Enrollment
```

**ER Representation**: Underlined attribute name.

### SVG: Attribute Types

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 360" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="360" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">ER Attribute Types</text>

  <!-- Simple -->
  <ellipse cx="100" cy="90" rx="75" ry="30" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="100" y="95" text-anchor="middle" font-size="13" fill="#2980b9">name</text>
  <text x="100" y="140" text-anchor="middle" font-size="11" fill="#555">Simple</text>
  <text x="100" y="155" text-anchor="middle" font-size="10" fill="#888">(atomic, indivisible)</text>

  <!-- Key -->
  <ellipse cx="290" cy="90" rx="75" ry="30" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="290" y="95" text-anchor="middle" font-size="13" fill="#2980b9" text-decoration="underline">student_id</text>
  <text x="290" y="140" text-anchor="middle" font-size="11" fill="#555">Key</text>
  <text x="290" y="155" text-anchor="middle" font-size="10" fill="#888">(unique identifier)</text>

  <!-- Multi-valued -->
  <ellipse cx="480" cy="90" rx="75" ry="30" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <ellipse cx="480" cy="90" rx="68" ry="24" fill="none" stroke="#f39c12" stroke-width="1.5"/>
  <text x="480" y="95" text-anchor="middle" font-size="13" fill="#e67e22">phone_nums</text>
  <text x="480" y="140" text-anchor="middle" font-size="11" fill="#555">Multi-Valued</text>
  <text x="480" y="155" text-anchor="middle" font-size="10" fill="#888">(double oval)</text>

  <!-- Derived -->
  <ellipse cx="660" cy="90" rx="65" ry="30" fill="#fdf2f8" stroke="#8e44ad" stroke-width="2" stroke-dasharray="6,3"/>
  <text x="660" y="95" text-anchor="middle" font-size="13" fill="#8e44ad">age</text>
  <text x="660" y="140" text-anchor="middle" font-size="11" fill="#555">Derived</text>
  <text x="660" y="155" text-anchor="middle" font-size="10" fill="#888">(dashed oval)</text>

  <!-- Composite -->
  <ellipse cx="260" cy="260" rx="85" ry="30" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="260" y="265" text-anchor="middle" font-size="13" fill="#27ae60">full_name</text>

  <line x1="200" y1="280" x2="120" y2="315" stroke="#27ae60" stroke-width="1.5"/>
  <line x1="260" y1="290" x2="260" y2="315" stroke="#27ae60" stroke-width="1.5"/>
  <line x1="320" y1="280" x2="400" y2="315" stroke="#27ae60" stroke-width="1.5"/>

  <ellipse cx="120" cy="335" rx="65" ry="20" fill="#d5f5e3" stroke="#27ae60" stroke-width="1.5"/>
  <text x="120" y="340" text-anchor="middle" font-size="11" fill="#1e8449">first_name</text>
  <ellipse cx="260" cy="335" rx="70" ry="20" fill="#d5f5e3" stroke="#27ae60" stroke-width="1.5"/>
  <text x="260" y="340" text-anchor="middle" font-size="11" fill="#1e8449">middle_name</text>
  <ellipse cx="400" cy="335" rx="65" ry="20" fill="#d5f5e3" stroke="#27ae60" stroke-width="1.5"/>
  <text x="400" y="340" text-anchor="middle" font-size="11" fill="#1e8449">last_name</text>

  <text x="260" y="210" text-anchor="middle" font-size="11" fill="#555">Composite</text>
  <text x="260" y="225" text-anchor="middle" font-size="10" fill="#888">(has sub-attributes)</text>
</svg>

---

## 5. Relationships and Relationship Sets

### Definition

A **relationship** is an association between two or more entities.

A **relationship set** is the set of all relationships of the same type.

```
Entity:        Alice (Student)
Entity:        DB101 (Course)
Relationship:  Alice ──enrolls_in──▶ DB101
```

### Degree of a Relationship

The **degree** is the number of entity sets participating.

| Degree | Name | Example |
|---|---|---|
| **2** | Binary | Student ──enrolls_in──▶ Course |
| **3** | Ternary | Supplier ──supplies──▶ Part ──to──▶ Project |
| **1** | Unary (recursive) | Employee ──manages──▶ Employee |

#### Binary Relationship (most common)
```
Student ◇─── enrolls_in ───◇ Course
```

#### Ternary Relationship
```
Supplier ──┐
            ├──◇ supplies ◇───▶ stores data about which supplier
Part    ──┤                     supplies which part to which project
            ├──
Project ──┘
```

#### Unary (Recursive) Relationship
```
Employee ──manages──▶ Employee
   │                      │
 manager               subordinate
```

```sql
-- Recursive relationship in SQL:
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(100),
    manager_id  INT REFERENCES employees(emp_id)  -- self-reference
);
```

### Relationship Attributes

Relationships themselves can have attributes. An attribute belongs to the **relationship**, not to either entity.

```
Student ──enrolls_in──▶ Course
                │
            grade, semester   ← these belong to the enrollment, not to the student or course
```

```sql
CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(8) REFERENCES courses(course_id),
    grade       CHAR(2),       -- relationship attribute
    semester    CHAR(6),       -- relationship attribute
    PRIMARY KEY (student_id, course_id, semester)
);
```

---

## 6. Cardinality Constraints

**Cardinality** defines the maximum number of relationship instances an entity can participate in. It's the most critical design decision in ER modeling.

### The Three Types

### 6.1 One-to-One (1:1)

Each entity on one side is associated with **at most one** entity on the other side.

```
Person ──1────has────1──▶ Passport
  │                           │
 One person has              One passport belongs
 at most one passport        to at most one person
```

**Real-world examples**:
- Person ↔ Passport
- Country ↔ Capital
- Employee ↔ Parking Spot (assigned spot)
- CEO ↔ Company (one CEO per company at a time)

```sql
-- 1:1 — FK on either side (put on the side with total participation if possible)
CREATE TABLE persons (
    person_id   INT PRIMARY KEY,
    name        VARCHAR(100)
);

CREATE TABLE passports (
    passport_id  CHAR(10) PRIMARY KEY,
    person_id    INT UNIQUE REFERENCES persons(person_id),  -- UNIQUE enforces 1:1
    issue_date   DATE,
    expiry_date  DATE
);
```

### 6.2 One-to-Many (1:N)

One entity on the "one" side can be associated with **many** entities on the "many" side, but each entity on the "many" side links to **at most one** on the "one" side.

```
Department ──1────has────N──▶ Employee
    │                            │
 One dept has many           Each employee belongs
 employees                   to at most one dept
```

**Real-world examples**:
- Department → Employees (one dept, many employees)
- Customer → Orders (one customer, many orders)
- Author → Books (one author, many books — simplified)
- Mother → Children

```sql
-- 1:N — FK goes on the "many" side
CREATE TABLE departments (
    dept_id    INT PRIMARY KEY,
    dept_name  VARCHAR(100)
);

CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(100),
    dept_id    INT REFERENCES departments(dept_id)  -- FK on "many" side
);
```

### 6.3 Many-to-Many (M:N)

Each entity on either side can be associated with **many** entities on the other side.

```
Student ──M────enrolls_in────N──▶ Course
   │                                  │
 A student enrolls in            A course has
 many courses                    many students
```

**Real-world examples**:
- Student ↔ Course
- Author ↔ Book (co-authored books)
- Actor ↔ Movie
- Doctor ↔ Patient

```sql
-- M:N — requires a JUNCTION TABLE (also called bridge/join/associative table)
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100)
);

CREATE TABLE courses (
    course_id   CHAR(8) PRIMARY KEY,
    title       VARCHAR(200)
);

-- Junction table that resolves M:N
CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(8) REFERENCES courses(course_id),
    grade       CHAR(2),
    semester    CHAR(6),
    PRIMARY KEY (student_id, course_id, semester)
);
```

### SVG: Cardinality Types

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 740 400" font-family="Segoe UI, Arial, sans-serif">
  <rect width="740" height="400" fill="#f8f9fa" rx="12"/>
  <text x="370" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Cardinality Constraints</text>

  <!-- 1:1 -->
  <rect x="20" y="50" width="220" height="100" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="130" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">1 : 1  (One-to-One)</text>
  <rect x="35" y="85" width="80" height="28" rx="4" fill="#aed6f1"/>
  <text x="75" y="104" text-anchor="middle" font-size="11" fill="#1a5276">Person</text>
  <rect x="145" y="85" width="80" height="28" rx="4" fill="#aed6f1"/>
  <text x="185" y="104" text-anchor="middle" font-size="11" fill="#1a5276">Passport</text>
  <line x1="115" y1="99" x2="145" y2="99" stroke="#2980b9" stroke-width="2"/>
  <text x="130" y="130" text-anchor="middle" font-size="10" fill="#555">1 person ↔ 1 passport</text>
  <text x="130" y="144" text-anchor="middle" font-size="10" fill="#888">FK (UNIQUE) on either side</text>

  <!-- 1:N -->
  <rect x="260" y="50" width="220" height="100" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="370" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">1 : N  (One-to-Many)</text>
  <rect x="275" y="85" width="80" height="28" rx="4" fill="#a9dfbf"/>
  <text x="315" y="104" text-anchor="middle" font-size="11" fill="#145a32">Dept</text>
  <rect x="385" y="78" width="80" height="28" rx="4" fill="#a9dfbf"/>
  <text x="425" y="97" text-anchor="middle" font-size="11" fill="#145a32">Emp 1</text>
  <rect x="385" y="108" width="80" height="28" rx="4" fill="#a9dfbf"/>
  <text x="425" y="127" text-anchor="middle" font-size="11" fill="#145a32">Emp 2</text>
  <line x1="355" y1="92" x2="385" y2="92" stroke="#27ae60" stroke-width="1.5"/>
  <line x1="355" y1="99" x2="385" y2="122" stroke="#27ae60" stroke-width="1.5"/>
  <text x="370" y="148" text-anchor="middle" font-size="10" fill="#888">FK on the "many" (N) side</text>

  <!-- M:N -->
  <rect x="500" y="50" width="220" height="100" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="610" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#e67e22">M : N  (Many-to-Many)</text>
  <rect x="515" y="78" width="75" height="24" rx="4" fill="#fdebd0"/>
  <text x="552" y="95" text-anchor="middle" font-size="10" fill="#784212">Student 1</text>
  <rect x="515" y="106" width="75" height="24" rx="4" fill="#fdebd0"/>
  <text x="552" y="123" text-anchor="middle" font-size="10" fill="#784212">Student 2</text>
  <rect x="635" y="78" width="75" height="24" rx="4" fill="#fdebd0"/>
  <text x="672" y="95" text-anchor="middle" font-size="10" fill="#784212">Course 1</text>
  <rect x="635" y="106" width="75" height="24" rx="4" fill="#fdebd0"/>
  <text x="672" y="123" text-anchor="middle" font-size="10" fill="#784212">Course 2</text>
  <line x1="590" y1="90" x2="635" y2="90" stroke="#e67e22" stroke-width="1"/>
  <line x1="590" y1="90" x2="635" y2="118" stroke="#e67e22" stroke-width="1"/>
  <line x1="590" y1="118" x2="635" y2="90" stroke="#e67e22" stroke-width="1"/>
  <line x1="590" y1="118" x2="635" y2="118" stroke="#e67e22" stroke-width="1"/>
  <text x="610" y="148" text-anchor="middle" font-size="10" fill="#888">Needs a junction table</text>

  <!-- SQL Mapping Summary -->
  <rect x="40" y="180" width="660" height="200" rx="10" fill="#fff" stroke="#ddd" stroke-width="1"/>
  <text x="370" y="205" text-anchor="middle" font-size="14" font-weight="bold" fill="#2c3e50">How Each Maps to SQL</text>

  <!-- 1:1 mapping -->
  <rect x="60" y="220" width="195" height="145" rx="6" fill="#ebf5fb"/>
  <text x="157" y="240" text-anchor="middle" font-size="12" font-weight="bold" fill="#2980b9">1:1 Mapping</text>
  <text x="70" y="260" font-size="10" fill="#333">FK with UNIQUE on either</text>
  <text x="70" y="275" font-size="10" fill="#333">side, OR merge into one</text>
  <text x="70" y="290" font-size="10" fill="#333">table if both are total.</text>
  <text x="70" y="315" font-size="10" fill="#2980b9" font-style="italic">passports.person_id</text>
  <text x="70" y="330" font-size="10" fill="#2980b9" font-style="italic">  UNIQUE FK → persons</text>

  <!-- 1:N mapping -->
  <rect x="272" y="220" width="195" height="145" rx="6" fill="#eafaf1"/>
  <text x="370" y="240" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">1:N Mapping</text>
  <text x="282" y="260" font-size="10" fill="#333">FK on the "many" (N)</text>
  <text x="282" y="275" font-size="10" fill="#333">side pointing to the</text>
  <text x="282" y="290" font-size="10" fill="#333">"one" (1) side's PK.</text>
  <text x="282" y="315" font-size="10" fill="#27ae60" font-style="italic">employees.dept_id</text>
  <text x="282" y="330" font-size="10" fill="#27ae60" font-style="italic">  FK → departments</text>

  <!-- M:N mapping -->
  <rect x="484" y="220" width="195" height="145" rx="6" fill="#fef9e7"/>
  <text x="582" y="240" text-anchor="middle" font-size="12" font-weight="bold" fill="#e67e22">M:N Mapping</text>
  <text x="494" y="260" font-size="10" fill="#333">New junction table with</text>
  <text x="494" y="275" font-size="10" fill="#333">FKs to both entities.</text>
  <text x="494" y="290" font-size="10" fill="#333">PK = composite of both FKs.</text>
  <text x="494" y="315" font-size="10" fill="#e67e22" font-style="italic">enrollments(</text>
  <text x="494" y="330" font-size="10" fill="#e67e22" font-style="italic">  student_id FK, course_id FK)</text>
</svg>

---

## 7. Participation Constraints

**Participation** defines whether **every** entity in an entity set must participate in a relationship or not.

### Total Participation (Mandatory)

**Every** entity in the set **must** participate in the relationship. Shown with a **double line** in ER diagrams.

```
Every employee MUST work in a department.
    Employee ═══════works_in═══════ Department
    (double line = total participation)
```

**SQL**: Enforced with `NOT NULL` on the foreign key:
```sql
CREATE TABLE employees (
    emp_id   INT PRIMARY KEY,
    name     VARCHAR(100),
    dept_id  INT NOT NULL REFERENCES departments(dept_id)  -- NOT NULL = total participation
);
```

### Partial Participation (Optional)

Some entities **may or may not** participate. Shown with a **single line**.

```
Not every employee manages a department.
    Employee ───────manages─────── Department
    (single line = partial participation)
```

**SQL**: FK allows NULL:
```sql
CREATE TABLE departments (
    dept_id       INT PRIMARY KEY,
    dept_name     VARCHAR(100),
    manager_id    INT REFERENCES employees(emp_id)  -- NULL allowed = partial participation
);
```

### Combining Cardinality and Participation

```
Notation:  (min, max)  on each side

Employee  (1,1) ═════works_in═════ (0,N) Department
   │                                        │
   Every employee works in                  A dept can have 0..N
   exactly 1 dept (total, 1:1)              employees (partial on this side)
```

| Notation | Meaning | SQL Enforcement |
|---|---|---|
| (0, 1) | Optional, at most one | FK allows NULL |
| (1, 1) | Mandatory, exactly one | FK NOT NULL |
| (0, N) | Optional, any number | Standard FK or junction table |
| (1, N) | Mandatory, at least one | Harder — needs triggers or app logic |

### SVG: Participation Constraints

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 260" font-family="Segoe UI, Arial, sans-serif">
  <rect width="720" height="260" fill="#f8f9fa" rx="12"/>
  <text x="360" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">Participation Constraints</text>

  <!-- Total Participation -->
  <rect x="20" y="50" width="330" height="190" rx="10" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="185" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">Total Participation (mandatory)</text>
  <text x="185" y="95" text-anchor="middle" font-size="11" fill="#555">Every entity MUST participate</text>

  <rect x="45" y="115" width="100" height="35" rx="5" fill="#a9dfbf"/>
  <text x="95" y="137" text-anchor="middle" font-size="12" fill="#145a32">Employee</text>

  <!-- Double line (total) -->
  <line x1="145" y1="130" x2="210" y2="130" stroke="#27ae60" stroke-width="3"/>
  <line x1="145" y1="135" x2="210" y2="135" stroke="#27ae60" stroke-width="3"/>

  <polygon points="210,120 245,132 210,145" fill="#a9dfbf" stroke="#27ae60" stroke-width="1.5"/>
  <text x="228" y="136" text-anchor="middle" font-size="9" fill="#145a32">works</text>

  <line x1="245" y1="132" x2="280" y2="132" stroke="#27ae60" stroke-width="2"/>

  <rect x="280" y="115" width="100" height="35" rx="5" fill="#a9dfbf"/>
  <text x="330" y="137" text-anchor="middle" font-size="12" fill="#145a32">Department</text>

  <text x="185" y="175" text-anchor="middle" font-size="11" fill="#27ae60">Double line ═══</text>
  <text x="185" y="195" text-anchor="middle" font-size="11" fill="#555">SQL: FK is NOT NULL</text>
  <text x="185" y="215" text-anchor="middle" font-size="10" fill="#888">Every employee must have a dept</text>

  <!-- Partial Participation -->
  <rect x="370" y="50" width="330" height="190" rx="10" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="535" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#e67e22">Partial Participation (optional)</text>
  <text x="535" y="95" text-anchor="middle" font-size="11" fill="#555">Some entities MAY participate</text>

  <rect x="395" y="115" width="100" height="35" rx="5" fill="#fdebd0"/>
  <text x="445" y="137" text-anchor="middle" font-size="12" fill="#784212">Employee</text>

  <!-- Single line (partial) -->
  <line x1="495" y1="132" x2="560" y2="132" stroke="#e67e22" stroke-width="2"/>

  <polygon points="560,122 595,132 560,145" fill="#fdebd0" stroke="#e67e22" stroke-width="1.5"/>
  <text x="577" y="136" text-anchor="middle" font-size="9" fill="#784212">mgrs</text>

  <line x1="595" y1="132" x2="630" y2="132" stroke="#e67e22" stroke-width="2"/>

  <rect x="630" y="115" width="100" height="35" rx="5" fill="#fdebd0"/>
  <text x="680" y="137" text-anchor="middle" font-size="12" fill="#784212">Department</text>

  <text x="535" y="175" text-anchor="middle" font-size="11" fill="#e67e22">Single line ───</text>
  <text x="535" y="195" text-anchor="middle" font-size="11" fill="#555">SQL: FK allows NULL</text>
  <text x="535" y="215" text-anchor="middle" font-size="10" fill="#888">Not every emp manages a dept</text>
</svg>

---

## 8. Weak Entities

### Definition

A **weak entity** is an entity that:
1. **Cannot be uniquely identified** by its own attributes alone
2. **Depends on a strong (owner) entity** for its existence and identification

### Components

| Term | Description | ER Symbol |
|---|---|---|
| **Weak entity** | The dependent entity | Double-line rectangle |
| **Owner/Identifying entity** | The strong entity it depends on | Regular rectangle |
| **Identifying relationship** | The relationship connecting them | Double-line diamond |
| **Partial key (discriminator)** | Attribute that distinguishes weak entities under the same owner | Dashed-underlined attribute |

### Example 1: Employee and Dependents

An employee's dependent (e.g., spouse, child) can't be identified just by name — there could be many "Sarah"s across all employees. We need `employee_id` + `dependent_name` together.

```
EMPLOYEE (strong)                 DEPENDENT (weak)
┌────────────────┐    has         ╔═════════════════╗
│  employee_id   │═══════════◆══▶ ║ dependent_name  ║  (partial key: dashed underline)
│  name          │  (identifying  ║ birth_date      ║
│  salary        │   relationship)║ relationship    ║
└────────────────┘                ╚═════════════════╝

PK of DEPENDENT = (employee_id, dependent_name)
```

```sql
CREATE TABLE dependents (
    employee_id     INT REFERENCES employees(employee_id) ON DELETE CASCADE,
    dependent_name  VARCHAR(100),
    birth_date      DATE,
    relationship    VARCHAR(20),
    PRIMARY KEY (employee_id, dependent_name)  -- composite key using owner's PK
);
```

### Example 2: Building and Rooms

A room number (e.g., "101") is meaningless without knowing which building. Room 101 in the Science building is not the same as Room 101 in the Library.

```
BUILDING (strong)                 ROOM (weak)
┌────────────────┐    contains    ╔═════════════════╗
│  building_id   │═══════════◆══▶ ║ room_number     ║  (partial key)
│  building_name │                ║ capacity        ║
│  floors        │                ║ room_type       ║
└────────────────┘                ╚═════════════════╝

PK of ROOM = (building_id, room_number)
```

```sql
CREATE TABLE rooms (
    building_id   INT REFERENCES buildings(building_id) ON DELETE CASCADE,
    room_number   VARCHAR(10),
    capacity      INT,
    room_type     VARCHAR(50),
    PRIMARY KEY (building_id, room_number)
);
```

### Key Rule

> **Identifying relationship** always has **total participation** from the weak entity side — every dependent MUST belong to an employee. If the employee is deleted, their dependents are meaningless → `ON DELETE CASCADE`.

---

## 9. ISA (Generalization/Specialization) Hierarchies

### Definition

**ISA (is-a)** represents inheritance in ER modeling. A parent entity (superclass) generalizes common attributes, and child entities (subclasses) add specialized attributes.

```
           PERSON
          ┌──────┐
          │ pid  │
          │ name │
          │ dob  │
          └──┬───┘
             │
            ISA
           / | \
          /  |  \
    STUDENT  FACULTY  STAFF
    ┌──────┐ ┌──────┐ ┌──────┐
    │ gpa  │ │ rank │ │ shift│
    │ year │ │tenure│ │ dept │
    └──────┘ └──────┘ └──────┘
```

### Types of ISA Constraints

#### Overlap Constraint: Disjoint vs. Overlapping

| Type | Meaning | Example |
|---|---|---|
| **Disjoint** (d) | Entity can be in **at most one** subclass | A person is either a Student OR Faculty, not both |
| **Overlapping** (o) | Entity can be in **multiple** subclasses | A person can be both a Student AND a Staff member |

#### Completeness Constraint: Total vs. Partial

| Type | Meaning | Example |
|---|---|---|
| **Total** | Every superclass entity **must** be in at least one subclass | Every person must be a Student, Faculty, or Staff |
| **Partial** | A superclass entity **may not** be in any subclass | A person might just be a "Person" without further classification |

### SQL Mapping Strategies for ISA

#### Strategy 1: Single Table (Table-per-Hierarchy)
All entities in one table with a `type` discriminator column. Unused columns are NULL.

```sql
CREATE TABLE persons (
    pid     INT PRIMARY KEY,
    name    VARCHAR(100),
    dob     DATE,
    type    VARCHAR(10) CHECK (type IN ('student', 'faculty', 'staff')),
    -- Student-specific:
    gpa     DECIMAL(3,2),
    year    INT,
    -- Faculty-specific:
    rank    VARCHAR(50),
    tenure  BOOLEAN,
    -- Staff-specific:
    shift   VARCHAR(20),
    dept    VARCHAR(50)
);
```
**Pros**: No joins needed, simple queries.  
**Cons**: Lots of NULLs, wastes space, hard to enforce subclass constraints.

#### Strategy 2: Separate Tables (Table-per-Type)
One table for superclass, one for each subclass. Subclass tables have FK to superclass.

```sql
CREATE TABLE persons (
    pid   INT PRIMARY KEY,
    name  VARCHAR(100),
    dob   DATE
);

CREATE TABLE students (
    pid   INT PRIMARY KEY REFERENCES persons(pid),
    gpa   DECIMAL(3,2),
    year  INT
);

CREATE TABLE faculty (
    pid    INT PRIMARY KEY REFERENCES persons(pid),
    rank   VARCHAR(50),
    tenure BOOLEAN
);

CREATE TABLE staff (
    pid    INT PRIMARY KEY REFERENCES persons(pid),
    shift  VARCHAR(20),
    dept   VARCHAR(50)
);
```
**Pros**: Clean, no NULLs, easy to add new subclasses.  
**Cons**: Requires joins to get full information.

#### Strategy 3: Subclass Tables Only (Table-per-Concrete-Class)
No superclass table. Each subclass table contains all attributes (inherited + specialized).

```sql
CREATE TABLE students (
    pid   INT PRIMARY KEY,
    name  VARCHAR(100),
    dob   DATE,
    gpa   DECIMAL(3,2),
    year  INT
);

CREATE TABLE faculty (
    pid    INT PRIMARY KEY,
    name   VARCHAR(100),
    dob    DATE,
    rank   VARCHAR(50),
    tenure BOOLEAN
);

-- No 'persons' table at all
```
**Pros**: No joins, no NULLs.  
**Cons**: Redundant columns, hard to query "all persons", hard to enforce disjointness.

### SVG: ISA Hierarchy

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 340" font-family="Segoe UI, Arial, sans-serif">
  <rect width="700" height="340" fill="#f8f9fa" rx="12"/>
  <text x="350" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">ISA (Generalization/Specialization)</text>

  <!-- Superclass: Person -->
  <rect x="260" y="48" width="180" height="65" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2"/>
  <text x="350" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">PERSON</text>
  <text x="350" y="92" text-anchor="middle" font-size="11" fill="#555">pid, name, dob</text>
  <text x="350" y="105" text-anchor="middle" font-size="10" fill="#888">(superclass)</text>

  <!-- ISA triangle -->
  <polygon points="350,130 310,170 390,170" fill="#fff" stroke="#8e44ad" stroke-width="2"/>
  <text x="350" y="162" text-anchor="middle" font-size="11" font-weight="bold" fill="#8e44ad">ISA</text>

  <!-- Lines from ISA to subclasses -->
  <line x1="310" y1="170" x2="130" y2="210" stroke="#8e44ad" stroke-width="1.5"/>
  <line x1="350" y1="170" x2="350" y2="210" stroke="#8e44ad" stroke-width="1.5"/>
  <line x1="390" y1="170" x2="570" y2="210" stroke="#8e44ad" stroke-width="1.5"/>

  <!-- Student -->
  <rect x="50" y="210" width="160" height="65" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2"/>
  <text x="130" y="234" text-anchor="middle" font-size="13" font-weight="bold" fill="#27ae60">STUDENT</text>
  <text x="130" y="254" text-anchor="middle" font-size="11" fill="#555">gpa, year</text>
  <text x="130" y="268" text-anchor="middle" font-size="10" fill="#888">(subclass)</text>

  <!-- Faculty -->
  <rect x="270" y="210" width="160" height="65" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
  <text x="350" y="234" text-anchor="middle" font-size="13" font-weight="bold" fill="#e67e22">FACULTY</text>
  <text x="350" y="254" text-anchor="middle" font-size="11" fill="#555">rank, tenure</text>
  <text x="350" y="268" text-anchor="middle" font-size="10" fill="#888">(subclass)</text>

  <!-- Staff -->
  <rect x="490" y="210" width="160" height="65" rx="8" fill="#f4ecf7" stroke="#8e44ad" stroke-width="2"/>
  <text x="570" y="234" text-anchor="middle" font-size="13" font-weight="bold" fill="#8e44ad">STAFF</text>
  <text x="570" y="254" text-anchor="middle" font-size="11" fill="#555">shift, dept</text>
  <text x="570" y="268" text-anchor="middle" font-size="10" fill="#888">(subclass)</text>

  <!-- Constraints -->
  <rect x="160" y="295" width="380" height="32" rx="6" fill="#2c3e50"/>
  <text x="350" y="316" text-anchor="middle" font-size="12" fill="white">Disjoint (d) = at most one subclass  |  Overlapping (o) = multiple</text>
</svg>

---

## 10. ER Diagram Notation — Complete Symbol Reference

| Symbol | Meaning | Used For |
|---|---|---|
| **Rectangle** (single line) | Strong Entity | Student, Course, Department |
| **Rectangle** (double line) | Weak Entity | Dependent, Room |
| **Oval** (single line) | Simple Attribute | name, price |
| **Oval** (underlined) | Key Attribute | student_id, course_id |
| **Oval** (double line) | Multi-valued Attribute | phone_numbers, colors |
| **Oval** (dashed line) | Derived Attribute | age, total_cost |
| **Oval** with sub-ovals | Composite Attribute | full_name → first, last |
| **Oval** (dashed underline) | Partial Key (weak entity discriminator) | dependent_name, room_number |
| **Diamond** (single line) | Relationship | enrolls_in, works_for |
| **Diamond** (double line) | Identifying Relationship | has (for weak entity) |
| **Single line** | Partial Participation | optional |
| **Double line** | Total Participation | mandatory |
| **1, N, M** near line | Cardinality | 1:1, 1:N, M:N |
| **Triangle** with "ISA" | Generalization/Specialization | Person → Student/Faculty |

---

## 11. Full ER Diagram — University System (SVG)

**Scenario**: A university database tracking departments, professors, students, courses, enrollments, and dependents.

**Entities**: DEPARTMENT, PROFESSOR, STUDENT, COURSE, ENROLLMENT (relationship), DEPENDENT (weak)

**Relationships**:
- Department *has* Professors (1:N)
- Department *offers* Courses (1:N)
- Professor *teaches* Courses (1:N)
- Student *enrolls_in* Courses (M:N → Enrollment)
- Student *belongs_to* Department (N:1)
- Professor *has* Dependents (1:N, identifying)

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 700" font-family="Segoe UI, Arial, sans-serif">
  <rect width="900" height="700" fill="#f8f9fa" rx="12"/>
  <text x="450" y="28" text-anchor="middle" font-size="18" font-weight="bold" fill="#2c3e50">University System — ER Diagram</text>

  <!-- ============ DEPARTMENT ============ -->
  <rect x="350" y="55" width="170" height="50" rx="8" fill="#ebf5fb" stroke="#2980b9" stroke-width="2.5"/>
  <text x="435" y="85" text-anchor="middle" font-size="14" font-weight="bold" fill="#2980b9">DEPARTMENT</text>
  <!-- Attributes -->
  <ellipse cx="290" cy="45" rx="55" ry="18" fill="#d6eaf8" stroke="#2980b9" stroke-width="1.5"/>
  <text x="290" y="50" text-anchor="middle" font-size="10" fill="#1a5276" text-decoration="underline">dept_id</text>
  <ellipse cx="435" cy="35" rx="55" ry="16" fill="#d6eaf8" stroke="#2980b9" stroke-width="1"/>
  <text x="435" y="40" text-anchor="middle" font-size="10" fill="#1a5276">dept_name</text>
  <ellipse cx="570" cy="45" rx="45" ry="18" fill="#d6eaf8" stroke="#2980b9" stroke-width="1"/>
  <text x="570" y="50" text-anchor="middle" font-size="10" fill="#1a5276">building</text>
  <line x1="340" y1="55" x2="290" y2="55" stroke="#2980b9" stroke-width="1"/>
  <line x1="435" y1="51" x2="435" y2="55" stroke="#2980b9" stroke-width="1"/>
  <line x1="520" y1="60" x2="545" y2="52" stroke="#2980b9" stroke-width="1"/>

  <!-- ============ PROFESSOR ============ -->
  <rect x="60" y="200" width="160" height="50" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="2.5"/>
  <text x="140" y="230" text-anchor="middle" font-size="14" font-weight="bold" fill="#27ae60">PROFESSOR</text>
  <!-- Attributes -->
  <ellipse cx="50" cy="170" rx="45" ry="16" fill="#d5f5e3" stroke="#27ae60" stroke-width="1"/>
  <text x="50" y="175" text-anchor="middle" font-size="10" fill="#145a32" text-decoration="underline">prof_id</text>
  <ellipse cx="140" cy="170" rx="40" ry="16" fill="#d5f5e3" stroke="#27ae60" stroke-width="1"/>
  <text x="140" y="175" text-anchor="middle" font-size="10" fill="#145a32">name</text>
  <ellipse cx="225" cy="175" rx="40" ry="16" fill="#d5f5e3" stroke="#27ae60" stroke-width="1"/>
  <text x="225" y="180" text-anchor="middle" font-size="10" fill="#145a32">rank</text>
  <line x1="80" y1="200" x2="60" y2="186" stroke="#27ae60" stroke-width="1"/>
  <line x1="140" y1="200" x2="140" y2="186" stroke="#27ae60" stroke-width="1"/>
  <line x1="190" y1="200" x2="210" y2="188" stroke="#27ae60" stroke-width="1"/>

  <!-- ============ Relationship: dept HAS professor (1:N) ============ -->
  <polygon points="280,135 320,110 360,135 320,160" fill="#fff" stroke="#7f8c8d" stroke-width="2"/>
  <text x="320" y="140" text-anchor="middle" font-size="10" fill="#333">has</text>
  <!-- Lines -->
  <line x1="380" y1="105" x2="360" y2="118" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="390" y="115" font-size="9" fill="#888">1</text>
  <line x1="180" y1="200" x2="305" y2="160" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="225" y="175" font-size="9" fill="#888"></text>
  <text x="270" y="168" font-size="9" fill="#888">N</text>

  <!-- ============ STUDENT ============ -->
  <rect x="610" y="200" width="160" height="50" rx="8" fill="#fef9e7" stroke="#f39c12" stroke-width="2.5"/>
  <text x="690" y="230" text-anchor="middle" font-size="14" font-weight="bold" fill="#e67e22">STUDENT</text>
  <!-- Attributes -->
  <ellipse cx="620" cy="170" rx="50" ry="16" fill="#fdebd0" stroke="#f39c12" stroke-width="1"/>
  <text x="620" y="175" text-anchor="middle" font-size="10" fill="#784212" text-decoration="underline">student_id</text>
  <ellipse cx="720" cy="170" rx="40" ry="16" fill="#fdebd0" stroke="#f39c12" stroke-width="1"/>
  <text x="720" y="175" text-anchor="middle" font-size="10" fill="#784212">name</text>
  <ellipse cx="810" cy="195" rx="35" ry="16" fill="#fdebd0" stroke="#f39c12" stroke-width="1"/>
  <text x="810" y="200" text-anchor="middle" font-size="10" fill="#784212">gpa</text>
  <line x1="640" y1="200" x2="630" y2="186" stroke="#f39c12" stroke-width="1"/>
  <line x1="710" y1="200" x2="720" y2="186" stroke="#f39c12" stroke-width="1"/>
  <line x1="770" y1="210" x2="790" y2="200" stroke="#f39c12" stroke-width="1"/>

  <!-- ============ Relationship: student BELONGS_TO dept (N:1) ============ -->
  <polygon points="555,135 595,110 635,135 595,160" fill="#fff" stroke="#7f8c8d" stroke-width="2"/>
  <text x="595" y="138" text-anchor="middle" font-size="9" fill="#333">belongs</text>
  <line x1="520" y1="100" x2="560" y2="120" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="530" y="110" font-size="9" fill="#888">1</text>
  <line x1="630" y1="150" x2="660" y2="200" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="655" y="175" font-size="9" fill="#888">N</text>

  <!-- ============ COURSE ============ -->
  <rect x="335" y="340" width="160" height="50" rx="8" fill="#f4ecf7" stroke="#8e44ad" stroke-width="2.5"/>
  <text x="415" y="370" text-anchor="middle" font-size="14" font-weight="bold" fill="#8e44ad">COURSE</text>
  <!-- Attributes -->
  <ellipse cx="310" cy="320" rx="50" ry="16" fill="#e8daef" stroke="#8e44ad" stroke-width="1"/>
  <text x="310" y="325" text-anchor="middle" font-size="10" fill="#4a235a" text-decoration="underline">course_id</text>
  <ellipse cx="420" cy="315" rx="40" ry="16" fill="#e8daef" stroke="#8e44ad" stroke-width="1"/>
  <text x="420" y="320" text-anchor="middle" font-size="10" fill="#4a235a">title</text>
  <ellipse cx="530" cy="325" rx="40" ry="16" fill="#e8daef" stroke="#8e44ad" stroke-width="1"/>
  <text x="530" y="330" text-anchor="middle" font-size="10" fill="#4a235a">credits</text>
  <line x1="350" y1="340" x2="335" y2="332" stroke="#8e44ad" stroke-width="1"/>
  <line x1="415" y1="340" x2="420" y2="331" stroke="#8e44ad" stroke-width="1"/>
  <line x1="495" y1="345" x2="510" y2="335" stroke="#8e44ad" stroke-width="1"/>

  <!-- ============ Relationship: dept OFFERS course (1:N) ============ -->
  <polygon points="370,250 410,225 450,250 410,275" fill="#fff" stroke="#7f8c8d" stroke-width="2"/>
  <text x="410" y="254" text-anchor="middle" font-size="10" fill="#333">offers</text>
  <line x1="435" y1="105" x2="415" y2="225" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="428" y="165" font-size="9" fill="#888">1</text>
  <line x1="415" y1="275" x2="415" y2="340" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="425" y="310" font-size="9" fill="#888">N</text>

  <!-- ============ Relationship: professor TEACHES course (1:N) ============ -->
  <polygon points="200,320 240,295 280,320 240,345" fill="#fff" stroke="#7f8c8d" stroke-width="2"/>
  <text x="240" y="324" text-anchor="middle" font-size="10" fill="#333">teaches</text>
  <line x1="165" y1="250" x2="220" y2="300" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="180" y="270" font-size="9" fill="#888">1</text>
  <line x1="280" y1="335" x2="335" y2="355" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="310" y="340" font-size="9" fill="#888">N</text>

  <!-- ============ Relationship: student ENROLLS_IN course (M:N) ============ -->
  <polygon points="600,380 640,355 680,380 640,405" fill="#fef9e7" stroke="#e67e22" stroke-width="2"/>
  <text x="640" y="382" text-anchor="middle" font-size="9" fill="#a04000">enrolls</text>
  <!-- Relationship attrs -->
  <ellipse cx="720" cy="365" rx="35" ry="14" fill="#fdebd0" stroke="#e67e22" stroke-width="1"/>
  <text x="720" y="370" text-anchor="middle" font-size="9" fill="#784212">grade</text>
  <line x1="690" y1="370" x2="700" y2="368" stroke="#e67e22" stroke-width="1"/>
  <ellipse cx="720" cy="400" rx="45" ry="14" fill="#fdebd0" stroke="#e67e22" stroke-width="1"/>
  <text x="720" y="405" text-anchor="middle" font-size="9" fill="#784212">semester</text>
  <line x1="688" y1="392" x2="700" y2="396" stroke="#e67e22" stroke-width="1"/>

  <line x1="495" y1="370" x2="600" y2="378" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="545" y="370" font-size="9" fill="#888">N</text>
  <line x1="690" y1="250" x2="660" y2="355" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="680" y="300" font-size="9" fill="#888">M</text>

  <!-- ============ DEPENDENT (weak entity) ============ -->
  <rect x="30" y="380" width="160" height="50" rx="6" fill="#fce4ec" stroke="#c0392b" stroke-width="2.5"/>
  <rect x="35" y="385" width="150" height="40" rx="4" fill="none" stroke="#c0392b" stroke-width="1.5"/>
  <text x="110" y="410" text-anchor="middle" font-size="13" font-weight="bold" fill="#c0392b">DEPENDENT</text>
  <!-- Attributes -->
  <ellipse cx="50" cy="455" rx="50" ry="14" fill="#fadbd8" stroke="#c0392b" stroke-width="1"/>
  <text x="50" y="460" text-anchor="middle" font-size="9" fill="#922b21" style="text-decoration: underline; text-decoration-style: dashed;">dep_name</text>
  <ellipse cx="160" cy="455" rx="45" ry="14" fill="#fadbd8" stroke="#c0392b" stroke-width="1"/>
  <text x="160" y="460" text-anchor="middle" font-size="9" fill="#922b21">birth_date</text>
  <line x1="80" y1="430" x2="60" y2="442" stroke="#c0392b" stroke-width="1"/>
  <line x1="130" y1="430" x2="150" y2="442" stroke="#c0392b" stroke-width="1"/>

  <!-- Identifying relationship: professor HAS dependent -->
  <polygon points="80,310 120,285 160,310 120,335" fill="#fce4ec" stroke="#c0392b" stroke-width="2"/>
  <polygon points="85,310 120,290 155,310 120,330" fill="none" stroke="#c0392b" stroke-width="1.5"/>
  <text x="120" y="314" text-anchor="middle" font-size="9" fill="#922b21">has</text>
  <!-- Double line to weak entity -->
  <line x1="110" y1="335" x2="110" y2="380" stroke="#c0392b" stroke-width="2.5"/>
  <line x1="118" y1="335" x2="118" y2="380" stroke="#c0392b" stroke-width="2.5"/>
  <text x="130" y="360" font-size="9" fill="#c0392b">N</text>
  <!-- Line to professor -->
  <line x1="140" y1="250" x2="130" y2="285" stroke="#c0392b" stroke-width="2"/>
  <text x="140" y="270" font-size="9" fill="#c0392b">1</text>

  <!-- Legend -->
  <rect x="30" y="520" width="840" height="170" rx="10" fill="#fff" stroke="#ddd" stroke-width="1"/>
  <text x="450" y="545" text-anchor="middle" font-size="14" font-weight="bold" fill="#2c3e50">Legend</text>
  <rect x="50" y="560" width="70" height="22" rx="4" fill="#ebf5fb" stroke="#2980b9" stroke-width="1.5"/>
  <text x="85" y="576" text-anchor="middle" font-size="9" fill="#2980b9">Entity</text>
  <text x="140" y="576" font-size="10" fill="#555">Strong Entity</text>

  <rect x="50" y="590" width="70" height="22" rx="3" fill="#fce4ec" stroke="#c0392b" stroke-width="1.5"/>
  <rect x="53" y="593" width="64" height="16" rx="2" fill="none" stroke="#c0392b" stroke-width="1"/>
  <text x="85" y="606" text-anchor="middle" font-size="9" fill="#c0392b">Weak</text>
  <text x="140" y="606" font-size="10" fill="#555">Weak Entity (double border)</text>

  <ellipse cx="395" cy="571" rx="40" ry="14" fill="#d6eaf8" stroke="#2980b9" stroke-width="1"/>
  <text x="395" y="576" text-anchor="middle" font-size="9" fill="#1a5276">attr</text>
  <text x="450" y="576" font-size="10" fill="#555">Attribute</text>

  <ellipse cx="395" cy="601" rx="40" ry="14" fill="#d6eaf8" stroke="#2980b9" stroke-width="1"/>
  <text x="395" y="606" text-anchor="middle" font-size="9" fill="#1a5276" text-decoration="underline">key</text>
  <text x="450" y="606" font-size="10" fill="#555">Key Attribute (underlined)</text>

  <polygon points="640,558 665,545 690,558 665,571" fill="#fff" stroke="#7f8c8d" stroke-width="1.5"/>
  <text x="665" y="563" text-anchor="middle" font-size="8" fill="#555">rel</text>
  <text x="710" y="563" font-size="10" fill="#555">Relationship</text>

  <line x1="640" y1="590" x2="690" y2="590" stroke="#7f8c8d" stroke-width="2"/>
  <text x="710" y="594" font-size="10" fill="#555">Partial participation</text>

  <line x1="640" y1="615" x2="690" y2="615" stroke="#27ae60" stroke-width="3"/>
  <line x1="640" y1="620" x2="690" y2="620" stroke="#27ae60" stroke-width="3"/>
  <text x="710" y="620" font-size="10" fill="#555">Total participation</text>

  <text x="50" y="665" font-size="11" fill="#555">Relationships: DEPARTMENT (1)──has──(N) PROFESSOR (1)──teaches──(N) COURSE (N)──enrolls──(M) STUDENT</text>
  <text x="50" y="683" font-size="11" fill="#555">DEPARTMENT (1)──offers──(N) COURSE  |  STUDENT (N)──belongs_to──(1) DEPARTMENT  |  PROFESSOR (1)──has──(N) DEPENDENT (weak)</text>
</svg>

---

## 12. ER → Relational Mapping Rules

These are the **systematic rules** for converting an ER diagram into a set of relational tables (SQL). Master these and you can translate any ER diagram into a schema.

### Rule 1: Strong Entity → Table

Create a table for each strong entity. Its attributes become columns. The key attribute becomes the primary key.

```
ER:  STUDENT(student_id, name, gpa)

SQL:
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    gpa         DECIMAL(3,2)
);
```

### Rule 2: Weak Entity → Table with Composite PK

Create a table with a composite primary key: owner's PK + partial key. Add a FK to the owner with `ON DELETE CASCADE`.

```
ER:  DEPENDENT(dependent_name, birth_date) — weak entity of PROFESSOR

SQL:
CREATE TABLE dependents (
    prof_id         INT REFERENCES professors(prof_id) ON DELETE CASCADE,
    dependent_name  VARCHAR(100),
    birth_date      DATE,
    PRIMARY KEY (prof_id, dependent_name)
);
```

### Rule 3: 1:1 Relationship → FK on either side (prefer total participation side)

Add the FK to the table with **total participation** (if one exists). Add a UNIQUE constraint to enforce the 1:1.

```
ER:  EMPLOYEE (1,total) ──manages── (1,partial) DEPARTMENT

SQL: Put FK in departments (the partial side), OR in employees — but prefer the total side:
CREATE TABLE departments (
    dept_id     INT PRIMARY KEY,
    dept_name   VARCHAR(100),
    manager_id  INT UNIQUE REFERENCES employees(emp_id)  -- UNIQUE = 1:1
);
```

### Rule 4: 1:N Relationship → FK on the "N" side

Add a foreign key in the table on the **"many" (N)** side, pointing to the PK of the "one" (1) side.

```
ER:  DEPARTMENT (1) ──has── (N) PROFESSOR

SQL:
CREATE TABLE professors (
    prof_id   INT PRIMARY KEY,
    name      VARCHAR(100),
    dept_id   INT REFERENCES departments(dept_id)  -- FK on the N side
);
```

### Rule 5: M:N Relationship → New Junction Table

Create a **new table** with FKs to both entity tables. The PK is typically the combination of both FKs (plus any additional discriminating attributes).

```
ER:  STUDENT (M) ──enrolls_in── (N) COURSE  [grade, semester]

SQL:
CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(8) REFERENCES courses(course_id),
    grade       CHAR(2),
    semester    CHAR(6),
    PRIMARY KEY (student_id, course_id, semester)
);
```

### Rule 6: Multi-valued Attribute → Separate Table

```
ER:  STUDENT has multi-valued attribute {phone_numbers}

SQL:
CREATE TABLE student_phones (
    student_id    INT REFERENCES students(student_id),
    phone_number  VARCHAR(20),
    PRIMARY KEY (student_id, phone_number)
);
```

### Rule 7: Composite Attribute → Flatten to columns

```
ER:  STUDENT has composite attribute address(street, city, state, zip)

SQL:
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    street      VARCHAR(200),
    city        VARCHAR(100),
    state       CHAR(2),
    zip         CHAR(10)
);
```

### Rule 8: Derived Attribute → Don't store (or use generated column)

```
ER:  STUDENT has derived attribute age (from birth_date)

SQL: Simply compute when needed:
SELECT name, EXTRACT(YEAR FROM AGE(birth_date)) AS age FROM students;
```

### Rule 9: ISA Hierarchy → Choose a strategy

(See Section 9 for the three strategies: single table, table-per-type, or table-per-concrete-class.)

### Mapping Decision Flowchart

```
Is it an entity?
├── YES → Is it a strong entity?
│   ├── YES → RULE 1: Create table, PK = key attribute
│   └── NO (weak) → RULE 2: Create table, PK = owner PK + partial key, FK CASCADE
└── NO → Is it a relationship?
    ├── 1:1 → RULE 3: FK (UNIQUE) on the total-participation side
    ├── 1:N → RULE 4: FK on the N side
    └── M:N → RULE 5: New junction table

Is it an attribute?
├── Simple → Column in the entity's table
├── Composite → RULE 7: Flatten sub-attributes as columns
├── Multi-valued → RULE 6: New separate table
├── Derived → RULE 8: Don't store (compute)
└── Key → RULE 1: Becomes PRIMARY KEY
```

---

## 13. Mapping the University ER Diagram to SQL

Let's apply all the mapping rules to the university ER diagram from Section 11.

```sql
-- ============================================================
-- RULE 1: Strong entities → Tables
-- ============================================================

CREATE TABLE departments (
    dept_id     CHAR(4) PRIMARY KEY,
    dept_name   VARCHAR(100) NOT NULL UNIQUE,
    building    VARCHAR(50)
);

CREATE TABLE professors (
    prof_id     INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    rank        VARCHAR(50),
    dept_id     CHAR(4) NOT NULL REFERENCES departments(dept_id)
    -- ↑ RULE 4: 1:N (department HAS professors) → FK on N side (professors)
    -- NOT NULL → total participation (every professor must be in a department)
);

CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    gpa         DECIMAL(3,2) CHECK (gpa >= 0 AND gpa <= 4.0),
    dept_id     CHAR(4) NOT NULL REFERENCES departments(dept_id)
    -- ↑ RULE 4: 1:N (department HAS students) → FK on N side
);

CREATE TABLE courses (
    course_id   CHAR(8) PRIMARY KEY,
    title       VARCHAR(200) NOT NULL,
    credits     INT CHECK (credits BETWEEN 1 AND 6),
    dept_id     CHAR(4) NOT NULL REFERENCES departments(dept_id),
    -- ↑ RULE 4: 1:N (department OFFERS courses) → FK on N side
    prof_id     INT REFERENCES professors(prof_id)
    -- ↑ RULE 4: 1:N (professor TEACHES courses) → FK on N side
    -- NULL allowed: a course might not have a professor assigned yet (partial)
);

-- ============================================================
-- RULE 5: M:N relationship → Junction table
-- ============================================================

CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(8) REFERENCES courses(course_id),
    semester    CHAR(6) NOT NULL,
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id, semester)
    -- ↑ Composite PK includes semester so a student can retake a course
);

-- ============================================================
-- RULE 2: Weak entity → Table with composite PK + CASCADE
-- ============================================================

CREATE TABLE dependents (
    prof_id         INT REFERENCES professors(prof_id) ON DELETE CASCADE,
    dependent_name  VARCHAR(100),
    birth_date      DATE,
    relationship    VARCHAR(20),
    PRIMARY KEY (prof_id, dependent_name)
    -- ↑ PK = owner's PK (prof_id) + partial key (dependent_name)
);
```

**Summary of which rule was applied where:**

| ER Element | Rule | SQL Result |
|---|---|---|
| DEPARTMENT (strong entity) | Rule 1 | `departments` table with PK `dept_id` |
| PROFESSOR (strong entity) | Rule 1 | `professors` table with PK `prof_id` |
| STUDENT (strong entity) | Rule 1 | `students` table with PK `student_id` |
| COURSE (strong entity) | Rule 1 | `courses` table with PK `course_id` |
| dept HAS professors (1:N) | Rule 4 | FK `dept_id` in `professors` |
| student BELONGS_TO dept (N:1) | Rule 4 | FK `dept_id` in `students` |
| dept OFFERS courses (1:N) | Rule 4 | FK `dept_id` in `courses` |
| prof TEACHES courses (1:N) | Rule 4 | FK `prof_id` in `courses` |
| student ENROLLS_IN course (M:N) | Rule 5 | `enrollments` junction table |
| DEPENDENT (weak entity of professor) | Rule 2 | `dependents` table, PK = (prof_id, dependent_name) |

---

## 14. Worked Examples — Beginner, Intermediate, Advanced

### Beginner: Library System

**Entities**: BOOK, MEMBER, LOAN

```
BOOK (book_id, title, author, isbn)
MEMBER (member_id, name, email, join_date)
LOAN relationship: MEMBER (1) ──borrows── (N) BOOK   [loan_date, return_date]
  Actually, a member can borrow many books, and a book can be borrowed by many members
  over time → M:N
```

```sql
CREATE TABLE books (
    book_id   SERIAL PRIMARY KEY,
    title     VARCHAR(300) NOT NULL,
    author    VARCHAR(200),
    isbn      CHAR(13) UNIQUE
);

CREATE TABLE members (
    member_id  SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(200) UNIQUE,
    join_date  DATE DEFAULT CURRENT_DATE
);

-- M:N relationship → junction table
CREATE TABLE loans (
    loan_id     SERIAL PRIMARY KEY,
    member_id   INT NOT NULL REFERENCES members(member_id),
    book_id     INT NOT NULL REFERENCES books(book_id),
    loan_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    return_date DATE,
    UNIQUE (member_id, book_id, loan_date)
);
```

---

### Intermediate: E-Commerce System

**Entities**: CUSTOMER, PRODUCT, CATEGORY, ORDER, ORDER_ITEM, REVIEW  
**Relationships**: Customer places Order (1:N), Order contains Products (M:N via ORDER_ITEM), Product belongs to Category (N:1), Customer writes Review for Product (ternary)

```sql
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL UNIQUE,
    parent_id     INT REFERENCES categories(category_id)  -- recursive: subcategories
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    description   TEXT,
    price         DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock         INT NOT NULL DEFAULT 0,
    category_id   INT REFERENCES categories(category_id)  -- 1:N, FK on N side
);

CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(200) UNIQUE NOT NULL,
    city          VARCHAR(100),
    joined        DATE DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INT NOT NULL REFERENCES customers(customer_id),  -- 1:N, FK on N side
    order_date    TIMESTAMP DEFAULT NOW(),
    status        VARCHAR(20) DEFAULT 'pending',
    total_amount  DECIMAL(10,2)
);

-- M:N: Order contains Products → junction table
CREATE TABLE order_items (
    order_id      INT REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id    INT REFERENCES products(product_id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    unit_price    DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Ternary relationship: Customer reviews Product
CREATE TABLE reviews (
    review_id     SERIAL PRIMARY KEY,
    customer_id   INT NOT NULL REFERENCES customers(customer_id),
    product_id    INT NOT NULL REFERENCES products(product_id),
    rating        INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text   TEXT,
    created_at    TIMESTAMP DEFAULT NOW(),
    UNIQUE (customer_id, product_id)  -- one review per customer per product
);
```

---

### Advanced: Hospital Management System

**Entities**: HOSPITAL, DEPARTMENT, DOCTOR, PATIENT, APPOINTMENT, MEDICAL_RECORD, PRESCRIPTION  
**Weak Entity**: PRESCRIPTION (identified by MEDICAL_RECORD)  
**ISA**: DOCTOR and NURSE are specializations of STAFF  
**Recursive**: DOCTOR supervises DOCTOR

```sql
-- Staff (superclass for ISA)
CREATE TABLE staff (
    staff_id      SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    dob           DATE NOT NULL,
    hire_date     DATE NOT NULL DEFAULT CURRENT_DATE,
    phone         VARCHAR(20),
    staff_type    VARCHAR(10) NOT NULL CHECK (staff_type IN ('doctor', 'nurse', 'admin'))
);

-- Doctor (subclass — table-per-type strategy)
CREATE TABLE doctors (
    staff_id       INT PRIMARY KEY REFERENCES staff(staff_id),
    specialization VARCHAR(100) NOT NULL,
    license_no     VARCHAR(50) UNIQUE NOT NULL,
    supervisor_id  INT REFERENCES doctors(staff_id)  -- recursive: doctor supervises doctor
);

-- Nurse (subclass)
CREATE TABLE nurses (
    staff_id     INT PRIMARY KEY REFERENCES staff(staff_id),
    ward         VARCHAR(50),
    shift        VARCHAR(20) CHECK (shift IN ('day', 'night', 'rotating'))
);

CREATE TABLE departments (
    dept_id       SERIAL PRIMARY KEY,
    dept_name     VARCHAR(100) NOT NULL,
    floor         INT,
    head_doctor   INT REFERENCES doctors(staff_id)  -- 1:1, partial
);

CREATE TABLE patients (
    patient_id    SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    dob           DATE NOT NULL,
    gender        CHAR(1) CHECK (gender IN ('M', 'F', 'O')),
    blood_type    VARCHAR(5),
    phone         VARCHAR(20),
    emergency_contact VARCHAR(100)
);

-- M:N: Doctor treats Patient → Appointments
CREATE TABLE appointments (
    appointment_id  SERIAL PRIMARY KEY,
    patient_id      INT NOT NULL REFERENCES patients(patient_id),
    doctor_id       INT NOT NULL REFERENCES doctors(staff_id),
    dept_id         INT REFERENCES departments(dept_id),
    appointment_date TIMESTAMP NOT NULL,
    reason          TEXT,
    status          VARCHAR(20) DEFAULT 'scheduled'
);

CREATE TABLE medical_records (
    record_id       SERIAL PRIMARY KEY,
    patient_id      INT NOT NULL REFERENCES patients(patient_id),
    doctor_id       INT NOT NULL REFERENCES doctors(staff_id),
    visit_date      DATE NOT NULL,
    diagnosis       TEXT NOT NULL,
    treatment       TEXT,
    notes           TEXT
);

-- Weak entity: Prescription is identified through Medical Record
CREATE TABLE prescriptions (
    record_id       INT REFERENCES medical_records(record_id) ON DELETE CASCADE,
    drug_name       VARCHAR(200),
    dosage          VARCHAR(100) NOT NULL,
    frequency       VARCHAR(100) NOT NULL,
    duration_days   INT,
    PRIMARY KEY (record_id, drug_name)
);
```

---

## 15. Common Mistakes in ER Modeling

| Mistake | Problem | Fix |
|---|---|---|
| **Redundant relationships** | "student belongs_to department" AND "student enrolled_in course in department" — redundant path | Remove one; derive the department from the course if possible |
| **Entity vs. attribute confusion** | Modeling "address" as a separate entity when it has no independent relationships | If address has its own relationships or is shared, make it an entity; otherwise keep as composite attribute |
| **Missing cardinality** | Drawing lines without 1, N, M labels | Always specify cardinality — it directly affects the SQL mapping |
| **Overusing weak entities** | Making everything a weak entity "just in case" | Only use weak entities when the entity genuinely can't be identified without its owner |
| **Ignoring ternary relationships** | Decomposing a ternary into three binaries, which loses information | If the relationship inherently involves 3+ entities together, model it as ternary |
| **Many-to-many with attributes on the wrong entity** | Putting "grade" on Student instead of on the enrollment relationship | Relationship attributes belong on the relationship (diamond), not on either entity |
| **No participation constraints** | Not indicating whether participation is total or partial | Always mark this — it directly affects whether FKs are NOT NULL |
| **Derived attributes stored** | Storing "age" when "birth_date" exists | Use derived attributes (dashed oval); store only the source data |

---

## 16. Extended ER (EER) Concepts

The **Extended Entity-Relationship** model adds these concepts on top of basic ER:

| EER Concept | Description | Example |
|---|---|---|
| **Specialization** | Top-down: divide an entity into subclasses based on distinguishing features | EMPLOYEE → ENGINEER, MANAGER |
| **Generalization** | Bottom-up: combine similar entities into a single superclass | CAR, TRUCK, MOTORCYCLE → VEHICLE |
| **Aggregation** | Treat a relationship as a higher-level entity, so it can participate in other relationships | (Student enrolls_in Course) → this enrollment can be sponsored_by a Scholarship |
| **Category (Union type)** | A subclass that can have entities from multiple unrelated superclasses | VEHICLE_OWNER can be either a PERSON or a COMPANY (unrelated superclasses) |

### Aggregation Example

```
Problem: A project is sponsored by a department, but the sponsorship applies
to the (Employee works_on Project) relationship, not just to the project.

Without aggregation:
  Employee ──works_on──▶ Project
  Department ──sponsors──▶ ???   (can't connect to a relationship)

With aggregation:
  ┌──────────────────────────────────────┐
  │  Employee ──works_on──▶ Project      │ ← treat this as a single "box"
  └──────────────────┬───────────────────┘
                     │
            Department ──sponsors──▶ (the aggregated relationship)
```

---

## 17. Common Interview Questions

### Q1: What is an ER diagram and why is it important?
> An ER diagram is a visual representation of entities, attributes, and relationships in a database. It's important because it provides a high-level design before implementation, ensures all stakeholders understand the data model, and serves as a blueprint for creating the relational schema.

### Q2: What is the difference between an entity and an attribute?
> An **entity** is an independent "thing" that has its own identity and can participate in relationships (Student, Course). An **attribute** is a property that describes an entity (name, GPA). If something needs its own relationships, it should be an entity, not an attribute.

### Q3: Explain the difference between strong and weak entities with an example.
> A **strong entity** has its own primary key and can exist independently (Employee). A **weak entity** cannot be uniquely identified by its own attributes — it depends on an "owner" entity. Example: a Dependent of an Employee. "Sarah" isn't unique across all dependents, but (emp_id=101, dependent_name="Sarah") is. The weak entity's PK = owner's PK + its own partial key.

### Q4: How do you map a M:N relationship to a relational schema?
> Create a **junction table** (also called bridge/associative table) with:  
> 1. Foreign keys to both participating entities  
> 2. Primary key = composite of both FKs (+ any additional discriminating attributes)  
> 3. Any relationship attributes as additional columns  
> Example: `enrollments(student_id FK, course_id FK, grade, semester)`.

### Q5: What is the difference between total and partial participation?
> **Total participation** means every entity in the entity set MUST participate in the relationship (enforced with NOT NULL FK). **Partial participation** means some entities may not participate (FK allows NULL). Example: every employee must work in a department (total), but not every employee manages a department (partial).

### Q6: What are the three strategies for mapping ISA hierarchies? When would you use each?
> 1. **Single table**: One table for all, discriminator column. Best when subclasses have few unique attributes. Simple queries but many NULLs.  
> 2. **Table-per-type**: Superclass table + subclass tables with FK. Best for most cases — clean design, no NULLs. Requires joins.  
> 3. **Table-per-concrete-class**: Only subclass tables, each with inherited attributes. Best when superclass is never queried directly. Avoids joins but duplicates columns.

### Q7: What is a composite attribute? How is it different from a multi-valued attribute?
> A **composite** attribute can be divided into sub-parts (address → street, city, state, zip) but each entity has **one** address. A **multi-valued** attribute has **multiple values** per entity (a student can have multiple phone numbers). In SQL, composites become multiple columns; multi-valued attributes become a separate table.

### Q8: What is aggregation in EER? Give an example.
> Aggregation treats a relationship as a higher-level entity that can participate in another relationship. Example: the "works_on" relationship between Employee and Project can be aggregated so that a Department can "sponsor" a specific (Employee, Project) working arrangement. Without aggregation, you can't connect a relationship to another relationship.

### Q9: Can a relationship have attributes? Give an example.
> Yes. Attributes that don't belong to either entity but to the **association** between them. Example: in "Student enrolls_in Course", the `grade` belongs to the enrollment, not to the student or the course. It only exists because of the specific (student, course) pair.

### Q10: How do you decide whether something should be an entity or an attribute?
> Make it an entity if it:  
> - Has its own relationships to other entities  
> - Has multiple attributes of its own  
> - Has multiple instances that need to be shared/referenced  
> - Needs its own identity  
> Make it an attribute if it's a simple property with no independent existence. Example: "department" can be an attribute of Employee (just the name) or a full entity (with budget, location, manager) — depends on requirements.

---

## 18. Key Takeaways

| # | Concept | One-Line Summary |
|---|---|---|
| 1 | **Data Modeling** | Design before code — ER diagram is the blueprint |
| 2 | **Entity** | A distinguishable real-world "thing" (Student, Course) |
| 3 | **Attribute types** | Simple, Composite, Multi-valued, Derived, Key |
| 4 | **Relationship** | An association between entities (enrolls_in, teaches) |
| 5 | **Degree** | Binary (2), Ternary (3), Unary/recursive (1) |
| 6 | **1:1** | FK with UNIQUE on either side; prefer total participation side |
| 7 | **1:N** | FK on the "many" (N) side |
| 8 | **M:N** | Junction table with FKs to both entities |
| 9 | **Total participation** | Every entity MUST participate → NOT NULL FK |
| 10 | **Partial participation** | Some entities MAY participate → FK allows NULL |
| 11 | **Weak entity** | Can't exist alone; PK = owner PK + partial key; CASCADE delete |
| 12 | **ISA hierarchy** | Superclass/subclass inheritance; three SQL strategies |
| 13 | **Multi-valued attr** | Separate table (student_phones) |
| 14 | **Composite attr** | Flatten into multiple columns |
| 15 | **Derived attr** | Don't store — compute in queries |
| 16 | **Relationship attrs** | Belong on the relationship, not on either entity (grade on enrollment) |
| 17 | **Mapping rules** | 9 systematic rules: entity→table, 1:1→FK UNIQUE, 1:N→FK on N, M:N→junction |
| 18 | **EER** | Adds specialization, generalization, aggregation, categories |

---

<p style="text-align: center; margin-top: 40px;">
  <a href="02_data_modeling_er.md" download="02_data_modeling_er.md"
     style="display: inline-block; padding: 14px 36px; font-size: 16px; font-weight: bold;
            color: #fff; background-color: #8e44ad; border-radius: 8px;
            text-decoration: none; box-shadow: 0 4px 6px rgba(0,0,0,0.15);">
    Download 02_data_modeling_er.md
  </a>
</p>

---

*Ready? Say **"next"** to proceed to **Phase 3 — Relational Model & Schema Design**.*
