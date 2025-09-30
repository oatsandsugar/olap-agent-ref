# OLAP Schema Type Rules & Linter Guide

See also: [ClickHouse Docs – Selecting Data Types](https://clickhouse.com/docs/best-practices/select-data-types) and [LowCardinality Documentation](https://clickhouse.com/docs/en/sql-reference/data-types/lowcardinality).

## 0. Core Rule: Use the strictest type that preserves semantics and range

OLAP engines like ClickHouse scale with **bytes processed**, not row count.
Choosing the smallest viable type directly reduces:

* I/O (less data read)
* CPU (faster decompression and vectorization)
* Storage (better compression)

**Goal:** smallest safe type for each column
**Trade-off:** do not truncate valid ranges; use detection queries below

## 1. Type Waterfalls (ClickHouse) — with Ranges

Agents should pick the **first** type that safely fits all observed values.

### 1.1 Unsigned Integers (non-negative)

| Type     | Range                          | Notes   |
| -------- | ------------------------------ | ------- |
| `UInt8`  | 0 – 255                        | 1 byte  |
| `UInt16` | 0 – 65,535                     | 2 bytes |
| `UInt32` | 0 – 4,294,967,295              | 4 bytes |
| `UInt64` | 0 – 18,446,744,073,709,551,615 | 8 bytes |

### 1.2 Signed Integers

| Type    | Range                          |
| ------- | ------------------------------ |
| `Int8`  | −128 – 127                     |
| `Int16` | −32,768 – 32,767               |
| `Int32` | −2,147,483,648 – 2,147,483,647 |
| `Int64` | −9.22e18 – 9.22e18             |

### 1.3 Floating / Real

* Use `Float32` unless measurable precision loss
* Use `Float64` only if required for precision or very large magnitudes

### 1.4 Fixed-Scale (Decimal)

* Use `Decimal(P,S)` where **P = total digits**, **S = scale**
* Example: `Decimal(9,2)` stores up to `±99,999,999.99`

### 1.5 Dates and Times

| Type            | Range / Precision       |
| --------------- | ----------------------- |
| `Date`          | 1970-01-01 – 2149-06-06 |
| `Date32`        | ±~5,000 years           |
| `DateTime`      | seconds precision       |
| `DateTime64(3)` | milliseconds precision  |
| `DateTime64(6)` | microseconds precision  |
| `DateTime64(9)` | nanoseconds precision   |

### 1.6 Boolean

* `Bool` (alias of `UInt8`)

### 1.7 Categorical

* `Enum8` → ≤127 unique values (stable set)
* `Enum16` → ≤32,767 values (stable set)
* `LowCardinality(String)` → for evolving low-distinct sets

### 1.8 Strings

* `FixedString(n)` if all rows share exact length *n*
* `String` otherwise
* Wrap with `LowCardinality(...)` if `distinct_count < 10 000` and `ratio ≤ 0.2`

### 1.9 JSON / Semi-Structured

* Light use → `String` + `JSONExtract*()` on read
* Heavy querying → `Object('json')` + materialized typed columns for hot paths

## 2. LowCardinality Rules (Dictionary Encoding)

Use cardinality thresholds to select the correct encoding strategy.
The right choice minimizes memory and lookup overhead while preserving performance.

Columns with repeating values benefit from dictionary encoding, which stores a small dictionary of unique values and replaces each occurrence with a compact surrogate key.

### 2.1 Heuristics

| Cardinality Range | Recommended Type / Encoding     | Notes                                            |
| ----------------- | ------------------------------- | ------------------------------------------------ |
| ≤ 10              | `Enum8`                         | Ideal for small, static sets                     |
| 10 – 10 000       | `LowCardinality(String)`        | Best trade-off for evolving categorical values   |
| 10 000 – 100 000  | `LowCardinality` (watch memory) | Benchmark — dictionary growth may outweigh gains |
| > 100 000         | Base `String` / numeric         | Avoid dictionary overhead; prefer raw types      |

**Rule of Thumb:** Use `LowCardinality` only when `distinct_count < 10k` and `distinct_ratio ≤ 0.2`

### 2.2 Rationale

* Dictionary encodings compress repeated strings via surrogate IDs
* They improve scan speed and compression but scale poorly with very high cardinality
* Thresholds balance I/O reduction with memory overhead

### 2.3 Guidelines

* Static sets (e.g. `status`, `day_of_week`) → `Enum8`
* Small evolving sets (e.g. `country`, `region`) → `LowCardinality(String)`
* Large ID sets (e.g. `user_id`, `session_id`) → plain numeric or string
* Use detection queries (§3) to monitor drift — columns can “age out” of LowCardinality fit

### 2.4 Pitfalls

* High `distinct_ratio` (> 0.2) with many rows → dictionary overhead dominates
* Frequent value churn → costly rebuilds
* Changing enum values requires schema migration
* Watch for memory blow-ups when `LowCardinality` is applied to unbounded text

### 2.5 Detection Query Reference

```sql
SELECT
  uniqExact(col) AS distinct_exact,
  count() AS n_rows,
  round(distinct_exact / n_rows, 4) AS distinct_ratio
FROM tbl;
```

### 2.6 Example Mappings

| Column Example | Distinct Count | Recommended Type         |
| -------------- | -------------- | ------------------------ |
| `status`       | 4              | `Enum8`                  |
| `country`      | 200            | `LowCardinality(String)` |
| `state`        | 50             | `Enum8` or `Enum16`      |
| `user_id`      | 50M            | Plain `UInt64`           |

---

## 3. Detection Queries (Plug-and-Play)

Replace `tbl`, `col` with actual names.

### 3.1 Profile Column

```sql
SELECT
  toTypeName(col) AS current_type,
  count() AS n_rows,
  countIf(isNull(col)) AS n_nulls,
  min(col) AS min_v,
  max(col) AS max_v,
  uniqExact(col) AS distinct_exact,
  round(distinct_exact / n_rows, 6) AS distinct_ratio
FROM tbl;
```
