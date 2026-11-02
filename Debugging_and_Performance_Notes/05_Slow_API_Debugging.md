# Debugging & Performance Notes — Slow API Debugging (p95/p99, Tracing, EXPLAIN ANALYZE, N+1, HikariCP Pool Exhaustion, Timeouts)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Measure first — p95/p99 per route

*Definition:* **p95 latency** is the response time under which 95% of requests finish; **p99** covers 99%. Measured **per route**, they tell you *which* endpoint is slow and how bad the tail is — averages hide it.

Real life: "Average delivery 2 days" sounds fine until you learn 5% of ShipRocket orders take 12 days. Those 5% write the 1-star reviews.

```
100 requests to /api/orders, sorted:
  p50 = 110 ms   → half are faster
  p95 = 420 ms   → slowest 5% are worse than this
  p99 = 2.1 s    → 1 in 100 users waits 2 s   ← the pain lives here
  avg = 160 ms   → looks "fine" ❌
```

```yaml
# Spring Boot: publish histogram buckets so Prometheus can compute percentiles
management.metrics.distribution.percentiles-histogram.http.server.requests: true
```

```promql
# p95 per route (Spring Boot / Micrometer)
histogram_quantile(0.95, sum by (le, uri) (rate(http_server_requests_seconds_bucket[5m])))
# NestJS with prom-client: same idea on your own http_request_duration_seconds_bucket{route=...}
```

> ⚠️ Label by **route template** (`/api/orders/{id}`), never raw URL (`/api/orders/98123`) — raw URLs explode Prometheus label cardinality.

**Interview one-liner:** I start with p95/p99 per route in Grafana, not the average, because the tail is what users feel and it tells me exactly which endpoint to dig into.

---

## 2. Tracing — find the slow span

*Definition:* **Distributed tracing** records one request as a tree of timed **spans** (HTTP handler, SQL query, Redis call, ShipRocket call). **OpenTelemetry** is the vendor-neutral standard; backends include **Jaeger, Grafana Tempo, Datadog APM, New Relic, Elastic APM**.

```
GET /api/orders/98123                                         total 1,840 ms
├─ OrderController.getOrder                                   1,838 ms
│  ├─ redis GET order:98123  (miss)                               2 ms
│  ├─ SELECT * FROM orders WHERE id=?                             4 ms
│  ├─ SELECT * FROM order_items WHERE order_id=?  × 48         ~190 ms   ← N+1 smell
│  ├─ HikariPool getConnection                                  610 ms   ← pool wait!
│  └─ POST apiv2.shiprocket.in/v1/external/courier/track      1,020 ms   ← slow downstream
```

```bash
# Spring Boot: zero-code auto-instrumentation via the OTel Java agent
OTEL_SERVICE_NAME=order-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318 \
java -javaagent:/otel/opentelemetry-javaagent.jar -jar order-service.jar

# NestJS
node --require @opentelemetry/auto-instrumentations-node/register dist/main.js
```

| Signal | Answers |
|--------|---------|
| **Metrics** (Prometheus) | *Which* route is slow, since when |
| **Traces** (OTel/APM) | *Where inside* the request the time goes |
| **Logs** (Loki/ELK, with `trace_id`) | *Why* — the exception, the params |

Memory trick: **"Metrics say WHEN, Traces say WHERE, Logs say WHY."**

**Interview one-liner:** Metrics tell me which route got slow, then a trace waterfall shows the one span eating the time — a query, a pool wait or a downstream call — so I fix that instead of guessing.

---

## 3. Find the slow queries — slow query log & pg_stat_statements

*Definition:* The **slow query log** records every statement slower than a threshold. **`pg_stat_statements`** (PostgreSQL extension) aggregates all queries by shape with call counts and total/mean time.

```sql
-- PostgreSQL (RDS: set in the parameter group)
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'; log_min_duration_statement = 500
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT left(query, 80) AS query, calls,
       round(mean_exec_time::numeric, 1)  AS mean_ms,
       round(total_exec_time::numeric)    AS total_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
--  query                                          | calls  | mean_ms | total_ms
--  SELECT * FROM order_items WHERE order_id = $1  | 912340 |     0.4 |   364936   ← cheap but called 900k×
--  SELECT * FROM orders WHERE customer_id = $1 .. |   8120 |   412.7 |  3351124   ← slow every time
```

