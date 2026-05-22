# Phase 14 — Stored Procedures, Functions, Triggers & Cursors

> **Server-side programmability**: moving logic from your application into the database engine for atomicity, reuse, and performance.

---

## Table of Contents

1. [Why Server-Side Code?](#1-why-server-side-code)
2. [Stored Procedures — Concept & Syntax](#2-stored-procedures)
3. [User-Defined Functions (UDFs)](#3-user-defined-functions)
4. [Procedures vs Functions — Comparison](#4-procedures-vs-functions)
5. [Triggers — Concept & Architecture](#5-triggers)
6. [Trigger Types — BEFORE / AFTER × ROW / STATEMENT](#6-trigger-types)
7. [Cursors — Row-by-Row Processing](#7-cursors)
8. [Error Handling in Procedures](#8-error-handling)
9. [Multi-Database Syntax Reference](#9-multi-database-syntax)
10. [Worked Examples (10)](#10-worked-examples)
11. [Common Interview Questions](#11-interview-questions)
12. [Key Takeaways](#12-key-takeaways)
13. [Download](#13-download)

---

## 1. Why Server-Side Code?

### Real-World Analogy

Think of a restaurant kitchen. You *could* instruct the waiter step-by-step: "Go get the chicken, now season it, now put it in the oven at 200°C for 25 minutes…" Or you could simply say **"One roast chicken, please"** and the chef (database engine) executes the entire recipe internally. Stored procedures are the recipes that live in the kitchen — they eliminate round-trips between the dining room (application) and the kitchen (database).

### Benefits of Server-Side Logic

| Benefit | Explanation |
|---|---|
| **Reduced network round-trips** | One call replaces many queries |
| **Atomic multi-step operations** | Entire recipe succeeds or rolls back |
| **Code reuse** | Multiple apps call the same procedure |
| **Security** | Grant EXECUTE without exposing tables |
| **Consistency** | Business rules enforced at the data layer |
| **Performance** | Precompiled execution plans (varies by RDBMS) |

### When to Keep Logic in the Application Instead

| Scenario | Why |
|---|---|
| Complex business rules that change frequently | Deployments are easier in app code |
| Heavy computation (ML, image processing) | DB engine is optimised for data, not compute |
| Portability across RDBMS vendors | PL/pgSQL ≠ T-SQL ≠ PL/SQL |
| Unit-testing requirements | Application frameworks have richer test tooling |

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <!-- Title -->
  <text x="375" y="28" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">App-Side Logic  vs  Server-Side Logic</text>

  <!-- App-side panel -->
  <rect x="20" y="50" width="340" height="270" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="190" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#e65100">Application-Side (Many Round-Trips)</text>

  <rect x="40" y="95" width="130" height="35" rx="6" fill="#ffe0b2" stroke="#e65100"/>
  <text x="105" y="117" text-anchor="middle" font-size="11" fill="#333">App: SELECT …</text>
  <line x1="170" y1="112" x2="210" y2="112" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowOrange)"/>
  <rect x="210" y="95" width="130" height="35" rx="6" fill="#ffe0b2" stroke="#e65100"/>
  <text x="275" y="117" text-anchor="middle" font-size="11" fill="#333">DB → result</text>

  <rect x="40" y="145" width="130" height="35" rx="6" fill="#ffe0b2" stroke="#e65100"/>
  <text x="105" y="167" text-anchor="middle" font-size="11" fill="#333">App: UPDATE …</text>
  <line x1="170" y1="162" x2="210" y2="162" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowOrange)"/>
  <rect x="210" y="145" width="130" height="35" rx="6" fill="#ffe0b2" stroke="#e65100"/>
  <text x="275" y="167" text-anchor="middle" font-size="11" fill="#333">DB → ok</text>

  <rect x="40" y="195" width="130" height="35" rx="6" fill="#ffe0b2" stroke="#e65100"/>
  <text x="105" y="217" text-anchor="middle" font-size="11" fill="#333">App: INSERT …</text>
  <line x1="170" y1="212" x2="210" y2="212" stroke="#e65100" stroke-width="1.5" marker-end="url(#arrowOrange)"/>
  <rect x="210" y="195" width="130" height="35" rx="6" fill="#ffe0b2" stroke="#e65100"/>
  <text x="275" y="217" text-anchor="middle" font-size="11" fill="#333">DB → ok</text>

  <text x="190" y="260" text-anchor="middle" font-size="12" fill="#bf360c" font-weight="bold">3 network round-trips</text>
  <text x="190" y="280" text-anchor="middle" font-size="11" fill="#bf360c">No atomicity guarantee</text>
  <text x="190" y="298" text-anchor="middle" font-size="11" fill="#bf360c">Logic duplicated across apps</text>

  <!-- Server-side panel -->
  <rect x="390" y="50" width="340" height="270" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="560" y="75" text-anchor="middle" font-size="14" font-weight="bold" fill="#2e7d32">Server-Side (Single Call)</text>

  <rect x="430" y="100" width="130" height="35" rx="6" fill="#c8e6c9" stroke="#2e7d32"/>
  <text x="495" y="122" text-anchor="middle" font-size="11" fill="#333">App: CALL proc()</text>
  <line x1="560" y1="117" x2="600" y2="117" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrowGreen)"/>
  <rect x="600" y="95" width="110" height="130" rx="6" fill="#a5d6a7" stroke="#2e7d32"/>
  <text x="655" y="130" text-anchor="middle" font-size="11" fill="#333">SELECT …</text>
  <text x="655" y="155" text-anchor="middle" font-size="11" fill="#333">UPDATE …</text>
  <text x="655" y="180" text-anchor="middle" font-size="11" fill="#333">INSERT …</text>
  <text x="655" y="210" text-anchor="middle" font-size="10" fill="#2e7d32" font-weight="bold">All inside DB</text>

  <text x="560" y="260" text-anchor="middle" font-size="12" fill="#1b5e20" font-weight="bold">1 network round-trip</text>
  <text x="560" y="280" text-anchor="middle" font-size="11" fill="#1b5e20">Atomic (runs in a transaction)</text>
  <text x="560" y="298" text-anchor="middle" font-size="11" fill="#1b5e20">Single source of truth</text>

  <!-- Arrow markers -->
  <defs>
    <marker id="arrowOrange" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#e65100"/>
    </marker>
    <marker id="arrowGreen" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#2e7d32"/>
    </marker>
  </defs>
</svg>

---

## 2. Stored Procedures

### What Is a Stored Procedure?

A **stored procedure** is a named block of SQL and procedural code stored in the database catalog. You invoke it with `CALL` (SQL standard / PostgreSQL / MySQL) or `EXEC` (SQL Server). It can:

- Accept **input** and **output** parameters
- Contain control-flow logic (`IF`, `LOOP`, `WHILE`)
- Execute DML, DDL, and even other procedures
- Return zero or more result sets (RDBMS-dependent)

### Lifecycle

<svg viewBox="0 0 750 130" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowBlue" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
  </defs>

  <!-- Step boxes -->
  <rect x="10" y="30" width="140" height="60" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="80" y="55" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">CREATE</text>
  <text x="80" y="72" text-anchor="middle" font-size="10" fill="#333">Parse + Store</text>

  <line x1="150" y1="60" x2="185" y2="60" stroke="#1565c0" stroke-width="2" marker-end="url(#arrowBlue)"/>

  <rect x="190" y="30" width="140" height="60" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="260" y="55" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">COMPILE</text>
  <text x="260" y="72" text-anchor="middle" font-size="10" fill="#333">Plan + Optimise</text>

  <line x1="330" y1="60" x2="365" y2="60" stroke="#1565c0" stroke-width="2" marker-end="url(#arrowBlue)"/>

  <rect x="370" y="30" width="140" height="60" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="440" y="55" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">CACHE</text>
  <text x="440" y="72" text-anchor="middle" font-size="10" fill="#333">Execution plan</text>

  <line x1="510" y1="60" x2="545" y2="60" stroke="#1565c0" stroke-width="2" marker-end="url(#arrowBlue)"/>

  <rect x="550" y="30" width="140" height="60" rx="8" fill="#bbdefb" stroke="#1565c0" stroke-width="2"/>
  <text x="620" y="55" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">EXECUTE</text>
  <text x="620" y="72" text-anchor="middle" font-size="10" fill="#333">CALL proc(…)</text>

  <text x="375" y="120" text-anchor="middle" font-size="11" fill="#666">Subsequent calls reuse the cached plan (plan cache / shared pool)</text>
</svg>

### PostgreSQL Syntax (PL/pgSQL)

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_acct  INT,
    p_to_acct    INT,
    p_amount     NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Debit source
    UPDATE accounts
       SET balance = balance - p_amount
     WHERE account_id = p_from_acct;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Source account % not found', p_from_acct;
    END IF;

    -- Credit destination
    UPDATE accounts
       SET balance = balance + p_amount
     WHERE account_id = p_to_acct;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Destination account % not found', p_to_acct;
    END IF;

    COMMIT;
END;
$$;

-- Call it
CALL transfer_funds(1001, 1002, 500.00);
```

### SQL Server Syntax (T-SQL)

```sql
CREATE OR ALTER PROCEDURE dbo.transfer_funds
    @from_acct  INT,
    @to_acct    INT,
    @amount     MONEY
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        UPDATE accounts
           SET balance = balance - @amount
         WHERE account_id = @from_acct;

        IF @@ROWCOUNT = 0
            THROW 50001, 'Source account not found', 1;

        UPDATE accounts
           SET balance = balance + @amount
         WHERE account_id = @to_acct;

        IF @@ROWCOUNT = 0
            THROW 50002, 'Destination account not found', 1;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;

-- Call it
EXEC dbo.transfer_funds @from_acct = 1001, @to_acct = 1002, @amount = 500.00;
```

### MySQL Syntax

```sql
DELIMITER $$

CREATE PROCEDURE transfer_funds(
    IN  p_from_acct INT,
    IN  p_to_acct   INT,
    IN  p_amount    DECIMAL(12,2)
)
BEGIN
    DECLARE v_rows INT;

    START TRANSACTION;

    UPDATE accounts
       SET balance = balance - p_amount
     WHERE account_id = p_from_acct;

    SET v_rows = ROW_COUNT();
    IF v_rows = 0 THEN
        SIGNAL SQLSTATE '45000'
          SET MESSAGE_TEXT = 'Source account not found';
    END IF;

    UPDATE accounts
       SET balance = balance + p_amount
     WHERE account_id = p_to_acct;

    SET v_rows = ROW_COUNT();
    IF v_rows = 0 THEN
        SIGNAL SQLSTATE '45000'
          SET MESSAGE_TEXT = 'Destination account not found';
    END IF;

    COMMIT;
END$$

DELIMITER ;

-- Call it
CALL transfer_funds(1001, 1002, 500.00);
```

---

## 3. User-Defined Functions

### What Is a UDF?

A **function** is similar to a procedure but is designed to **return a value** and be used **inside SQL expressions** (SELECT, WHERE, etc.). Functions must be deterministic or at least side-effect-free in many RDBMS to be callable in queries.

### Function Categories

| Category | Returns | Example |
|---|---|---|
| **Scalar** | Single value | `get_tax_rate(state)` → `0.07` |
| **Table-valued** | A result set (table) | `get_orders_by_customer(42)` → rows |
| **Aggregate** | Single value from a set | Custom `MEDIAN()` |

### PostgreSQL — Scalar Function

```sql
CREATE OR REPLACE FUNCTION calculate_discount(
    p_total  NUMERIC,
    p_tier   TEXT
) RETURNS NUMERIC
LANGUAGE plpgsql
IMMUTABLE            -- no side effects, same input = same output
AS $$
BEGIN
    RETURN CASE p_tier
        WHEN 'gold'     THEN p_total * 0.20
        WHEN 'silver'   THEN p_total * 0.10
        WHEN 'bronze'   THEN p_total * 0.05
        ELSE 0
    END;
END;
$$;

-- Usage in a query
SELECT order_id,
       total,
       calculate_discount(total, customer_tier) AS discount
  FROM orders;
```

### PostgreSQL — Table-Valued Function (RETURNS TABLE)

```sql
CREATE OR REPLACE FUNCTION get_overdue_invoices(p_days INT)
RETURNS TABLE(invoice_id INT, customer TEXT, amount NUMERIC, days_overdue INT)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT i.invoice_id,
           c.name,
           i.amount,
           (CURRENT_DATE - i.due_date)::INT
      FROM invoices i
      JOIN customers c ON c.id = i.customer_id
     WHERE i.due_date < CURRENT_DATE - p_days
       AND i.paid = FALSE;
END;
$$;

-- Call like a table
SELECT * FROM get_overdue_invoices(30);
```

### SQL Server — Inline Table-Valued Function

```sql
CREATE FUNCTION dbo.get_overdue_invoices(@days INT)
RETURNS TABLE
AS
RETURN (
    SELECT i.invoice_id,
           c.name        AS customer,
           i.amount,
           DATEDIFF(DAY, i.due_date, GETDATE()) AS days_overdue
      FROM invoices i
      JOIN customers c ON c.id = i.customer_id
     WHERE i.due_date < DATEADD(DAY, -@days, GETDATE())
       AND i.paid = 0
);

-- Call it
SELECT * FROM dbo.get_overdue_invoices(30);
```

---

## 4. Procedures vs Functions — Comparison

<svg viewBox="0 0 750 420" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <!-- Title -->
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Stored Procedure vs Function — Side by Side</text>

  <!-- Header row -->
  <rect x="20" y="45" width="220" height="35" rx="4" fill="#37474f"/>
  <text x="130" y="68" text-anchor="middle" font-size="13" fill="#fff" font-weight="bold">Feature</text>
  <rect x="245" y="45" width="235" height="35" rx="4" fill="#1565c0"/>
  <text x="362" y="68" text-anchor="middle" font-size="13" fill="#fff" font-weight="bold">Stored Procedure</text>
  <rect x="485" y="45" width="245" height="35" rx="4" fill="#00695c"/>
  <text x="607" y="68" text-anchor="middle" font-size="13" fill="#fff" font-weight="bold">Function (UDF)</text>

  <!-- Row 1 -->
  <rect x="20" y="85" width="220" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="130" y="107" text-anchor="middle" font-size="11" fill="#333">Return value</text>
  <rect x="245" y="85" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="107" text-anchor="middle" font-size="11" fill="#333">Optional (OUT params / result sets)</text>
  <rect x="485" y="85" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="107" text-anchor="middle" font-size="11" fill="#333">Required (scalar / table / set)</text>

  <!-- Row 2 -->
  <rect x="20" y="120" width="220" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="130" y="142" text-anchor="middle" font-size="11" fill="#333">Callable in SELECT</text>
  <rect x="245" y="120" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="142" text-anchor="middle" font-size="11" fill="#c62828" font-weight="bold">No</text>
  <rect x="485" y="120" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="142" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">Yes</text>

  <!-- Row 3 -->
  <rect x="20" y="155" width="220" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="130" y="177" text-anchor="middle" font-size="11" fill="#333">Side effects (DML)</text>
  <rect x="245" y="155" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="177" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">Yes</text>
  <rect x="485" y="155" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="177" text-anchor="middle" font-size="11" fill="#333">Restricted in many RDBMS</text>

  <!-- Row 4 -->
  <rect x="20" y="190" width="220" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="130" y="212" text-anchor="middle" font-size="11" fill="#333">Transaction control</text>
  <rect x="245" y="190" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="212" text-anchor="middle" font-size="11" fill="#333">COMMIT / ROLLBACK allowed</text>
  <rect x="485" y="190" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="212" text-anchor="middle" font-size="11" fill="#333">Generally not allowed</text>

  <!-- Row 5 -->
  <rect x="20" y="225" width="220" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="130" y="247" text-anchor="middle" font-size="11" fill="#333">Invocation</text>
  <rect x="245" y="225" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="247" text-anchor="middle" font-size="11" fill="#333">CALL / EXEC</text>
  <rect x="485" y="225" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="247" text-anchor="middle" font-size="11" fill="#333">SELECT func() / in expressions</text>

  <!-- Row 6 -->
  <rect x="20" y="260" width="220" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="130" y="282" text-anchor="middle" font-size="11" fill="#333">Composability</text>
  <rect x="245" y="260" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="282" text-anchor="middle" font-size="11" fill="#333">Can call functions & procedures</text>
  <rect x="485" y="260" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="282" text-anchor="middle" font-size="11" fill="#333">Can call other functions</text>

  <!-- Row 7 -->
  <rect x="20" y="295" width="220" height="35" fill="#f5f5f5" stroke="#ddd"/>
  <text x="130" y="317" text-anchor="middle" font-size="11" fill="#333">Use in WHERE / JOIN</text>
  <rect x="245" y="295" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="317" text-anchor="middle" font-size="11" fill="#c62828" font-weight="bold">No</text>
  <rect x="485" y="295" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="317" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">Yes (table-valued → FROM)</text>

  <!-- Row 8 -->
  <rect x="20" y="330" width="220" height="35" fill="#fafafa" stroke="#ddd"/>
  <text x="130" y="352" text-anchor="middle" font-size="11" fill="#333">Error handling</text>
  <rect x="245" y="330" width="235" height="35" fill="#e3f2fd" stroke="#ddd"/>
  <text x="362" y="352" text-anchor="middle" font-size="11" fill="#333">TRY/CATCH or EXCEPTION block</text>
  <rect x="485" y="330" width="245" height="35" fill="#e0f2f1" stroke="#ddd"/>
  <text x="607" y="352" text-anchor="middle" font-size="11" fill="#333">Same, but txn control limited</text>

  <!-- Decision rule -->
  <rect x="70" y="380" width="610" height="30" rx="6" fill="#fff8e1" stroke="#f9a825" stroke-width="1.5"/>
  <text x="375" y="400" text-anchor="middle" font-size="12" fill="#e65100" font-weight="bold">Rule of thumb: If you need the result in a query → Function.  If you need side effects → Procedure.</text>
</svg>

---

## 5. Triggers — Concept & Architecture

### What Is a Trigger?

A **trigger** is a named block of procedural code that the database engine **fires automatically** in response to a specific DML event (`INSERT`, `UPDATE`, `DELETE`) on a table (or view). Triggers are the database equivalent of **event listeners** in UI frameworks — you don't call them directly; they react to data changes.

### Real-World Analogy

A **smoke detector** is a trigger. You don't press a button to activate it — it fires automatically when smoke (an event) is detected, and it executes a response (sounding the alarm). In the same way, a database trigger fires automatically when a row is inserted, updated, or deleted.

### Trigger Architecture

<svg viewBox="0 0 750 300" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowT" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#5c6bc0"/>
    </marker>
    <marker id="arrowR" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#e53935"/>
    </marker>
  </defs>

  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Trigger Execution Flow</text>

  <!-- DML Event -->
  <rect x="30" y="60" width="140" height="50" rx="8" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="100" y="82" text-anchor="middle" font-size="12" font-weight="bold" fill="#3949ab">DML Statement</text>
  <text x="100" y="98" text-anchor="middle" font-size="10" fill="#555">INSERT / UPDATE / DELETE</text>

  <line x1="170" y1="85" x2="210" y2="85" stroke="#5c6bc0" stroke-width="2" marker-end="url(#arrowT)"/>

  <!-- BEFORE trigger -->
  <rect x="215" y="45" width="140" height="80" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="285" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">BEFORE Trigger</text>
  <text x="285" y="90" text-anchor="middle" font-size="10" fill="#555">• Validate data</text>
  <text x="285" y="105" text-anchor="middle" font-size="10" fill="#555">• Modify NEW row</text>
  <text x="285" y="118" text-anchor="middle" font-size="10" fill="#555">• Cancel via RAISE</text>

  <line x1="355" y1="85" x2="395" y2="85" stroke="#5c6bc0" stroke-width="2" marker-end="url(#arrowT)"/>

  <!-- Actual DML -->
  <rect x="400" y="55" width="130" height="60" rx="8" fill="#c8e6c9" stroke="#2e7d32" stroke-width="2"/>
  <text x="465" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Execute DML</text>
  <text x="465" y="98" text-anchor="middle" font-size="10" fill="#555">Write to table</text>

  <line x1="530" y1="85" x2="570" y2="85" stroke="#5c6bc0" stroke-width="2" marker-end="url(#arrowT)"/>

  <!-- AFTER trigger -->
  <rect x="575" y="45" width="145" height="80" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="647" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#c62828">AFTER Trigger</text>
  <text x="647" y="90" text-anchor="middle" font-size="10" fill="#555">• Audit logging</text>
  <text x="647" y="105" text-anchor="middle" font-size="10" fill="#555">• Cascade updates</text>
  <text x="647" y="118" text-anchor="middle" font-size="10" fill="#555">• Notify systems</text>

  <!-- Constraint check -->
  <rect x="400" y="150" width="130" height="40" rx="6" fill="#f3e5f5" stroke="#7b1fa2" stroke-width="1.5"/>
  <text x="465" y="175" text-anchor="middle" font-size="11" fill="#7b1fa2" font-weight="bold">Constraint Check</text>
  <line x1="465" y1="115" x2="465" y2="150" stroke="#7b1fa2" stroke-width="1.5" stroke-dasharray="4,3"/>

  <!-- NEW / OLD references -->
  <rect x="130" y="200" width="490" height="80" rx="8" fill="#f5f5f5" stroke="#666" stroke-width="1"/>
  <text x="375" y="225" text-anchor="middle" font-size="13" font-weight="bold" fill="#333">Special Row Variables</text>
  <text x="260" y="248" text-anchor="middle" font-size="11" fill="#1565c0" font-weight="bold">NEW</text>
  <text x="260" y="265" text-anchor="middle" font-size="10" fill="#555">The row AFTER the operation</text>
  <text x="490" y="248" text-anchor="middle" font-size="11" fill="#c62828" font-weight="bold">OLD</text>
  <text x="490" y="265" text-anchor="middle" font-size="10" fill="#555">The row BEFORE the operation</text>

  <line x1="295" y1="248" x2="450" y2="248" stroke="#ccc" stroke-width="1" stroke-dasharray="3,3"/>
</svg>

### NEW and OLD Variables by Event

| Event | OLD | NEW |
|---|---|---|
| `INSERT` | Not available | The row being inserted |
| `UPDATE` | Row before the change | Row after the change |
| `DELETE` | The row being deleted | Not available |

---

## 6. Trigger Types — BEFORE / AFTER × ROW / STATEMENT

### The 2×2 Matrix

<svg viewBox="0 0 650 350" xmlns="http://www.w3.org/2000/svg" style="max-width:650px; font-family:Arial,sans-serif;">
  <text x="325" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Trigger Type Matrix</text>

  <!-- Axis labels -->
  <text x="325" y="55" text-anchor="middle" font-size="13" font-weight="bold" fill="#555">Timing</text>
  <text x="15" y="190" text-anchor="middle" font-size="13" font-weight="bold" fill="#555" transform="rotate(-90 15 190)">Granularity</text>

  <!-- Column headers -->
  <rect x="170" y="65" width="200" height="35" rx="6" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="270" y="88" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">BEFORE</text>
  <rect x="385" y="65" width="200" height="35" rx="6" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="485" y="88" text-anchor="middle" font-size="12" font-weight="bold" fill="#c62828">AFTER</text>

  <!-- Row headers -->
  <rect x="30" y="115" width="125" height="95" rx="6" fill="#e8eaf6" stroke="#3949ab" stroke-width="1.5"/>
  <text x="92" y="155" text-anchor="middle" font-size="12" font-weight="bold" fill="#3949ab">FOR EACH</text>
  <text x="92" y="172" text-anchor="middle" font-size="12" font-weight="bold" fill="#3949ab">ROW</text>

  <rect x="30" y="225" width="125" height="95" rx="6" fill="#e8eaf6" stroke="#3949ab" stroke-width="1.5"/>
  <text x="92" y="265" text-anchor="middle" font-size="12" font-weight="bold" fill="#3949ab">FOR EACH</text>
  <text x="92" y="282" text-anchor="middle" font-size="12" font-weight="bold" fill="#3949ab">STATEMENT</text>

  <!-- Cell: BEFORE ROW -->
  <rect x="170" y="115" width="200" height="95" rx="4" fill="#fff8e1" stroke="#ddd"/>
  <text x="270" y="140" text-anchor="middle" font-size="11" font-weight="bold" fill="#e65100">BEFORE ROW</text>
  <text x="270" y="158" text-anchor="middle" font-size="10" fill="#555">Fires per row, before write</text>
  <text x="270" y="175" text-anchor="middle" font-size="10" fill="#555">Can modify NEW values</text>
  <text x="270" y="192" text-anchor="middle" font-size="10" fill="#388e3c">✓ Validation, auto-fill</text>

  <!-- Cell: AFTER ROW -->
  <rect x="385" y="115" width="200" height="95" rx="4" fill="#fce4ec" stroke="#ddd"/>
  <text x="485" y="140" text-anchor="middle" font-size="11" font-weight="bold" fill="#c62828">AFTER ROW</text>
  <text x="485" y="158" text-anchor="middle" font-size="10" fill="#555">Fires per row, after write</text>
  <text x="485" y="175" text-anchor="middle" font-size="10" fill="#555">OLD/NEW are final</text>
  <text x="485" y="192" text-anchor="middle" font-size="10" fill="#388e3c">✓ Audit log, cascade</text>

  <!-- Cell: BEFORE STATEMENT -->
  <rect x="170" y="225" width="200" height="95" rx="4" fill="#fff8e1" stroke="#ddd"/>
  <text x="270" y="250" text-anchor="middle" font-size="11" font-weight="bold" fill="#e65100">BEFORE STATEMENT</text>
  <text x="270" y="268" text-anchor="middle" font-size="10" fill="#555">Fires once before entire stmt</text>
  <text x="270" y="285" text-anchor="middle" font-size="10" fill="#555">No access to NEW/OLD</text>
  <text x="270" y="302" text-anchor="middle" font-size="10" fill="#388e3c">✓ Permission checks</text>

  <!-- Cell: AFTER STATEMENT -->
  <rect x="385" y="225" width="200" height="95" rx="4" fill="#fce4ec" stroke="#ddd"/>
  <text x="485" y="250" text-anchor="middle" font-size="11" font-weight="bold" fill="#c62828">AFTER STATEMENT</text>
  <text x="485" y="268" text-anchor="middle" font-size="10" fill="#555">Fires once after entire stmt</text>
  <text x="485" y="285" text-anchor="middle" font-size="10" fill="#555">No access to NEW/OLD</text>
  <text x="485" y="302" text-anchor="middle" font-size="10" fill="#388e3c">✓ Summary/aggregate</text>
</svg>

### PostgreSQL Examples

**BEFORE ROW — Auto-fill `updated_at` timestamp:**

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;          -- must RETURN NEW or the row is discarded
END;
$$;

CREATE TRIGGER trg_set_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

**AFTER ROW — Audit log:**

```sql
CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salary_audit(employee_id, old_salary, new_salary, changed_at)
    VALUES (OLD.id, OLD.salary, NEW.salary, NOW());
    RETURN NULL;        -- return value is ignored in AFTER triggers
END;
$$;

CREATE TRIGGER trg_salary_audit
    AFTER UPDATE OF salary ON employees
    FOR EACH ROW
    WHEN (OLD.salary IS DISTINCT FROM NEW.salary)
    EXECUTE FUNCTION log_salary_change();
```

**INSTEAD OF — Updatable view trigger:**

```sql
CREATE OR REPLACE FUNCTION update_active_employees_view()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE employees
       SET name   = NEW.name,
           salary = NEW.salary
     WHERE id = OLD.id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_update_active_emp
    INSTEAD OF UPDATE ON active_employees_view
    FOR EACH ROW
    EXECUTE FUNCTION update_active_employees_view();
```

### SQL Server — AFTER Trigger (T-SQL)

```sql
CREATE TRIGGER trg_audit_order_status
ON orders
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO order_audit(order_id, old_status, new_status, changed_at, changed_by)
    SELECT i.order_id, d.status, i.status, GETDATE(), SYSTEM_USER
      FROM inserted i
      JOIN deleted  d ON d.order_id = i.order_id
     WHERE i.status <> d.status;
END;
```

> **SQL Server note:** Uses the virtual tables `inserted` and `deleted` (analogous to `NEW` and `OLD`) and fires once per statement — containing all affected rows.

### Trigger Execution Order (PostgreSQL)

1. **BEFORE STATEMENT** triggers fire
2. For each affected row:
   - **BEFORE ROW** triggers fire (can modify NEW)
   - Constraint checks
   - The actual data modification
   - **AFTER ROW** triggers fire
3. **AFTER STATEMENT** triggers fire

Multiple triggers on the same event fire in **alphabetical order** in PostgreSQL. In SQL Server, you can use `sp_settriggerorder` to control order.

### Trigger Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|---|---|---|
| Trigger calls another trigger (cascading) | Hard to debug, infinite loops | Consolidate logic or use queues |
| Heavy computation in a trigger | Blocks the DML transaction | Async processing (LISTEN/NOTIFY) |
| Business logic in triggers | Invisible to developers, hard to test | Move to stored procedures / app layer |
| Trigger modifies its own table | Recursive trigger problem | Use `pg_trigger_depth()` guard |

---

## 7. Cursors — Row-by-Row Processing

### What Is a Cursor?

A **cursor** is a database object that lets you fetch and process rows **one at a time** (or in small batches) from a result set, rather than operating on the entire set at once.

### Real-World Analogy

A cursor is like a **bookmark** in a long document. The result set is the entire book — the cursor tracks which page you're on and lets you read one page at a time, move forward, or go back.

### Cursor Lifecycle

<svg viewBox="0 0 750 150" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowCur" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#6a1b9a"/>
    </marker>
  </defs>

  <text x="375" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Cursor Lifecycle</text>

  <!-- DECLARE -->
  <rect x="15" y="45" width="120" height="55" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="75" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">DECLARE</text>
  <text x="75" y="85" text-anchor="middle" font-size="10" fill="#555">Define query</text>

  <line x1="135" y1="72" x2="170" y2="72" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowCur)"/>

  <!-- OPEN -->
  <rect x="175" y="45" width="120" height="55" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="235" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">OPEN</text>
  <text x="235" y="85" text-anchor="middle" font-size="10" fill="#555">Execute query</text>

  <line x1="295" y1="72" x2="330" y2="72" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowCur)"/>

  <!-- FETCH -->
  <rect x="335" y="45" width="120" height="55" rx="8" fill="#e1bee7" stroke="#6a1b9a" stroke-width="2"/>
  <text x="395" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">FETCH</text>
  <text x="395" y="85" text-anchor="middle" font-size="10" fill="#555">Get next row</text>

  <!-- Loop arrow -->
  <path d="M 455 60 C 480 40, 480 40, 455 50" fill="none" stroke="#6a1b9a" stroke-width="1.5"/>
  <path d="M 455 72 Q 490 72 490 50 Q 490 30 395 30 Q 340 30 335 55" fill="none" stroke="#6a1b9a" stroke-width="1.5" stroke-dasharray="4,3" marker-end="url(#arrowCur)"/>
  <text x="460" y="30" font-size="9" fill="#6a1b9a">LOOP</text>

  <line x1="455" y1="72" x2="505" y2="72" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowCur)"/>

  <!-- CLOSE -->
  <rect x="510" y="45" width="100" height="55" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="560" y="68" text-anchor="middle" font-size="12" font-weight="bold" fill="#6a1b9a">CLOSE</text>
  <text x="560" y="85" text-anchor="middle" font-size="10" fill="#555">Release rows</text>

  <line x1="610" y1="72" x2="640" y2="72" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowCur)"/>

  <!-- DEALLOCATE -->
  <rect x="645" y="45" width="95" height="55" rx="8" fill="#ce93d8" stroke="#6a1b9a" stroke-width="2"/>
  <text x="692" y="68" text-anchor="middle" font-size="11" font-weight="bold" fill="#fff">DEALLOC</text>
  <text x="692" y="85" text-anchor="middle" font-size="10" fill="#fff">Free memory</text>

  <text x="375" y="135" text-anchor="middle" font-size="11" fill="#777">PostgreSQL auto-closes at END of block; SQL Server requires explicit CLOSE + DEALLOCATE</text>
</svg>

### Implicit vs Explicit Cursors

| Aspect | Implicit Cursor | Explicit Cursor |
|---|---|---|
| **Declaration** | Auto-created by engine for every SQL statement | Manually declared by programmer |
| **Control** | Engine manages open/fetch/close | You control each step |
| **Use case** | Single-row queries, `FOR rec IN query` loops | Complex row-by-row processing |
| **Performance** | Engine can optimise automatically | Risk of slow row-by-row processing |
| **Example** | PL/pgSQL `FOR` loop, Oracle implicit cursor | `DECLARE CURSOR`, `FETCH NEXT` |

### PostgreSQL — Explicit Cursor

```sql
DO $$
DECLARE
    cur_emp CURSOR FOR
        SELECT id, name, salary FROM employees WHERE department = 'Engineering';
    rec RECORD;
BEGIN
    OPEN cur_emp;
    LOOP
        FETCH cur_emp INTO rec;
        EXIT WHEN NOT FOUND;

        IF rec.salary < 50000 THEN
            UPDATE employees SET salary = salary * 1.10 WHERE id = rec.id;
            RAISE NOTICE 'Raised salary for %', rec.name;
        END IF;
    END LOOP;
    CLOSE cur_emp;
END;
$$;
```

### PostgreSQL — Implicit Cursor (FOR loop)

```sql
DO $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN SELECT id, name, salary FROM employees WHERE department = 'Engineering'
    LOOP
        IF rec.salary < 50000 THEN
            UPDATE employees SET salary = salary * 1.10 WHERE id = rec.id;
            RAISE NOTICE 'Raised salary for %', rec.name;
        END IF;
    END LOOP;
    -- No OPEN/CLOSE needed — handled implicitly
END;
$$;
```

### SQL Server — Cursor

```sql
DECLARE @id INT, @name VARCHAR(100), @salary MONEY;

DECLARE emp_cursor CURSOR FAST_FORWARD FOR
    SELECT id, name, salary FROM employees WHERE department = 'Engineering';

OPEN emp_cursor;
FETCH NEXT FROM emp_cursor INTO @id, @name, @salary;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @salary < 50000
    BEGIN
        UPDATE employees SET salary = salary * 1.10 WHERE id = @id;
        PRINT 'Raised salary for ' + @name;
    END;
    FETCH NEXT FROM emp_cursor INTO @id, @name, @salary;
END;

CLOSE emp_cursor;
DEALLOCATE emp_cursor;
```

### Cursor Types (SQL Server)

| Type | Scrollable | Sensitivity | Performance |
|---|---|---|---|
| `STATIC` | Yes | Snapshot at open time | Medium |
| `DYNAMIC` | Yes | Reflects real-time changes | Slowest |
| `FAST_FORWARD` | Forward-only | Read-only | Fastest |
| `KEYSET` | Yes | Keys fixed, values dynamic | Medium |

### When to Avoid Cursors

Cursors process rows **one at a time** — the antithesis of set-based SQL. In most cases, a single `UPDATE` with a `WHERE` clause is orders of magnitude faster.

```sql
-- ❌ Cursor approach: ~10 sec for 100k rows
FETCH → IF → UPDATE → FETCH → IF → UPDATE → ...

-- ✅ Set-based approach: ~100 ms for 100k rows
UPDATE employees
   SET salary = salary * 1.10
 WHERE department = 'Engineering'
   AND salary < 50000;
```

**Use cursors only when:**
- You need to call an external procedure per row
- Row-by-row ordering logic that can't be expressed in SQL
- Processing requires `COMMIT` every N rows (batch commits)
- Iterating over databases/tables in administrative scripts

---

## 8. Error Handling in Procedures

### PostgreSQL — EXCEPTION Block

```sql
CREATE OR REPLACE PROCEDURE safe_insert_customer(
    p_id   INT,
    p_name TEXT,
    p_email TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO customers(id, name, email)
    VALUES (p_id, p_name, p_email);

    RAISE NOTICE 'Customer % inserted successfully', p_id;

EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Customer % already exists — skipping', p_id;
    WHEN check_violation THEN
        RAISE WARNING 'Check constraint failed for customer %', p_id;
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Unexpected error: % — %', SQLSTATE, SQLERRM;
END;
$$;
```

### PostgreSQL — Named Error Conditions

| Condition Name | SQLSTATE | Description |
|---|---|---|
| `division_by_zero` | 22012 | Divide by zero |
| `unique_violation` | 23505 | Duplicate key |
| `foreign_key_violation` | 23503 | FK constraint broken |
| `check_violation` | 23514 | CHECK constraint failed |
| `not_null_violation` | 23502 | NULL in NOT NULL column |
| `raise_exception` | P0001 | User-raised exception |
| `no_data_found` | P0002 | Query returned no rows |
| `too_many_rows` | P0003 | Expected one row, got many |

### SQL Server — TRY / CATCH

```sql
CREATE PROCEDURE dbo.safe_insert_customer
    @id    INT,
    @name  NVARCHAR(100),
    @email NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        INSERT INTO customers(id, name, email)
        VALUES (@id, @name, @email);

        COMMIT TRANSACTION;
        PRINT 'Customer inserted successfully';
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        DECLARE @msg   NVARCHAR(4000) = ERROR_MESSAGE(),
                @sev   INT            = ERROR_SEVERITY(),
                @state INT            = ERROR_STATE();

        RAISERROR(@msg, @sev, @state);
    END CATCH
END;
```

### MySQL — DECLARE HANDLER

```sql
DELIMITER $$

CREATE PROCEDURE safe_insert_customer(
    IN p_id   INT,
    IN p_name VARCHAR(100),
    IN p_email VARCHAR(255)
)
BEGIN
    DECLARE EXIT HANDLER FOR 1062  -- duplicate key
    BEGIN
        SELECT CONCAT('Customer ', p_id, ' already exists') AS message;
    END;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT 'Unexpected error occurred' AS message;
    END;

    START TRANSACTION;
    INSERT INTO customers(id, name, email)
    VALUES (p_id, p_name, p_email);
    COMMIT;
    SELECT 'Customer inserted successfully' AS message;
END$$

DELIMITER ;
```

### Error-Handling Strategy Comparison

<svg viewBox="0 0 750 260" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Error Handling Across RDBMS</text>

  <!-- PostgreSQL -->
  <rect x="20" y="50" width="220" height="190" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="130" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">PostgreSQL</text>
  <text x="130" y="100" text-anchor="middle" font-size="11" fill="#333">BEGIN … EXCEPTION</text>
  <text x="130" y="120" text-anchor="middle" font-size="11" fill="#333">WHEN condition THEN</text>
  <text x="130" y="145" text-anchor="middle" font-size="10" fill="#555">✓ Named conditions</text>
  <text x="130" y="163" text-anchor="middle" font-size="10" fill="#555">✓ SQLSTATE codes</text>
  <text x="130" y="181" text-anchor="middle" font-size="10" fill="#555">✓ GET STACKED DIAGNOSTICS</text>
  <text x="130" y="199" text-anchor="middle" font-size="10" fill="#555">✓ Sub-transaction per block</text>
  <text x="130" y="225" text-anchor="middle" font-size="10" fill="#c62828">⚠ Each EXCEPTION creates a savepoint</text>

  <!-- SQL Server -->
  <rect x="265" y="50" width="220" height="190" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="375" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">SQL Server</text>
  <text x="375" y="100" text-anchor="middle" font-size="11" fill="#333">BEGIN TRY … END TRY</text>
  <text x="375" y="120" text-anchor="middle" font-size="11" fill="#333">BEGIN CATCH … END CATCH</text>
  <text x="375" y="145" text-anchor="middle" font-size="10" fill="#555">✓ ERROR_MESSAGE()</text>
  <text x="375" y="163" text-anchor="middle" font-size="10" fill="#555">✓ ERROR_NUMBER()</text>
  <text x="375" y="181" text-anchor="middle" font-size="10" fill="#555">✓ ERROR_SEVERITY()</text>
  <text x="375" y="199" text-anchor="middle" font-size="10" fill="#555">✓ XACT_STATE()</text>
  <text x="375" y="225" text-anchor="middle" font-size="10" fill="#c62828">⚠ Must check @@TRANCOUNT</text>

  <!-- MySQL -->
  <rect x="510" y="50" width="220" height="190" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="620" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">MySQL</text>
  <text x="620" y="100" text-anchor="middle" font-size="11" fill="#333">DECLARE … HANDLER</text>
  <text x="620" y="120" text-anchor="middle" font-size="11" fill="#333">FOR condition / SQLSTATE</text>
  <text x="620" y="145" text-anchor="middle" font-size="10" fill="#555">✓ CONTINUE handler</text>
  <text x="620" y="163" text-anchor="middle" font-size="10" fill="#555">✓ EXIT handler</text>
  <text x="620" y="181" text-anchor="middle" font-size="10" fill="#555">✓ SIGNAL / RESIGNAL</text>
  <text x="620" y="199" text-anchor="middle" font-size="10" fill="#555">✓ Named conditions</text>
  <text x="620" y="225" text-anchor="middle" font-size="10" fill="#c62828">⚠ No nested handler scopes</text>
</svg>

---

## 9. Multi-Database Syntax Reference

| Feature | PostgreSQL | SQL Server | MySQL | Oracle |
|---|---|---|---|---|
| **Create procedure** | `CREATE PROCEDURE` | `CREATE PROCEDURE` | `CREATE PROCEDURE` | `CREATE PROCEDURE` |
| **Create function** | `CREATE FUNCTION` | `CREATE FUNCTION` | `CREATE FUNCTION` | `CREATE FUNCTION` |
| **Language** | `LANGUAGE plpgsql` | T-SQL (default) | SQL (built-in) | PL/SQL (default) |
| **Call procedure** | `CALL proc()` | `EXEC proc` | `CALL proc()` | `EXEC proc` or `CALL proc()` |
| **Variable declaration** | `DECLARE x INT;` in body | `DECLARE @x INT;` | `DECLARE x INT;` | `x NUMBER;` in DECLARE block |
| **IF syntax** | `IF … THEN … END IF;` | `IF … BEGIN … END` | `IF … THEN … END IF;` | `IF … THEN … END IF;` |
| **Loop** | `LOOP … END LOOP;` | `WHILE … BEGIN … END` | `LOOP … END LOOP;` | `LOOP … END LOOP;` |
| **Raise error** | `RAISE EXCEPTION` | `THROW` / `RAISERROR` | `SIGNAL SQLSTATE` | `RAISE_APPLICATION_ERROR` |
| **Error handling** | `EXCEPTION WHEN …` | `TRY … CATCH` | `DECLARE HANDLER` | `EXCEPTION WHEN …` |
| **Output param** | `OUT p_x INT` | `@x INT OUTPUT` | `OUT p_x INT` | `p_x OUT NUMBER` |
| **Trigger NEW/OLD** | `NEW.col` / `OLD.col` | `inserted` / `deleted` tables | `NEW.col` / `OLD.col` | `:NEW.col` / `:OLD.col` |
| **Create trigger** | `CREATE TRIGGER … EXECUTE FUNCTION f()` | `CREATE TRIGGER … AS BEGIN … END` | `CREATE TRIGGER … BEGIN … END` | `CREATE TRIGGER … BEGIN … END` |

---

## 10. Worked Examples

### Example 1 — Beginner: Auto-Increment Surrogate with Procedure

**Scenario:** Student registration system — insert a student and return the generated ID.

```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE register_student(
    p_name   TEXT,
    p_email  TEXT,
    OUT p_id INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO students(name, email)
    VALUES (p_name, p_email)
    RETURNING id INTO p_id;
END;
$$;

-- Call
CALL register_student('Alice Johnson', 'alice@university.edu', NULL);
-- Returns: p_id = 1
```

### Example 2 — Beginner: Simple Validation Trigger

**Scenario:** Prevent inserting negative grades.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION check_grade()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.grade < 0 OR NEW.grade > 100 THEN
        RAISE EXCEPTION 'Grade must be between 0 and 100, got %', NEW.grade;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_check_grade
    BEFORE INSERT OR UPDATE ON student_grades
    FOR EACH ROW
    EXECUTE FUNCTION check_grade();

-- Test
INSERT INTO student_grades(student_id, course_id, grade) VALUES (1, 101, 150);
-- ERROR: Grade must be between 0 and 100, got 150
```

### Example 3 — Beginner: Scalar Function in Query

**Scenario:** Format full name from first/last.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION full_name(p_first TEXT, p_last TEXT)
RETURNS TEXT
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT INITCAP(p_first) || ' ' || INITCAP(p_last);
$$;

SELECT student_id,
       full_name(first_name, last_name) AS student_name,
       grade
  FROM student_grades sg
  JOIN students s ON s.id = sg.student_id;
```

| student_id | student_name | grade |
|---|---|---|
| 1 | Alice Johnson | 92 |
| 2 | Bob Smith | 85 |
| 3 | Carol Davis | 78 |

### Example 4 — Intermediate: Inventory Reorder Trigger

**Scenario:** E-commerce system — when stock drops below a threshold, auto-create a purchase order.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION auto_reorder()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_threshold INT;
    v_reorder_qty INT;
BEGIN
    SELECT reorder_point, reorder_quantity
      INTO v_threshold, v_reorder_qty
      FROM products
     WHERE product_id = NEW.product_id;

    IF NEW.quantity_on_hand <= v_threshold THEN
        INSERT INTO purchase_orders(product_id, quantity, status, created_at)
        VALUES (NEW.product_id, v_reorder_qty, 'pending', NOW());

        RAISE NOTICE 'Auto-reorder created for product %: % units',
                     NEW.product_id, v_reorder_qty;
    END IF;

    RETURN NULL;  -- AFTER trigger, return value ignored
END;
$$;

CREATE TRIGGER trg_auto_reorder
    AFTER UPDATE OF quantity_on_hand ON inventory
    FOR EACH ROW
    WHEN (NEW.quantity_on_hand < OLD.quantity_on_hand)  -- only on decrease
    EXECUTE FUNCTION auto_reorder();
```

### Example 5 — Intermediate: Procedure with Output Params & Error Handling

**Scenario:** Place an order — validate stock, decrement inventory, create order, return order ID.

```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE place_order(
    p_customer_id INT,
    p_product_id  INT,
    p_quantity    INT,
    OUT p_order_id INT,
    OUT p_message  TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_stock   INT;
    v_price   NUMERIC;
BEGIN
    -- Check stock
    SELECT quantity_on_hand, unit_price
      INTO v_stock, v_price
      FROM inventory i
      JOIN products p ON p.product_id = i.product_id
     WHERE i.product_id = p_product_id
       FOR UPDATE;      -- lock the row

    IF NOT FOUND THEN
        p_message := 'Product not found';
        RETURN;
    END IF;

    IF v_stock < p_quantity THEN
        p_message := format('Insufficient stock: have %s, need %s', v_stock, p_quantity);
        RETURN;
    END IF;

    -- Decrement inventory
    UPDATE inventory
       SET quantity_on_hand = quantity_on_hand - p_quantity
     WHERE product_id = p_product_id;

    -- Create order
    INSERT INTO orders(customer_id, product_id, quantity, total, status)
    VALUES (p_customer_id, p_product_id, p_quantity, v_price * p_quantity, 'confirmed')
    RETURNING order_id INTO p_order_id;

    p_message := 'Order placed successfully';

EXCEPTION
    WHEN lock_not_available THEN
        p_message := 'Could not lock inventory — try again';
    WHEN OTHERS THEN
        p_message := format('Error: %s', SQLERRM);
END;
$$;

-- Call
CALL place_order(42, 1001, 3, NULL, NULL);
-- Returns: p_order_id = 5847, p_message = 'Order placed successfully'
```

### Example 6 — Intermediate: Table-Valued Function for Reporting

**Scenario:** Monthly sales summary that can be used in JOINs and WHERE clauses.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION monthly_sales_report(p_year INT, p_month INT)
RETURNS TABLE(
    product_name  TEXT,
    units_sold    BIGINT,
    total_revenue NUMERIC,
    avg_order_value NUMERIC
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT p.name,
           COUNT(*)::BIGINT,
           SUM(o.total),
           ROUND(AVG(o.total), 2)
      FROM orders o
      JOIN products p ON p.product_id = o.product_id
     WHERE EXTRACT(YEAR FROM o.order_date) = p_year
       AND EXTRACT(MONTH FROM o.order_date) = p_month
       AND o.status = 'confirmed'
     GROUP BY p.name
     ORDER BY SUM(o.total) DESC;
END;
$$;

-- Use in a query
SELECT *
  FROM monthly_sales_report(2026, 3)
 WHERE total_revenue > 10000;
```

| product_name | units_sold | total_revenue | avg_order_value |
|---|---|---|---|
| Laptop Pro 16 | 142 | 284000.00 | 2000.00 |
| Wireless Headphones | 389 | 38900.00 | 100.00 |
| USB-C Hub | 267 | 13350.00 | 50.00 |

### Example 7 — Intermediate: Cursor for Batch Email Notifications

**Scenario:** Process overdue invoices in batches, committing every 100 rows to avoid long transactions.

```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE send_overdue_notices()
LANGUAGE plpgsql
AS $$
DECLARE
    cur CURSOR FOR
        SELECT i.invoice_id, c.email, i.amount, i.due_date
          FROM invoices i
          JOIN customers c ON c.id = i.customer_id
         WHERE i.due_date < CURRENT_DATE - INTERVAL '30 days'
           AND i.paid = FALSE
           AND i.notice_sent = FALSE
         ORDER BY i.due_date;
    rec RECORD;
    v_count INT := 0;
BEGIN
    OPEN cur;
    LOOP
        FETCH cur INTO rec;
        EXIT WHEN NOT FOUND;

        -- Mark as notified (actual email sent by app layer)
        UPDATE invoices
           SET notice_sent = TRUE,
               notice_date = NOW()
         WHERE invoice_id = rec.invoice_id;

        INSERT INTO notification_queue(email, subject, body, created_at)
        VALUES (
            rec.email,
            'Overdue Invoice #' || rec.invoice_id,
            format('Amount $%s was due on %s', rec.amount, rec.due_date),
            NOW()
        );

        v_count := v_count + 1;

        IF v_count % 100 = 0 THEN
            COMMIT;          -- batch commit every 100 rows
            RAISE NOTICE 'Processed % invoices', v_count;
        END IF;
    END LOOP;
    CLOSE cur;

    IF v_count % 100 <> 0 THEN
        COMMIT;              -- final batch
    END IF;

    RAISE NOTICE 'Total overdue notices queued: %', v_count;
END;
$$;

CALL send_overdue_notices();
```

### Example 8 — Advanced: Audit Trail with JSON Diff (Banking)

**Scenario:** Full audit trail recording old and new values as JSON for any column change on the `accounts` table.

```sql
-- PostgreSQL
CREATE TABLE account_audit (
    audit_id    BIGSERIAL PRIMARY KEY,
    account_id  INT NOT NULL,
    operation   TEXT NOT NULL,
    old_data    JSONB,
    new_data    JSONB,
    changed_cols TEXT[],
    changed_by  TEXT DEFAULT current_user,
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION audit_account_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_old JSONB;
    v_new JSONB;
    v_diff TEXT[];
    k TEXT;
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO account_audit(account_id, operation, old_data)
        VALUES (OLD.account_id, 'DELETE', to_jsonb(OLD));
        RETURN OLD;
    END IF;

    IF TG_OP = 'INSERT' THEN
        INSERT INTO account_audit(account_id, operation, new_data)
        VALUES (NEW.account_id, 'INSERT', to_jsonb(NEW));
        RETURN NEW;
    END IF;

    -- UPDATE: compute diff
    v_old := to_jsonb(OLD);
    v_new := to_jsonb(NEW);
    v_diff := ARRAY[]::TEXT[];

    FOR k IN SELECT jsonb_object_keys(v_new)
    LOOP
        IF v_old->k IS DISTINCT FROM v_new->k THEN
            v_diff := v_diff || k;
        END IF;
    END LOOP;

    IF array_length(v_diff, 1) > 0 THEN
        INSERT INTO account_audit(account_id, operation, old_data, new_data, changed_cols)
        VALUES (NEW.account_id, 'UPDATE', v_old, v_new, v_diff);
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_audit_accounts
    AFTER INSERT OR UPDATE OR DELETE ON accounts
    FOR EACH ROW
    EXECUTE FUNCTION audit_account_changes();
```

**Query the audit trail:**

```sql
SELECT audit_id, account_id, operation, changed_cols, changed_at,
       old_data->>'balance' AS old_balance,
       new_data->>'balance' AS new_balance
  FROM account_audit
 WHERE account_id = 1001
 ORDER BY changed_at DESC
 LIMIT 5;
```

| audit_id | account_id | operation | changed_cols | changed_at | old_balance | new_balance |
|---|---|---|---|---|---|---|
| 892 | 1001 | UPDATE | {balance} | 2026-04-29 14:32:01 | 5000.00 | 4500.00 |
| 891 | 1001 | UPDATE | {balance,status} | 2026-04-29 10:15:33 | 4200.00 | 5000.00 |
| 887 | 1001 | INSERT | | 2026-04-28 09:00:00 | | 4200.00 |

### Example 9 — Advanced: Recursive Procedure for Hierarchical Processing

**Scenario:** Calculate total budget for a department including all sub-departments (tree rollup).

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION rollup_department_budget(p_dept_id INT)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_own_budget NUMERIC;
    v_child_sum  NUMERIC;
BEGIN
    -- Own budget
    SELECT budget INTO v_own_budget
      FROM departments
     WHERE dept_id = p_dept_id;

    IF NOT FOUND THEN
        RETURN 0;
    END IF;

    -- Sum children recursively
    SELECT COALESCE(SUM(rollup_department_budget(dept_id)), 0)
      INTO v_child_sum
      FROM departments
     WHERE parent_dept_id = p_dept_id;

    RETURN v_own_budget + v_child_sum;
END;
$$;

-- Alternatively, using a CTE (set-based, generally preferred)
WITH RECURSIVE dept_tree AS (
    SELECT dept_id, name, budget, parent_dept_id
      FROM departments
     WHERE dept_id = 1          -- root department
    UNION ALL
    SELECT d.dept_id, d.name, d.budget, d.parent_dept_id
      FROM departments d
      JOIN dept_tree dt ON dt.dept_id = d.parent_dept_id
)
SELECT SUM(budget) AS total_budget FROM dept_tree;
```

### Example 10 — Advanced: Dynamic SQL in a Maintenance Procedure

**Scenario:** DBA utility — rebuild indexes on all tables in a schema that are bloated above a threshold.

```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE rebuild_bloated_indexes(
    p_schema     TEXT DEFAULT 'public',
    p_threshold  FLOAT DEFAULT 30.0  -- % bloat threshold
)
LANGUAGE plpgsql
AS $$
DECLARE
    rec RECORD;
    v_count INT := 0;
BEGIN
    FOR rec IN
        SELECT schemaname, tablename, indexname,
               pg_relation_size(indexrelid) AS index_size,
               100.0 * (1 - pg_relation_size(indexrelid)::float /
                   NULLIF(pg_total_relation_size(indrelid), 0)) AS est_bloat
          FROM pg_stat_user_indexes
          JOIN pg_index ON pg_index.indexrelid = pg_stat_user_indexes.indexrelid
         WHERE schemaname = p_schema
           AND idx_scan > 0  -- only indexes that are actually used
         ORDER BY est_bloat DESC
    LOOP
        IF rec.est_bloat > p_threshold THEN
            RAISE NOTICE 'Rebuilding % (est. bloat: %s%%)', rec.indexname, ROUND(rec.est_bloat::numeric, 1);

            EXECUTE format('REINDEX INDEX CONCURRENTLY %I.%I', rec.schemaname, rec.indexname);

            v_count := v_count + 1;
            COMMIT;  -- release locks between rebuilds
        END IF;
    END LOOP;

    RAISE NOTICE 'Rebuilt % indexes in schema %', v_count, p_schema;
END;
$$;

-- Run it
CALL rebuild_bloated_indexes('public', 25.0);
```

---

## 11. Common Interview Questions

### Q1: What is the difference between a stored procedure and a function?

**Answer:** A **stored procedure** is invoked with `CALL`/`EXEC`, can have side effects (DML, DDL, transaction control), and returns results via OUT parameters or result sets. A **function** must return a value, can be used inside SQL expressions (`SELECT`, `WHERE`), and is generally restricted from having side effects. The rule of thumb: if you need the result embedded in a query, use a function; if you need to perform actions, use a procedure.

### Q2: What are BEFORE and AFTER triggers? When would you use each?

**Answer:**
- **BEFORE triggers** fire before the DML operation commits the row. They can inspect and *modify* the NEW row values (e.g., auto-fill timestamps, enforce validation). Returning NULL from a BEFORE ROW trigger cancels the operation.
- **AFTER triggers** fire after the row has been written. They are used for side effects like audit logging, cascading updates to other tables, or sending notifications. They cannot modify the row that was just written.

### Q3: Can a trigger call another trigger? What problems can this cause?

**Answer:** Yes — if trigger A's action causes a DML event that trigger B is listening for, trigger B fires. This is called **cascading triggers**. Problems include:
- **Infinite loops** (trigger A fires trigger B which fires trigger A)
- **Hard to debug** — the execution path is invisible to the application
- **Performance degradation** — a single INSERT could spawn dozens of hidden operations
- **Mitigation:** Use `pg_trigger_depth()` in PostgreSQL or `TRIGGER_NESTLEVEL()` in SQL Server to detect and break cycles. Better yet, avoid cascading triggers by consolidating logic.

### Q4: What is the difference between ROW-level and STATEMENT-level triggers?

**Answer:**
- **ROW-level** (`FOR EACH ROW`): fires once per affected row. Has access to `OLD` and `NEW` row variables. An `UPDATE` affecting 1,000 rows fires the trigger 1,000 times.
- **STATEMENT-level** (`FOR EACH STATEMENT`): fires once per DML statement, regardless of how many rows are affected. Does not have access to individual row values. Useful for aggregate operations or permission checks.

### Q5: Why are cursors generally considered bad practice? When are they justified?

**Answer:** Cursors process rows **one at a time**, which defeats SQL's set-based optimization. A cursor loop with per-row `UPDATE` can be 100x slower than an equivalent single `UPDATE … WHERE` statement because:
- Each FETCH is a context switch
- The optimizer cannot batch I/O or use parallel plans
- Lock acquisition happens per row instead of in bulk

Cursors are justified when:
- You need to call an external procedure per row
- Complex sequential logic can't be expressed in a single statement
- Batch commits are required (e.g., `COMMIT` every 1,000 rows to limit undo log growth)
- Administrative scripts iterating over system catalog entries

### Q6: How does error handling work in PL/pgSQL?

**Answer:** PL/pgSQL uses a `BEGIN … EXCEPTION … END` block. When an error occurs, control jumps to the `EXCEPTION` section. You can catch specific conditions (`WHEN unique_violation`, `WHEN foreign_key_violation`) or use `WHEN OTHERS` as a catch-all. Key detail: PostgreSQL internally creates a **savepoint** at the start of each block with an `EXCEPTION` clause, so if an error occurs, only that block's changes are rolled back — not the entire transaction.

### Q7: What is an INSTEAD OF trigger?

**Answer:** An `INSTEAD OF` trigger replaces the default DML action on a **view**. Since complex views (with JOINs, aggregates) are not directly updatable, you can define an INSTEAD OF trigger that intercepts `INSERT`, `UPDATE`, or `DELETE` on the view and translates it into the correct operations on the underlying base tables. This makes the view appear updatable to the application.

### Q8: What is dynamic SQL in a stored procedure? What are the risks?

**Answer:** Dynamic SQL means constructing SQL strings at runtime using `EXECUTE` (PostgreSQL) or `EXEC`/`sp_executesql` (SQL Server). It's needed when table names, column names, or SQL structure aren't known at compile time.

**Risks:**
- **SQL injection** if user input is concatenated directly — always use `format('%I', identifier)` or parameterized queries (`USING` clause in PL/pgSQL, `sp_executesql` parameters in T-SQL)
- **Plan cache inefficiency** — each unique string generates a new plan
- **Harder to debug** — no compile-time syntax checking

### Q9: How do `inserted` and `deleted` work in SQL Server triggers?

**Answer:** SQL Server triggers fire once per statement (not per row) and provide two virtual tables:
- **`inserted`**: contains all new/updated rows (like NEW but for the entire batch)
- **`deleted`**: contains all old/deleted rows (like OLD but for the entire batch)

For an `INSERT`, only `inserted` has rows. For a `DELETE`, only `deleted` has rows. For an `UPDATE`, both have rows — you `JOIN` them on the primary key to compare old vs new values. This set-based model is fundamentally different from PostgreSQL's per-row `NEW`/`OLD` approach.

### Q10: What is the difference between IMMUTABLE, STABLE, and VOLATILE in PostgreSQL functions?

**Answer:**
- **IMMUTABLE**: Same input always returns the same output; cannot access the database. The optimizer may pre-evaluate at plan time. Example: `lower('ABC')` → `'abc'`.
- **STABLE**: Returns the same result within a single statement/transaction but may change across statements (because data can change). Safe to call in index scans. Example: `NOW()`, lookup functions.
- **VOLATILE** (default): May return different results on each call, even within the same statement. Cannot be used in index scans. Example: `random()`, `nextval()`.

Marking a function correctly is critical for performance — an IMMUTABLE function can be folded into a constant; a STABLE function can be used with index scans; a VOLATILE function forces sequential evaluation.

### Q11: Can you modify a table that a trigger is defined on, from within that trigger?

**Answer:** This depends on the RDBMS:
- **PostgreSQL**: Yes, but you need to be careful of infinite recursion. Use `pg_trigger_depth()` to prevent it.
- **SQL Server**: The `inserted`/`deleted` tables are read-only, but you can modify the base table directly. Recursive triggers are off by default; enable with `ALTER DATABASE SET RECURSIVE_TRIGGERS ON`.
- **MySQL**: No — a trigger cannot modify the same table that fired it (the "mutating table" restriction, borrowed from Oracle's model).
- **Oracle**: Raises a "mutating table" error for row-level triggers that query or modify the triggering table.

### Q12: How would you migrate business logic from application code to a stored procedure?

**Answer:** A phased approach:
1. **Identify** the logic to migrate — must be data-centric (validation, calculations, multi-table operations)
2. **Write the procedure** with proper error handling and OUT parameters for status
3. **Add logging** in the procedure (e.g., insert into an audit table) to match application-level logging
4. **Test side-by-side** — run both app logic and procedure, compare results
5. **Switch the application** to call the procedure instead of executing inline SQL
6. **Remove the application logic** once the procedure is validated in production
7. **Document** the procedure — unlike app code, database objects don't live in a git repo by default. Use schema migration tools (Flyway, Liquibase, sqitch).

---

## 12. Key Takeaways

<svg viewBox="0 0 750 440" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">Phase 14 — Key Takeaways</text>

  <!-- Card 1 -->
  <rect x="20" y="50" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="40" y="75" font-size="13" font-weight="bold" fill="#1565c0">Procedures = Actions</text>
  <text x="40" y="95" font-size="11" fill="#333">Use for multi-step DML with transaction</text>
  <text x="40" y="112" font-size="11" fill="#333">control. Called with CALL / EXEC.</text>

  <!-- Card 2 -->
  <rect x="390" y="50" width="340" height="80" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="410" y="75" font-size="13" font-weight="bold" fill="#00695c">Functions = Computations</text>
  <text x="410" y="95" font-size="11" fill="#333">Use for returning values inside queries.</text>
  <text x="410" y="112" font-size="11" fill="#333">Callable in SELECT, WHERE, JOIN.</text>

  <!-- Card 3 -->
  <rect x="20" y="150" width="340" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="40" y="175" font-size="13" font-weight="bold" fill="#e65100">Triggers = Event Listeners</text>
  <text x="40" y="195" font-size="11" fill="#333">Fire automatically on DML events.</text>
  <text x="40" y="212" font-size="11" fill="#333">BEFORE for validation; AFTER for auditing.</text>

  <!-- Card 4 -->
  <rect x="390" y="150" width="340" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="410" y="175" font-size="13" font-weight="bold" fill="#6a1b9a">Cursors = Last Resort</text>
  <text x="410" y="195" font-size="11" fill="#333">Row-by-row is 10–100× slower than</text>
  <text x="410" y="212" font-size="11" fill="#333">set-based. Use only when unavoidable.</text>

  <!-- Card 5 -->
  <rect x="20" y="250" width="340" height="80" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="40" y="275" font-size="13" font-weight="bold" fill="#c62828">Error Handling Is Non-Negotiable</text>
  <text x="40" y="295" font-size="11" fill="#333">Always wrap DML in error handlers.</text>
  <text x="40" y="312" font-size="11" fill="#333">PG: EXCEPTION; MSSQL: TRY/CATCH.</text>

  <!-- Card 6 -->
  <rect x="390" y="250" width="340" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="410" y="275" font-size="13" font-weight="bold" fill="#2e7d32">Avoid Trigger Anti-Patterns</text>
  <text x="410" y="295" font-size="11" fill="#333">No cascading triggers. No heavy logic.</text>
  <text x="410" y="312" font-size="11" fill="#333">No business rules hidden in triggers.</text>

  <!-- Card 7 - Decision Guide -->
  <rect x="120" y="350" width="510" height="70" rx="10" fill="#fff8e1" stroke="#f9a825" stroke-width="2"/>
  <text x="375" y="375" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Decision Guide</text>
  <text x="375" y="395" text-anchor="middle" font-size="11" fill="#333">Need it in a query? → Function  |  Need side effects? → Procedure</text>
  <text x="375" y="410" text-anchor="middle" font-size="11" fill="#333">Auto-react to data changes? → Trigger  |  Row-by-row batch? → Cursor (last resort)</text>
</svg>

---

## 13. Download

<a href="14_programmability.md" download="14_programmability.md" style="display:inline-block;padding:14px 28px;background:linear-gradient(135deg,#6a1b9a,#4a148c);color:#fff;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;box-shadow:0 4px 15px rgba(106,27,154,0.3);margin:10px 0;">Download 14_programmability.md</a>
