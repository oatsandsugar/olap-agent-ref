# 3. LLM Agent Guide — Modeling Nested and Variably-Shaped Data (Consumption-First)

**Purpose:**
Instruct agents to land and shape messy payloads (nested JSON, heterogeneous logs) for OLAP based on **consumption patterns**.

**Core rule:**
**Schema = Query Plan.** Model from **queries outward**, not payload inward.

---

## 3.1 Inputs the Agent Must Gather

* **Top query patterns:** Most common filters, groupings, joins, and time windows.
* **Intended analytical grain:** Per item, per order, per event, per device, etc.
* **Payload stability:** Stable arrays vs evolving/ragged JSON paths.
* **Data volumes and SLAs:** Insert rates, concurrency, latency/cost targets.

---

## 3.2 Three Questions Before Designing

1. What is the **grain** of the questions I need to answer?
2. Which fields are **filtered or grouped by** most often?
3. Which **paths are hot today (and likely tomorrow)** and should be real columns?

---

## 3.3 Decision Rules (General)

* **One grain per table.** Do not mix levels of detail.
* **Promote hot paths:** Any field used in `WHERE`, `GROUP BY`, or `JOIN` becomes a **typed column** (`MATERIALIZED` if derived).
* **Choose shape by stability:**

  * Predictable repeated structures → `Nested(...)` columns (arrays of fixed objects).
  * Evolving or sparse structures → `JSON` with materialized subpaths.
  * Item-centric analytics → Flatten to item grain; produce rollups via materialized views.
* **Keep raw payloads** for debugging or evolution, but not for hot queries.
* Align `ORDER BY` and `PARTITION BY` to dominant filters and time scans.

---

## 3.4 Checklist (Agent Execution)

* Select and enforce a **single analytical grain**.
* Map payload to schema: nested vs JSON vs flattened, tied to consumption.
* Create **typed columns** for hot paths; **materialize** derived fields.
* Build **materialized views** for common rollups and cohorts.
* Retain **raw JSON** for audit and evolution.
* Validate that critical queries **avoid per-row JSON parsing** and **deep joins**.

---

## 3.5 Guardrails and Red Flags

* Mixed grains in one table.
* Repeated `JSONExtract` in production queries.
* Join depth > 2 on hot paths.
* `ORDER BY` / partition keys that don’t match common filters.
* High-cardinality attributes in sort keys without justification.

---

## 3.6 Descriptive Examples (Illustrative Only)

These examples clarify the rules. Agents should generalize them to any domain.

---

### Example A: Item-Centric Analytics (Flattened)

If most questions are about **item performance**, use an **item-level grain**.
Keep frequently filtered attributes (e.g., product, channel, customer) as **typed columns**.
Build an order-level rollup via a **materialized view**.

**DDL Example (ClickHouse):**

```sql
CREATE TABLE fact_line_items (
  ts              DateTime64(3, 'UTC'),
  transaction_id  String,
  customer_id     UInt64,
  channel         LowCardinality(String),
  sku             String,
  quantity        UInt32,
  unit_price      Decimal(10,2),
  line_revenue    Decimal(12,2) MATERIALIZED quantity * unit_price
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (sku, ts);
```

**Rollup View (Order-Level):**

```sql
CREATE TABLE fact_orders (
  ts              DateTime64(3, 'UTC'),
  transaction_id  String,
  customer_id     UInt64,
  channel         LowCardinality(String),
  line_item_count UInt16,
  total_amount    Decimal(12,2)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, customer_id);

CREATE MATERIALIZED VIEW mv_line_to_orders
TO fact_orders AS
SELECT
  min(ts) AS ts,
  transaction_id,
  any(customer_id) AS customer_id,
  any(channel) AS channel,
  count() AS line_item_count,
  sum(line_revenue) AS total_amount
FROM fact_line_items
GROUP BY transaction_id;
```

---

### Example B: Predictable Nested Structures (`Nested(...)`)

If each record contains a **predictable array of objects** (e.g., `line_items`),
store them as **typed nested columns** for compact storage and ad-hoc flattening.