MySQL: `SET GLOBAL slow_query_log = 'ON'; SET GLOBAL long_query_time = 0.5;` then `mysqldumpslow -s t slow.log | head`.

> ⚠️ Sort by **total** time, not mean. A 0.4 ms query called 900k times (N+1) costs more than a 400 ms report run twice.

**Interview one-liner:** I enable the slow query log and `pg_stat_statements`, sort by total execution time, and that ranks both the genuinely slow queries and the cheap-but-chatty N+1 ones.

---

## 4. EXPLAIN ANALYZE & the missing index

*Definition:* **`EXPLAIN ANALYZE`** runs the query and shows the real plan with actual times and row counts. A **Seq Scan** filtering away millions of rows usually means a **missing index**.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 4821 AND status = 'PENDING'
ORDER BY created_at DESC LIMIT 20;

-- ❌ before
Limit  (actual time=412.551..412.556 rows=20 loops=1)
  ->  Sort  (Sort Key: created_at DESC)
        ->  Seq Scan on orders  (actual time=0.031..410.882 rows=1290 loops=1)
              Filter: ((customer_id = 4821) AND (status = 'PENDING'))
              Rows Removed by Filter: 1198710
Execution Time: 412.700 ms

-- ✅ fix: composite index matching WHERE (equality first) + ORDER BY
CREATE INDEX CONCURRENTLY idx_orders_cust_status_created
    ON orders (customer_id, status, created_at DESC);

Limit  (actual time=0.040..0.090 rows=20 loops=1)
  ->  Index Scan using idx_orders_cust_status_created on orders
Execution Time: 0.150 ms
```

| Plan node | Meaning |
|-----------|---------|
| **Seq Scan** + big "Rows Removed by Filter" | Reading whole table → index candidate |
| **Index Scan / Index Only Scan** | Using index ✅ (Index Only = no table visit) |
| **Nested Loop** with high `loops=` | Inner side runs many times — check inner index |
| Estimated rows far from actual rows | Stale stats → `ANALYZE orders;` |
| MySQL `type: ALL` | Full table scan |

> ⚠️ `EXPLAIN ANALYZE` **executes** the statement. For `UPDATE/DELETE` wrap in `BEGIN; ... ROLLBACK;`. Use `CREATE INDEX CONCURRENTLY` in production so you don't lock writes on `orders`.

**Interview one-liner:** I run `EXPLAIN ANALYZE` on the top offenders; a Seq Scan throwing away most rows means a missing index, so I add a composite index matching the filter and sort columns, created concurrently.

---

## 5. N+1 queries — detect and fix

*Definition:* **N+1** is when the ORM runs **1 query** for the parent list, then **1 extra query per row** to load a relation. 100 orders → 101 queries.

```
SELECT * FROM orders LIMIT 100;                 -- 1
SELECT * FROM customers WHERE id = 7;           -- +1
SELECT * FROM customers WHERE id = 9;           -- +1 ... ×100   = 101 round trips
```

**Detect:**
```properties
# Spring Boot / Hibernate
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
#  → Session Metrics { ... 98765432 nanoseconds spent executing 101 JDBC statements; ... }
```
```ts
// NestJS / TypeORM: log every query, and anything slower than 500 ms
TypeOrmModule.forRoot({ ..., logging: ['query', 'error'], maxQueryExecutionTime: 500 })
```
Tests can assert it: **Hypersistence Utils** `SQLStatementCountValidator` / **datasource-proxy** (Java).

**Fix:**
```java
// ✅ JPQL JOIN FETCH
@Query("select o from Order o join fetch o.customer where o.status = :s")
List<Order> findWithCustomer(@Param("s") OrderStatus s);

// ✅ @EntityGraph on a derived query
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus s);

