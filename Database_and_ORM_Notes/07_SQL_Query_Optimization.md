# Database Notes — SQL Query Optimization (EXPLAIN, slow query log)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is query optimization?

*Definition:* **Query optimization** is the process of making your SQL queries run **faster** and use **fewer resources** — by understanding how the database executes them and removing the slow parts (full scans, missing indexes, fetching too much data).

The workflow is always:
```
1. FIND the slow query      → slow query log / monitoring
2. UNDERSTAND why it's slow → EXPLAIN (the execution plan)
3. FIX it                   → indexes, rewrite, fetch less
4. VERIFY                   → EXPLAIN again, measure
```

---

## 2. EXPLAIN — see how the DB runs your query

*Definition:* **`EXPLAIN`** shows the database's **execution plan** — the step-by-step strategy it will use to run a query (which indexes it uses, how it joins tables, how many rows it scans). It's your #1 tool for diagnosing slow queries.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'd@app.com';
```

### Key things to read in the output (MySQL)
| Column | What it tells you | Want to see |
|--------|-------------------|-------------|
| **type** | How rows are accessed | `const`/`ref`/`range` good; **`ALL` = full scan = bad** |
| **key** | Which index is used | A real index name, not `NULL` |
| **rows** | Estimated rows examined | As **low** as possible |
| **Extra** | Notes | `Using index` (covering) good; **`Using filesort`/`Using temporary` = warning** |

```
type = ALL          → full table scan → add an index!
key  = NULL         → no index used   → add an index!
Using filesort      → sorting in memory/disk → index the ORDER BY column
Using index         → covering index → great
```

> Use **`EXPLAIN ANALYZE`** (Postgres, MySQL 8+) to actually *run* the query and show **real** timing and row counts, not just estimates. Even more accurate.

**Interview one-liner:** EXPLAIN reveals the query's execution plan — watch for `type=ALL` (full scan) and `key=NULL` (no index); those signal where to add indexes.

---

## 3. Slow Query Log — find which queries are slow

*Definition:* The **slow query log** is a database feature that **records every query taking longer than a set time threshold**, so you know exactly which queries to optimize (instead of guessing).

```sql
-- MySQL: turn it on, log queries slower than 1 second
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;          -- seconds
```
Then analyze the log (e.g. with `mysqldumpslow` or `pt-query-digest`) to find the worst offenders.

> Postgres equivalent: `log_min_duration_statement = 1000` (ms). Also `pg_stat_statements` aggregates query stats over time — great for spotting the heaviest queries.

**Interview one-liner:** The slow query log records queries exceeding a time threshold, so you can target the actual slow ones instead of guessing.

---

## 4. The most common optimizations (interview gold)

### a) Add the right index
```sql
-- Slow: full scan on a big table
SELECT * FROM orders WHERE customer_id = 42;
-- Fix:
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

### b) Don't use `SELECT *` — fetch only needed columns
```sql
SELECT * FROM users;              -- ❌ pulls every column (more I/O, no covering index)
SELECT id, name FROM users;       -- ✅ only what you need
```

### c) Avoid functions on indexed columns (kills the index)
```sql
WHERE YEAR(created_at) = 2026     -- ❌ index on created_at NOT used
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'  -- ✅ index used
```

### d) Avoid leading wildcards in LIKE
```sql
WHERE name LIKE '%danish'   -- ❌ can't use index (no known prefix)
WHERE name LIKE 'danish%'   -- ✅ index usable (prefix known)
```

### e) Use LIMIT for large result sets / pagination
```sql
SELECT id, name FROM users ORDER BY id LIMIT 20 OFFSET 0;
```

### f) Prefer joins over correlated subqueries (often faster)
```sql
-- Often slower (runs subquery per row):
SELECT * FROM users u WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
-- Often faster:
SELECT DISTINCT u.* FROM users u JOIN orders o ON o.user_id = u.id;
```

### g) Fix N+1 (ORM) — the app-level killer
Covered in the Lazy vs Eager file: use `JOIN FETCH`/`@EntityGraph` instead of one query per row.

---

## 5. Quick diagnosis table

| Symptom in EXPLAIN | Likely cause | Fix |
|--------------------|--------------|-----|
| `type = ALL` | No usable index | Add index on filtered column |
| `key = NULL` | Index not used | Add/adjust index; avoid functions on column |
| High `rows` | Scanning too much | Better index, add filters |
| `Using filesort` | Sorting without index | Index the `ORDER BY` column(s) |
| `Using temporary` | Temp table for grouping | Index `GROUP BY`, simplify query |

---

## 6. Beyond single queries (scaling tactics)

When indexing/rewriting isn't enough:
- **Caching** (Redis) → avoid hitting the DB for repeated reads.
- **Read replicas** → spread read load (see Read Replicas file).
- **Partitioning / sharding** → split huge tables (see Sharding file).
- **Connection pooling** → reuse connections (see Connection Pooling file).
- **Denormalization** → store pre-joined/computed data to skip expensive joins.

**Easy rule of thumb:** First **measure** (slow log + EXPLAIN), then **index** and **rewrite**, then **cache**, then **scale**. Never optimize blindly.

---

## Quick Revision Cheat Sheet

- **Workflow**: find slow query → EXPLAIN → fix → verify. Measure, don't guess.
- **EXPLAIN** = execution plan; watch `type=ALL` (full scan) & `key=NULL` (no index); `EXPLAIN ANALYZE` for real timings.
- **Slow query log** (`long_query_time`) records queries over a threshold to find offenders.
- **Top fixes**: add indexes; avoid `SELECT *`; no functions on indexed columns; no leading `%` in LIKE; use LIMIT; joins over correlated subqueries; fix ORM N+1.
- **EXPLAIN warnings**: `Using filesort`/`Using temporary` → index the ORDER BY/GROUP BY.
- **Scale further**: caching (Redis), read replicas, partitioning/sharding, connection pooling, denormalization.
