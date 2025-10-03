# OLAP Schema Design Guide

See also:

* [ClickHouse Docs – Schema Design Overview](https://clickhouse.com/docs/en/guides/best-practices/schema-design)
* [DuckDB Docs – Data Modeling](https://duckdb.org/docs/sql/introduction.html)

---

## 0. Core Rule: Schema = Query Plan

In **OLAP systems** (e.g. ClickHouse, DuckDB), your **schema design is your query plan**.

Each design choice — table width, nesting, relationships — directly shapes **query speed**, **compression**, and **cost**.

**Goal:** Design schemas for **query patterns**, not **mutation patterns**.
**Trade-off:** Denormalization increases storage size, but compute cost dominates in OLAP workloads.

---

## 1. OLAP vs OLTP Schema Principles

| Aspect           | OLTP (e.g. Postgres)              | OLAP (e.g. ClickHouse, DuckDB)         |
| ---------------- | --------------------------------- | -------------------------------------- |
| **Goal**         | Data integrity, transaction speed | Analytical throughput, scan speed      |
| **Design**       | Normalized (3NF)                  | Denormalized (Wide, Flat)              |
| **Data Access**  | Row-level lookups                 | Columnar scans, GROUP BY, aggregations |
| **Joins**        | Frequent, cheap                   | Expensive, avoid when possible         |
| **Mutations**    | UPDATE / DELETE                   | Append-only, periodic rebuilds         |
| **Duplication**  | Avoid duplication                 | Embrace duplication for speed          |
| **Optimization** | Indexes, normalization            | ORDER BY, partitioning, compression    |
| **Evolution**    | Frequent schema changes           | Evolve via ETL pipelines               |

---

## 2. Design Heuristics

### 2.1 Table Structure

| Principle                    | Guidance                                                    |
| ---------------------------- | ----------------------------------------------------------- |
| **Fewer tables**             | Reduce join depth; flatten where possible                   |
| **Wider tables**             | Add columns to avoid joins; unused columns don’t slow scans |
| **Flatter structure**        | Flatten nested JSON or arrays on ingest                     |
| **Precompute relationships** | Denormalize derived attributes (e.g. city/state, category)  |

**Heuristic:**

> If a column appears frequently in `WHERE`, `GROUP BY`, or `SELECT`, it likely belongs in the main table.

---

### 2.2 Duplication vs Integrity

Duplication increases storage, but compression offsets cost.

```sql
-- OLTP normalized example
SELECT o.id, c.city
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```sql
-- OLAP denormalized example
SELECT id, city
FROM orders_flat;
```

OLAP engines compress repeated values (e.g. `city`) efficiently; retrieval avoids joins entirely.

---

### 2.3 Mutability

Denormalized tables are **append-only**; avoid frequent `UPDATE`/`DELETE`.

**Pattern:**

* Rebuild tables in **ETL batches** (`CREATE TABLE AS SELECT`)
* Use **versioning or snapshots** rather than row-level updates

```sql
CREATE TABLE orders_flat_v2 AS
SELECT *, now() AS version_ts
FROM source_orders
JOIN source_customers USING (customer_id);
```

---

## 3. Star Schema vs Flat Schema

### 3.1 Star Schema (OLTP or Legacy Analytics)

**Use Case:** OLTP or historical warehouse with expensive storage, cheap joins.

```sql
-- fact_orders
CREATE TABLE fact_orders (
  order_id UInt32,
  customer_key UInt32,
  product_key UInt32,
  date_key UInt32,
  quantity UInt8,
  total_amount Float32
);

-- dim_customer
CREATE TABLE dim_customer (
  customer_key UInt32,
  customer_name String,
  city String,
  state String
);
```

Query:

```sql
SELECT c.state, sum(o.total_amount)
FROM fact_orders o
JOIN dim_customer c USING (customer_key)
GROUP BY c.state;
```

**Cost:** Multi-join aggregation → expensive in columnar systems.

---

### 3.2 Flat Schema (OLAP)

**Use Case:** Analytical queries scanning millions of rows.

```sql
CREATE TABLE orders_flat
(
  order_id UInt32,
  date Date,
  day_of_week String,
  month String,
  year UInt16,
  customer_name String,
  city String,
  state String,
  product_name String,
  category String,
  unit_price Float32,
  quantity UInt8,
  total_amount Float32
)
ENGINE = MergeTree
ORDER BY (state, date);
```

Query:

```sql
SELECT state, sum(total_amount)
FROM orders_flat
WHERE date >= '2025-01-01'
GROUP BY state;
```

**Cost:** Single scan, no joins, SIMD-optimized computation.

---

## 4. Nesting and JSON

Flatten nested JSON at ingest where possible:

| Pattern                     | Recommendation                             |
| --------------------------- | ------------------------------------------ |
| Predictable JSON structure  | Use `JSONEachRow` or `JSONExtract` columns |
| Unpredictable / sparse JSON | Store as `JSON` type; extract selectively  |
| Repeated fields             | Flatten to separate columns                |

```sql
-- Example: flatten predictable nested JSON
SELECT
  JSONExtractString(data, 'user.city') AS city,
  JSONExtractString(data, 'user.state') AS state
FROM events_json;
```

ClickHouse can **auto-generate subcolumns** for typed JSON:

```sql
CREATE TABLE events
(
  data JSON
)
ENGINE = MergeTree
ORDER BY tuple();
```

Then query `data.user.city` directly.

Docs: [ClickHouse JSON Type](https://clickhouse.com/docs/en/sql-reference/data-types/newjson)

---

## 5. Exceptions

Keep separate tables only when:

* Data is **rarely used** in analytical queries
* Data has **very high cardinality**
* Join cost < storage duplication

Pattern:

```sql
SELECT *
FROM orders_flat o
LEFT JOIN metadata m ON o.metadata_id = m.id
WHERE m.flag = 'X';
```

---

## 6. Compression and Columnar Efficiency

In columnar engines:

* **Wide tables** are efficient: unused columns are skipped at read time
* **Low-cardinality columns** compress extremely well
* **SIMD** allows CPU to aggregate millions of values per tick

**Implication:**
Adding columns is cheap if they’re often filtered or aggregated.

---

## 7. Ingestion Strategy

### 7.1 Enrichment at Ingest

Perform joins or lookups during ETL:

```sql
INSERT INTO orders_flat
SELECT
  o.id,
  o.date,
  c.city,
  c.state,
  p.category,
  o.quantity,
  o.total_amount
FROM raw_orders o
JOIN customers c USING (customer_id)
JOIN products p USING (product_id);
```

Avoid runtime joins in queries.

### 7.2 Periodic Rebuilds

Rebuild wide tables in batches to incorporate updates:

```sql
CREATE TABLE orders_flat_v20251003 AS
SELECT *
FROM staging.orders_enriched;
```

Drop or archive older versions when safe.

---

## 8. Validation Queries

### 8.1 Detect Join Depth

Check how many joins a query uses:

```sql
EXPLAIN SYNTAX
SELECT ...
```

If join depth > 2, consider flattening.

---

### 8.2 Identify High Cardinality Columns

```sql
SELECT
  name,
  uniqExact(value) AS distinct_values,
  count() AS total_rows,
  round(distinct_values / total_rows, 4) AS distinct_ratio
FROM system.columns
ARRAY JOIN [col1, col2, col3] AS value
WHERE table = 'orders_flat'
GROUP BY name;
```

Columns with `distinct_ratio < 0.01` → strong candidates for early placement and compression.

---

## 9. Summary

| Category               | Heuristic                                               |
| ---------------------- | ------------------------------------------------------- |
| **Tables**             | Fewer, wider, flatter                                   |
| **Joins**              | Avoid; precompute at ingest                             |
| **Duplication**        | Accept it; compression mitigates cost                   |
| **Nesting**            | Flatten JSON; predictable types can be auto-subcolumned |
| **Ingestion**          | Enrich early, rebuild periodically                      |
| **Mutability**         | Append-only; use versioning                             |
| **Query Focus**        | Design for `WHERE`, `GROUP BY`, `SELECT` usage          |
| **Storage vs Compute** | Storage is cheap; compute isn’t                         |

---

**Agent Tip:**

> Think like a query planner: *What columns are filtered, grouped, or aggregated most often?*
> Model your schema to make those operations effortless — not clever.