// ✅ batch fetching: N queries → N/50 queries using IN (...)
// spring.jpa.properties.hibernate.default_batch_fetch_size=50   (or @BatchSize(size = 50))
```
```ts
// ✅ TypeORM: opt-in per query, no eager: true on heavy relations
orderRepo.find({ where: { status: 'PENDING' }, relations: { customer: true } });
orderRepo.createQueryBuilder('o').leftJoinAndSelect('o.customer', 'c').getMany();
// GraphQL resolvers → DataLoader batches per-field loads into one IN query per tick
```

| Fix | Best for | Watch out |
|-----|----------|-----------|
| **JOIN FETCH / leftJoinAndSelect** | To-one relations, one screen | Fetching a collection + pagination → Hibernate warns `HHH90003004` and paginates **in memory** |
| **@EntityGraph** | Same, but declarative per repo method | Same pagination caveat |
| **Batch size** | Collections, pagination | 1 + N/size queries, not 1 |
| **DataLoader** | GraphQL / NestJS resolvers | Per-request instance, never global |
| **DTO projection** | Read-only list pages | Select only needed columns |

> ⚠️ Real case from my TypeORM work: `eager: true` on `visits`/`agent` relations dragged joins into every list endpoint. Turning eager off and loading relations per query removed the N+1 and the extra data — p95 on list endpoints dropped noticeably.

**Interview one-liner:** I detect N+1 with Hibernate statistics or TypeORM query logging, then fix it with JOIN FETCH/`@EntityGraph`, batch fetching for paginated collections, per-query relations in TypeORM, or DataLoader in GraphQL.

---

## 6. Connection pool exhaustion (HikariCP)

*Definition:* **Pool exhaustion** is when every DB connection in the pool is busy, so new requests **wait** for one and finally fail. The query isn't slow — the **wait for a connection** is.

Real life: A petrol pump with 10 nozzles. Cars aren't slow to fill; there's a queue because 3 drivers went shopping while parked at the nozzle.

```
HikariPool-1 - Connection is not available, request timed out after 30000ms
  (total=10, active=10, idle=0, waiting=37)
```

```promql
hikaricp_connections_active / hikaricp_connections_max      # near 1.0 = saturated
hikaricp_connections_pending                                 # threads waiting > 0 sustained = bad
hikaricp_connections_acquire_seconds_max                     # worst wait for a connection
hikaricp_connections_usage_seconds_max                       # longest a borrow was held
rate(hikaricp_connections_timeout_total[5m])                 # requests that gave up waiting
```

```yaml
spring.datasource.hikari:
  maximum-pool-size: 10
  connection-timeout: 3000          # fail fast (default 30 s hangs threads)
  leak-detection-threshold: 20000   # log stack trace if a connection is held > 20 s
