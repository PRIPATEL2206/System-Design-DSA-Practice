# Phase 20 — Data Warehousing & OLAP

> A **data warehouse** is a central repository that stores integrated, historical data from multiple sources — optimised not for running the business (OLTP), but for *understanding* the business through analytical queries, dashboards, and reports (OLAP).

---

## Table of Contents

1. [OLTP vs OLAP](#1-oltp-vs-olap)
2. [What Is a Data Warehouse?](#2-what-is-a-data-warehouse)
3. [Dimensional Modeling](#3-dimensional-modeling)
4. [Star Schema](#4-star-schema)
5. [Snowflake Schema](#5-snowflake-schema)
6. [Galaxy Schema (Fact Constellation)](#6-galaxy-schema)
7. [Fact Tables](#7-fact-tables)
8. [Dimension Tables](#8-dimension-tables)
9. [Slowly Changing Dimensions (SCD)](#9-slowly-changing-dimensions)
10. [ETL vs ELT Pipelines](#10-etl-vs-elt)
11. [Columnar Storage Engines](#11-columnar-storage)
12. [ROLLUP, CUBE & GROUPING SETS](#12-rollup-cube-grouping-sets)
13. [Worked Examples](#13-worked-examples)
14. [Common Interview Questions](#14-interview-questions)
15. [Key Takeaways](#15-key-takeaways)
16. [Download](#16-download)

---

## 1. OLTP vs OLAP

### Fundamental Difference

**OLTP** (Online Transaction Processing) systems *run* the business — they handle the individual transactions your application performs every second. **OLAP** (Online Analytical Processing) systems *analyse* the business — they answer questions like "What were our top-selling products in Q3 by region?"

### Real-World Analogy

Think of a **cash register** vs a **boardroom dashboard**. The cash register (OLTP) records each sale — fast, isolated, up-to-the-second. The boardroom dashboard (OLAP) aggregates all sales data, slices it by time, product, region, and customer — slow-to-build but powerful for decision-making.

### Comparison

| Dimension | OLTP | OLAP |
|---|---|---|
| **Purpose** | Run day-to-day operations | Analyse business trends |
| **Users** | Application layer, clerks, customers | Analysts, executives, data scientists |
| **Queries** | Simple: `INSERT`, `UPDATE`, `SELECT` by PK | Complex: multi-table `JOIN`, aggregations |
| **Rows per query** | Few (1–100) | Millions to billions |
| **Schema** | Highly normalised (3NF+) | Denormalised (star/snowflake) |
| **Data currency** | Real-time, up-to-the-second | Periodic loads (minutes to daily) |
| **Concurrency** | Thousands of short txns | Fewer long-running analytics |
| **Optimised for** | Write throughput, low latency | Read throughput, scan speed |
| **Storage** | Row-oriented | Column-oriented |
| **Typical size** | GB–low TB | TB–PB |

<svg viewBox="0 0 780 320" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">OLTP vs OLAP Spectrum</text>

  <!-- OLTP side -->
  <rect x="30" y="50" width="320" height="250" rx="14" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="190" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#1565c0">OLTP</text>
  <text x="190" y="98" text-anchor="middle" font-size="10" fill="#555">Online Transaction Processing</text>

  <rect x="50" y="115" width="280" height="28" rx="6" fill="#bbdefb"/>
  <text x="60" y="134" font-size="11" fill="#333">Normalised schema (3NF / BCNF)</text>
  <rect x="50" y="150" width="280" height="28" rx="6" fill="#bbdefb"/>
  <text x="60" y="169" font-size="11" fill="#333">Row-oriented storage</text>
  <rect x="50" y="185" width="280" height="28" rx="6" fill="#bbdefb"/>
  <text x="60" y="204" font-size="11" fill="#333">Short, frequent transactions</text>
  <rect x="50" y="220" width="280" height="28" rx="6" fill="#bbdefb"/>
  <text x="60" y="239" font-size="11" fill="#333">INSERT / UPDATE / DELETE heavy</text>
  <rect x="50" y="255" width="280" height="28" rx="6" fill="#bbdefb"/>
  <text x="60" y="274" font-size="11" fill="#333">Examples: PostgreSQL, MySQL, Oracle</text>

  <!-- Arrow -->
  <line x1="365" y1="175" x2="415" y2="175" stroke="#666" stroke-width="2" marker-end="url(#arrowDual)"/>
  <line x1="415" y1="175" x2="365" y2="175" stroke="#666" stroke-width="2" marker-end="url(#arrowDual)"/>
  <defs><marker id="arrowDual" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#666"/></marker></defs>
  <text x="390" y="160" text-anchor="middle" font-size="9" fill="#666">ETL / ELT</text>
  <text x="390" y="195" text-anchor="middle" font-size="9" fill="#666">feeds data</text>

  <!-- OLAP side -->
  <rect x="430" y="50" width="320" height="250" rx="14" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="590" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#e65100">OLAP</text>
  <text x="590" y="98" text-anchor="middle" font-size="10" fill="#555">Online Analytical Processing</text>

  <rect x="450" y="115" width="280" height="28" rx="6" fill="#ffe0b2"/>
  <text x="460" y="134" font-size="11" fill="#333">Denormalised schema (star/snowflake)</text>
  <rect x="450" y="150" width="280" height="28" rx="6" fill="#ffe0b2"/>
  <text x="460" y="169" font-size="11" fill="#333">Column-oriented storage</text>
  <rect x="450" y="185" width="280" height="28" rx="6" fill="#ffe0b2"/>
  <text x="460" y="204" font-size="11" fill="#333">Long-running analytical scans</text>
  <rect x="450" y="220" width="280" height="28" rx="6" fill="#ffe0b2"/>
  <text x="460" y="239" font-size="11" fill="#333">SELECT / JOIN / GROUP BY heavy</text>
  <rect x="450" y="255" width="280" height="28" rx="6" fill="#ffe0b2"/>
  <text x="460" y="274" font-size="11" fill="#333">Examples: Redshift, BigQuery, Snowflake</text>
</svg>

---

## 2. What Is a Data Warehouse?

### Bill Inmon's Definition

> "A data warehouse is a **subject-oriented**, **integrated**, **time-variant**, **non-volatile** collection of data in support of management's decision-making process."

| Property | Meaning |
|---|---|
| **Subject-oriented** | Organised around business subjects (sales, customers, products), not application processes |
| **Integrated** | Data from CRM, ERP, web logs, etc. is cleaned, standardised, and merged |
| **Time-variant** | Maintains historical snapshots — you can query last year's data |
| **Non-volatile** | Data is loaded and queried, not updated in place (no `UPDATE` / `DELETE` in normal flow) |

### Architecture Layers

<svg viewBox="0 0 800 300" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="22" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">Data Warehouse Architecture</text>

  <!-- Sources -->
  <rect x="15" y="50" width="120" height="210" rx="10" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="75" y="72" text-anchor="middle" font-size="11" font-weight="bold" fill="#3949ab">Sources</text>
  <rect x="25" y="85" width="100" height="24" rx="4" fill="#c5cae9"/>
  <text x="75" y="102" text-anchor="middle" font-size="9" fill="#333">OLTP DBs</text>
  <rect x="25" y="115" width="100" height="24" rx="4" fill="#c5cae9"/>
  <text x="75" y="132" text-anchor="middle" font-size="9" fill="#333">CSV / API feeds</text>
  <rect x="25" y="145" width="100" height="24" rx="4" fill="#c5cae9"/>
  <text x="75" y="162" text-anchor="middle" font-size="9" fill="#333">Web / App logs</text>
  <rect x="25" y="175" width="100" height="24" rx="4" fill="#c5cae9"/>
  <text x="75" y="192" text-anchor="middle" font-size="9" fill="#333">SaaS exports</text>
  <rect x="25" y="205" width="100" height="24" rx="4" fill="#c5cae9"/>
  <text x="75" y="222" text-anchor="middle" font-size="9" fill="#333">IoT streams</text>

  <!-- ETL -->
  <polygon points="160,105 160,205 235,155" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="180" y="150" font-size="9" font-weight="bold" fill="#f57f17">E</text>
  <text x="190" y="160" font-size="9" font-weight="bold" fill="#f57f17">T L</text>

  <!-- Staging -->
  <rect x="250" y="95" width="110" height="120" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="305" y="118" text-anchor="middle" font-size="11" font-weight="bold" fill="#c62828">Staging</text>
  <text x="305" y="140" text-anchor="middle" font-size="9" fill="#555">Raw landing zone</text>
  <text x="305" y="155" text-anchor="middle" font-size="9" fill="#555">Cleaning</text>
  <text x="305" y="170" text-anchor="middle" font-size="9" fill="#555">Deduplication</text>
  <text x="305" y="185" text-anchor="middle" font-size="9" fill="#555">Type conversion</text>

  <!-- Arrow -->
  <line x1="365" y1="155" x2="400" y2="155" stroke="#666" stroke-width="2" marker-end="url(#arrRight)"/>
  <defs><marker id="arrRight" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#666"/></marker></defs>

  <!-- DW core -->
  <rect x="405" y="55" width="160" height="200" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2.5"/>
  <text x="485" y="80" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">Data Warehouse</text>
  <rect x="420" y="95" width="130" height="22" rx="4" fill="#c8e6c9"/>
  <text x="485" y="111" text-anchor="middle" font-size="9" fill="#333">Fact Tables</text>
  <rect x="420" y="123" width="130" height="22" rx="4" fill="#c8e6c9"/>
  <text x="485" y="139" text-anchor="middle" font-size="9" fill="#333">Dimension Tables</text>
  <rect x="420" y="151" width="130" height="22" rx="4" fill="#c8e6c9"/>
  <text x="485" y="167" text-anchor="middle" font-size="9" fill="#333">Star / Snowflake</text>
  <rect x="420" y="179" width="130" height="22" rx="4" fill="#c8e6c9"/>
  <text x="485" y="195" text-anchor="middle" font-size="9" fill="#333">Historical data</text>
  <rect x="420" y="207" width="130" height="22" rx="4" fill="#c8e6c9"/>
  <text x="485" y="223" text-anchor="middle" font-size="9" fill="#333">Columnar storage</text>

  <!-- Arrow to data marts -->
  <line x1="570" y1="155" x2="605" y2="155" stroke="#666" stroke-width="2" marker-end="url(#arrRight)"/>

  <!-- Data Marts + Consumers -->
  <rect x="610" y="50" width="170" height="210" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="695" y="75" text-anchor="middle" font-size="11" font-weight="bold" fill="#6a1b9a">Consumption</text>
  <rect x="625" y="88" width="140" height="22" rx="4" fill="#e1bee7"/>
  <text x="695" y="104" text-anchor="middle" font-size="9" fill="#333">Data Marts (per dept)</text>
  <rect x="625" y="116" width="140" height="22" rx="4" fill="#e1bee7"/>
  <text x="695" y="132" text-anchor="middle" font-size="9" fill="#333">BI Dashboards</text>
  <rect x="625" y="144" width="140" height="22" rx="4" fill="#e1bee7"/>
  <text x="695" y="160" text-anchor="middle" font-size="9" fill="#333">Tableau / Power BI</text>
  <rect x="625" y="172" width="140" height="22" rx="4" fill="#e1bee7"/>
  <text x="695" y="188" text-anchor="middle" font-size="9" fill="#333">Ad-hoc SQL queries</text>
  <rect x="625" y="200" width="140" height="22" rx="4" fill="#e1bee7"/>
  <text x="695" y="216" text-anchor="middle" font-size="9" fill="#333">ML / AI pipelines</text>
</svg>

### Inmon vs Kimball

| Aspect | **Inmon** (top-down) | **Kimball** (bottom-up) |
|---|---|---|
| Build order | Enterprise DW first, then data marts | Build data marts first, integrate later |
| Schema | 3NF in core warehouse | Star schema throughout |
| Complexity | Higher initial effort | Faster time-to-value |
| Audience | Enterprise-wide consistency | Department-level quick wins |
| Modern trend | Used in data lakes / lakehouses | Dominant in cloud DWs (Snowflake, BigQuery) |

---

## 3. Dimensional Modeling

Dimensional modeling organises data into **facts** (what happened — measurable events) and **dimensions** (the context — who, what, when, where). This approach was pioneered by Ralph Kimball and is the foundation of most warehouse schemas.

### The Grain

The **grain** defines what a single row in a fact table represents. Setting the grain correctly is the most critical decision in dimensional modeling.

| Example Domain | Grain |
|---|---|
| Retail sales | One line-item per transaction |
| Web analytics | One page view per session |
| Telecom billing | One call per subscriber per day |
| Hospital admissions | One diagnosis per patient visit |

### Bus Matrix

A **bus matrix** maps business processes (rows) to shared dimensions (columns), ensuring consistent dimension definitions across fact tables.

| Business Process | Date | Product | Customer | Store | Employee | Promotion |
|---|---|---|---|---|---|---|
| **Sales** | X | X | X | X | X | X |
| **Inventory** | X | X | — | X | — | — |
| **Returns** | X | X | X | X | — | — |
| **Purchasing** | X | X | — | — | X | — |

---

## 4. Star Schema

The star schema is the simplest and most widely used dimensional model. A central **fact table** connects to multiple **dimension tables** via foreign keys — like a star.

### Why Star?

- Simple queries with few JOINs
- Excellent for BI tools (they auto-detect star schemas)
- Aggregation-friendly layout
- Optimisers recognise star joins (star transformation)

<svg viewBox="0 0 760 500" xmlns="http://www.w3.org/2000/svg" style="max-width:760px; font-family:Arial,sans-serif;">
  <text x="380" y="22" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Star Schema — Retail Sales</text>

  <!-- Fact table (center) -->
  <rect x="255" y="175" width="250" height="180" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2.5"/>
  <rect x="255" y="175" width="250" height="30" rx="10" fill="#f57f17"/>
  <text x="380" y="195" text-anchor="middle" font-size="13" font-weight="bold" fill="white">fact_sales</text>
  <text x="270" y="222" font-size="10" fill="#333">sale_id (PK)</text>
  <text x="270" y="238" font-size="10" fill="#1565c0">date_key (FK) ─────────────────→</text>
  <text x="270" y="254" font-size="10" fill="#2e7d32">product_key (FK) ──────────────→</text>
  <text x="270" y="270" font-size="10" fill="#c62828">customer_key (FK) ─────────────→</text>
  <text x="270" y="286" font-size="10" fill="#6a1b9a">store_key (FK) ────────────────→</text>
  <text x="270" y="306" font-size="10" fill="#555">quantity_sold</text>
  <text x="270" y="322" font-size="10" fill="#555">unit_price</text>
  <text x="270" y="338" font-size="10" fill="#555">discount_amount</text>
  <text x="270" y="354" font-size="10" fill="#555">total_amount</text>

  <!-- dim_date (top) -->
  <rect x="270" y="40" width="220" height="110" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <rect x="270" y="40" width="220" height="26" rx="8" fill="#1565c0"/>
  <text x="380" y="58" text-anchor="middle" font-size="11" font-weight="bold" fill="white">dim_date</text>
  <text x="280" y="80" font-size="9" fill="#333">date_key (PK)</text>
  <text x="280" y="94" font-size="9" fill="#333">full_date, day_of_week</text>
  <text x="280" y="108" font-size="9" fill="#333">month, quarter, year</text>
  <text x="280" y="122" font-size="9" fill="#333">is_holiday, fiscal_year</text>
  <line x1="380" y1="150" x2="380" y2="175" stroke="#1565c0" stroke-width="2"/>

  <!-- dim_product (right) -->
  <rect x="550" y="185" width="190" height="120" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <rect x="550" y="185" width="190" height="26" rx="8" fill="#2e7d32"/>
  <text x="645" y="203" text-anchor="middle" font-size="11" font-weight="bold" fill="white">dim_product</text>
  <text x="560" y="225" font-size="9" fill="#333">product_key (PK)</text>
  <text x="560" y="239" font-size="9" fill="#333">product_name, sku</text>
  <text x="560" y="253" font-size="9" fill="#333">category, subcategory</text>
  <text x="560" y="267" font-size="9" fill="#333">brand, supplier</text>
  <text x="560" y="281" font-size="9" fill="#333">unit_cost</text>

  <!-- dim_customer (bottom) -->
  <rect x="270" y="390" width="220" height="100" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <rect x="270" y="390" width="220" height="26" rx="8" fill="#c62828"/>
  <text x="380" y="408" text-anchor="middle" font-size="11" font-weight="bold" fill="white">dim_customer</text>
  <text x="280" y="430" font-size="9" fill="#333">customer_key (PK)</text>
  <text x="280" y="444" font-size="9" fill="#333">name, email, phone</text>
  <text x="280" y="458" font-size="9" fill="#333">city, state, country</text>
  <text x="280" y="472" font-size="9" fill="#333">loyalty_tier, signup_date</text>
  <line x1="380" y1="355" x2="380" y2="390" stroke="#c62828" stroke-width="2"/>

  <!-- dim_store (left) -->
  <rect x="15" y="200" width="190" height="110" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <rect x="15" y="200" width="190" height="26" rx="8" fill="#6a1b9a"/>
  <text x="110" y="218" text-anchor="middle" font-size="11" font-weight="bold" fill="white">dim_store</text>
  <text x="25" y="240" font-size="9" fill="#333">store_key (PK)</text>
  <text x="25" y="254" font-size="9" fill="#333">store_name, store_type</text>
  <text x="25" y="268" font-size="9" fill="#333">city, state, region</text>
  <text x="25" y="282" font-size="9" fill="#333">manager, open_date</text>
  <text x="25" y="296" font-size="9" fill="#333">square_footage</text>
  <line x1="205" y1="260" x2="255" y2="260" stroke="#6a1b9a" stroke-width="2"/>
</svg>

### SQL — Star Schema DDL

```sql
-- Dimension: date
CREATE TABLE dim_date (
    date_key     INT         PRIMARY KEY,
    full_date    DATE        NOT NULL,
    day_of_week  VARCHAR(10),
    month_name   VARCHAR(10),
    quarter      SMALLINT,
    year         SMALLINT,
    is_holiday   BOOLEAN     DEFAULT FALSE,
    fiscal_year  SMALLINT
);

-- Dimension: product
CREATE TABLE dim_product (
    product_key  INT         PRIMARY KEY,
    product_name VARCHAR(200),
    sku          VARCHAR(50),
    category     VARCHAR(100),
    subcategory  VARCHAR(100),
    brand        VARCHAR(100),
    supplier     VARCHAR(100),
    unit_cost    NUMERIC(10,2)
);

-- Dimension: customer
CREATE TABLE dim_customer (
    customer_key INT         PRIMARY KEY,
    full_name    VARCHAR(150),
    email        VARCHAR(200),
    city         VARCHAR(100),
    state        VARCHAR(50),
    country      VARCHAR(50),
    loyalty_tier VARCHAR(20),
    signup_date  DATE
);

-- Dimension: store
CREATE TABLE dim_store (
    store_key      INT         PRIMARY KEY,
    store_name     VARCHAR(150),
    store_type     VARCHAR(50),
    city           VARCHAR(100),
    state          VARCHAR(50),
    region         VARCHAR(50),
    manager        VARCHAR(150),
    open_date      DATE,
    square_footage INT
);

-- Fact: sales
CREATE TABLE fact_sales (
    sale_id         BIGINT      PRIMARY KEY,
    date_key        INT         REFERENCES dim_date(date_key),
    product_key     INT         REFERENCES dim_product(product_key),
    customer_key    INT         REFERENCES dim_customer(customer_key),
    store_key       INT         REFERENCES dim_store(store_key),
    quantity_sold   INT,
    unit_price      NUMERIC(10,2),
    discount_amount NUMERIC(10,2) DEFAULT 0,
    total_amount    NUMERIC(12,2)
);
```

### Typical Star Query

```sql
SELECT
    d.year,
    d.quarter,
    p.category,
    s.region,
    SUM(f.total_amount)   AS revenue,
    SUM(f.quantity_sold)   AS units_sold
FROM fact_sales f
JOIN dim_date     d ON f.date_key     = d.date_key
JOIN dim_product  p ON f.product_key  = p.product_key
JOIN dim_store    s ON f.store_key    = s.store_key
WHERE d.year = 2025
GROUP BY d.year, d.quarter, p.category, s.region
ORDER BY d.quarter, revenue DESC;
```

---

## 5. Snowflake Schema

A snowflake schema **normalises the dimension tables** into sub-dimensions. Where a star schema keeps everything flat inside each dimension, the snowflake breaks hierarchies (e.g., product → category → department) into separate tables.

<svg viewBox="0 0 800 440" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="22" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Snowflake Schema — Normalised Dimensions</text>

  <!-- Fact -->
  <rect x="295" y="170" width="210" height="130" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2.5"/>
  <rect x="295" y="170" width="210" height="28" rx="10" fill="#f57f17"/>
  <text x="400" y="189" text-anchor="middle" font-size="12" font-weight="bold" fill="white">fact_sales</text>
  <text x="310" y="215" font-size="9" fill="#333">date_key (FK)</text>
  <text x="310" y="230" font-size="9" fill="#333">product_key (FK)</text>
  <text x="310" y="245" font-size="9" fill="#333">customer_key (FK)</text>
  <text x="310" y="260" font-size="9" fill="#333">store_key (FK)</text>
  <text x="310" y="278" font-size="9" fill="#555">quantity, total_amount</text>

  <!-- dim_product -->
  <rect x="560" y="100" width="160" height="90" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <rect x="560" y="100" width="160" height="24" rx="8" fill="#2e7d32"/>
  <text x="640" y="117" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_product</text>
  <text x="570" y="138" font-size="9" fill="#333">product_key (PK)</text>
  <text x="570" y="153" font-size="9" fill="#1565c0">subcategory_key (FK)</text>
  <text x="570" y="168" font-size="9" fill="#333">product_name, sku, brand</text>
  <line x1="505" y1="220" x2="560" y2="145" stroke="#2e7d32" stroke-width="1.5"/>

  <!-- dim_subcategory -->
  <rect x="560" y="210" width="160" height="70" rx="8" fill="#c8e6c9" stroke="#2e7d32" stroke-width="1.5"/>
  <rect x="560" y="210" width="160" height="22" rx="8" fill="#43a047"/>
  <text x="640" y="226" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_subcategory</text>
  <text x="570" y="248" font-size="9" fill="#333">subcategory_key (PK)</text>
  <text x="570" y="263" font-size="9" fill="#1565c0">category_key (FK)</text>
  <line x1="640" y1="190" x2="640" y2="210" stroke="#2e7d32" stroke-width="1.5"/>

  <!-- dim_category -->
  <rect x="560" y="300" width="160" height="60" rx="8" fill="#a5d6a7" stroke="#2e7d32" stroke-width="1.5"/>
  <rect x="560" y="300" width="160" height="22" rx="8" fill="#66bb6a"/>
  <text x="640" y="316" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_category</text>
  <text x="570" y="338" font-size="9" fill="#333">category_key (PK)</text>
  <text x="570" y="353" font-size="9" fill="#333">category_name, department</text>
  <line x1="640" y1="280" x2="640" y2="300" stroke="#2e7d32" stroke-width="1.5"/>

  <!-- dim_date (top) -->
  <rect x="200" y="45" width="160" height="80" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <rect x="200" y="45" width="160" height="22" rx="8" fill="#1565c0"/>
  <text x="280" y="61" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_date</text>
  <text x="210" y="82" font-size="9" fill="#333">date_key (PK)</text>
  <text x="210" y="97" font-size="9" fill="#333">full_date, month, quarter</text>
  <text x="210" y="112" font-size="9" fill="#333">year, is_holiday</text>
  <line x1="350" y1="105" x2="350" y2="170" stroke="#1565c0" stroke-width="1.5"/>

  <!-- dim_customer -->
  <rect x="40" y="125" width="160" height="75" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <rect x="40" y="125" width="160" height="22" rx="8" fill="#c62828"/>
  <text x="120" y="141" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_customer</text>
  <text x="50" y="162" font-size="9" fill="#333">customer_key (PK)</text>
  <text x="50" y="177" font-size="9" fill="#1565c0">city_key (FK)</text>
  <text x="50" y="192" font-size="9" fill="#333">name, email, loyalty_tier</text>
  <line x1="200" y1="175" x2="295" y2="240" stroke="#c62828" stroke-width="1.5"/>

  <!-- dim_city -->
  <rect x="40" y="225" width="160" height="70" rx="8" fill="#f8bbd0" stroke="#c62828" stroke-width="1.5"/>
  <rect x="40" y="225" width="160" height="22" rx="8" fill="#e57373"/>
  <text x="120" y="241" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_city</text>
  <text x="50" y="262" font-size="9" fill="#333">city_key (PK)</text>
  <text x="50" y="277" font-size="9" fill="#333">city, state, country, region</text>
  <line x1="120" y1="200" x2="120" y2="225" stroke="#c62828" stroke-width="1.5"/>

  <!-- dim_store (bottom-left) -->
  <rect x="40" y="330" width="160" height="75" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <rect x="40" y="330" width="160" height="22" rx="8" fill="#6a1b9a"/>
  <text x="120" y="346" text-anchor="middle" font-size="10" font-weight="bold" fill="white">dim_store</text>
  <text x="50" y="368" font-size="9" fill="#333">store_key (PK)</text>
  <text x="50" y="383" font-size="9" fill="#1565c0">city_key (FK)</text>
  <text x="50" y="398" font-size="9" fill="#333">store_name, store_type</text>
  <line x1="200" y1="370" x2="335" y2="300" stroke="#6a1b9a" stroke-width="1.5"/>

  <!-- Labels -->
  <rect x="560" y="395" width="220" height="40" rx="6" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="570" y="412" font-size="9" fill="#555">Normalised dimensions form</text>
  <text x="570" y="426" font-size="9" fill="#555">"snowflake arms" branching outward</text>
</svg>

### Star vs Snowflake

| Aspect | Star | Snowflake |
|---|---|---|
| **Dimension structure** | Flat, denormalised | Normalised into sub-tables |
| **Number of JOINs** | Fewer (simpler queries) | More (dimension → sub-dimension) |
| **Query performance** | Faster scans, fewer JOINs | Slightly slower (more JOINs) |
| **Storage** | More redundancy in dimensions | Less redundancy |
| **ETL complexity** | Simpler loads | More complex (maintain relationships) |
| **BI tool compatibility** | Excellent (auto-detected) | Good but may need configuration |
| **When to choose** | Most cases, modern DWs | Very large dimensions with update-heavy hierarchies |

---

## 6. Galaxy Schema (Fact Constellation)

A **galaxy schema** (or **fact constellation**) has **multiple fact tables** sharing common dimension tables. This models multiple business processes that intersect on the same dimensions.

<svg viewBox="0 0 780 380" xmlns="http://www.w3.org/2000/svg" style="max-width:780px; font-family:Arial,sans-serif;">
  <text x="390" y="22" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Galaxy Schema — Fact Constellation</text>

  <!-- Shared dims at top -->
  <rect x="235" y="48" width="130" height="55" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="300" y="72" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">dim_date</text>
  <text x="300" y="90" text-anchor="middle" font-size="9" fill="#555">Shared by both facts</text>

  <rect x="415" y="48" width="130" height="55" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="480" y="72" text-anchor="middle" font-size="10" font-weight="bold" fill="#2e7d32">dim_product</text>
  <text x="480" y="90" text-anchor="middle" font-size="9" fill="#555">Shared by both facts</text>

  <!-- Fact Sales (left) -->
  <rect x="80" y="160" width="210" height="130" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2.5"/>
  <rect x="80" y="160" width="210" height="26" rx="10" fill="#f57f17"/>
  <text x="185" y="178" text-anchor="middle" font-size="11" font-weight="bold" fill="white">fact_sales</text>
  <text x="95" y="203" font-size="9" fill="#333">date_key (FK), product_key (FK)</text>
  <text x="95" y="218" font-size="9" fill="#333">customer_key (FK), store_key (FK)</text>
  <text x="95" y="238" font-size="9" fill="#555">quantity, discount, total_amount</text>
  <text x="95" y="253" font-size="9" fill="#555">promotion_key (FK)</text>
  <text x="95" y="268" font-size="9" fill="#555">salesperson_key (FK)</text>

  <!-- Fact Inventory (right) -->
  <rect x="490" y="160" width="210" height="130" rx="10" fill="#e1f5fe" stroke="#0277bd" stroke-width="2.5"/>
  <rect x="490" y="160" width="210" height="26" rx="10" fill="#0277bd"/>
  <text x="595" y="178" text-anchor="middle" font-size="11" font-weight="bold" fill="white">fact_inventory</text>
  <text x="505" y="203" font-size="9" fill="#333">date_key (FK), product_key (FK)</text>
  <text x="505" y="218" font-size="9" fill="#333">warehouse_key (FK)</text>
  <text x="505" y="238" font-size="9" fill="#555">quantity_on_hand</text>
  <text x="505" y="253" font-size="9" fill="#555">quantity_on_order</text>
  <text x="505" y="268" font-size="9" fill="#555">reorder_point</text>

  <!-- Lines from shared dims to facts -->
  <line x1="265" y1="103" x2="185" y2="160" stroke="#1565c0" stroke-width="1.5" stroke-dasharray="5,3"/>
  <line x1="335" y1="103" x2="595" y2="160" stroke="#1565c0" stroke-width="1.5" stroke-dasharray="5,3"/>
  <line x1="445" y1="103" x2="185" y2="160" stroke="#2e7d32" stroke-width="1.5" stroke-dasharray="5,3"/>
  <line x1="510" y1="103" x2="595" y2="160" stroke="#2e7d32" stroke-width="1.5" stroke-dasharray="5,3"/>

  <!-- Private dims -->
  <rect x="15" y="320" width="120" height="50" rx="8" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="75" y="342" text-anchor="middle" font-size="9" font-weight="bold" fill="#c62828">dim_customer</text>
  <text x="75" y="358" text-anchor="middle" font-size="8" fill="#555">Sales only</text>
  <line x1="130" y1="340" x2="185" y2="290" stroke="#c62828" stroke-width="1.2"/>

  <rect x="165" y="320" width="120" height="50" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="225" y="342" text-anchor="middle" font-size="9" font-weight="bold" fill="#6a1b9a">dim_store</text>
  <text x="225" y="358" text-anchor="middle" font-size="8" fill="#555">Sales only</text>
  <line x1="225" y1="320" x2="200" y2="290" stroke="#6a1b9a" stroke-width="1.2"/>

  <rect x="560" y="320" width="130" height="50" rx="8" fill="#e0f2f1" stroke="#00695c" stroke-width="1.5"/>
  <text x="625" y="342" text-anchor="middle" font-size="9" font-weight="bold" fill="#00695c">dim_warehouse</text>
  <text x="625" y="358" text-anchor="middle" font-size="8" fill="#555">Inventory only</text>
  <line x1="600" y1="320" x2="595" y2="290" stroke="#00695c" stroke-width="1.2"/>

  <!-- Legend -->
  <line x1="310" y1="345" x2="360" y2="345" stroke="#999" stroke-width="1.5" stroke-dasharray="5,3"/>
  <text x="368" y="349" font-size="9" fill="#555">Shared dimension link</text>
  <line x1="310" y1="365" x2="360" y2="365" stroke="#999" stroke-width="1.5"/>
  <text x="368" y="369" font-size="9" fill="#555">Private dimension link</text>
</svg>

### When to Use Galaxy

- You have **multiple related business processes** (sales + inventory + returns)
- Dimensions like `date`, `product`, `customer` are **shared** across processes
- Need cross-process analysis (e.g., correlate sales spikes with inventory depletion)

---

## 7. Fact Tables

### Types of Fact Tables

| Type | Description | Example |
|---|---|---|
| **Transaction fact** | One row per event at the atomic grain | Each line-item in a sale |
| **Periodic snapshot** | One row per entity per time period | Monthly account balance |
| **Accumulating snapshot** | One row per process instance, updated as milestones pass | Order: placed → shipped → delivered |
| **Factless fact** | Records events with no measures — only dimension keys | Student attendance, promotion coverage |

### Measures / Metrics

| Additivity | Meaning | Examples |
|---|---|---|
| **Additive** | Can SUM across all dimensions | Revenue, quantity, cost |
| **Semi-additive** | Can SUM across some but not all dimensions (typically not across time) | Account balance, inventory count |
| **Non-additive** | Cannot be meaningfully summed — must AVG, ratio, etc. | Unit price, temperature, percentages |

### Example — Accumulating Snapshot

```sql
CREATE TABLE fact_order_fulfillment (
    order_key        BIGINT  PRIMARY KEY,
    customer_key     INT     REFERENCES dim_customer(customer_key),
    product_key      INT     REFERENCES dim_product(product_key),
    order_date_key   INT     REFERENCES dim_date(date_key),
    ship_date_key    INT     REFERENCES dim_date(date_key),
    deliver_date_key INT     REFERENCES dim_date(date_key),
    return_date_key  INT     REFERENCES dim_date(date_key),
    order_amount     NUMERIC(12,2),
    shipping_cost    NUMERIC(10,2),
    days_to_ship     INT,
    days_to_deliver  INT,
    current_status   VARCHAR(20)    -- placed, shipped, delivered, returned
);
```

---

## 8. Dimension Tables

### Characteristics

- Wide (many columns, 50–100+ attributes is normal)
- Short (thousands to low millions of rows)
- Descriptive text attributes that analysts filter on
- Surrogate keys (integer PKs) instead of natural keys
- Contain the "who / what / when / where / why / how"

### Surrogate Keys vs Natural Keys

| Aspect | Surrogate Key | Natural Key |
|---|---|---|
| Format | Auto-increment integer | Business code (SKU, email, etc.) |
| Change handling | Stays stable; history via SCD | Changes break joins |
| Performance | Narrow, fast joins | Wider, slower joins |
| Source integration | Single key across multiple sources | Each source may differ |
| Warehouse recommendation | **Always use** | Use as an attribute, not PK |

### Role-Playing Dimensions

A single dimension table can appear multiple times in a fact with different roles:

```sql
SELECT
    od.full_date   AS order_date,
    sd.full_date   AS ship_date,
    dd.full_date   AS delivery_date,
    dd.full_date - od.full_date AS days_to_deliver
FROM fact_order_fulfillment f
JOIN dim_date od ON f.order_date_key   = od.date_key
JOIN dim_date sd ON f.ship_date_key    = sd.date_key
JOIN dim_date dd ON f.deliver_date_key = dd.date_key
WHERE od.year = 2025 AND od.quarter = 4;
```

### Junk Dimensions

Low-cardinality flags (e.g., `is_gift_wrap`, `payment_method`, `shipping_type`) that don't deserve their own dimension can be combined into a single **junk dimension**:

```sql
CREATE TABLE dim_transaction_flags (
    flag_key        INT PRIMARY KEY,
    is_gift_wrap    BOOLEAN,
    payment_method  VARCHAR(20),   -- 'credit','debit','cash','mobile'
    shipping_type   VARCHAR(20),   -- 'standard','express','overnight'
    is_online       BOOLEAN
);
```

---

## 9. Slowly Changing Dimensions (SCD)

When dimension attributes change over time (a customer moves city, a product changes category), how do we handle the history?

<svg viewBox="0 0 800 480" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Slowly Changing Dimensions (SCD Types)</text>

  <!-- SCD Type 0 -->
  <rect x="20" y="50" width="370" height="80" rx="10" fill="#f5f5f5" stroke="#9e9e9e" stroke-width="2"/>
  <text x="35" y="72" font-size="12" font-weight="bold" fill="#616161">Type 0 — Retain Original</text>
  <text x="35" y="90" font-size="10" fill="#555">Never update. Original value preserved forever.</text>
  <text x="35" y="108" font-size="10" fill="#555">Use: date of birth, original signup date.</text>

  <!-- SCD Type 1 -->
  <rect x="410" y="50" width="370" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="425" y="72" font-size="12" font-weight="bold" fill="#1565c0">Type 1 — Overwrite</text>
  <text x="425" y="90" font-size="10" fill="#555">Simply UPDATE the old value. No history kept.</text>
  <text x="425" y="108" font-size="10" fill="#555">Use: fixing typos, correcting data errors.</text>

  <!-- SCD Type 2 -->
  <rect x="20" y="150" width="760" height="140" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2.5"/>
  <text x="35" y="175" font-size="13" font-weight="bold" fill="#2e7d32">Type 2 — Add New Row (Most Common)</text>
  <text x="35" y="195" font-size="10" fill="#555">Insert a new row with new surrogate key. Mark old row as expired.</text>
  <text x="35" y="215" font-size="10" fill="#555">Maintains full history. Facts join to the version that was current at transaction time.</text>

  <!-- Type 2 example table -->
  <rect x="45" y="230" width="720" height="48" rx="4" fill="#c8e6c9"/>
  <text x="55" y="248" font-size="9" font-weight="bold" fill="#333">customer_key</text>
  <text x="155" y="248" font-size="9" font-weight="bold" fill="#333">customer_id</text>
  <text x="250" y="248" font-size="9" font-weight="bold" fill="#333">city</text>
  <text x="340" y="248" font-size="9" font-weight="bold" fill="#333">effective_from</text>
  <text x="460" y="248" font-size="9" font-weight="bold" fill="#333">effective_to</text>
  <text x="575" y="248" font-size="9" font-weight="bold" fill="#333">is_current</text>
  <text x="665" y="248" font-size="9" font-weight="bold" fill="#333">version</text>
  <text x="55" y="268" font-size="9" fill="#333">1001</text>
  <text x="155" y="268" font-size="9" fill="#333">C-42</text>
  <text x="250" y="268" font-size="9" fill="#c62828">New York</text>
  <text x="340" y="268" font-size="9" fill="#333">2022-01-15</text>
  <text x="460" y="268" font-size="9" fill="#c62828">2024-06-30</text>
  <text x="575" y="268" font-size="9" fill="#c62828">FALSE</text>
  <text x="665" y="268" font-size="9" fill="#333">1</text>
  <text x="55" y="268" font-size="9" fill="#333">1001</text>
  <text x="155" y="268" font-size="9" fill="#333">C-42</text>

  <!-- Second row of type 2 -->
  <text x="55" y="278" font-size="9" fill="#2e7d32">1002</text>
  <text x="155" y="278" font-size="9" fill="#333">C-42</text>
  <text x="250" y="278" font-size="9" fill="#2e7d32">Chicago</text>
  <text x="340" y="278" font-size="9" fill="#333">2024-07-01</text>
  <text x="460" y="278" font-size="9" fill="#2e7d32">9999-12-31</text>
  <text x="575" y="278" font-size="9" fill="#2e7d32">TRUE</text>
  <text x="665" y="278" font-size="9" fill="#333">2</text>

  <!-- SCD Type 3 -->
  <rect x="20" y="310" width="370" height="80" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="35" y="332" font-size="12" font-weight="bold" fill="#e65100">Type 3 — Add Column</text>
  <text x="35" y="350" font-size="10" fill="#555">Keep current + previous value in same row.</text>
  <text x="35" y="368" font-size="10" fill="#555">Only one level of history (current + original).</text>

  <!-- SCD Type 4 -->
  <rect x="410" y="310" width="370" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="425" y="332" font-size="12" font-weight="bold" fill="#6a1b9a">Type 4 — History Table</text>
  <text x="425" y="350" font-size="10" fill="#555">Current dim + separate history table.</text>
  <text x="425" y="368" font-size="10" fill="#555">Fast lookups on current; full audit trail separate.</text>

  <!-- SCD Type 6 -->
  <rect x="20" y="410" width="760" height="60" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="35" y="432" font-size="12" font-weight="bold" fill="#00695c">Type 6 — Hybrid (1 + 2 + 3 combined)</text>
  <text x="35" y="452" font-size="10" fill="#555">Type 2 rows with a Type 3 "current" column overwritten (Type 1) on each change. Most flexible but most complex.</text>
</svg>

### SCD Type 2 Implementation

```sql
-- When customer C-42 moves from New York to Chicago:

-- Step 1: Expire the current row
UPDATE dim_customer
SET effective_to = CURRENT_DATE - INTERVAL '1 day',
    is_current   = FALSE
WHERE customer_id = 'C-42'
  AND is_current = TRUE;

-- Step 2: Insert a new version
INSERT INTO dim_customer
    (customer_key, customer_id, full_name, city, state,
     effective_from, effective_to, is_current, version)
VALUES
    (nextval('customer_key_seq'), 'C-42', 'Jane Doe', 'Chicago', 'IL',
     CURRENT_DATE, '9999-12-31', TRUE, 2);
```

### SCD Type 6 (Hybrid) Example

```sql
CREATE TABLE dim_customer_type6 (
    customer_key      INT PRIMARY KEY,
    customer_id       VARCHAR(20),
    full_name         VARCHAR(150),
    current_city      VARCHAR(100),   -- Type 1: always overwritten to latest
    historical_city   VARCHAR(100),   -- Type 3: the city AT the time of this version
    effective_from    DATE,           -- Type 2: versioning dates
    effective_to      DATE,
    is_current        BOOLEAN
);
```

---

## 10. ETL vs ELT Pipelines

### ETL — Extract, Transform, Load

Data is **extracted** from sources, **transformed** (cleaned, aggregated, mapped) in a separate processing layer, then **loaded** into the warehouse.

### ELT — Extract, Load, Transform

Data is **extracted** from sources, **loaded** raw into the warehouse (or data lake), then **transformed** using the warehouse's own compute engine (SQL, dbt, etc.).

<svg viewBox="0 0 800 400" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="22" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">ETL vs ELT Pipelines</text>

  <!-- ETL (top) -->
  <text x="400" y="52" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">ETL — Traditional</text>

  <rect x="30" y="65" width="140" height="55" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="100" y="88" text-anchor="middle" font-size="11" font-weight="bold" fill="#1565c0">Extract</text>
  <text x="100" y="106" text-anchor="middle" font-size="9" fill="#555">Pull from sources</text>

  <line x1="175" y1="92" x2="230" y2="92" stroke="#1565c0" stroke-width="2" marker-end="url(#etlArr)"/>
  <defs><marker id="etlArr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#1565c0"/></marker></defs>

  <rect x="235" y="65" width="220" height="55" rx="8" fill="#fff9c4" stroke="#f57f17" stroke-width="2.5"/>
  <text x="345" y="88" text-anchor="middle" font-size="11" font-weight="bold" fill="#f57f17">Transform</text>
  <text x="345" y="106" text-anchor="middle" font-size="9" fill="#555">External engine (Informatica, SSIS, Spark)</text>

  <line x1="460" y1="92" x2="515" y2="92" stroke="#1565c0" stroke-width="2" marker-end="url(#etlArr)"/>

  <rect x="520" y="65" width="140" height="55" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="590" y="88" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">Load</text>
  <text x="590" y="106" text-anchor="middle" font-size="9" fill="#555">Into data warehouse</text>

  <rect x="690" y="70" width="90" height="45" rx="6" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="735" y="90" text-anchor="middle" font-size="9" fill="#555">Clean data</text>
  <text x="735" y="105" text-anchor="middle" font-size="9" fill="#555">in warehouse</text>

  <!-- Divider -->
  <line x1="30" y1="150" x2="770" y2="150" stroke="#ddd" stroke-width="1" stroke-dasharray="6,4"/>

  <!-- ELT (bottom) -->
  <text x="400" y="180" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">ELT — Modern</text>

  <rect x="30" y="195" width="140" height="55" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="100" y="218" text-anchor="middle" font-size="11" font-weight="bold" fill="#1565c0">Extract</text>
  <text x="100" y="236" text-anchor="middle" font-size="9" fill="#555">Pull from sources</text>

  <line x1="175" y1="222" x2="230" y2="222" stroke="#e65100" stroke-width="2" marker-end="url(#eltArr)"/>
  <defs><marker id="eltArr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#e65100"/></marker></defs>

  <rect x="235" y="195" width="140" height="55" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="305" y="218" text-anchor="middle" font-size="11" font-weight="bold" fill="#2e7d32">Load</text>
  <text x="305" y="236" text-anchor="middle" font-size="9" fill="#555">Raw into warehouse</text>

  <line x1="380" y1="222" x2="435" y2="222" stroke="#e65100" stroke-width="2" marker-end="url(#eltArr)"/>

  <rect x="440" y="195" width="220" height="55" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="2.5"/>
  <text x="550" y="218" text-anchor="middle" font-size="11" font-weight="bold" fill="#e65100">Transform</text>
  <text x="550" y="236" text-anchor="middle" font-size="9" fill="#555">Inside warehouse (SQL, dbt, Dataform)</text>

  <rect x="690" y="200" width="90" height="45" rx="6" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="735" y="218" text-anchor="middle" font-size="9" fill="#555">Clean data</text>
  <text x="735" y="233" text-anchor="middle" font-size="9" fill="#555">in warehouse</text>

  <!-- Comparison table -->
  <rect x="30" y="275" width="740" height="115" rx="10" fill="#fafafa" stroke="#ccc" stroke-width="1"/>
  <text x="400" y="295" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Comparison</text>

  <text x="50" y="318" font-size="10" font-weight="bold" fill="#1565c0">ETL:</text>
  <text x="95" y="318" font-size="9" fill="#555">Transform before loading. Good when warehouse compute is expensive. Tools: Informatica, Talend, SSIS, Apache NiFi.</text>
  <text x="95" y="336" font-size="9" fill="#555">Better for strict data quality gates. Data arrives pre-cleaned. Traditional on-premise approach.</text>

  <text x="50" y="360" font-size="10" font-weight="bold" fill="#e65100">ELT:</text>
  <text x="95" y="360" font-size="9" fill="#555">Load raw, transform in-place. Leverages cheap cloud compute (BigQuery, Snowflake, Redshift). Tools: dbt, Dataform, Spark SQL.</text>
  <text x="95" y="378" font-size="9" fill="#555">Keeps raw data for re-processing. Schema-on-read flexibility. Modern cloud-native approach.</text>
</svg>

### Common ETL/ELT Tools

| Category | ETL (external transform) | ELT (in-warehouse transform) |
|---|---|---|
| **Orchestration** | Apache Airflow, Prefect, Dagster | Apache Airflow, Prefect, Dagster |
| **Ingestion** | Fivetran, Airbyte, Stitch, Matillion | Fivetran, Airbyte, Stitch |
| **Transformation** | Informatica, Talend, SSIS, Spark | **dbt**, Dataform, Spark SQL |
| **Warehouse** | On-premise DW | Snowflake, BigQuery, Redshift, Databricks |

### dbt (data build tool) — The ELT Standard

```sql
-- models/marts/sales/fct_daily_sales.sql (dbt model)
WITH source AS (
    SELECT * FROM {{ ref('stg_sales') }}
),

enriched AS (
    SELECT
        s.sale_date,
        s.product_id,
        p.category,
        p.brand,
        s.quantity,
        s.total_amount
    FROM source s
    JOIN {{ ref('dim_product') }} p
      ON s.product_id = p.product_id
)

SELECT
    sale_date,
    category,
    brand,
    COUNT(*)            AS num_transactions,
    SUM(quantity)       AS total_units,
    SUM(total_amount)   AS total_revenue
FROM enriched
GROUP BY sale_date, category, brand
```

---

## 11. Columnar Storage Engines

### Row-Oriented vs Column-Oriented

Traditional OLTP databases store data **row by row** — great for reading entire rows (e.g., `SELECT * WHERE id = 42`). Columnar engines store data **column by column** — great for scanning a few columns across millions of rows (e.g., `SELECT SUM(revenue) GROUP BY region`).

<svg viewBox="0 0 800 380" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="25" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Row Store vs Column Store</text>

  <!-- Row store -->
  <text x="200" y="55" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Row-Oriented (OLTP)</text>

  <!-- Row 1 -->
  <rect x="30" y="70" width="55" height="28" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="57" y="89" text-anchor="middle" font-size="9" fill="#333">Alice</text>
  <rect x="85" y="70" width="55" height="28" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="112" y="89" text-anchor="middle" font-size="9" fill="#333">NYC</text>
  <rect x="140" y="70" width="55" height="28" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="167" y="89" text-anchor="middle" font-size="9" fill="#333">$120</text>
  <rect x="195" y="70" width="55" height="28" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="222" y="89" text-anchor="middle" font-size="9" fill="#333">2025</text>
  <rect x="250" y="70" width="55" height="28" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="277" y="89" text-anchor="middle" font-size="9" fill="#333">Widget</text>
  <text x="330" y="89" font-size="9" fill="#888">← Row 1</text>

  <!-- Row 2 -->
  <rect x="30" y="100" width="55" height="28" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="57" y="119" text-anchor="middle" font-size="9" fill="#333">Bob</text>
  <rect x="85" y="100" width="55" height="28" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="112" y="119" text-anchor="middle" font-size="9" fill="#333">LA</text>
  <rect x="140" y="100" width="55" height="28" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="167" y="119" text-anchor="middle" font-size="9" fill="#333">$85</text>
  <rect x="195" y="100" width="55" height="28" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="222" y="119" text-anchor="middle" font-size="9" fill="#333">2025</text>
  <rect x="250" y="100" width="55" height="28" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="277" y="119" text-anchor="middle" font-size="9" fill="#333">Gadget</text>
  <text x="330" y="119" font-size="9" fill="#888">← Row 2</text>

  <!-- Row 3 -->
  <rect x="30" y="130" width="55" height="28" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="57" y="149" text-anchor="middle" font-size="9" fill="#333">Carol</text>
  <rect x="85" y="130" width="55" height="28" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="112" y="149" text-anchor="middle" font-size="9" fill="#333">NYC</text>
  <rect x="140" y="130" width="55" height="28" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="167" y="149" text-anchor="middle" font-size="9" fill="#333">$200</text>
  <rect x="195" y="130" width="55" height="28" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="222" y="149" text-anchor="middle" font-size="9" fill="#333">2024</text>
  <rect x="250" y="130" width="55" height="28" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="277" y="149" text-anchor="middle" font-size="9" fill="#333">Widget</text>
  <text x="330" y="149" font-size="9" fill="#888">← Row 3</text>

  <text x="200" y="180" text-anchor="middle" font-size="9" fill="#888">Disk layout: [Alice|NYC|$120|2025|Widget][Bob|LA|$85|2025|Gadget]...</text>
  <text x="200" y="194" text-anchor="middle" font-size="9" fill="#c62828">SUM(amount) → must read ALL columns of ALL rows</text>

  <!-- Column store -->
  <text x="600" y="55" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">Column-Oriented (OLAP)</text>

  <rect x="440" y="70" width="60" height="88" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.5"/>
  <text x="470" y="89" text-anchor="middle" font-size="9" fill="#333">Alice</text>
  <text x="470" y="119" text-anchor="middle" font-size="9" fill="#333">Bob</text>
  <text x="470" y="149" text-anchor="middle" font-size="9" fill="#333">Carol</text>

  <rect x="510" y="70" width="50" height="88" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.5"/>
  <text x="535" y="89" text-anchor="middle" font-size="9" fill="#333">NYC</text>
  <text x="535" y="119" text-anchor="middle" font-size="9" fill="#333">LA</text>
  <text x="535" y="149" text-anchor="middle" font-size="9" fill="#333">NYC</text>

  <rect x="570" y="70" width="50" height="88" fill="#fff3e0" stroke="#e65100" stroke-width="2.5"/>
  <text x="595" y="89" text-anchor="middle" font-size="9" fill="#333">$120</text>
  <text x="595" y="119" text-anchor="middle" font-size="9" fill="#333">$85</text>
  <text x="595" y="149" text-anchor="middle" font-size="9" fill="#333">$200</text>

  <rect x="630" y="70" width="50" height="88" fill="#fce4ec" stroke="#c62828" stroke-width="1.5"/>
  <text x="655" y="89" text-anchor="middle" font-size="9" fill="#333">2025</text>
  <text x="655" y="119" text-anchor="middle" font-size="9" fill="#333">2025</text>
  <text x="655" y="149" text-anchor="middle" font-size="9" fill="#333">2024</text>

  <rect x="690" y="70" width="60" height="88" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.5"/>
  <text x="720" y="89" text-anchor="middle" font-size="9" fill="#333">Widget</text>
  <text x="720" y="119" text-anchor="middle" font-size="9" fill="#333">Gadget</text>
  <text x="720" y="149" text-anchor="middle" font-size="9" fill="#333">Widget</text>

  <text x="600" y="180" text-anchor="middle" font-size="9" fill="#888">Disk layout: [Alice|Bob|Carol] [NYC|LA|NYC] [$120|$85|$200]...</text>
  <text x="600" y="194" text-anchor="middle" font-size="9" fill="#2e7d32">SUM(amount) → reads ONLY the amount column</text>

  <!-- Benefits comparison -->
  <rect x="30" y="215" width="350" height="150" rx="10" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="205" y="238" text-anchor="middle" font-size="11" font-weight="bold" fill="#333">Row Store Advantages</text>
  <text x="45" y="258" font-size="10" fill="#555">Fast single-row lookups by PK</text>
  <text x="45" y="276" font-size="10" fill="#555">Efficient INSERT / UPDATE / DELETE</text>
  <text x="45" y="294" font-size="10" fill="#555">Low overhead for point queries</text>
  <text x="45" y="312" font-size="10" fill="#555">Better for mixed workloads</text>
  <text x="45" y="335" font-size="9" fill="#888">Examples: PostgreSQL, MySQL, Oracle, SQL Server</text>

  <rect x="420" y="215" width="350" height="150" rx="10" fill="#f5f5f5" stroke="#999" stroke-width="1"/>
  <text x="595" y="238" text-anchor="middle" font-size="11" font-weight="bold" fill="#333">Column Store Advantages</text>
  <text x="435" y="258" font-size="10" fill="#555">10-100x faster analytical scans</text>
  <text x="435" y="276" font-size="10" fill="#555">Excellent compression (same-type data)</text>
  <text x="435" y="294" font-size="10" fill="#555">Read only needed columns (I/O savings)</text>
  <text x="435" y="312" font-size="10" fill="#555">Vectorised execution on modern CPUs</text>
  <text x="435" y="335" font-size="9" fill="#888">Examples: Redshift, BigQuery, ClickHouse, Parquet, DuckDB</text>
</svg>

### Why Columnar Compresses Better

Same-type values stored together have high redundancy:

| Column | Values | Compression Technique |
|---|---|---|
| `country` | USA, USA, USA, UK, UK, USA, ... | Dictionary encoding → [0,0,0,1,1,0,...] |
| `year` | 2025, 2025, 2025, 2025, ... | Run-length encoding → (2025, 10000) |
| `amount` | 120.50, 85.00, 200.00, ... | Delta encoding, bit-packing |
| `is_returned` | FALSE, FALSE, TRUE, FALSE, ... | Bitmap → 0010... |

Typical compression ratios: **5:1 to 10:1** for columnar vs **2:1 to 3:1** for row-based.

### Column Store Engines and Formats

| Engine / Format | Type | Notes |
|---|---|---|
| **Amazon Redshift** | Cloud DW | Columnar with zone maps, sort keys |
| **Google BigQuery** | Cloud DW | Capacitor columnar format, serverless |
| **Snowflake** | Cloud DW | Micro-partitions, automatic clustering |
| **ClickHouse** | Open source | MergeTree engine, extremely fast aggregations |
| **Apache Parquet** | File format | Columnar for data lakes (S3, HDFS) |
| **Apache ORC** | File format | Optimised Row Columnar (Hive ecosystem) |
| **DuckDB** | Embedded | "SQLite for analytics" — in-process columnar |

---

## 12. ROLLUP, CUBE & GROUPING SETS

These SQL extensions generate **multiple levels of aggregation** in a single query — replacing several UNION ALL queries.

### GROUPING SETS — Explicit Control

```sql
-- Generate subtotals for specific grouping combinations
SELECT
    d.year,
    p.category,
    s.region,
    SUM(f.total_amount) AS revenue
FROM fact_sales f
JOIN dim_date     d ON f.date_key    = d.date_key
JOIN dim_product  p ON f.product_key = p.product_key
JOIN dim_store    s ON f.store_key   = s.store_key
GROUP BY GROUPING SETS (
    (d.year, p.category, s.region),   -- detail
    (d.year, p.category),              -- by year + category
    (d.year),                          -- by year only
    ()                                 -- grand total
)
ORDER BY d.year, p.category, s.region;
```

| year | category | region | revenue |
|---|---|---|---|
| 2025 | Electronics | East | 1,200,000 |
| 2025 | Electronics | West | 980,000 |
| 2025 | Electronics | NULL | 2,180,000 |
| 2025 | Clothing | East | 560,000 |
| 2025 | Clothing | West | 430,000 |
| 2025 | Clothing | NULL | 990,000 |
| 2025 | NULL | NULL | 3,170,000 |
| NULL | NULL | NULL | 12,500,000 |

### ROLLUP — Hierarchical Subtotals

`ROLLUP` generates subtotals from right to left, producing a **hierarchy** of aggregations:

```sql
-- ROLLUP(a, b, c) generates:
-- (a, b, c)  → detail
-- (a, b)     → subtotal removing c
-- (a)        → subtotal removing b, c
-- ()         → grand total

SELECT
    d.year,
    d.quarter,
    d.month_name,
    SUM(f.total_amount) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year IN (2024, 2025)
GROUP BY ROLLUP (d.year, d.quarter, d.month_name)
ORDER BY d.year, d.quarter, d.month_name;
```

### CUBE — All Combinations

`CUBE` generates subtotals for **every possible combination** of the grouped columns:

```sql
-- CUBE(a, b, c) generates 2^3 = 8 grouping sets:
-- (a, b, c), (a, b), (a, c), (b, c), (a), (b), (c), ()

SELECT
    p.category,
    s.region,
    SUM(f.total_amount)  AS revenue,
    COUNT(*)             AS num_sales
FROM fact_sales f
JOIN dim_product  p ON f.product_key = p.product_key
JOIN dim_store    s ON f.store_key   = s.store_key
GROUP BY CUBE (p.category, s.region)
ORDER BY p.category, s.region;
```

| category | region | revenue | num_sales |
|---|---|---|---|
| Clothing | East | 560,000 | 12,400 |
| Clothing | West | 430,000 | 9,800 |
| Clothing | NULL | 990,000 | 22,200 |
| Electronics | East | 1,200,000 | 8,100 |
| Electronics | West | 980,000 | 7,300 |
| Electronics | NULL | 2,180,000 | 15,400 |
| NULL | East | 1,760,000 | 20,500 |
| NULL | West | 1,410,000 | 17,100 |
| NULL | NULL | 3,170,000 | 37,600 |

### GROUPING() Function — Detecting Subtotal Rows

```sql
SELECT
    CASE WHEN GROUPING(p.category) = 1 THEN 'ALL Categories'
         ELSE p.category END AS category,
    CASE WHEN GROUPING(s.region) = 1 THEN 'ALL Regions'
         ELSE s.region END AS region,
    SUM(f.total_amount) AS revenue,
    GROUPING(p.category) AS is_category_total,
    GROUPING(s.region)   AS is_region_total
FROM fact_sales f
JOIN dim_product  p ON f.product_key = p.product_key
JOIN dim_store    s ON f.store_key   = s.store_key
GROUP BY CUBE (p.category, s.region)
ORDER BY GROUPING(p.category), GROUPING(s.region), p.category, s.region;
```

<svg viewBox="0 0 760 300" xmlns="http://www.w3.org/2000/svg" style="max-width:760px; font-family:Arial,sans-serif;">
  <text x="380" y="22" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">ROLLUP vs CUBE vs GROUPING SETS</text>

  <!-- ROLLUP -->
  <rect x="20" y="50" width="220" height="230" rx="12" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="130" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">ROLLUP(a, b, c)</text>
  <text x="130" y="95" text-anchor="middle" font-size="10" fill="#555">Hierarchical drill-down</text>
  <rect x="40" y="108" width="180" height="22" rx="4" fill="#bbdefb"/>
  <text x="130" y="124" text-anchor="middle" font-size="10" fill="#333">(a, b, c) — detail</text>
  <rect x="40" y="135" width="180" height="22" rx="4" fill="#90caf9"/>
  <text x="130" y="151" text-anchor="middle" font-size="10" fill="#333">(a, b) — sub-total</text>
  <rect x="40" y="162" width="180" height="22" rx="4" fill="#64b5f6"/>
  <text x="130" y="178" text-anchor="middle" font-size="10" fill="#fff">(a) — sub-total</text>
  <rect x="40" y="189" width="180" height="22" rx="4" fill="#1565c0"/>
  <text x="130" y="205" text-anchor="middle" font-size="10" fill="#fff">() — grand total</text>
  <text x="130" y="235" text-anchor="middle" font-size="10" font-weight="bold" fill="#1565c0">n + 1 sets</text>
  <text x="130" y="252" text-anchor="middle" font-size="9" fill="#555">Best for: time hierarchies</text>
  <text x="130" y="268" text-anchor="middle" font-size="9" fill="#555">(year → quarter → month)</text>

  <!-- CUBE -->
  <rect x="270" y="50" width="220" height="230" rx="12" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="380" y="75" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">CUBE(a, b, c)</text>
  <text x="380" y="95" text-anchor="middle" font-size="10" fill="#555">Every combination</text>
  <rect x="290" y="108" width="180" height="22" rx="4" fill="#ffe0b2"/>
  <text x="380" y="124" text-anchor="middle" font-size="10" fill="#333">(a,b,c) (a,b) (a,c) (b,c)</text>
  <rect x="290" y="135" width="180" height="22" rx="4" fill="#ffcc80"/>
  <text x="380" y="151" text-anchor="middle" font-size="10" fill="#333">(a) (b) (c)</text>
  <rect x="290" y="162" width="180" height="22" rx="4" fill="#e65100"/>
  <text x="380" y="178" text-anchor="middle" font-size="10" fill="#fff">() — grand total</text>
  <text x="380" y="210" text-anchor="middle" font-size="10" font-weight="bold" fill="#e65100">2^n sets</text>
  <text x="380" y="232" text-anchor="middle" font-size="9" fill="#555">Best for: cross-tab reports,</text>
  <text x="380" y="248" text-anchor="middle" font-size="9" fill="#555">pivot-style analysis</text>

  <!-- GROUPING SETS -->
  <rect x="520" y="50" width="220" height="230" rx="12" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="630" y="75" text-anchor="middle" font-size="12" font-weight="bold" fill="#2e7d32">GROUPING SETS(...)</text>
  <text x="630" y="95" text-anchor="middle" font-size="10" fill="#555">You pick exactly which</text>
  <rect x="540" y="108" width="180" height="22" rx="4" fill="#c8e6c9"/>
  <text x="630" y="124" text-anchor="middle" font-size="10" fill="#333">(a, b) — you choose</text>
  <rect x="540" y="135" width="180" height="22" rx="4" fill="#a5d6a7"/>
  <text x="630" y="151" text-anchor="middle" font-size="10" fill="#333">(c) — you choose</text>
  <rect x="540" y="162" width="180" height="22" rx="4" fill="#2e7d32"/>
  <text x="630" y="178" text-anchor="middle" font-size="10" fill="#fff">() — if you want it</text>
  <text x="630" y="210" text-anchor="middle" font-size="10" font-weight="bold" fill="#2e7d32">Exactly what you list</text>
  <text x="630" y="232" text-anchor="middle" font-size="9" fill="#555">Best for: custom reports</text>
  <text x="630" y="248" text-anchor="middle" font-size="9" fill="#555">with specific subtotals</text>
</svg>

### ROLLUP and CUBE Are Shortcuts

```sql
-- These are equivalent:
GROUP BY ROLLUP (a, b, c)
GROUP BY GROUPING SETS ((a,b,c), (a,b), (a), ())

-- These are equivalent:
GROUP BY CUBE (a, b)
GROUP BY GROUPING SETS ((a,b), (a), (b), ())
```

---

## 13. Worked Examples

### Example 1 — Beginner: Basic Star Schema Query

**Scenario**: A small bookshop wants monthly revenue by genre.

```sql
-- DDL
CREATE TABLE dim_date (
    date_key   INT PRIMARY KEY,
    full_date  DATE,
    month_name VARCHAR(10),
    year       SMALLINT
);

CREATE TABLE dim_genre (
    genre_key  INT PRIMARY KEY,
    genre_name VARCHAR(50)
);

CREATE TABLE fact_book_sales (
    sale_id    BIGINT PRIMARY KEY,
    date_key   INT REFERENCES dim_date(date_key),
    genre_key  INT REFERENCES dim_genre(genre_key),
    quantity   INT,
    revenue    NUMERIC(10,2)
);

-- Query: Monthly revenue by genre for 2025
SELECT
    d.month_name,
    g.genre_name,
    SUM(f.revenue)  AS total_revenue,
    SUM(f.quantity)  AS total_units
FROM fact_book_sales f
JOIN dim_date  d ON f.date_key  = d.date_key
JOIN dim_genre g ON f.genre_key = g.genre_key
WHERE d.year = 2025
GROUP BY d.month_name, g.genre_name
ORDER BY d.month_name, total_revenue DESC;
```

| month_name | genre_name | total_revenue | total_units |
|---|---|---|---|
| January | Fiction | 4,520.00 | 301 |
| January | Non-Fiction | 3,180.00 | 212 |
| January | Sci-Fi | 1,890.00 | 126 |
| February | Fiction | 5,010.00 | 334 |
| ... | ... | ... | ... |

---

### Example 2 — Beginner: ROLLUP for Time Hierarchy

```sql
-- Year → Quarter → Month revenue breakdown with subtotals
SELECT
    d.year,
    d.quarter,
    d.month_name,
    SUM(f.total_amount) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2025
GROUP BY ROLLUP (d.year, d.quarter, d.month_name)
ORDER BY d.year, d.quarter, d.month_name;
```

**Result highlights:**

| year | quarter | month_name | revenue |
|---|---|---|---|
| 2025 | 1 | January | 1,050,000 |
| 2025 | 1 | February | 980,000 |
| 2025 | 1 | March | 1,120,000 |
| 2025 | 1 | **NULL** | **3,150,000** |
| 2025 | 2 | April | 1,200,000 |
| ... | ... | ... | ... |
| 2025 | **NULL** | **NULL** | **12,500,000** |
| **NULL** | **NULL** | **NULL** | **12,500,000** |

Rows with `NULL` are subtotals: quarter subtotals, year subtotals, and the grand total.

---

### Example 3 — Intermediate: SCD Type 2 with Fact Lookup

**Scenario**: E-commerce warehouse. Customer moves from NYC to Chicago. Sales before the move should still report as NYC.

```sql
-- Customer dimension with SCD Type 2
CREATE TABLE dim_customer (
    customer_key    SERIAL PRIMARY KEY,
    customer_id     VARCHAR(20) NOT NULL,
    full_name       VARCHAR(150),
    city            VARCHAR(100),
    state           VARCHAR(50),
    loyalty_tier    VARCHAR(20),
    effective_from  DATE NOT NULL,
    effective_to    DATE NOT NULL DEFAULT '9999-12-31',
    is_current      BOOLEAN DEFAULT TRUE
);

-- Current state:
-- customer_key=1001, customer_id='C-42', city='New York',
--   effective_from='2022-01-15', effective_to='2024-06-30', is_current=FALSE
-- customer_key=1002, customer_id='C-42', city='Chicago',
--   effective_from='2024-07-01', effective_to='9999-12-31', is_current=TRUE

-- Revenue by customer city — respects historical truth
SELECT
    c.city,
    d.year,
    SUM(f.total_amount) AS revenue
FROM fact_sales f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_date     d ON f.date_key     = d.date_key
WHERE c.customer_id = 'C-42'
GROUP BY c.city, d.year
ORDER BY d.year;
```

| city | year | revenue |
|---|---|---|
| New York | 2023 | 3,200.00 |
| New York | 2024 | 1,800.00 |
| Chicago | 2024 | 2,100.00 |
| Chicago | 2025 | 4,500.00 |

The 2024 revenue splits correctly between both cities.

---

### Example 4 — Intermediate: CUBE Cross-Tab Report

**Scenario**: Marketing wants revenue broken down by every combination of category and region, including all subtotals.

```sql
SELECT
    COALESCE(p.category, '** ALL **')  AS category,
    COALESCE(s.region, '** ALL **')    AS region,
    SUM(f.total_amount)                AS revenue,
    COUNT(DISTINCT f.customer_key)     AS unique_customers,
    GROUPING(p.category, s.region)     AS grouping_id
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_store   s ON f.store_key   = s.store_key
JOIN dim_date    d ON f.date_key    = d.date_key
WHERE d.year = 2025
GROUP BY CUBE (p.category, s.region)
ORDER BY GROUPING(p.category, s.region), p.category, s.region;
```

The `GROUPING()` function with multiple arguments returns a bit-vector encoded as an integer:
- `0` = detail row (both grouped)
- `1` = region is ALL
- `2` = category is ALL
- `3` = grand total (both ALL)

---

### Example 5 — Advanced: Full ETL Pipeline with dbt-Style Models

**Scenario**: Building a warehouse for a SaaS analytics platform. Raw events land in a staging schema; dbt transforms them into dimensional models.

```sql
-- Stage 1: Staging model (raw → cleaned)
-- stg_events.sql
CREATE TABLE staging.stg_events AS
SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp::TIMESTAMP     AS event_at,
    (properties->>'plan')::TEXT    AS plan_name,
    (properties->>'amount')::NUMERIC(10,2) AS amount,
    CURRENT_TIMESTAMP              AS _loaded_at
FROM raw.events
WHERE event_timestamp >= CURRENT_DATE - INTERVAL '2 days';

-- Stage 2: Dimension — dim_user (SCD Type 1 for simplicity)
CREATE TABLE warehouse.dim_user AS
SELECT
    ROW_NUMBER() OVER (ORDER BY user_id) AS user_key,
    user_id,
    name,
    email,
    company,
    plan_name,
    signup_date,
    country
FROM (
    SELECT DISTINCT ON (user_id) *
    FROM staging.stg_users
    ORDER BY user_id, _loaded_at DESC
) latest;

-- Stage 3: Dimension — dim_date (generate via generate_series)
CREATE TABLE warehouse.dim_date AS
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INT      AS date_key,
    d::DATE                            AS full_date,
    EXTRACT(DOW FROM d)::INT           AS day_of_week,
    TO_CHAR(d, 'Month')               AS month_name,
    EXTRACT(QUARTER FROM d)::INT       AS quarter,
    EXTRACT(YEAR FROM d)::INT          AS year,
    d IN ('2025-01-01','2025-12-25')   AS is_holiday
FROM generate_series('2020-01-01', '2030-12-31', '1 day'::INTERVAL) d;

-- Stage 4: Fact table
CREATE TABLE warehouse.fact_events AS
SELECT
    e.event_id,
    dd.date_key,
    du.user_key,
    e.event_type,
    e.amount
FROM staging.stg_events e
JOIN warehouse.dim_date dd ON e.event_at::DATE = dd.full_date
JOIN warehouse.dim_user du ON e.user_id        = du.user_id;

-- Stage 5: Analytical mart — Monthly Recurring Revenue
CREATE TABLE warehouse.mart_mrr AS
SELECT
    dd.year,
    dd.month_name,
    du.plan_name,
    du.country,
    COUNT(DISTINCT du.user_key)  AS active_users,
    SUM(f.amount)                AS mrr
FROM warehouse.fact_events f
JOIN warehouse.dim_date dd ON f.date_key = dd.date_key
JOIN warehouse.dim_user du ON f.user_key = du.user_key
WHERE f.event_type = 'subscription_payment'
GROUP BY dd.year, dd.month_name, du.plan_name, du.country;
```

---

### Example 6 — Advanced: Columnar Optimisation with Redshift Sort/Dist Keys

**Scenario**: Optimising a billion-row fact table on Amazon Redshift.

```sql
-- Redshift-specific DDL with columnar optimisations
CREATE TABLE fact_web_events (
    event_id        BIGINT       IDENTITY(1,1),
    session_id      VARCHAR(64)  ENCODE zstd,
    user_id         INT          ENCODE az64,
    date_key        INT          ENCODE az64,
    page_key        INT          ENCODE az64,
    event_type      VARCHAR(30)  ENCODE bytedict,  -- low cardinality
    device_type     VARCHAR(20)  ENCODE bytedict,
    duration_ms     INT          ENCODE az64,
    is_conversion   BOOLEAN      ENCODE raw
)
DISTSTYLE KEY                    -- distribute rows across nodes
DISTKEY (user_id)                -- co-locate same user on same node
SORTKEY (date_key, event_type);  -- optimise range-scan + filter

-- Redshift automatically compresses columns.
-- The ENCODE hints tune compression per-column:
--   bytedict: for < 256 distinct values
--   az64:     for integers (adaptive)
--   zstd:     general-purpose for strings
--   raw:      no compression (tiny columns)

-- Query leveraging sort key for partition elimination
SELECT
    d.year,
    d.month_name,
    we.event_type,
    COUNT(*)                   AS event_count,
    AVG(we.duration_ms)        AS avg_duration,
    SUM(we.is_conversion::INT) AS conversions
FROM fact_web_events we
JOIN dim_date d ON we.date_key = d.date_key
WHERE we.date_key BETWEEN 20250101 AND 20250331  -- sort key prunes blocks
  AND we.event_type IN ('page_view', 'add_to_cart', 'purchase')
GROUP BY d.year, d.month_name, we.event_type
ORDER BY d.month_name, event_count DESC;

-- Zone maps on the sort key let Redshift skip 
-- entire 1MB blocks that fall outside the date range,
-- often scanning < 5% of the table for a quarterly query.
```

---

## 14. Common Interview Questions

### Q1: What is the difference between OLTP and OLAP?

**A:** OLTP handles day-to-day transactions — short queries, high concurrency, normalised schemas, row storage. OLAP handles analytical queries — long scans, aggregations over billions of rows, denormalised star/snowflake schemas, columnar storage. OLTP is the source of truth for operations; OLAP is the source of insight for decision-making.

---

### Q2: Explain the star schema. Why is it preferred for data warehousing?

**A:** A star schema has a central fact table surrounded by dimension tables joined via foreign keys. It's preferred because: (1) fewer JOINs than normalised schemas, (2) simple for analysts and BI tools to understand, (3) query optimisers recognise star joins and apply star transformations, (4) denormalised dimensions enable fast lookups without cascading joins.

---

### Q3: Star schema vs snowflake schema — when would you choose snowflake?

**A:** Snowflake normalises dimensions into sub-tables (e.g., product → subcategory → category). Choose snowflake when: (1) dimension tables are very large and updates to hierarchies are frequent, (2) storage savings matter, (3) data quality requires enforced referential integrity at each hierarchy level. In practice, star is preferred because modern columnar engines handle the denormalised redundancy efficiently and queries are simpler.

---

### Q4: What are slowly changing dimensions? Compare SCD Types 1, 2, and 3.

**A:** SCDs handle attribute changes in dimensions over time. **Type 1** overwrites — no history, simplest. **Type 2** adds a new row with version dates and a surrogate key — full history, most common. **Type 3** adds columns for "current" and "previous" values — limited history (one level). The choice depends on whether the business needs historical truth (Type 2) or just the latest state (Type 1).

---

### Q5: What is the difference between additive, semi-additive, and non-additive facts?

**A:** **Additive** facts (revenue, quantity) can be summed across all dimensions. **Semi-additive** facts (account balance, inventory count) can be summed across some dimensions but not time (you'd average or take a snapshot). **Non-additive** facts (ratios, percentages, unit price) can't be meaningfully summed — they need weighted averages or other calculations.

---

### Q6: Explain ETL vs ELT. Which is better for modern cloud warehouses?

**A:** ETL transforms data in an external engine before loading into the warehouse. ELT loads raw data first and transforms inside the warehouse using SQL (often dbt). ELT is preferred for modern cloud DWs (Snowflake, BigQuery, Redshift) because: (1) cloud compute is elastic and cheap, (2) raw data is preserved for re-processing, (3) SQL-based transforms are version-controllable and testable, (4) schema-on-read provides flexibility.

---

### Q7: Why are columnar storage engines faster for analytical queries?

**A:** Columnar engines: (1) read only the columns referenced in the query, skipping irrelevant data, (2) achieve 5-10x compression because same-type values compress well (dictionary, RLE, delta encoding), (3) enable vectorised processing where CPU operates on batches of values from one column, (4) leverage zone maps / min-max metadata to skip entire data blocks. For a query like `SUM(amount) WHERE year=2025`, a columnar engine might read 1% of the data a row store would.

---

### Q8: What is a fact constellation (galaxy schema)?

**A:** A galaxy schema has multiple fact tables sharing common (conformed) dimensions. For example, `fact_sales` and `fact_inventory` both share `dim_date` and `dim_product`, but `fact_sales` also connects to `dim_customer` while `fact_inventory` connects to `dim_warehouse`. It models multiple business processes with shared context, enabling cross-process analytics.

---

### Q9: Explain ROLLUP, CUBE, and GROUPING SETS with an example.

**A:** `ROLLUP(year, quarter, month)` produces hierarchical subtotals: `(y,q,m)`, `(y,q)`, `(y)`, `()` — 4 sets, good for drill-down reports. `CUBE(category, region)` produces every combination: `(cat,reg)`, `(cat)`, `(reg)`, `()` — 2^n sets, good for cross-tab analysis. `GROUPING SETS` lets you specify exactly which combinations you want, offering full control. All three replace multiple `UNION ALL` queries.

---

### Q10: What is the grain of a fact table and why is it important?

**A:** The grain defines what one row represents — e.g., one line-item per sale, one page view per session. It's the most important decision because: (1) it determines what questions the fact table can answer, (2) too coarse a grain loses detail (can't drill down), (3) too fine a grain wastes storage and query time, (4) all measures and dimensions must be consistent with the declared grain.

---

### Q11: What are junk dimensions and role-playing dimensions?

**A:** **Junk dimensions** combine low-cardinality flags (is_gift_wrap, payment_method, shipping_type) into a single dimension table rather than cluttering the fact table with many boolean columns. **Role-playing dimensions** are a single dimension table that joins to a fact multiple times in different roles — e.g., `dim_date` joining as `order_date`, `ship_date`, and `delivery_date`.

---

### Q12: Compare Inmon and Kimball approaches to data warehouse design.

**A:** **Inmon** (top-down): build an enterprise-normalised (3NF) data warehouse first, then derive departmental data marts. Ensures consistency but has high upfront cost. **Kimball** (bottom-up): build star-schema data marts for each business process first, integrate later via conformed dimensions. Faster time-to-value, lower initial effort. Most modern cloud warehouses follow Kimball's dimensional approach, though data lakehouses borrow ideas from both.

---

## 15. Key Takeaways

<svg viewBox="0 0 800 520" xmlns="http://www.w3.org/2000/svg" style="max-width:800px; font-family:Arial,sans-serif;">
  <text x="400" y="28" text-anchor="middle" font-size="16" font-weight="bold" fill="#1a1a2e">Data Warehousing & OLAP — Key Takeaways</text>

  <rect x="20" y="48" width="370" height="62" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="2"/>
  <text x="35" y="70" font-size="11" font-weight="bold" fill="#1565c0">1. OLTP ≠ OLAP</text>
  <text x="35" y="86" font-size="9" fill="#555">OLTP runs the business (transactions). OLAP analyses it</text>
  <text x="35" y="100" font-size="9" fill="#555">(aggregations). Don't use an OLTP DB for heavy analytics.</text>

  <rect x="410" y="48" width="370" height="62" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="2"/>
  <text x="425" y="70" font-size="11" font-weight="bold" fill="#2e7d32">2. Star Schema Is King</text>
  <text x="425" y="86" font-size="9" fill="#555">Central fact table + dimension tables. Simple, fast,</text>
  <text x="425" y="100" font-size="9" fill="#555">BI-tool friendly. Use snowflake only when justified.</text>

  <rect x="20" y="120" width="370" height="62" rx="10" fill="#fff3e0" stroke="#e65100" stroke-width="2"/>
  <text x="35" y="142" font-size="11" font-weight="bold" fill="#e65100">3. Get the Grain Right</text>
  <text x="35" y="158" font-size="9" fill="#555">The grain is what one fact row represents. Decide this</text>
  <text x="35" y="172" font-size="9" fill="#555">first — everything else (dims, measures) follows.</text>

  <rect x="410" y="120" width="370" height="62" rx="10" fill="#fce4ec" stroke="#c62828" stroke-width="2"/>
  <text x="425" y="142" font-size="11" font-weight="bold" fill="#c62828">4. SCD Type 2 for History</text>
  <text x="425" y="158" font-size="9" fill="#555">Use surrogate keys + effective dates to track dimension</text>
  <text x="425" y="172" font-size="9" fill="#555">changes over time. Facts join to the correct version.</text>

  <rect x="20" y="192" width="370" height="62" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="2"/>
  <text x="35" y="214" font-size="11" font-weight="bold" fill="#6a1b9a">5. ELT > ETL for Cloud</text>
  <text x="35" y="230" font-size="9" fill="#555">Load raw, transform in-warehouse with dbt/SQL.</text>
  <text x="35" y="244" font-size="9" fill="#555">Raw data preserved. Cloud compute is elastic.</text>

  <rect x="410" y="192" width="370" height="62" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="425" y="214" font-size="11" font-weight="bold" fill="#00695c">6. Columnar = Analytics Fast Lane</text>
  <text x="425" y="230" font-size="9" fill="#555">Column-oriented storage reads less data, compresses</text>
  <text x="425" y="244" font-size="9" fill="#555">better, and enables vectorised CPU operations.</text>

  <rect x="20" y="264" width="370" height="62" rx="10" fill="#fff9c4" stroke="#f57f17" stroke-width="2"/>
  <text x="35" y="286" font-size="11" font-weight="bold" fill="#f57f17">7. ROLLUP / CUBE / GROUPING SETS</text>
  <text x="35" y="302" font-size="9" fill="#555">Generate multi-level aggregations in one query.</text>
  <text x="35" y="316" font-size="9" fill="#555">ROLLUP for hierarchies, CUBE for cross-tabs.</text>

  <rect x="410" y="264" width="370" height="62" rx="10" fill="#e8eaf6" stroke="#3949ab" stroke-width="2"/>
  <text x="425" y="286" font-size="11" font-weight="bold" fill="#3949ab">8. Conformed Dimensions</text>
  <text x="425" y="302" font-size="9" fill="#555">Share dimensions across fact tables (galaxy schema)</text>
  <text x="425" y="316" font-size="9" fill="#555">for consistent cross-process analytics.</text>

  <rect x="20" y="336" width="370" height="62" rx="10" fill="#fbe9e7" stroke="#bf360c" stroke-width="2"/>
  <text x="35" y="358" font-size="11" font-weight="bold" fill="#bf360c">9. Fact Table Types Matter</text>
  <text x="35" y="374" font-size="9" fill="#555">Transaction (event-level), periodic snapshot (balances),</text>
  <text x="35" y="388" font-size="9" fill="#555">accumulating snapshot (milestones), factless (events).</text>

  <rect x="410" y="336" width="370" height="62" rx="10" fill="#e0f7fa" stroke="#006064" stroke-width="2"/>
  <text x="425" y="358" font-size="11" font-weight="bold" fill="#006064">10. Surrogate Keys Always</text>
  <text x="425" y="374" font-size="9" fill="#555">Use integer surrogate keys in the warehouse — never</text>
  <text x="425" y="388" font-size="9" fill="#555">join on natural keys. Stable, narrow, SCD-compatible.</text>

  <!-- Decision guide -->
  <rect x="20" y="420" width="760" height="85" rx="12" fill="#f5f5f5" stroke="#666" stroke-width="1.5"/>
  <text x="400" y="442" text-anchor="middle" font-size="12" font-weight="bold" fill="#333">Quick Decision Guide</text>
  <text x="40" y="462" font-size="10" fill="#555">Need sub-second point lookups? → OLTP (row store)</text>
  <text x="40" y="478" font-size="10" fill="#555">Need aggregated reports over TB+ data? → OLAP (columnar DW)</text>
  <text x="40" y="494" font-size="10" fill="#555">Need both? → OLTP for operations + OLAP for analytics, connected by ETL/ELT pipeline</text>
</svg>

---

## 16. Download

<a href="20_data_warehousing.md" download="20_data_warehousing.md" style="display:inline-block;padding:14px 28px;background:#e65100;color:white;text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;cursor:pointer;">Download 20_data_warehousing.md</a>
