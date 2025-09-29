# 🧠 ClickHouse Schema Type Rules & Linter Guide
> Based on [ClickHouse Docs – Selecting Data Types](https://clickhouse.com/docs/best-practices/select-data-types) and [LowCardinality Documentation](https://clickhouse.com/docs/en/sql-reference/data-types/lowcardinality).

---

## 0️⃣ Core Rule: “Use the strictest type that preserves semantics and range”

OLAP engines like ClickHouse scale with **bytes processed**, not row count.  
Choosing the smallest viable type directly reduces:
- **I/O** (less data read)
- **CPU** (faster decompression/vectorization)
- **Storage** (better compression)

> ✅ **Goal:** smallest safe type for each column  
> ⚠️ **Trade-off:** don’t truncate valid ranges; use detection queries below.

---

## 1️⃣ Type Waterfalls (ClickHouse) — with Ranges

Agents: pick the **first** type that safely fits all observed values.

### 🔢 Unsigned Integers (non-negative)
| Type | Range | Notes |
|------|--------|-------|
| `UInt8`  | 0 – 255 | 1 byte |
| `UInt16` | 0 – 65,535 | 2 bytes |
| `UInt32` | 0 – 4,294,967,295 | 4 bytes |
| `UInt64` | 0 – 18,446,744,073,709,551,615 | 8 bytes |

### 🔢 Signed Integers
| Type | Range |
|------|--------|
| `Int8`  | −128 – 127 |
| `Int16` | −32,768 – 32,767 |
| `Int32` | −2,147,483,648 – 2,147,483,647 |
| `Int64` | −9.22e18 – 9.22e18 |

### 🌊 Floating / Real
- `Float32` → prefer unless measurable precision loss  
- `Float64` → only if required for precision or very large magnitudes

### 💰 Fixed-Scale (Decimal)
- Use `Decimal(P,S)` where **P = total digits**, **S = scale**
- Example: `Decimal(9,2)` → up to `±99,999,999.99`

### 📅 Dates & ⏱ Times

| Type             | Range / Precision                                      |
|------------------|--------------------------------------------------------|
| `Date`           | 1970-01-01 – 2149-06-06                                |
| `Date32`         | ±~5,000 years                                          |
| `DateTime`       | seconds precision                                      |
| `DateTime64(3)`  | milliseconds precision                                 |
| `DateTime64(6)`  | microseconds precision                                 |
| `DateTime64(9)`  | nanoseconds precision                                  |


### ⚙️ Boolean
- `Bool` (alias of `UInt8`)

### 🏷 Categorical
- `Enum8` → ≤127 unique values (stable set)
- `Enum16` → ≤32,767 values (stable set)
- `LowCardinality(String)` → for evolving low-distinct sets (preferred)

### 🔠 Strings
- `FixedString(n)` if all rows share exact length *n*  
- `String` otherwise  
- Wrap with `LowCardinality(...)` if **distinct_count < ~10 000** and **ratio ≤ 0.2**

### 🌐 JSON / Semi-Structured
- Light use → `String` + `JSONExtract*()` on read  
- Heavy querying → `Object('json')` + **materialized typed columns** for hot paths

---

## 2️⃣ LowCardinality Rules (Dictionary Encoding)

| Heuristic | Action |
|------------|--------|
| distinct < 10 000 **and** ratio ≤ 0.2 | ✅ Use `LowCardinality(String)` |
| 10 000 – 100 000 | ⚖️ Benchmark first |
| > 100 000 | 🚫 Use plain `String` |

> Docs note: large dictionaries (>100 k) can increase memory + indirection cost. 

---

## 3️⃣ Detection Queries (plug-and-play)

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
