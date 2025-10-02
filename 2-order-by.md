# OLAP ORDER BY Rules & Linter Guide

See also: [ClickHouse Docs – MergeTree ORDER BY](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree) and [ClickHouse Best Practices](https://clickhouse.com/docs/best-practices/merge-tree/).

---

## 0. Core Rule: ORDER BY defines physical clustering

In **ClickHouse**, `ORDER BY` determines how data is **sorted on disk**, forming the basis of:

* **Skip Indexes** → enable data pruning by min/max per block
* **Compression** → adjacent similar values compress better
* **Query Speed** → range scans on key columns are efficient

Unlike OLTP **indexes**, which are lookup maps, `ORDER BY` affects **physical layout**.
A well-chosen key = fewer bytes read, less CPU, faster queries.

**Goal:** Design `ORDER BY` keys aligned with **common filters** and **data shape**
**Trade-off:** Sorting costs CPU during merges; poor keys increase scan cost

---

## 1. ORDER BY Column Selection Rules

### 1.1 General Heuristic

> **Low → High Cardinality, Frequent Filters First**

| Priority | Rule                                  | Example                           |
| -------- | ------------------------------------- | --------------------------------- |
| 1️⃣      | Low cardinality columns first         | `country`, `on_ground`, `status`  |
| 2️⃣      | Columns used in `WHERE` often         | `region`, `category`, `on_ground` |
| 3️⃣      | Time columns last (for mixed filters) | `ts`, `event_date`                |
| 4️⃣      | Time-first for append-only logs       | `ts` (natural insert order)       |
| 5️⃣      | Avoid pure high-card keys             | `uuid`, `id`                      |

---

### 1.2 Common Patterns

| Pattern           | When to Use                           | Example          |
| ----------------- | ------------------------------------- | ---------------- |
| `(on_ground, ts)` | Boolean or enum filters + time ranges | Telemetry, IoT   |
| `(country, ts)`   | Regional aggregates                   | Web analytics    |
| `(ts, service)`   | Append-only logs                      | Event streams    |
| `(user_id, ts)`   | Per-user time-series                  | Activity history |

---

### 1.3 Anti-Patterns

| Issue               | Symptom                      | Example              |
| ------------------- | ---------------------------- | -------------------- |
| High-card key first | No pruning, poor compression | `(uuid)`             |
| Randomized order    | Merge thrashing              | `(rand())`           |
| Unused columns      | Wasteful sorting             | `(col_not_in_where)` |

---

## 2. ORDER BY + Partitioning

`PARTITION BY` works at **file-level granularity**, `ORDER BY` inside each part.

Use both:

```sql
PARTITION BY toYYYYMM(ts)
ORDER BY (country, ts)
```

This enables:

* **Partition pruning** (e.g. `WHERE ts >= now() - INTERVAL 30 DAY`)
* **Intra-part skipping** (e.g. `WHERE country = 'US'`)

---

## 3. Time Granularity Optimization

High-cardinality timestamps slow merges and weaken compression.
Reduce granularity for clustering (not semantics):

```sql
ORDER BY (on_ground, toStartOfMinute(ts))
```

**Guideline:**

| Precision             | Use Case           | Comment               |
| --------------------- | ------------------ | --------------------- |
| `DateTime64(3)`       | ms precision       | Default               |
| `toStartOfSecond(ts)` | 1000x fewer keys   | Better clustering     |
| `toStartOfMinute(ts)` | 60,000x fewer keys | Excellent compression |

---

## 4. LowCardinality Columns in ORDER BY

`LowCardinality(String)` is **fully supported** in `ORDER BY`.

✅ Use when distinct count < 10k
⚠️ Avoid when cardinality > 100k (dictionary overhead)
⚠️ Avoid volatile values (churn increases merge cost)

```sql
ORDER BY (LowCardinality(country), ts)
```

Compression improves when the leading key groups similar strings.

---

## 5. Linter Heuristics

Agents should **analyze each column** and propose keys based on usage and shape.

| Signal                               | Linter Action                    |
| ------------------------------------ | -------------------------------- |
| `col_used_in_where = true`           | Prioritize in key                |
| `cardinality <= 100`                 | Candidate for early key position |
| `type = DateTime`                    | Place last unless append-only    |
| `type = UUID` or `cardinality > 1e6` | Exclude or last position         |
| `distinct_ratio < 0.01`              | Strong clustering candidate      |

---

## 6. Detection Queries

### 6.1 Profile Columns for Key Selection

```sql
SELECT
  name,
  toTypeName(value) AS type,
  uniqExact(value) AS distinct_exact,
  count() AS n_rows,
  round(distinct_exact / n_rows, 4) AS distinct_ratio,
  countIf(isNull(value)) AS n_nulls,
  any(value) AS example
FROM system.columns
ARRAY JOIN [col1, col2, col3] AS value
WHERE table = 'tbl'
GROUP BY name, type;
```

---

### 6.2 Assess Key Efficiency

Estimate pruning benefit:

```sql
SELECT
  min(ts) AS part_min,
  max(ts) AS part_max,
  uniqExact(country) AS n_country,
  count() AS rows
FROM tbl
GROUP BY part_min, part_max;
```

If `n_country` ≪ total distinct → strong clustering.

---

## 7. Examples

### 7.1 Good Key: Regional Telemetry

```sql
CREATE TABLE telemetry
(
  on_ground Bool,
  ts DateTime64(3),
  altitude Float32,
  speed Float32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (on_ground, toStartOfMinute(ts));
```

### 7.2 Good Key: Web Analytics

```sql
CREATE TABLE pageviews
(
  country LowCardinality(String),
  ts DateTime,
  site_id UInt32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (country, ts);
```

### 7.3 Good Key: Append-Only Logs

```sql
CREATE TABLE logs
(
  ts DateTime,
  service String,
  message String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, service);
```

---

## 8. Pitfalls

| Pitfall                 | Why It Hurts                |
| ----------------------- | --------------------------- |
| Sorting by UUID         | No compression, no skipping |
| Overly wide keys        | Merge CPU ↑, insert speed ↓ |
| Ignoring WHERE patterns | Full scans                  |
| High-cardinality time   | Merge thrashing             |
| Randomized order        | Fragmentation               |

---

## 9. Summary

| Category         | Rule of Thumb                            |
| ---------------- | ---------------------------------------- |
| Purpose          | Defines clustering, pruning, compression |
| Key Order        | Low → High cardinality                   |
| Filter Alignment | WHERE columns first                      |
| Time Columns     | Last (mixed), first (append-only)        |
| Partitioning     | Coarse grain; complements ORDER BY       |
| LowCardinality   | Great for small dictionaries             |

---

**Agent Tip:**

> Think like the query planner: *What filters most often? What groups compress best?*
> Recommend keys that match **query shape**, **data shape**, and **insert pattern**.