# log → "Connection leak detection triggered for org.postgresql.jdbc.PgConnection@3f2a on thread http-nio-8080-exec-17, stack trace follows"
```

Root causes, most common first:
```java
// ❌ long transaction: holds a DB connection during a 3 s ShipRocket call
@Transactional
public void confirm(Long id) {
    Order o = repo.findById(id).orElseThrow();
    shiprocket.createShipment(o);   // network call INSIDE the transaction
    o.setStatus(SHIPPED);
}
// ✅ do the remote call outside; keep the transaction to the DB writes only
```
- `spring.jpa.open-in-view=true` (default) keeps the connection for the **whole request** incl. JSON rendering → set `false`.
- Leaks: manual `getConnection()` / TypeORM `queryRunner.connect()` without `release()` in `finally`.
- Check the DB side: `SELECT state, count(*) FROM pg_stat_activity GROUP BY state;` — many `idle in transaction` = app holding transactions open.

**Pool sizing formula (HikariCP / PostgreSQL wiki):** `connections = (core_count × 2) + effective_spindle_count` → a 4-vCPU DB ≈ **9–10** connections total.

> ⚠️ The limit is **per DB, not per pod**: 12 pods × pool 20 = 240 connections on an RDS that allows 200 → errors when HPA scales out. Bigger pools usually make it **slower**. Use RDS Proxy / PgBouncer when pods × pool gets large.

**Interview one-liner:** "Connection is not available" means the pool is saturated; I check Hikari active/pending metrics, use leak detection, shorten transactions (no remote calls inside, OSIV off), and size the pool with (cores × 2) + spindles across all pods.

---

## 7. Downstream calls & timeouts

*Definition:* A **downstream dependency** (ShipRocket, Razorpay, another microservice) adds its latency to yours. Without a **timeout**, one slow dependency holds your threads and connections forever.

```java
// Spring Boot 3.2+ RestClient with explicit timeouts
var f = new SimpleClientHttpRequestFactory();
f.setConnectTimeout(Duration.ofSeconds(1));  f.setReadTimeout(Duration.ofSeconds(3));
RestClient shiprocket = RestClient.builder().requestFactory(f).baseUrl("https://apiv2.shiprocket.in").build();
// NestJS HttpModule (axios): default timeout is 0 = wait forever ❌ → HttpModule.register({ timeout: 3000 })
```

| Fix | When |
|-----|------|
| Timeouts (connect ~1 s, read 2–5 s) | Always |
| Cache (Redis) | Rates/serviceability by pincode that change rarely |
| Parallelise | Independent calls (`CompletableFuture.allOf`, `Promise.all`) |
| Async via queue | Shipment creation, emails, invoices — don't block checkout |
| Circuit breaker + bulkhead | Dependency often slow/down (see Circuit Breaker notes) |

**Interview one-liner:** Every outbound call gets a connect and read timeout; then I cache, parallelise or move slow calls to a queue, and wrap flaky dependencies in a circuit breaker.

---

## 8. How I'd actually debug a slow API in production

1. **Scope it** — Grafana p95/p99 per route: one route or all? Since a deploy, a traffic spike, or a time of day?
2. **All routes slow?** → shared resource: DB CPU, Hikari pending, GC pauses (`jvm_gc_pause_seconds_max`), pod CPU throttling.
3. **One route slow?** → open a slow trace in Tempo/Datadog, find the fattest span.
4. **Span is SQL** → `pg_stat_statements` + `EXPLAIN ANALYZE` → index / rewrite. Many tiny identical SQL spans → N+1.
5. **Span is `getConnection`** → pool exhaustion → leak detection, long transactions, OSIV, `pg_stat_activity`.
6. **Span is HTTP to a partner** → timeouts, cache, async, breaker.
7. **No span stands out** → app CPU: thread dump (`jcmd <pid> Thread.print`) or async-profiler/JFR flame graph.
8. **Fix one thing, re-measure**, load-test with **k6**, add an alert on the route's p95 SLO.

| Interviewer asks | Strong answer |
|------------------|---------------|
| "API is slow, where do you start?" | p95/p99 per route → trace → slowest span; don't guess |
| "How do you find a missing index?" | `EXPLAIN ANALYZE`: Seq Scan + many rows removed by filter |
| "What is N+1 and how do you fix it?" | 1 + N queries; JOIN FETCH / `@EntityGraph` / batch size / DataLoader |
| "JOIN FETCH with pagination?" | Collection fetch paginates in memory (HHH90003004) → batch size or two-step id query |
| "Connection is not available — cause?" | Pool saturated: leaks, long transactions, remote calls in `@Transactional`, OSIV |
| "Just increase pool size?" | No — DB has limited cores; pods × pool must fit `max_connections` |
| "Fast locally, slow in prod?" | Data volume (missing index), N+1 at scale, network latency, pool contention |

**Interview one-liner:** I scope with per-route percentiles, find the slow span in a trace, then branch: SQL → EXPLAIN and indexes, many small queries → N+1, connection wait → pool, partner call → timeouts — fix one thing and re-measure.

---

## Quick Revision Cheat Sheet

- **p95/p99 per route**: `histogram_quantile` over route-templated buckets; averages hide the tail.
- **Tracing**: OTel agent → Tempo/Jaeger/Datadog; Metrics = WHEN, Traces = WHERE, Logs = WHY.
- **Slow query log / pg_stat_statements**: sort by **total** time to catch both slow and chatty queries.
- **EXPLAIN ANALYZE**: Seq Scan + rows removed → composite index (equality cols, then sort), `CONCURRENTLY`; it executes the query.
- **N+1**: detect with Hibernate stats / TypeORM logging; fix with JOIN FETCH, `@EntityGraph`, batch size, per-query relations, DataLoader.
- **Pool exhaustion**: Hikari active/pending, `leak-detection-threshold`, no remote calls in `@Transactional`, OSIV off, size = (cores × 2) + spindles across all pods.
- **Downstream**: connect + read timeouts always; cache, parallelise, queue, circuit breaker.
- **Prod flow**: scope → all-routes vs one-route → slowest span → branch (SQL / N+1 / pool / partner / CPU) → fix one → re-measure → alert.
