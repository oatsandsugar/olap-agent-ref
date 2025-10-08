# olap-agent-ref
Reference documents for my agents building OLAP systems.

Proposed syllabus; I am most sure about the ones coming up soonest. Super open to feedback.

## Phase 1 — Data Modeling & Table / Database Design 

1. ✅ Types (strictest type that fits; bytes processed rule)
2. ✅ LowCardinality & Enums (dictionary encoding thresholds and trade‑offs)
3. ✅ ORDER BY keys (cluster low→high cardinality; align with filters)
4. ✅ Denormalization (wide/flat beats joins for OLAP reads)
5. ✅ Flattening and nesting (Nested vs JSON vs flatten; when to explode)
6. ✅ Nullability vs defaults (avoid null bitmaps; when NULL is justified)
7. Compression & encoding overview (codecs, sort effects, how to measure)
8. ClickHouse codecs (ZSTD/LZ4/Delta/DoubleDelta/T64; per‑column picks)
9. Numeric & date precision (Decimal vs Float; DateTime64; codec synergy)
10. Strings & categoricals (LowCardinality vs Enum vs String; memory impact)
11. JSON type & subcolumns (new JSON; typed paths; dynamic path limits)
12. NamedTuple vs Nested (single object vs repeated arrays; query cost)
13. One table = one grain (line‑item vs order; dedupe and keys)
14. Partitioning strategy (time‑based pruning and lifecycle)
15. ORDER BY × PARTITION BY (skip indexes, merges, hot/cold access)
16. Surrogate keys & IDs (UInt sizes, hashing, collisions)
17. Time‑series modeling (bucketing, sessions, downsampling)
18. Schema evolution (ALTER safety; promote hot paths; land raw JSON)
19. Constraints & immutability (why no FKs; append‑only patterns)
20. Modeling recap & linter rules (agent checks; detection SQL)

## Phase 2 — Query Optimization 

21. Query planning & read path (marks, granules, pruning, EXPLAIN)
22. Predicate pushdown & filter order (min/max indexes; early pruning)
23. Aggregation optimization (GROUP BY keys; combinators; partials)
24. Join strategies (hash vs merge joins; collocation; prejoin choices)
25. Projections (alternate physical layouts for fast paths)
26. Sampling & approximations (SAMPLE; quantiles; uniq variants)
27. Materialized views (schema‑on‑write; incremental rollups; pitfalls)
28. Aggregating/SummingMergeTree (design pre‑agg tables correctly)
29. Arrays at scale (ARRAY JOIN vs array functions; when to flatten)
30. Low‑latency dashboards (buckets, precompute, refresh patterns)
31. Compression tuning in practice (per‑column experiments, benchmarks)
32. Caching & memory locality (mark cache, OS cache, query cache)
33. Parallelism & threads (max_threads; read concurrency; I/O)
34. Memory limits & spills (max_memory_usage; OOM mitigation)
35. High‑cardinality handling (distinct strategies; sketches; indexes)
36. Secondary/skip indexes (bloom, token, set; when they help)
37. Lambda/UDF patterns (vectorized transforms vs row‑by‑row)
38. Anti‑patterns (SELECT *; random ORDER BY; over‑wide ORDER BY)
39. Distributed queries & sharding (clusters, locality, rebalance)
40. Optimization playbook (checklist; query_log metrics; guardrails)

## Phase 3 — Updates & Migration 

41. Immutability & merges (why UPDATE/DELETE is different in OLAP)
42. Lightweight deletes/updates (mutation engine; safe patterns)
43. ReplacingMergeTree for dedupe (version column; correctness)
44. Backfills & rebuilds (CTAS; chunked INSERT SELECT; copy‑on‑write)
45. TTL & retention (expire/move by partition; cold storage tiers)
46. Schema registry & contracts (versioned schemas; compatibility)
47. Migration safety (shadow tables; dual‑write; cutover/rollback)
48. Data validation (counts, checksums, sampling; dbt tests)
49. Late‑arriving data (upserts, re‑aggregation, idempotency)
50. Migration toolkit (automation scripts; playbooks; SLAs)

## Phase 4 — Integrations & Operations

51. Streaming ingest (Kafka; MV routers; backpressure)
52. CDC from OLTP (Debezium; transforms; ordering)
53. APIs & webhooks (direct inserts; dedupe; rate limits)
54. Data lake interop (S3/Parquet; external tables; staging)
55. Real‑time dashboards (freshness targets; rollup layers)
56. Observability (system tables; query_log; tracing; alerts)
57. Cost management (codecs, TTL, merges, part counts, storage)
58. Security & RBAC (roles, row policies, masking)
59. Backups & disaster recovery (BACKUP/RESTORE; snapshots)
60. Capstone & linter pack (agent rules; end‑to‑end review)

