# 4. LLM Agent Guide — Nullability & Defaults (OLAP Read Path Optimization)

**Purpose:**
Teach agents when to allow `NULL` vs assign `DEFAULT` values in OLAP schemas, balancing **semantic clarity** with **performance**.

**Core Rule:**
In OLAP, `NULL` ≠ free. Every nullable column adds a **null bitmap**, breaks **vectorization**, and slows scans.
Prefer **typed defaults** unless nulls are rare *or* semantically essential.

---

## 4.1 Inputs the Agent Must Gather

* Column purpose (key, metric, dimension, metadata)
* Expected **null ratio** (% of rows missing data)
* Query usage: filtered? grouped? aggregated?
* Semantic importance of `unknown` vs `zero` or placeholder

---

## 4.2 Three Questions Before Designing

1. Will this column ever appear in `WHERE`, `GROUP BY`, `ORDER BY`, or joins?
2. Is `NULL` semantically distinct from a default (e.g., `0`, `'UNKNOWN'`)?
3. How often is it missing? (<5%, ~50%, >95%)

---

## 4.3 Decision Rules (General)

* OLAP favors **non-nullable** columns with **defaults**.
* Use `Nullable(Type)` only if nulls are semantically meaningful and rare.
* Keys, partitions, and sort columns **must not** be nullable.
* For aggregates, `NULL` is skipped; `0` is included — know the difference.
* For nested and array types, use **empty structures**, not nulls.

---

## 4.4 Checklist (Agent Execution)

* Define **`NOT NULL DEFAULT <value>`** unless strong reason otherwise.
* Avoid nulls in **ORDER BY** or **key columns**.
* Replace nullable metrics with **sentinels** (`0`, `-1`, `'UNKNOWN'`).
* Use **nullable** only when 95%+ rows are null (bitmap compression wins).
* For JSON, use **subcolumn defaults** or `assumeNotNull()` on read.
* Document sentinel meaning in schema comments.

---

## 4.5 Heuristics Table

| Scenario          | Column Type                             | Notes                            |
| ----------------- | --------------------------------------- | -------------------------------- |
| Always present    | `NOT NULL`                              | Fastest, simplest                |
| <5% nulls         | `NOT NULL DEFAULT`                      | Avoid bitmap overhead            |
| 5–95% nulls       | Rethink                                 | Hybrid model or precompute       |
| >95% nulls        | `Nullable(Type)`                        | Bitmap cheaper than zeros        |
| Key / ORDER BY    | `NOT NULL`                              | ClickHouse disallows `NULL` keys |
| Metric aggregates | `NOT NULL DEFAULT 0`                    | Ensures sum/avg correctness      |
| Text dimension    | `NOT NULL DEFAULT 'UNKNOWN'`            | Keeps filters fast               |
| Nested / Array    | `[]` or `{}`                            | `NULL` not supported             |
| JSON              | `assumeNotNull()` or default on subpath | Avoid per-row null checks        |

---

## 4.6 Example DDL Patterns

**Non-nullable with Default:**

```sql
CREATE TABLE fact_sales (
  ts DateTime64(3, 'UTC'),
  region LowCardinality(String) DEFAULT 'UNKNOWN',
  units UInt32 DEFAULT 0,
  revenue Decimal(10,2) DEFAULT 0.00
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (region, ts);
```

**Nullable (rarely used):**

```sql
CREATE TABLE fact_measurements (
  ts DateTime64(3, 'UTC'),
  sensor_id UInt32,
  temperature Nullable(Float32)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (sensor_id, ts);
```

**Querying Nullable Columns:**

```sql
-- Skip nulls in aggregate
SELECT avgOrNull(temperature) FROM fact_measurements;

-- Coalesce to default for arithmetic
SELECT avg(coalesce(temperature, 0)) FROM fact_measurements;
```

---

## 4.7 Prompt Templates

* **Select Null Strategy:**
  “Given null frequency and query usage, decide whether each column should be `Nullable(Type)` or `NOT NULL DEFAULT`. Justify in terms of scan cost and semantics.”

* **Rewrite Schema:**
  “Refactor this DDL to eliminate nullable columns in keys, sort orders, and high-frequency filters. Replace with typed defaults.”

* **Optimize Queries:**
  “Review queries using `IS NULL` or `JSONExtract`. Suggest schema or default changes to remove per-row null checks.”