```sql
CREATE TABLE fact_orders_wide (
  ts              DateTime64(3, 'UTC'),
  transaction_id  String,
  customer_id     UInt64,
  channel         LowCardinality(String),

  items Nested(
    sku        String,
    quantity   UInt32,
    unit_price Decimal(10,2)
  ),

  line_item_count UInt16,
  total_amount    Decimal(12,2)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, customer_id);
```

**Explode on Demand:**

```sql
SELECT items.sku, sum(items.quantity) AS q
FROM fact_orders_wide
ARRAY JOIN items
WHERE ts >= today() - 30
GROUP BY items.sku
ORDER BY q DESC;
```

---

### Example C: Evolving Payloads (`JSON` + Hot Paths)

If structure changes over time (e.g., new metadata fields), use `JSON`
and **materialize hot paths** you query often.

```sql
CREATE TABLE fact_orders_json (
  ts              DateTime64(3, 'UTC'),
  transaction_id  String,
  customer_id     UInt64,
  channel         LowCardinality(String),
  data            JSON,

  -- Precomputed hot fields
  line_item_count UInt16 MATERIALIZED length(data.line_items),
  total_amount    Decimal(12,2) MATERIALIZED data.total_amount:Decimal(12,2)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, customer_id);
```

ClickHouse auto-generates **typed subcolumns** (e.g. `data.line_items.sku`),
so you can scan them **columnarly** without per-row parsing.

---

### Example D: Heterogeneous Event Logs

If a log stream mixes shapes (e.g., metrics + incidents),
land a **common envelope** (e.g., timestamp, device_id, event_type),
store payload in `JSON`, and **materialize hot keys**.

```sql
CREATE TABLE device_events (
  ts         DateTime64(3, 'UTC'),
  device_id  String,
  event_type LowCardinality(String),
  data       JSON,

  cpu        Nullable(Float32)      MATERIALIZED data.cpu:Float32,
  reason     LowCardinality(Nullable(String)) MATERIALIZED data.reason:String,
  latency_ms Nullable(UInt32)       MATERIALIZED data.latency_ms:UInt32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (event_type, device_id, ts);
```

**Query Fast Paths:**

```sql
SELECT reason, count()
FROM device_events
WHERE event_type = 'reboot'
GROUP BY reason
ORDER BY count() DESC;
```

**Split High-Volume Event Types:**

```sql
CREATE MATERIALIZED VIEW mv_route_metrics
TO device_metrics
AS SELECT * FROM device_events WHERE event_type = 'metrics';
```

---

### Example E: Ingest Configuration

Import predictable nested JSON directly into `Nested(...)` columns:

```sql
SET input_format_import_nested_json = 1;
```

Materialize on write (schema-on-write):

```sql
ALTER TABLE fact_orders_json
ADD COLUMN line_item_count UInt16 MATERIALIZED length(data.line_items);
```

---

## 3.7 Prompt Templates for the Agent

* **Select grain and shape:**
  “Given these representative queries and dashboards, identify the dominant analytical grain and recommend a table design. Justify `Nested` vs `JSON` vs flattened strictly in terms of `WHERE` / `GROUP BY` / `JOIN` usage.”

* **Promote hot paths:**
  “From the query corpus, list fields most used in `WHERE` / `GROUP BY` / `JOIN`. Propose DDL to add typed (and where appropriate, `MATERIALIZED`) columns. Rewrite sample queries to remove JSON parsing.”

* **Build rollups:**
  “Define materialized views for the common rollups implied by the queries (e.g., totals per subject, temporal buckets, cohorts) so they incur ingest-time cost rather than query-time cost.”

* **Validate plans:**
  “Explain the execution plans for N critical queries. Flag JSON parsing or deep joins, and propose schema or ORDER BY changes to eliminate them.”

---

## 3.8 Quick Reference

* Model for **how data is consumed**, not how it arrives.
* One **grain per table**; derive others via **materialized views**.
* **Flatten or materialize hot paths**; keep the raw payload only for audit.
* Prefer **typed nested columns** for predictable repetition; **JSON with promoted subpaths** for evolving shapes.
* Partition and order to match **dominant filters and time scans**.
* Critical queries should **not parse JSON** or depend on **deep joins**.
