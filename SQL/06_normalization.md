# Phase 6 — Normalization (1NF → 5NF)

> **Goal**: Understand functional dependencies, master every normal form from 1NF through 5NF, learn lossless-join and dependency-preserving decomposition, and know when to deliberately denormalize. By the end you will look at any table and instantly spot the normalization violations.

---

## Table of Contents

1. [Why Normalize?](#1-why-normalize)
2. [Functional Dependencies (FDs)](#2-functional-dependencies-fds)
3. [Attribute Closure & Candidate Keys](#3-attribute-closure--candidate-keys)
4. [Canonical Cover (Minimal Cover)](#4-canonical-cover-minimal-cover)
5. [First Normal Form (1NF)](#5-first-normal-form-1nf)
6. [Second Normal Form (2NF)](#6-second-normal-form-2nf)
7. [Third Normal Form (3NF)](#7-third-normal-form-3nf)
8. [Boyce-Codd Normal Form (BCNF)](#8-boyce-codd-normal-form-bcnf)
9. [Multi-Valued Dependencies & Fourth Normal Form (4NF)](#9-multi-valued-dependencies--fourth-normal-form-4nf)
10. [Join Dependencies & Fifth Normal Form (5NF)](#10-join-dependencies--fifth-normal-form-5nf)
11. [Normalization Roadmap — SVG](#11-normalization-roadmap--svg)
12. [Decomposition Theory](#12-decomposition-theory)
13. [Denormalization — When & Why](#13-denormalization--when--why)
14. [Worked Examples — Beginner, Intermediate, Advanced](#14-worked-examples--beginner-intermediate-advanced)
15. [Common Interview Questions](#15-common-interview-questions)
16. [Key Takeaways](#16-key-takeaways)

---

## 1. Why Normalize?

### The Problem: Redundancy

Consider a single flat table that stores everything about students, courses, and enrollments:

```
student_enrollment_flat
┌────────────┬──────────┬─────────┬───────────┬─────────┬──────────────┬─────────┐
│ student_id │ name     │ dept    │ course_id │ title   │ instructor   │ grade   │
├────────────┼──────────┼─────────┼───────────┼─────────┼──────────────┼─────────┤
│ 101        │ Alice    │ CS      │ CS201     │ DBMS    │ Dr. Smith    │ A       │
│ 101        │ Alice    │ CS      │ CS301     │ OS      │ Dr. Patel    │ B+      │
│ 102        │ Bob      │ Math    │ MA101     │ Calc I  │ Dr. Lee      │ A-      │
│ 102        │ Bob      │ Math    │ CS201     │ DBMS    │ Dr. Smith    │ B       │
│ 103        │ Carol    │ CS      │ CS201     │ DBMS    │ Dr. Smith    │ A+      │
└────────────┴──────────┴─────────┴───────────┴─────────┴──────────────┴─────────┘
```

**Three deadly anomalies arise:**

| Anomaly | Example | What Goes Wrong |
|---|---|---|
| **Update anomaly** | Change Dr. Smith's name → must update every row where he appears | Miss one row and data is inconsistent |
| **Insertion anomaly** | Add a new course with no enrolled student → can't insert (student_id is part of PK) | Can't record facts independently |
| **Deletion anomaly** | Delete Bob's last enrollment → lose the fact that MA101 exists and Dr. Lee teaches it | Deleting one fact destroys unrelated facts |

### The Solution: Normalization

**Normalization** is the process of decomposing a relation into smaller, well-structured relations that:
- Eliminate redundancy
- Prevent anomalies
- Preserve all information (lossless join)
- Ideally preserve all functional dependencies

> **Real-world analogy**: Instead of writing your full address on every form you fill out, you write your address once in a master record and reference it by an ID everywhere else.

---

## 2. Functional Dependencies (FDs)

### Definition

A **functional dependency** X → Y means: for any two tuples, if they have the same value of X, they **must** have the same value of Y.

> "X functionally determines Y" — knowing X, you can determine exactly one value of Y.

```
student_id → name, dept          (knowing the ID tells you the name and department)
course_id  → title, instructor   (knowing the course tells you its title and instructor)
student_id, course_id → grade    (knowing who + which course tells you the grade)
```

### Types of FDs

| Type | Definition | Example |
|---|---|---|
| **Trivial** | Y ⊆ X (right side is subset of left) | {student_id, name} → {name} |
| **Non-trivial** | Y ⊄ X (right side has something new) | student_id → name |
| **Completely non-trivial** | X ∩ Y = ∅ (no overlap) | student_id → dept |
| **Full FD** | Y depends on all of X, not a proper subset | {student_id, course_id} → grade (need both) |
| **Partial FD** | Y depends on a proper subset of X | {student_id, course_id} → name (only need student_id) |
| **Transitive FD** | X → Z via X → Y → Z | student_id → dept → dept_building |

### Armstrong's Axioms

The **sound and complete** set of rules for reasoning about FDs:

| Axiom | Rule | Example |
|---|---|---|
| **Reflexivity** | If Y ⊆ X, then X → Y | {A, B} → A |
| **Augmentation** | If X → Y, then XZ → YZ | A → B implies AC → BC |
| **Transitivity** | If X → Y and Y → Z, then X → Z | A → B, B → C ⟹ A → C |

**Derived rules** (provable from the axioms):

| Rule | Statement |
|---|---|
| **Union** | If X → Y and X → Z, then X → YZ |
| **Decomposition** | If X → YZ, then X → Y and X → Z |
| **Pseudo-transitivity** | If X → Y and WY → Z, then WX → Z |

---

## 3. Attribute Closure & Candidate Keys

### Attribute Closure (X⁺)

The **closure** of a set of attributes X under a set of FDs F is the set of all attributes that X can determine.

**Algorithm**:

```
Input: attribute set X, FD set F
Output: X⁺ (closure of X)

1. result = X
2. Repeat:
     For each FD (A → B) in F:
       If A ⊆ result, then result = result ∪ B
   Until result doesn't change
3. Return result
```

**Example**:

```
Relation: R(A, B, C, D, E)
FDs: { A → B,  B → C,  A → D,  D → E }

Compute {A}⁺:
  Start:  {A}
  A → B:  {A, B}
  B → C:  {A, B, C}
  A → D:  {A, B, C, D}
  D → E:  {A, B, C, D, E}  ← all attributes!

{A}⁺ = {A, B, C, D, E} = R  ⟹  A is a superkey
Since no proper subset of {A} can determine all of R, A is a candidate key.
```

### Finding Candidate Keys

1. Compute closure of every subset of attributes (start from single attributes, then pairs, etc.)
2. A **superkey** is any set X where X⁺ = R (all attributes)
3. A **candidate key** is a **minimal** superkey — no proper subset is also a superkey

**Shortcut**: Attributes that never appear on the right side of any FD must be in every candidate key.

```
FDs: { A → B,  B → C,  CD → A }
Right-side attributes: B, C, A
Attributes never on right: D

D must be in every candidate key.
{D}⁺ = {D} ≠ R → not a superkey
{A, D}⁺ = {A, D, B, C} = R → AD is a superkey → check subsets → A alone? No. D alone? No.
→ AD is a candidate key.
{B, D}⁺ = {B, D, C, A} = R → BD is also a candidate key.
```

---

## 4. Canonical Cover (Minimal Cover)

A **canonical cover** Fc of an FD set F is a simplified equivalent set where:
1. No FD has extraneous attributes on the left
2. No FD has extraneous attributes on the right (right side is a single attribute)
3. No redundant FD can be removed

**Algorithm**:

```
1. Decompose: split all FDs so each right side has a single attribute
   A → BC  becomes  A → B, A → C

2. Minimize left sides: for each FD X → A,
   for each attribute B in X:
     if A ∈ (X - B)⁺ under current FDs, remove B from X

3. Remove redundant FDs: for each FD X → A,
   if A ∈ X⁺ under (F - {X → A}), remove the FD
```

**Example**:

```
F = { A → BC,  B → C,  AB → D }

Step 1: Decompose right sides
  A → B, A → C, B → C, AB → D

Step 2: Minimize left sides
  AB → D: Is D in {A}⁺? {A}⁺ = {A,B,C,D} (via A→B, B→C, and now checking AB→D: A→B so A→AB→D)
  Yes! A⁺ includes D. So simplify AB → D to A → D.

Step 3: Remove redundant FDs
  A → C: Is C in {A}⁺ under {A→B, B→C, A→D}? A→B→C. Yes! Remove A → C.

Canonical cover: { A → B,  B → C,  A → D }
```

---

## 5. First Normal Form (1NF)

### Definition

A relation is in **1NF** if and only if:
1. Every column contains only **atomic** (indivisible) values
2. No **repeating groups** or arrays
3. Each row is unique (has a primary key)

### The Problem: Non-1NF

```
student_courses (NOT in 1NF)
┌────────────┬──────────┬──────────────────────────────┐
│ student_id │ name     │ courses                      │
├────────────┼──────────┼──────────────────────────────┤
│ 101        │ Alice    │ CS201, CS301, MA101           │
│ 102        │ Bob      │ CS201                        │
│ 103        │ Carol    │ CS201, CS301                 │
└────────────┴──────────┴──────────────────────────────┘
Problem: "courses" column stores multiple values — not atomic!
```

### SVG: 1NF Before → After

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 340" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="340" fill="#f8f9fa" rx="12"/>
  <text x="410" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">1NF — Eliminate Non-Atomic Values</text>

  <!-- BEFORE -->
  <text x="175" y="58" text-anchor="middle" font-size="13" font-weight="bold" fill="#e74c3c">BEFORE (Not 1NF)</text>
  <rect x="30" y="68" width="290" height="28" rx="4" fill="#e74c3c"/>
  <text x="55" y="87" font-size="11" fill="white" font-weight="bold">student_id</text>
  <text x="145" y="87" font-size="11" fill="white" font-weight="bold">name</text>
  <text x="235" y="87" font-size="11" fill="white" font-weight="bold">courses</text>

  <rect x="30" y="96" width="290" height="24" fill="#fdf2f2"/>
  <text x="55" y="113" font-size="11" fill="#333">101</text>
  <text x="145" y="113" font-size="11" fill="#333">Alice</text>
  <text x="235" y="113" font-size="11" fill="#e74c3c" font-weight="bold">CS201, CS301</text>

  <rect x="30" y="120" width="290" height="24" fill="#fff"/>
  <text x="55" y="137" font-size="11" fill="#333">102</text>
  <text x="145" y="137" font-size="11" fill="#333">Bob</text>
  <text x="235" y="137" font-size="11" fill="#e74c3c" font-weight="bold">CS201</text>

  <rect x="30" y="144" width="290" height="24" fill="#fdf2f2"/>
  <text x="55" y="161" font-size="11" fill="#333">103</text>
  <text x="145" y="161" font-size="11" fill="#333">Carol</text>
  <text x="235" y="161" font-size="11" fill="#e74c3c" font-weight="bold">CS201, CS301</text>

  <!-- Arrow -->
  <line x1="340" y1="130" x2="460" y2="130" stroke="#2c3e50" stroke-width="2.5" marker-end="url(#arrowhead1nf)"/>
  <text x="400" y="118" text-anchor="middle" font-size="11" fill="#2c3e50" font-weight="bold">Split into</text>
  <text x="400" y="148" text-anchor="middle" font-size="11" fill="#2c3e50" font-weight="bold">atomic rows</text>
  <defs><marker id="arrowhead1nf" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#2c3e50"/></marker></defs>

  <!-- AFTER -->
  <text x="620" y="58" text-anchor="middle" font-size="13" font-weight="bold" fill="#27ae60">AFTER (1NF)</text>
  <rect x="480" y="68" width="310" height="28" rx="4" fill="#27ae60"/>
  <text x="510" y="87" font-size="11" fill="white" font-weight="bold">student_id</text>
  <text x="590" y="87" font-size="11" fill="white" font-weight="bold">name</text>
  <text x="690" y="87" font-size="11" fill="white" font-weight="bold">course_id</text>

  <rect x="480" y="96" width="310" height="24" fill="#eafaf1"/>
  <text x="510" y="113" font-size="11" fill="#333">101</text>
  <text x="590" y="113" font-size="11" fill="#333">Alice</text>
  <text x="690" y="113" font-size="11" fill="#27ae60" font-weight="bold">CS201</text>

  <rect x="480" y="120" width="310" height="24" fill="#fff"/>
  <text x="510" y="137" font-size="11" fill="#333">101</text>
  <text x="590" y="137" font-size="11" fill="#333">Alice</text>
  <text x="690" y="137" font-size="11" fill="#27ae60" font-weight="bold">CS301</text>

  <rect x="480" y="144" width="310" height="24" fill="#eafaf1"/>
  <text x="510" y="161" font-size="11" fill="#333">102</text>
  <text x="590" y="161" font-size="11" fill="#333">Bob</text>
  <text x="690" y="161" font-size="11" fill="#27ae60" font-weight="bold">CS201</text>

  <rect x="480" y="168" width="310" height="24" fill="#fff"/>
  <text x="510" y="185" font-size="11" fill="#333">103</text>
  <text x="590" y="185" font-size="11" fill="#333">Carol</text>
  <text x="690" y="185" font-size="11" fill="#27ae60" font-weight="bold">CS201</text>

  <rect x="480" y="192" width="310" height="24" fill="#eafaf1"/>
  <text x="510" y="209" font-size="11" fill="#333">103</text>
  <text x="590" y="209" font-size="11" fill="#333">Carol</text>
  <text x="690" y="209" font-size="11" fill="#27ae60" font-weight="bold">CS301</text>

  <!-- Note -->
  <text x="410" y="260" text-anchor="middle" font-size="12" fill="#555">PK changes from {student_id} to {student_id, course_id}</text>
  <text x="410" y="280" text-anchor="middle" font-size="12" fill="#555">Each cell now contains exactly one value</text>

  <!-- Violation label -->
  <rect x="210" y="175" width="108" height="22" rx="4" fill="#e74c3c" opacity="0.9"/>
  <text x="264" y="190" text-anchor="middle" font-size="10" fill="white" font-weight="bold">Multi-valued!</text>
</svg>

### SQL Fix

```sql
-- Non-1NF (conceptual — most DBMS won't let you do this directly)
-- courses stored as comma-separated string = bad

-- 1NF: one row per student-course pair
CREATE TABLE student_courses (
    student_id  INT,
    name        VARCHAR(100),
    course_id   CHAR(6),
    PRIMARY KEY (student_id, course_id)
);
```

> **Note**: The 1NF table still has redundancy (name repeats for each student). That's addressed in 2NF.

---

## 6. Second Normal Form (2NF)

### Definition

A relation is in **2NF** if and only if:
1. It is in 1NF
2. **No non-prime attribute** is **partially dependent** on any candidate key

In other words: every non-key column must depend on the **whole** key, not just part of it.

> **Only matters for composite keys.** If your PK is a single column, you're automatically in 2NF (there's no "part" of a single-column key).

### The Problem: Partial Dependencies

```
student_courses (1NF but NOT 2NF)
PK: {student_id, course_id}
FDs:
  {student_id, course_id} → grade         ← full dependency (OK)
  student_id → name, dept                  ← partial dependency (BAD — only part of PK)
  course_id → title, instructor            ← partial dependency (BAD — only part of PK)
```

```
┌────────────┬──────────┬─────────┬───────────┬─────────┬──────────────┬─────────┐
│ student_id │ name     │ dept    │ course_id │ title   │ instructor   │ grade   │
├────────────┼──────────┼─────────┼───────────┼─────────┼──────────────┼─────────┤
│ 101        │ Alice    │ CS      │ CS201     │ DBMS    │ Dr. Smith    │ A       │
│ 101        │ Alice    │ CS      │ CS301     │ OS      │ Dr. Patel    │ B+      │
│ 102        │ Bob      │ Math    │ CS201     │ DBMS    │ Dr. Smith    │ B       │
└────────────┴──────────┴─────────┴───────────┴─────────┴──────────────┴─────────┘

"Alice" and "CS" repeat because name/dept depend only on student_id, not on the full key.
"DBMS" and "Dr. Smith" repeat because title/instructor depend only on course_id.
```

### SVG: 2NF Before → After

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 480" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="480" fill="#f8f9fa" rx="12"/>
  <text x="410" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">2NF — Eliminate Partial Dependencies</text>

  <!-- BEFORE table -->
  <text x="410" y="55" text-anchor="middle" font-size="13" font-weight="bold" fill="#e74c3c">BEFORE (1NF, not 2NF)</text>
  <rect x="80" y="65" width="660" height="28" rx="4" fill="#e74c3c"/>
  <text x="110" y="84" font-size="10" fill="white" font-weight="bold">student_id</text>
  <text x="200" y="84" font-size="10" fill="white" font-weight="bold">name</text>
  <text x="275" y="84" font-size="10" fill="white" font-weight="bold">dept</text>
  <text x="360" y="84" font-size="10" fill="white" font-weight="bold">course_id</text>
  <text x="455" y="84" font-size="10" fill="white" font-weight="bold">title</text>
  <text x="555" y="84" font-size="10" fill="white" font-weight="bold">instructor</text>
  <text x="665" y="84" font-size="10" fill="white" font-weight="bold">grade</text>

  <rect x="80" y="93" width="660" height="22" fill="#fdf2f2"/>
  <text x="110" y="108" font-size="10" fill="#333">101</text>
  <text x="200" y="108" font-size="10" fill="#c0392b">Alice</text>
  <text x="275" y="108" font-size="10" fill="#c0392b">CS</text>
  <text x="360" y="108" font-size="10" fill="#333">CS201</text>
  <text x="455" y="108" font-size="10" fill="#8e44ad">DBMS</text>
  <text x="555" y="108" font-size="10" fill="#8e44ad">Dr. Smith</text>
  <text x="665" y="108" font-size="10" fill="#333">A</text>

  <rect x="80" y="115" width="660" height="22" fill="#fff"/>
  <text x="110" y="130" font-size="10" fill="#333">101</text>
  <text x="200" y="130" font-size="10" fill="#c0392b">Alice</text>
  <text x="275" y="130" font-size="10" fill="#c0392b">CS</text>
  <text x="360" y="130" font-size="10" fill="#333">CS301</text>
  <text x="455" y="130" font-size="10" fill="#333">OS</text>
  <text x="555" y="130" font-size="10" fill="#333">Dr. Patel</text>
  <text x="665" y="130" font-size="10" fill="#333">B+</text>

  <rect x="80" y="137" width="660" height="22" fill="#fdf2f2"/>
  <text x="110" y="152" font-size="10" fill="#333">102</text>
  <text x="200" y="152" font-size="10" fill="#333">Bob</text>
  <text x="275" y="152" font-size="10" fill="#333">Math</text>
  <text x="360" y="152" font-size="10" fill="#333">CS201</text>
  <text x="455" y="152" font-size="10" fill="#8e44ad">DBMS</text>
  <text x="555" y="152" font-size="10" fill="#8e44ad">Dr. Smith</text>
  <text x="665" y="152" font-size="10" fill="#333">B</text>

  <!-- Arrows down -->
  <line x1="200" y1="175" x2="200" y2="210" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr2nf)"/>
  <line x1="410" y1="175" x2="410" y2="210" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr2nf)"/>
  <line x1="620" y1="175" x2="620" y2="210" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr2nf)"/>
  <defs><marker id="arr2nf" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0,10 3.5,0 7" fill="#2c3e50"/></marker></defs>
  <text x="410" y="195" text-anchor="middle" font-size="11" fill="#2c3e50" font-weight="bold">Decompose</text>

  <!-- AFTER: 3 tables -->
  <!-- students -->
  <text x="130" y="230" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">students</text>
  <rect x="40" y="238" width="180" height="24" rx="4" fill="#27ae60"/>
  <text x="70" y="255" font-size="10" fill="white" font-weight="bold">student_id</text>
  <text x="145" y="255" font-size="10" fill="white" font-weight="bold">name</text>
  <text x="195" y="255" font-size="10" fill="white" font-weight="bold">dept</text>

  <rect x="40" y="262" width="180" height="22" fill="#eafaf1"/>
  <text x="70" y="278" font-size="10" fill="#333">101</text>
  <text x="145" y="278" font-size="10" fill="#333">Alice</text>
  <text x="195" y="278" font-size="10" fill="#333">CS</text>

  <rect x="40" y="284" width="180" height="22" fill="#fff"/>
  <text x="70" y="300" font-size="10" fill="#333">102</text>
  <text x="145" y="300" font-size="10" fill="#333">Bob</text>
  <text x="195" y="300" font-size="10" fill="#333">Math</text>

  <!-- courses -->
  <text x="415" y="230" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">courses</text>
  <rect x="310" y="238" width="210" height="24" rx="4" fill="#27ae60"/>
  <text x="340" y="255" font-size="10" fill="white" font-weight="bold">course_id</text>
  <text x="420" y="255" font-size="10" fill="white" font-weight="bold">title</text>
  <text x="490" y="255" font-size="10" fill="white" font-weight="bold">instructor</text>

  <rect x="310" y="262" width="210" height="22" fill="#eafaf1"/>
  <text x="340" y="278" font-size="10" fill="#333">CS201</text>
  <text x="420" y="278" font-size="10" fill="#333">DBMS</text>
  <text x="490" y="278" font-size="10" fill="#333">Dr. Smith</text>

  <rect x="310" y="284" width="210" height="22" fill="#fff"/>
  <text x="340" y="300" font-size="10" fill="#333">CS301</text>
  <text x="420" y="300" font-size="10" fill="#333">OS</text>
  <text x="490" y="300" font-size="10" fill="#333">Dr. Patel</text>

  <!-- enrollments -->
  <text x="670" y="230" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">enrollments</text>
  <rect x="580" y="238" width="210" height="24" rx="4" fill="#27ae60"/>
  <text x="610" y="255" font-size="10" fill="white" font-weight="bold">student_id</text>
  <text x="690" y="255" font-size="10" fill="white" font-weight="bold">course_id</text>
  <text x="765" y="255" font-size="10" fill="white" font-weight="bold">grade</text>

  <rect x="580" y="262" width="210" height="22" fill="#eafaf1"/>
  <text x="610" y="278" font-size="10" fill="#333">101</text>
  <text x="690" y="278" font-size="10" fill="#333">CS201</text>
  <text x="765" y="278" font-size="10" fill="#333">A</text>

  <rect x="580" y="284" width="210" height="22" fill="#fff"/>
  <text x="610" y="300" font-size="10" fill="#333">101</text>
  <text x="690" y="300" font-size="10" fill="#333">CS301</text>
  <text x="765" y="300" font-size="10" fill="#333">B+</text>

  <rect x="580" y="306" width="210" height="22" fill="#eafaf1"/>
  <text x="610" y="322" font-size="10" fill="#333">102</text>
  <text x="690" y="322" font-size="10" fill="#333">CS201</text>
  <text x="765" y="322" font-size="10" fill="#333">B</text>

  <!-- Labels -->
  <text x="130" y="330" text-anchor="middle" font-size="10" fill="#27ae60">PK: student_id</text>
  <text x="415" y="330" text-anchor="middle" font-size="10" fill="#27ae60">PK: course_id</text>
  <text x="670" y="350" text-anchor="middle" font-size="10" fill="#27ae60">PK: {student_id, course_id}</text>

  <!-- Explanation -->
  <rect x="80" y="380" width="660" height="55" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="1"/>
  <text x="410" y="400" text-anchor="middle" font-size="11" fill="#333">Partial dependency student_id → name, dept removed → separate students table</text>
  <text x="410" y="418" text-anchor="middle" font-size="11" fill="#333">Partial dependency course_id → title, instructor removed → separate courses table</text>
  <text x="410" y="436" text-anchor="middle" font-size="11" fill="#333">Only the full dependency {student_id, course_id} → grade remains in enrollments</text>

  <!-- Rule box -->
  <rect x="130" y="448" width="560" height="24" rx="6" fill="#2c3e50"/>
  <text x="410" y="465" text-anchor="middle" font-size="11" fill="white" font-weight="bold">Rule: Every non-key attribute must depend on the WHOLE key</text>
</svg>

### SQL Fix

```sql
CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    dept        VARCHAR(50) NOT NULL
);

CREATE TABLE courses (
    course_id   CHAR(6) PRIMARY KEY,
    title       VARCHAR(200) NOT NULL,
    instructor  VARCHAR(100) NOT NULL
);

CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(6) REFERENCES courses(course_id),
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

---

## 7. Third Normal Form (3NF)

### Definition

A relation is in **3NF** if and only if:
1. It is in 2NF
2. **No non-prime attribute** is **transitively dependent** on any candidate key

Formal definition: For every non-trivial FD X → A, at least one of these holds:
- X is a superkey, **OR**
- A is a prime attribute (part of some candidate key)

> **Informal rule (Bill Kent's definition)**: "Every non-key attribute must depend on the key, the whole key, and nothing but the key — so help me Codd."

### The Problem: Transitive Dependencies

```
students (2NF but NOT 3NF)
PK: student_id
FDs:
  student_id → dept            (direct)
  dept → dept_building         (dept determines building)
  ∴ student_id → dept → dept_building   (TRANSITIVE — student_id determines building through dept)

┌────────────┬──────────┬─────────┬───────────────┐
│ student_id │ name     │ dept    │ dept_building  │
├────────────┼──────────┼─────────┼───────────────┤
│ 101        │ Alice    │ CS      │ Turing Hall    │
│ 102        │ Bob      │ Math    │ Euler Hall     │
│ 103        │ Carol    │ CS      │ Turing Hall    │
└────────────┴──────────┴─────────┴───────────────┘

"Turing Hall" repeats because it depends on dept, not directly on student_id.
```

### SVG: 3NF Before → After

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 410" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="410" fill="#f8f9fa" rx="12"/>
  <text x="410" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">3NF — Eliminate Transitive Dependencies</text>

  <!-- BEFORE -->
  <text x="410" y="55" text-anchor="middle" font-size="13" font-weight="bold" fill="#e74c3c">BEFORE (2NF, not 3NF)</text>
  <rect x="150" y="65" width="520" height="26" rx="4" fill="#e74c3c"/>
  <text x="195" y="83" font-size="10" fill="white" font-weight="bold">student_id</text>
  <text x="295" y="83" font-size="10" fill="white" font-weight="bold">name</text>
  <text x="395" y="83" font-size="10" fill="white" font-weight="bold">dept</text>
  <text x="530" y="83" font-size="10" fill="white" font-weight="bold">dept_building</text>

  <rect x="150" y="91" width="520" height="22" fill="#fdf2f2"/>
  <text x="195" y="106" font-size="10" fill="#333">101</text>
  <text x="295" y="106" font-size="10" fill="#333">Alice</text>
  <text x="395" y="106" font-size="10" fill="#333">CS</text>
  <text x="530" y="106" font-size="10" fill="#c0392b">Turing Hall</text>

  <rect x="150" y="113" width="520" height="22" fill="#fff"/>
  <text x="195" y="128" font-size="10" fill="#333">102</text>
  <text x="295" y="128" font-size="10" fill="#333">Bob</text>
  <text x="395" y="128" font-size="10" fill="#333">Math</text>
  <text x="530" y="128" font-size="10" fill="#333">Euler Hall</text>

  <rect x="150" y="135" width="520" height="22" fill="#fdf2f2"/>
  <text x="195" y="150" font-size="10" fill="#333">103</text>
  <text x="295" y="150" font-size="10" fill="#333">Carol</text>
  <text x="395" y="150" font-size="10" fill="#333">CS</text>
  <text x="530" y="150" font-size="10" fill="#c0392b">Turing Hall</text>

  <!-- FD chain -->
  <text x="410" y="185" text-anchor="middle" font-size="11" fill="#e74c3c" font-weight="bold">student_id → dept → dept_building (transitive!)</text>

  <line x1="410" y1="192" x2="410" y2="215" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr3nf)"/>
  <defs><marker id="arr3nf" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0,10 3.5,0 7" fill="#2c3e50"/></marker></defs>

  <!-- AFTER: 2 tables -->
  <text x="250" y="238" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">students</text>
  <rect x="110" y="246" width="280" height="24" rx="4" fill="#27ae60"/>
  <text x="145" y="263" font-size="10" fill="white" font-weight="bold">student_id</text>
  <text x="250" y="263" font-size="10" fill="white" font-weight="bold">name</text>
  <text x="345" y="263" font-size="10" fill="white" font-weight="bold">dept</text>

  <rect x="110" y="270" width="280" height="22" fill="#eafaf1"/>
  <text x="145" y="286" font-size="10" fill="#333">101</text>
  <text x="250" y="286" font-size="10" fill="#333">Alice</text>
  <text x="345" y="286" font-size="10" fill="#333">CS</text>

  <rect x="110" y="292" width="280" height="22" fill="#fff"/>
  <text x="145" y="308" font-size="10" fill="#333">102</text>
  <text x="250" y="308" font-size="10" fill="#333">Bob</text>
  <text x="345" y="308" font-size="10" fill="#333">Math</text>

  <rect x="110" y="314" width="280" height="22" fill="#eafaf1"/>
  <text x="145" y="330" font-size="10" fill="#333">103</text>
  <text x="250" y="330" font-size="10" fill="#333">Carol</text>
  <text x="345" y="330" font-size="10" fill="#333">CS</text>

  <!-- departments -->
  <text x="600" y="238" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">departments</text>
  <rect x="480" y="246" width="240" height="24" rx="4" fill="#27ae60"/>
  <text x="530" y="263" font-size="10" fill="white" font-weight="bold">dept</text>
  <text x="645" y="263" font-size="10" fill="white" font-weight="bold">dept_building</text>

  <rect x="480" y="270" width="240" height="22" fill="#eafaf1"/>
  <text x="530" y="286" font-size="10" fill="#333">CS</text>
  <text x="645" y="286" font-size="10" fill="#333">Turing Hall</text>

  <rect x="480" y="292" width="240" height="22" fill="#fff"/>
  <text x="530" y="308" font-size="10" fill="#333">Math</text>
  <text x="645" y="308" font-size="10" fill="#333">Euler Hall</text>

  <!-- Rule -->
  <rect x="130" y="365" width="560" height="24" rx="6" fill="#2c3e50"/>
  <text x="410" y="382" text-anchor="middle" font-size="11" fill="white" font-weight="bold">Rule: No non-key attribute may depend on another non-key attribute</text>
</svg>

### SQL Fix

```sql
CREATE TABLE departments (
    dept          VARCHAR(50) PRIMARY KEY,
    dept_building VARCHAR(100) NOT NULL
);

CREATE TABLE students (
    student_id  INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    dept        VARCHAR(50) NOT NULL REFERENCES departments(dept)
);
```

---

## 8. Boyce-Codd Normal Form (BCNF)

### Definition

A relation is in **BCNF** if and only if:

For every non-trivial FD X → Y, **X is a superkey**.

> BCNF is stricter than 3NF. The 3NF exception ("or A is prime") is removed. In BCNF, the **only** functional dependencies allowed are those where the left side is a superkey.

### 3NF vs. BCNF — The Difference

3NF allows a non-trivial FD X → A if A is a prime attribute (part of a candidate key). BCNF does not allow this exception.

```
Relation: R(student, subject, professor)
FDs:
  {student, subject} → professor      (a student in a subject has one professor)
  professor → subject                  (each professor teaches exactly one subject)

Candidate keys: {student, subject}  (also {student, professor})
```

The FD `professor → subject` violates BCNF because `professor` is **not** a superkey. But it satisfies 3NF because `subject` is a **prime attribute** (part of candidate key {student, subject}).

### SVG: BCNF Before → After

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 430" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="430" fill="#f8f9fa" rx="12"/>
  <text x="410" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">BCNF — Every Determinant Must Be a Superkey</text>

  <!-- BEFORE -->
  <text x="410" y="58" text-anchor="middle" font-size="13" font-weight="bold" fill="#e74c3c">BEFORE (3NF, not BCNF)</text>
  <rect x="180" y="68" width="460" height="26" rx="4" fill="#e74c3c"/>
  <text x="250" y="86" font-size="10" fill="white" font-weight="bold">student</text>
  <text x="370" y="86" font-size="10" fill="white" font-weight="bold">subject</text>
  <text x="510" y="86" font-size="10" fill="white" font-weight="bold">professor</text>

  <rect x="180" y="94" width="460" height="22" fill="#fdf2f2"/>
  <text x="250" y="110" font-size="10" fill="#333">Alice</text>
  <text x="370" y="110" font-size="10" fill="#333">DBMS</text>
  <text x="510" y="110" font-size="10" fill="#333">Dr. Smith</text>

  <rect x="180" y="116" width="460" height="22" fill="#fff"/>
  <text x="250" y="132" font-size="10" fill="#333">Alice</text>
  <text x="370" y="132" font-size="10" fill="#333">OS</text>
  <text x="510" y="132" font-size="10" fill="#333">Dr. Patel</text>

  <rect x="180" y="138" width="460" height="22" fill="#fdf2f2"/>
  <text x="250" y="154" font-size="10" fill="#333">Bob</text>
  <text x="370" y="154" font-size="10" fill="#333">DBMS</text>
  <text x="510" y="154" font-size="10" fill="#333">Dr. Smith</text>

  <rect x="180" y="160" width="460" height="22" fill="#fff"/>
  <text x="250" y="176" font-size="10" fill="#333">Carol</text>
  <text x="370" y="176" font-size="10" fill="#333">DBMS</text>
  <text x="510" y="176" font-size="10" fill="#333">Dr. Lee</text>

  <!-- Violation label -->
  <text x="410" y="205" text-anchor="middle" font-size="11" fill="#e74c3c" font-weight="bold">professor → subject : professor is NOT a superkey!</text>

  <line x1="410" y1="212" x2="410" y2="240" stroke="#2c3e50" stroke-width="2" marker-end="url(#arrbcnf)"/>
  <defs><marker id="arrbcnf" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0,10 3.5,0 7" fill="#2c3e50"/></marker></defs>

  <!-- AFTER -->
  <!-- prof_subject -->
  <text x="240" y="262" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">prof_subject</text>
  <rect x="120" y="270" width="240" height="24" rx="4" fill="#27ae60"/>
  <text x="190" y="287" font-size="10" fill="white" font-weight="bold">professor</text>
  <text x="300" y="287" font-size="10" fill="white" font-weight="bold">subject</text>

  <rect x="120" y="294" width="240" height="22" fill="#eafaf1"/>
  <text x="190" y="310" font-size="10" fill="#333">Dr. Smith</text>
  <text x="300" y="310" font-size="10" fill="#333">DBMS</text>

  <rect x="120" y="316" width="240" height="22" fill="#fff"/>
  <text x="190" y="332" font-size="10" fill="#333">Dr. Patel</text>
  <text x="300" y="332" font-size="10" fill="#333">OS</text>

  <rect x="120" y="338" width="240" height="22" fill="#eafaf1"/>
  <text x="190" y="354" font-size="10" fill="#333">Dr. Lee</text>
  <text x="300" y="354" font-size="10" fill="#333">DBMS</text>

  <!-- student_prof -->
  <text x="580" y="262" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">student_prof</text>
  <rect x="460" y="270" width="240" height="24" rx="4" fill="#27ae60"/>
  <text x="530" y="287" font-size="10" fill="white" font-weight="bold">student</text>
  <text x="640" y="287" font-size="10" fill="white" font-weight="bold">professor</text>

  <rect x="460" y="294" width="240" height="22" fill="#eafaf1"/>
  <text x="530" y="310" font-size="10" fill="#333">Alice</text>
  <text x="640" y="310" font-size="10" fill="#333">Dr. Smith</text>

  <rect x="460" y="316" width="240" height="22" fill="#fff"/>
  <text x="530" y="332" font-size="10" fill="#333">Alice</text>
  <text x="640" y="332" font-size="10" fill="#333">Dr. Patel</text>

  <rect x="460" y="338" width="240" height="22" fill="#eafaf1"/>
  <text x="530" y="354" font-size="10" fill="#333">Bob</text>
  <text x="640" y="354" font-size="10" fill="#333">Dr. Smith</text>

  <rect x="460" y="360" width="240" height="22" fill="#fff"/>
  <text x="530" y="376" font-size="10" fill="#333">Carol</text>
  <text x="640" y="376" font-size="10" fill="#333">Dr. Lee</text>

  <!-- Labels -->
  <text x="240" y="378" text-anchor="middle" font-size="10" fill="#27ae60">PK: professor</text>
  <text x="580" y="398" text-anchor="middle" font-size="10" fill="#27ae60">PK: {student, professor}</text>

  <!-- Rule -->
  <rect x="130" y="404" width="560" height="22" rx="6" fill="#2c3e50"/>
  <text x="410" y="419" text-anchor="middle" font-size="11" fill="white" font-weight="bold">Rule: Left side of every non-trivial FD must be a superkey</text>
</svg>

### The Trade-Off: BCNF May Lose Dependency Preservation

In the decomposition above, the FD `{student, subject} → professor` is **lost** — it can't be checked within either table alone. This is the classic BCNF vs. 3NF trade-off:

| Property | 3NF Decomposition | BCNF Decomposition |
|---|---|---|
| Lossless join | Always achievable | Always achievable |
| Dependency preserving | Always achievable | **Not always** |
| Redundancy | Some may remain | Minimal |

> **Practical guideline**: Most real schemas reach BCNF without losing dependency preservation. Only when the two conflict (rare) do you choose between 3NF (preserve all FDs) and BCNF (eliminate all redundancy).

---

## 9. Multi-Valued Dependencies & Fourth Normal Form (4NF)

### Multi-Valued Dependencies (MVDs)

An **MVD** X ↠ Y means: for a given X, the set of Y values is independent of the other attributes (R - X - Y).

> "The set of Y-values associated with X doesn't depend on what Z-values exist."

**Example**:

```
employee_skills_languages
┌──────────┬───────────┬──────────┐
│ emp_id   │ skill     │ language │
├──────────┼───────────┼──────────┤
│ 101      │ Java      │ English  │
│ 101      │ Java      │ French   │
│ 101      │ Python    │ English  │
│ 101      │ Python    │ French   │
│ 102      │ SQL       │ English  │
└──────────┴───────────┴──────────┘

MVDs: emp_id ↠ skill  and  emp_id ↠ language
Skills and languages are independent of each other — yet we must store every combination.
```

### 4NF Definition

A relation is in **4NF** if and only if:
1. It is in BCNF
2. For every non-trivial MVD X ↠ Y, **X is a superkey**

### SVG: 4NF Before → After

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 420" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="420" fill="#f8f9fa" rx="12"/>
  <text x="410" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">4NF — Eliminate Multi-Valued Dependencies</text>

  <!-- BEFORE -->
  <text x="410" y="58" text-anchor="middle" font-size="13" font-weight="bold" fill="#e74c3c">BEFORE (BCNF, not 4NF)</text>
  <rect x="220" y="68" width="380" height="26" rx="4" fill="#e74c3c"/>
  <text x="290" y="86" font-size="10" fill="white" font-weight="bold">emp_id</text>
  <text x="390" y="86" font-size="10" fill="white" font-weight="bold">skill</text>
  <text x="510" y="86" font-size="10" fill="white" font-weight="bold">language</text>

  <rect x="220" y="94" width="380" height="20" fill="#fdf2f2"/>
  <text x="290" y="108" font-size="10" fill="#333">101</text>
  <text x="390" y="108" font-size="10" fill="#c0392b">Java</text>
  <text x="510" y="108" font-size="10" fill="#8e44ad">English</text>

  <rect x="220" y="114" width="380" height="20" fill="#fff"/>
  <text x="290" y="128" font-size="10" fill="#333">101</text>
  <text x="390" y="128" font-size="10" fill="#c0392b">Java</text>
  <text x="510" y="128" font-size="10" fill="#8e44ad">French</text>

  <rect x="220" y="134" width="380" height="20" fill="#fdf2f2"/>
  <text x="290" y="148" font-size="10" fill="#333">101</text>
  <text x="390" y="148" font-size="10" fill="#c0392b">Python</text>
  <text x="510" y="148" font-size="10" fill="#8e44ad">English</text>

  <rect x="220" y="154" width="380" height="20" fill="#fff"/>
  <text x="290" y="168" font-size="10" fill="#333">101</text>
  <text x="390" y="168" font-size="10" fill="#c0392b">Python</text>
  <text x="510" y="168" font-size="10" fill="#8e44ad">French</text>

  <!-- Explanation -->
  <text x="410" y="200" text-anchor="middle" font-size="11" fill="#e74c3c" font-weight="bold">2 skills x 2 languages = 4 rows! Skills and languages are independent.</text>

  <line x1="410" y1="210" x2="410" y2="240" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr4nf)"/>
  <defs><marker id="arr4nf" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0,10 3.5,0 7" fill="#2c3e50"/></marker></defs>

  <!-- AFTER: 2 tables -->
  <!-- emp_skills -->
  <text x="240" y="262" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">emp_skills</text>
  <rect x="130" y="270" width="220" height="24" rx="4" fill="#27ae60"/>
  <text x="195" y="287" font-size="10" fill="white" font-weight="bold">emp_id</text>
  <text x="295" y="287" font-size="10" fill="white" font-weight="bold">skill</text>

  <rect x="130" y="294" width="220" height="22" fill="#eafaf1"/>
  <text x="195" y="310" font-size="10" fill="#333">101</text>
  <text x="295" y="310" font-size="10" fill="#333">Java</text>

  <rect x="130" y="316" width="220" height="22" fill="#fff"/>
  <text x="195" y="332" font-size="10" fill="#333">101</text>
  <text x="295" y="332" font-size="10" fill="#333">Python</text>

  <!-- emp_languages -->
  <text x="580" y="262" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">emp_languages</text>
  <rect x="470" y="270" width="220" height="24" rx="4" fill="#27ae60"/>
  <text x="535" y="287" font-size="10" fill="white" font-weight="bold">emp_id</text>
  <text x="635" y="287" font-size="10" fill="white" font-weight="bold">language</text>

  <rect x="470" y="294" width="220" height="22" fill="#eafaf1"/>
  <text x="535" y="310" font-size="10" fill="#333">101</text>
  <text x="635" y="310" font-size="10" fill="#333">English</text>

  <rect x="470" y="316" width="220" height="22" fill="#fff"/>
  <text x="535" y="332" font-size="10" fill="#333">101</text>
  <text x="635" y="332" font-size="10" fill="#333">French</text>

  <!-- Result -->
  <text x="410" y="365" text-anchor="middle" font-size="11" fill="#27ae60" font-weight="bold">2 + 2 = 4 rows instead of 2 x 2 = 4 rows (savings grow with data)</text>
  <text x="410" y="385" text-anchor="middle" font-size="11" fill="#555">With 10 skills and 5 languages: 50 rows → 15 rows</text>

  <!-- Rule -->
  <rect x="130" y="398" width="560" height="22" rx="6" fill="#2c3e50"/>
  <text x="410" y="413" text-anchor="middle" font-size="11" fill="white" font-weight="bold">Rule: Independent multi-valued facts must be stored in separate tables</text>
</svg>

### SQL Fix

```sql
CREATE TABLE emp_skills (
    emp_id  INT,
    skill   VARCHAR(50),
    PRIMARY KEY (emp_id, skill)
);

CREATE TABLE emp_languages (
    emp_id    INT,
    language  VARCHAR(50),
    PRIMARY KEY (emp_id, language)
);
```

---

## 10. Join Dependencies & Fifth Normal Form (5NF)

### Join Dependencies (JDs)

A **join dependency** ✱{R1, R2, ..., Rn} holds if R can be reconstructed by naturally joining R1, R2, ..., Rn without losing or gaining any tuples.

5NF handles cases where a relation can't be decomposed into two tables without loss, but **can** be decomposed into three or more.

### 5NF Definition

A relation is in **5NF** (also called **Project-Join Normal Form — PJNF**) if and only if:
1. It is in 4NF
2. Every non-trivial join dependency is implied by the candidate keys

### The Classic Example: Suppliers–Parts–Projects

```
supplies (4NF but NOT 5NF)
┌──────────┬──────┬─────────┐
│ supplier │ part │ project │
├──────────┼──────┼─────────┤
│ S1       │ P1   │ J1      │
│ S1       │ P2   │ J1      │
│ S2       │ P1   │ J1      │
│ S1       │ P1   │ J2      │
└──────────┴──────┴─────────┘

Business rule: if supplier S supplies part P, and S supplies to project J,
and someone supplies part P to project J — then S supplies P to J.

This "cyclic" constraint means the table can be losslessly decomposed into THREE projections:
  SP(supplier, part)
  PJ(part, project)
  SJ(supplier, project)

Joining SP ⋈ PJ ⋈ SJ reconstructs the original exactly.
```

### SVG: 5NF Decomposition

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 400" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="400" fill="#f8f9fa" rx="12"/>
  <text x="410" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#2c3e50">5NF — Eliminate Join Dependencies</text>

  <!-- BEFORE -->
  <text x="410" y="55" text-anchor="middle" font-size="13" font-weight="bold" fill="#e74c3c">BEFORE (4NF, not 5NF)</text>
  <rect x="240" y="65" width="340" height="24" rx="4" fill="#e74c3c"/>
  <text x="310" y="82" font-size="10" fill="white" font-weight="bold">supplier</text>
  <text x="400" y="82" font-size="10" fill="white" font-weight="bold">part</text>
  <text x="500" y="82" font-size="10" fill="white" font-weight="bold">project</text>

  <rect x="240" y="89" width="340" height="20" fill="#fdf2f2"/>
  <text x="310" y="103" font-size="10" fill="#333">S1</text>
  <text x="400" y="103" font-size="10" fill="#333">P1</text>
  <text x="500" y="103" font-size="10" fill="#333">J1</text>

  <rect x="240" y="109" width="340" height="20" fill="#fff"/>
  <text x="310" y="123" font-size="10" fill="#333">S1</text>
  <text x="400" y="123" font-size="10" fill="#333">P2</text>
  <text x="500" y="123" font-size="10" fill="#333">J1</text>

  <rect x="240" y="129" width="340" height="20" fill="#fdf2f2"/>
  <text x="310" y="143" font-size="10" fill="#333">S2</text>
  <text x="400" y="143" font-size="10" fill="#333">P1</text>
  <text x="500" y="143" font-size="10" fill="#333">J1</text>

  <rect x="240" y="149" width="340" height="20" fill="#fff"/>
  <text x="310" y="163" font-size="10" fill="#333">S1</text>
  <text x="400" y="163" font-size="10" fill="#333">P1</text>
  <text x="500" y="163" font-size="10" fill="#333">J2</text>

  <!-- Can't split into 2 — must split into 3 -->
  <text x="410" y="195" text-anchor="middle" font-size="11" fill="#e74c3c" font-weight="bold">Cannot decompose into 2 tables losslessly — need 3!</text>

  <line x1="250" y1="208" x2="150" y2="245" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr5nf)"/>
  <line x1="410" y1="208" x2="410" y2="245" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr5nf)"/>
  <line x1="570" y1="208" x2="670" y2="245" stroke="#2c3e50" stroke-width="2" marker-end="url(#arr5nf)"/>
  <defs><marker id="arr5nf" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0,10 3.5,0 7" fill="#2c3e50"/></marker></defs>

  <!-- SP -->
  <text x="130" y="260" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">SP</text>
  <rect x="50" y="268" width="160" height="22" rx="4" fill="#27ae60"/>
  <text x="100" y="284" font-size="10" fill="white" font-weight="bold">supplier</text>
  <text x="170" y="284" font-size="10" fill="white" font-weight="bold">part</text>

  <rect x="50" y="290" width="160" height="18" fill="#eafaf1"/>
  <text x="100" y="303" font-size="10" fill="#333">S1</text>
  <text x="170" y="303" font-size="10" fill="#333">P1</text>
  <rect x="50" y="308" width="160" height="18" fill="#fff"/>
  <text x="100" y="321" font-size="10" fill="#333">S1</text>
  <text x="170" y="321" font-size="10" fill="#333">P2</text>
  <rect x="50" y="326" width="160" height="18" fill="#eafaf1"/>
  <text x="100" y="339" font-size="10" fill="#333">S2</text>
  <text x="170" y="339" font-size="10" fill="#333">P1</text>

  <!-- PJ -->
  <text x="410" y="260" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">PJ</text>
  <rect x="330" y="268" width="160" height="22" rx="4" fill="#27ae60"/>
  <text x="380" y="284" font-size="10" fill="white" font-weight="bold">part</text>
  <text x="450" y="284" font-size="10" fill="white" font-weight="bold">project</text>

  <rect x="330" y="290" width="160" height="18" fill="#eafaf1"/>
  <text x="380" y="303" font-size="10" fill="#333">P1</text>
  <text x="450" y="303" font-size="10" fill="#333">J1</text>
  <rect x="330" y="308" width="160" height="18" fill="#fff"/>
  <text x="380" y="321" font-size="10" fill="#333">P2</text>
  <text x="450" y="321" font-size="10" fill="#333">J1</text>
  <rect x="330" y="326" width="160" height="18" fill="#eafaf1"/>
  <text x="380" y="339" font-size="10" fill="#333">P1</text>
  <text x="450" y="339" font-size="10" fill="#333">J2</text>

  <!-- SJ -->
  <text x="690" y="260" text-anchor="middle" font-size="12" font-weight="bold" fill="#27ae60">SJ</text>
  <rect x="610" y="268" width="160" height="22" rx="4" fill="#27ae60"/>
  <text x="660" y="284" font-size="10" fill="white" font-weight="bold">supplier</text>
  <text x="730" y="284" font-size="10" fill="white" font-weight="bold">project</text>

  <rect x="610" y="290" width="160" height="18" fill="#eafaf1"/>
  <text x="660" y="303" font-size="10" fill="#333">S1</text>
  <text x="730" y="303" font-size="10" fill="#333">J1</text>
  <rect x="610" y="308" width="160" height="18" fill="#fff"/>
  <text x="660" y="321" font-size="10" fill="#333">S2</text>
  <text x="730" y="321" font-size="10" fill="#333">J1</text>
  <rect x="610" y="326" width="160" height="18" fill="#eafaf1"/>
  <text x="660" y="339" font-size="10" fill="#333">S1</text>
  <text x="730" y="339" font-size="10" fill="#333">J2</text>

  <!-- Reconstruction -->
  <rect x="130" y="365" width="560" height="28" rx="6" fill="#2c3e50"/>
  <text x="410" y="378" text-anchor="middle" font-size="11" fill="white" font-weight="bold">SP ⋈ PJ ⋈ SJ = original table (lossless)</text>
  <text x="410" y="393" text-anchor="middle" font-size="10" fill="#bbb">5NF is rarely needed in practice — most schemas stop at BCNF or 3NF</text>
</svg>

---

## 11. Normalization Roadmap — SVG

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 560" font-family="Segoe UI, Arial, sans-serif">
  <rect width="820" height="560" fill="#f8f9fa" rx="12"/>
  <text x="410" y="30" text-anchor="middle" font-size="18" font-weight="bold" fill="#2c3e50">Normalization Roadmap</text>

  <!-- UNF -->
  <rect x="295" y="48" width="230" height="38" rx="8" fill="#95a5a6"/>
  <text x="410" y="72" text-anchor="middle" fill="white" font-size="13" font-weight="bold">Unnormalized Form (UNF)</text>
  <line x1="410" y1="86" x2="410" y2="108" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- Test 1NF -->
  <rect x="245" y="108" width="330" height="28" rx="6" fill="#e67e22"/>
  <text x="410" y="127" text-anchor="middle" fill="white" font-size="11">Remove non-atomic values &amp; repeating groups</text>
  <line x1="410" y1="136" x2="410" y2="158" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- 1NF -->
  <rect x="315" y="158" width="190" height="36" rx="8" fill="#2980b9"/>
  <text x="410" y="181" text-anchor="middle" fill="white" font-size="13" font-weight="bold">1NF</text>
  <line x1="410" y1="194" x2="410" y2="216" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- Test 2NF -->
  <rect x="245" y="216" width="330" height="28" rx="6" fill="#e67e22"/>
  <text x="410" y="235" text-anchor="middle" fill="white" font-size="11">Remove partial dependencies on candidate keys</text>
  <line x1="410" y1="244" x2="410" y2="266" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- 2NF -->
  <rect x="315" y="266" width="190" height="36" rx="8" fill="#2980b9"/>
  <text x="410" y="289" text-anchor="middle" fill="white" font-size="13" font-weight="bold">2NF</text>
  <line x1="410" y1="302" x2="410" y2="324" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- Test 3NF -->
  <rect x="245" y="324" width="330" height="28" rx="6" fill="#e67e22"/>
  <text x="410" y="343" text-anchor="middle" fill="white" font-size="11">Remove transitive dependencies on candidate keys</text>
  <line x1="410" y1="352" x2="410" y2="374" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- 3NF -->
  <rect x="315" y="374" width="190" height="36" rx="8" fill="#27ae60"/>
  <text x="410" y="397" text-anchor="middle" fill="white" font-size="13" font-weight="bold">3NF</text>
  <line x1="410" y1="410" x2="410" y2="428" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- Test BCNF -->
  <rect x="245" y="428" width="330" height="28" rx="6" fill="#e67e22"/>
  <text x="410" y="447" text-anchor="middle" fill="white" font-size="11">Every determinant is a superkey</text>
  <line x1="410" y1="456" x2="410" y2="478" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>

  <!-- BCNF -->
  <rect x="315" y="478" width="190" height="36" rx="8" fill="#27ae60"/>
  <text x="410" y="501" text-anchor="middle" fill="white" font-size="13" font-weight="bold">BCNF</text>

  <!-- 4NF and 5NF side branch -->
  <line x1="505" y1="496" x2="590" y2="496" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>
  <rect x="590" y="478" width="190" height="36" rx="8" fill="#8e44ad"/>
  <text x="685" y="492" text-anchor="middle" fill="white" font-size="11" font-weight="bold">4NF</text>
  <text x="685" y="507" text-anchor="middle" fill="white" font-size="9">No non-trivial MVDs</text>

  <line x1="685" y1="514" x2="685" y2="533" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrmap)"/>
  <rect x="590" y="533" width="190" height="24" rx="8" fill="#8e44ad"/>
  <text x="685" y="550" text-anchor="middle" fill="white" font-size="11" font-weight="bold">5NF (PJNF)</text>

  <!-- Side labels -->
  <text x="108" y="183" text-anchor="middle" font-size="10" fill="#555">Atomic values</text>
  <text x="108" y="196" text-anchor="middle" font-size="10" fill="#555">only</text>
  <text x="108" y="291" text-anchor="middle" font-size="10" fill="#555">Full FDs on</text>
  <text x="108" y="304" text-anchor="middle" font-size="10" fill="#555">whole key</text>
  <text x="108" y="397" text-anchor="middle" font-size="10" fill="#555">No transitive</text>
  <text x="108" y="410" text-anchor="middle" font-size="10" fill="#555">FDs</text>
  <text x="108" y="501" text-anchor="middle" font-size="10" fill="#555">All determinants</text>
  <text x="108" y="514" text-anchor="middle" font-size="10" fill="#555">are superkeys</text>

  <!-- Practical target -->
  <rect x="25" y="374" width="80" height="50" rx="6" fill="#27ae60" opacity="0.2" stroke="#27ae60" stroke-width="2"/>
  <text x="65" y="395" text-anchor="middle" font-size="9" fill="#27ae60" font-weight="bold">Practical</text>
  <text x="65" y="408" text-anchor="middle" font-size="9" fill="#27ae60" font-weight="bold">Target</text>
  <rect x="25" y="478" width="80" height="36" rx="6" fill="#27ae60" opacity="0.1" stroke="#27ae60" stroke-width="1" stroke-dasharray="4"/>
  <text x="65" y="500" text-anchor="middle" font-size="9" fill="#27ae60">Ideal</text>

  <defs><marker id="arrmap" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0,10 3.5,0 7" fill="#7f8c8d"/></marker></defs>
</svg>

---

## 12. Decomposition Theory

### Lossless-Join Decomposition

A decomposition of R into R1 and R2 is **lossless** (no spurious tuples) if and only if:

```
R1 ∩ R2 → R1   OR   R1 ∩ R2 → R2
```

In other words, the common attributes must be a superkey of at least one of the resulting tables.

**Example**:

```
R(student_id, name, dept, dept_building)
Decompose into:
  R1(student_id, name, dept)     — PK: student_id
  R2(dept, dept_building)         — PK: dept

R1 ∩ R2 = {dept}
dept → dept_building  means  dept is a key of R2  ✓  → Lossless!
```

### Dependency-Preserving Decomposition

A decomposition is **dependency preserving** if every FD in the original set F can be verified using only the attributes of a single decomposed relation (without needing to join tables).

**Example**:

```
R(A, B, C)  FDs: { A → B, B → C }
Decompose: R1(A, B), R2(B, C)

A → B: can check in R1 ✓
B → C: can check in R2 ✓
→ Dependency preserving!
```

**Counter-example (BCNF decomposition that loses an FD)**:

```
R(student, subject, professor)
FDs: {student, subject} → professor,  professor → subject

BCNF decompose:
  R1(professor, subject)      — checks professor → subject ✓
  R2(student, professor)      — can't check {student, subject} → professor ✗

The FD {student, subject} → professor is LOST — requires joining R1 and R2 to verify.
```

### Decomposition Algorithm for BCNF

```
Algorithm: BCNF Decomposition
Input: Relation R with FD set F

1. If R is in BCNF, return {R}
2. Find a non-trivial FD X → Y that violates BCNF (X is not a superkey)
3. Decompose R into:
     R1 = X ∪ Y           (the violating FD becomes the key of R1)
     R2 = R - Y + X       (everything else, keeping X as the link)
4. Recursively apply to R1 and R2
```

### Decomposition Algorithm for 3NF (Synthesis)

```
Algorithm: 3NF Synthesis (always dependency-preserving + lossless)
Input: Relation R with FD set F

1. Compute canonical cover Fc of F
2. For each FD X → A in Fc:
     Create a relation Ri(X, A)
3. If no relation contains a candidate key of R:
     Add a relation with a candidate key
4. Remove any relation that is a subset of another
```

### Comparison

| Property | BCNF Decomposition | 3NF Synthesis |
|---|---|---|
| Lossless join | Always | Always |
| Dependency preserving | **Not always** | **Always** |
| Redundancy | None from FDs | May have slight redundancy |
| Algorithm | Iterative decomposition | Synthesis from canonical cover |
| When to prefer | Most schemas | When BCNF loses an FD |

---

## 13. Denormalization — When & Why

### Why Denormalize?

Normalization optimizes for **write correctness** and **storage efficiency**. But in read-heavy workloads, excessive joins can destroy query performance.

**Denormalization** deliberately reintroduces controlled redundancy to speed up reads.

### When to Denormalize

| Scenario | Example | Denormalization Approach |
|---|---|---|
| **Read-heavy dashboards** | Analytics reporting page | Pre-computed summary tables |
| **Expensive joins** | A query joining 8 tables for a single display | Store derived columns in the parent table |
| **Caching frequently-accessed data** | User profile + latest order | Embed latest_order_id in users table |
| **OLAP / Data warehousing** | Star schema for BI tools | Fact + dimension tables with intentional redundancy |
| **Document-oriented apps** | Blog post with author info | Embed author_name in posts (accept staleness) |

### Common Denormalization Techniques

```sql
-- 1. Storing a derived/computed column
ALTER TABLE orders ADD COLUMN item_count INT;
-- Updated by trigger or application when order_items change

-- 2. Duplicating a lookup value
ALTER TABLE order_items ADD COLUMN product_name VARCHAR(200);
-- Copied from products at order time (snapshot — price at time of order)

-- 3. Pre-aggregated summary table
CREATE TABLE daily_sales_summary (
    sale_date    DATE PRIMARY KEY,
    total_orders INT NOT NULL,
    total_revenue DECIMAL(15,2) NOT NULL
);
-- Populated by a nightly batch job or trigger

-- 4. Materialized view (PostgreSQL)
CREATE MATERIALIZED VIEW mv_product_stats AS
SELECT p.product_id, p.name,
       COUNT(oi.order_id) AS order_count,
       SUM(oi.quantity) AS total_sold
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.name;

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_stats;
```

### The Trade-Off

| Aspect | Normalized | Denormalized |
|---|---|---|
| Write performance | Fast (update one place) | Slower (update multiple copies) |
| Read performance | Slower (many joins) | Fast (pre-joined data) |
| Data integrity | Guaranteed (no redundancy) | Risk of inconsistency |
| Storage | Minimal | Increased |
| Schema complexity | More tables | Fewer tables, more triggers |
| Best for | OLTP (transactions) | OLAP (analytics), caching |

> **Golden rule**: Normalize first, denormalize only with evidence (slow query logs, EXPLAIN ANALYZE output). Never skip normalization and jump straight to a denormalized design.

---

## 14. Worked Examples — Beginner, Intermediate, Advanced

### Beginner: Student-Course Database

**Starting flat table:**

```
student_enrollment(student_id, student_name, dept, dept_phone,
                   course_id, course_title, instructor, grade)

FDs:
  student_id → student_name, dept
  dept → dept_phone
  course_id → course_title, instructor
  {student_id, course_id} → grade
```

**Step-by-step normalization:**

```
1NF: Already atomic — no multi-valued columns. ✓

2NF: Candidate key is {student_id, course_id}.
     Partial FDs:
       student_id → student_name, dept      (only part of key)
       course_id → course_title, instructor  (only part of key)

     Decompose:
       students(student_id, student_name, dept)
       courses(course_id, course_title, instructor)
       enrollments(student_id, course_id, grade)

3NF: Check students: student_id → dept → dept_phone (transitive!)
     Decompose:
       students(student_id, student_name, dept)
       departments(dept, dept_phone)
       courses(course_id, course_title, instructor)
       enrollments(student_id, course_id, grade)

BCNF: Check all — every left side is a superkey. ✓
```

```sql
CREATE TABLE departments (
    dept        VARCHAR(50) PRIMARY KEY,
    dept_phone  VARCHAR(20) NOT NULL
);

CREATE TABLE students (
    student_id    INT PRIMARY KEY,
    student_name  VARCHAR(100) NOT NULL,
    dept          VARCHAR(50) NOT NULL REFERENCES departments(dept)
);

CREATE TABLE courses (
    course_id   CHAR(6) PRIMARY KEY,
    course_title VARCHAR(200) NOT NULL,
    instructor  VARCHAR(100) NOT NULL
);

CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   CHAR(6) REFERENCES courses(course_id),
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

---

### Intermediate: E-Commerce System

**Starting flat table:**

```
order_details(order_id, order_date, customer_id, customer_name, customer_email,
              product_id, product_name, category_id, category_name,
              quantity, unit_price, discount)

FDs:
  order_id → order_date, customer_id
  customer_id → customer_name, customer_email
  product_id → product_name, category_id, unit_price
  category_id → category_name
  {order_id, product_id} → quantity, discount
```

**Step-by-step normalization:**

```
1NF: Already atomic. ✓

2NF: Candidate key is {order_id, product_id}.
     Partial FDs:
       order_id → order_date, customer_id
       product_id → product_name, category_id, unit_price

     Decompose:
       orders(order_id, order_date, customer_id)
       products(product_id, product_name, category_id, unit_price)
       order_items(order_id, product_id, quantity, discount)

     Plus carry forward: customer_id → customer_name, customer_email
                         category_id → category_name

3NF: Check orders: order_id → customer_id → customer_name, customer_email (transitive!)
     Check products: product_id → category_id → category_name (transitive!)

     Decompose further:
       customers(customer_id, customer_name, customer_email)
       categories(category_id, category_name)
       orders(order_id, order_date, customer_id)
       products(product_id, product_name, category_id, unit_price)
       order_items(order_id, product_id, quantity, discount)

BCNF: Every determinant is a superkey. ✓

Final schema: 5 tables, zero redundancy, all FDs preserved.
```

```sql
CREATE TABLE customers (
    customer_id    SERIAL PRIMARY KEY,
    customer_name  VARCHAR(100) NOT NULL,
    customer_email VARCHAR(200) NOT NULL UNIQUE
);

CREATE TABLE categories (
    category_id    SERIAL PRIMARY KEY,
    category_name  VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_name  VARCHAR(200) NOT NULL,
    category_id   INT NOT NULL REFERENCES categories(category_id),
    unit_price    DECIMAL(10,2) NOT NULL CHECK (unit_price > 0)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    order_date  TIMESTAMP NOT NULL DEFAULT NOW(),
    customer_id INT NOT NULL REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_id    INT REFERENCES orders(order_id),
    product_id  INT REFERENCES products(product_id),
    quantity    INT NOT NULL CHECK (quantity > 0),
    discount    DECIMAL(5,2) NOT NULL DEFAULT 0 CHECK (discount >= 0),
    PRIMARY KEY (order_id, product_id)
);
```

---

### Advanced: Banking System — Normalization with Denormalization Decision

**Starting flat table:**

```
bank_transaction_log(
    txn_id, txn_date, txn_type,
    account_id, account_type, holder_name, holder_ssn,
    branch_id, branch_name, branch_city,
    amount, balance_after,
    category_code, category_desc
)

FDs:
  txn_id → txn_date, txn_type, account_id, amount, balance_after, category_code
  account_id → account_type, holder_name, holder_ssn, branch_id
  branch_id → branch_name, branch_city
  category_code → category_desc
  holder_ssn → holder_name                  (SSN uniquely identifies a person)
```

**Normalization walkthrough:**

```
1NF: Atomic. ✓

2NF: PK is {txn_id} — single column, so no partial FDs. ✓

3NF: Transitive chains:
  txn_id → account_id → holder_name, holder_ssn, branch_id, account_type
  txn_id → account_id → branch_id → branch_name, branch_city
  txn_id → category_code → category_desc
  account_id → holder_ssn → holder_name

  Decompose:
    transactions(txn_id, txn_date, txn_type, account_id, amount, balance_after, category_code)
    accounts(account_id, account_type, holder_ssn, branch_id)
    holders(holder_ssn, holder_name)
    branches(branch_id, branch_name, branch_city)
    categories(category_code, category_desc)

BCNF: Every determinant is a superkey in each table. ✓

4NF: No MVDs — accounts don't have independent multi-valued facts. ✓

5NF: No cyclic join dependencies. ✓
```

**Denormalization decision:**

The reporting team runs this query 10,000 times per day:

```sql
SELECT t.txn_date, t.amount, a.account_type, b.branch_name
FROM transactions t
JOIN accounts a ON t.account_id = a.account_id
JOIN branches b ON a.branch_id = b.branch_id
WHERE t.txn_date >= CURRENT_DATE - INTERVAL '30 days';
```

EXPLAIN shows this 3-table join takes 800ms on 50M rows. Solution: a **materialized view**:

```sql
CREATE MATERIALIZED VIEW mv_recent_txn_report AS
SELECT t.txn_id, t.txn_date, t.txn_type, t.amount, t.balance_after,
       a.account_type, h.holder_name,
       b.branch_name, b.branch_city,
       c.category_desc
FROM transactions t
JOIN accounts a ON t.account_id = a.account_id
JOIN holders h ON a.holder_ssn = h.holder_ssn
JOIN branches b ON a.branch_id = b.branch_id
JOIN categories c ON t.category_code = c.category_code
WHERE t.txn_date >= CURRENT_DATE - INTERVAL '90 days';

CREATE INDEX idx_mv_txn_date ON mv_recent_txn_report(txn_date);

-- Refresh nightly or on schedule
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_recent_txn_report;
```

**Result**: The underlying schema stays in BCNF (writes are clean), but reads hit the denormalized materialized view (fast). Best of both worlds.

---

## 15. Common Interview Questions

### Q1: What is normalization? Why do we do it?
> Normalization is the process of organizing a database to reduce redundancy and prevent update/insertion/deletion anomalies. We decompose large tables into smaller ones linked by foreign keys, ensuring each fact is stored exactly once.

### Q2: What is a functional dependency?
> X → Y means that for any two tuples with the same X value, they must have the same Y value. "Knowing X uniquely determines Y." Example: student_id → name (knowing the ID tells you the name).

### Q3: What are partial and transitive dependencies?
> **Partial**: A non-key attribute depends on only part of a composite candidate key. Violates 2NF. Example: {student_id, course_id} is the key but name depends only on student_id.
> **Transitive**: A non-key attribute depends on the key through another non-key attribute. Violates 3NF. Example: student_id → dept → dept_building.

### Q4: Explain 1NF, 2NF, 3NF in one sentence each.
> **1NF**: Every column is atomic (no lists, no repeating groups).
> **2NF**: 1NF + no partial dependencies on any candidate key.
> **3NF**: 2NF + no transitive dependencies on any candidate key.

### Q5: What is the difference between 3NF and BCNF?
> 3NF allows a non-trivial FD X → A if A is a prime attribute (part of some candidate key). BCNF requires that X is a superkey for **every** non-trivial FD — no exceptions. BCNF is strictly stronger. Most schemas that are in 3NF are also in BCNF; the difference only arises when a non-key attribute determines a prime attribute.

### Q6: Can you always achieve BCNF with dependency preservation?
> No. Some schemas cannot be decomposed into BCNF while preserving all functional dependencies. In such cases, you choose between BCNF (no redundancy, but lose an FD) or 3NF (preserve all FDs, but tolerate slight redundancy). In practice this conflict is rare.

### Q7: What is a multi-valued dependency? Give an example.
> An MVD X ↠ Y means: for a given X, the set of Y values is independent of the other attributes. Example: emp_id ↠ skill and emp_id ↠ language — an employee's skills and languages are independent facts. Storing them in one table forces a Cartesian product of skills × languages per employee.

### Q8: When should you denormalize?
> Denormalize when you have measured evidence that joins are causing unacceptable query performance, typically in read-heavy OLAP/reporting workloads. Common techniques: materialized views, pre-computed aggregates, duplicating lookup values at insert time. Always normalize first, denormalize with evidence.

### Q9: What is lossless-join decomposition?
> A decomposition is lossless if joining the decomposed tables always reproduces the original data exactly — no spurious tuples, no lost tuples. The test: the common attributes of R1 and R2 must be a superkey of at least one of them.

### Q10: What is a canonical cover and why does it matter?
> A canonical cover is a minimal equivalent set of FDs — no redundant FDs, no extraneous attributes. It matters for the 3NF synthesis algorithm: you create one table per FD in the canonical cover, guaranteeing lossless-join + dependency-preserving decomposition.

### Q11: Walk through normalizing this table: R(A, B, C, D, E) with FDs {A → B, BC → D, D → E}.
> Candidate key: {A, C} (because {A,C}⁺ = {A,C,B,D,E}).
> **2NF violation**: A → B is a partial FD (A is part of the key {A,C}, and B depends on A alone).
> Decompose: R1(A, B), R2(A, C, D, E).
> In R2: {A,C} → D (via BC→D and A→B so AC→BC→D), D → E is transitive.
> **3NF violation**: D → E is transitive in R2.
> Decompose: R2a(A, C, D), R2b(D, E).
> Final: R1(A, B), R2a(A, C, D), R2b(D, E) — all in BCNF.

### Q12: Is a single-column PK table automatically in 2NF?
> Yes. 2NF violations only occur with composite candidate keys (partial dependencies). If every candidate key is a single attribute, no partial dependencies can exist, so the table is automatically in 2NF.

---

## 16. Key Takeaways

| # | Concept | One-Line Summary |
|---|---|---|
| 1 | **Functional Dependency** | X → Y: knowing X uniquely determines Y |
| 2 | **Armstrong's Axioms** | Reflexivity, Augmentation, Transitivity — sound & complete for reasoning about FDs |
| 3 | **Attribute Closure (X⁺)** | All attributes determinable from X — used to find candidate keys |
| 4 | **Canonical Cover** | Minimal equivalent FD set — foundation for 3NF synthesis |
| 5 | **1NF** | Atomic values only — no lists, arrays, or repeating groups |
| 6 | **2NF** | 1NF + no partial dependencies on any candidate key |
| 7 | **3NF** | 2NF + no transitive dependencies; "the key, the whole key, nothing but the key" |
| 8 | **BCNF** | Every determinant is a superkey — stricter than 3NF |
| 9 | **4NF** | BCNF + no non-trivial multi-valued dependencies |
| 10 | **5NF (PJNF)** | 4NF + no non-trivial join dependencies — the ultimate form |
| 11 | **Partial Dependency** | Non-key attribute depends on part of a composite key (violates 2NF) |
| 12 | **Transitive Dependency** | Non-key attribute depends on another non-key attribute (violates 3NF) |
| 13 | **Lossless Join** | Common attributes must be a key of at least one decomposed table |
| 14 | **Dependency Preservation** | Every FD checkable in a single decomposed table (always achievable in 3NF) |
| 15 | **BCNF vs 3NF Trade-off** | BCNF eliminates all FD redundancy; 3NF always preserves dependencies |
| 16 | **Denormalization** | Controlled redundancy for read performance — normalize first, denormalize with evidence |
| 17 | **Practical Target** | Most production schemas aim for 3NF or BCNF |
| 18 | **4NF/5NF** | Rarely needed — address independent multi-valued facts and cyclic join dependencies |
| 19 | **Anomalies** | Update, insertion, deletion — the three evils that normalization prevents |
| 20 | **Normalize writes, denormalize reads** | Use materialized views / summary tables to get the best of both worlds |

---

<p style="text-align: center; margin-top: 40px;">
  <a href="06_normalization.md" download="06_normalization.md"
     style="display: inline-block; padding: 14px 36px; font-size: 16px; font-weight: bold;
            color: #fff; background-color: #8e44ad; border-radius: 8px;
            text-decoration: none; box-shadow: 0 4px 6px rgba(0,0,0,0.15);">
    Download 06_normalization.md
  </a>
</p>

---

*Ready? Say **"next"** to proceed to **Phase 7 — Joins (All Types)**.*