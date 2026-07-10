# Scaling & Production Troubleshooting

> Format: **Restate the question** → **What I'd actually do (first → next → fix)** → **Interview one-liner**
> Grounded in the real stack: NestJS + TypeORM, PostgreSQL/MySQL, Redis, Docker, AWS, Nginx, Prometheus + Grafana.

---

## 1. Traffic suddenly jumps 100 → 10,000 users and the system is slow

**The mindset:** don't guess, don't randomly restart. **Measure first, then relieve the hottest constraint.** A 100× jump almost never means "add more servers" as step one — it usually means one shared resource (the DB) is now the choke point.

**First — confirm where the time is going (2 minutes, not 2 hours):**
- Look at **p95/p99 per route** in Grafana (`histogram_quantile(0.95, ...)`), not the average — averages hide the pain. Which endpoints got slow?
- Check the obvious resource meters: **CPU, memory, DB connections, DB CPU/IO**. In a NestJS app under load, the app servers are usually fine and the **database is pegged**.

**Next — relieve the pressure in the order that gives the most relief per minute of work:**
1. **Cache the hot reads.** The same product/catalog/config is being read 10,000× — put it in **Redis** with the **cache-aside** pattern (check cache → miss → DB → store). This alone often drops DB load 80%+ for read-heavy traffic.
2. **Fix the queries the traffic exposed.** A 100× jump turns a hidden **N+1** into 10,000 extra queries. Kill eager loading on list endpoints, load relations per-query, add the **missing index** on the columns you filter/sort by.
3. **Scale reads horizontally.** Add **read replicas** and route "browse" reads to them; keep writes on the primary.
4. **Scale the app tier.** Put app instances behind a **load balancer** and turn on **auto-scaling** (AWS ASG / K8s HPA) so more pods spin up with load. This is safe *because* the app is stateless — sessions live in Redis/JWT, not local memory.
5. **Shed and smooth load.** **Rate-limit** abusive clients, and push slow side-effects (emails, PDF/report generation, webhooks) onto a **message queue** so checkout doesn't wait on them.
6. **Put a CDN in front** of static assets/images so those 10,000 users don't all hit the origin.

**⚠️ Gotcha:** adding more app servers when the DB is the bottleneck makes things *worse* — every new instance opens more DB connections and finishes off the database. Always scale the constrained layer, not the comfortable one.

**Interview one-liner:**
> "First I look at p95 per route and resource meters to find the actual bottleneck — under a sudden 100× spike it's almost always the database. I relieve it in order: cache hot reads in Redis, fix the N+1/missing-index queries the load exposed, add read replicas, then autoscale the stateless app tier behind a load balancer and offload slow work to a queue."

---

## 2. A page crashed while thousands of users were actively using it

**The mindset:** **stop the bleeding first, root-cause second.** Restore service, then investigate — don't debug a live outage with users watching.

**Immediate (stop the bleeding):**
- **Roll back** to the last known-good deploy if the crash started right after a release — that's the single most likely cause, and rollback is faster than a hotfix.
- If it's one crashing instance, let the **load balancer health check** pull it out and route around it while the rest serve traffic.

**Investigate (in this order):**
1. **Read the logs / error tracker (Sentry).** Get the actual **stack trace** and the **first** occurrence timestamp — that timestamp usually points straight at the trigger (a deploy, a config change, a traffic threshold).
2. **Correlate with the timeline:** did it start at a deploy? A migration? A spike? A dependency (payment gateway, ShipRocket, third-party API) going down?
3. **Classify the crash:**
   - **Unhandled exception / promise rejection** → a code path that only fires under real data/scale (e.g. a `null` relation, a bad edge case).
   - **Out of memory** → a **memory leak** or loading too much into memory (unbounded query, no pagination) — the process gets OOM-killed under sustained load.
   - **Resource exhaustion** → **DB connection pool** drained, file handles, or a downstream timeout cascading into the app.
4. **Reproduce** against staging with production-like data/volume — a crash that only shows "at thousands of users" is almost always **data- or concurrency-dependent**, not random.

**Resolve & prevent recurrence:**
- Ship the targeted fix (guard the null, paginate the query, add the missing timeout, fix the pool size).
- Add the **guardrails** that would have contained it: global exception filter so one bad request returns 500 instead of killing the process, health checks + auto-restart, a **circuit breaker** on the flaky dependency, and an **alert** on the error-rate/p99 that would've paged you *before* users noticed.

**Interview one-liner:**
> "Stabilize first — roll back or let the load balancer route around the bad instance — then root-cause from the stack trace and the first-occurrence timestamp, correlating it with deploys and traffic. Crashes 'at thousands of users' are almost always data- or memory-related, not random, so I reproduce with production-like load, fix the specific path, and add the guardrail (exception filter, health check, circuit breaker, alert) that stops it recurring."

---

## 3. How I'd identify production bottlenecks

**The principle:** a bottleneck is the **one resource everything else waits on.** You find it with data, not intuition — measure end-to-end, then zoom into the slowest layer.

**Layer by layer, what to look at:**
- **Metrics (Prometheus + Grafana):** p95/p99 latency per route, request rate, error rate — the classic **RED** signals. Plus **USE** on the boxes: CPU, memory, disk IO, network **U**tilization / **S**aturation / **E**rrors. Whatever is saturated (100% CPU, connection pool full, disk IO maxed) is your bottleneck.
- **Distributed tracing / APM (Datadog, New Relic, Sentry Performance):** shows *where inside a request* the time goes — app code vs DB vs an external API call. This is what separates "the endpoint is slow" from "the endpoint is slow **because** of one query."
- **Database:** slow-query log, `EXPLAIN ANALYZE` on the worst offenders, **N+1** detection, missing indexes, lock contention, connection-pool saturation. In a CRUD/NestJS+TypeORM app, **this is the #1 source of bottlenecks.**
- **Cache:** hit/miss ratio — a low hit rate means load is falling through to the DB.
- **External dependencies:** payment gateway, ShipRocket, any third-party API — their latency becomes your latency unless you time-out and isolate it.

**Method:** don't optimize blind. **Measure → find the biggest wait → fix that one thing → re-measure.** Fixing anything other than the current bottleneck moves zero needles (Amdahl's law in practice). Load tools like **k6 / autocannon** let you push synthetic load and watch which resource tips over first.

**Interview one-liner:**
> "I identify bottlenecks with data, not guesses: RED metrics (rate/errors/latency) and USE metrics (utilization/saturation/errors) in Grafana to find the saturated resource, then APM/tracing to see where inside the request the time goes. In our stack it's usually the database — slow queries, N+1s, missing indexes — so I confirm with EXPLAIN, fix the single biggest wait, and re-measure."

---

## 4. Strategies to improve scalability, performance, and reliability

**Scalability — handle more load without a rewrite:**
- **Stateless app servers** (sessions in Redis/JWT) so you can add instances freely.
- **Horizontal scaling + load balancer + autoscaling** (AWS ASG / K8s HPA) instead of buying one giant box.
- **Database scaling:** read replicas for read-heavy traffic; sharding/partitioning when a single DB can't hold or serve the data.
- **Async + message queues** (Kafka / SQS / RabbitMQ) to decouple services and absorb spikes instead of crashing under them.

**Performance — make each request faster/cheaper:**
- **Caching** at every sensible layer: Redis for DB results, CDN for static assets, HTTP caching for responses.
- **Query optimization:** proper indexes, kill N+1s, paginate, select only needed columns, denormalize hot read paths.
- **Connection pooling** and sensible timeouts so slow dependencies don't tie up resources.

**Reliability — stay up and recover gracefully:**
- **Redundancy / no single point of failure:** multiple instances, multi-AZ, DB failover/replication.
- **Graceful degradation:** circuit breakers, timeouts, and retries **with idempotency keys** so retries are safe and one failing dependency doesn't take the whole system down.
- **Observability + alerting:** metrics, logs, traces, and alerts on p99/error-rate so you know *before* users do.
- **Health checks + auto-restart + rollbacks** so a bad instance or deploy self-heals.

**⚠️ The honest trade-off:** these pull against each other — strong consistency vs availability (**CAP**), caching vs freshness, more nodes vs more operational complexity. The skill is choosing **per feature**: a bank ledger wants consistency (CP); a like-count wants availability (AP).

**Interview one-liner:**
> "Scalability comes from stateless services scaling horizontally behind a load balancer with the DB scaled via replicas/sharding and queues absorbing spikes. Performance comes from caching, query/index optimization, and pooling. Reliability comes from redundancy, circuit breakers with idempotent retries, and observability with alerting. The real skill is making the trade-offs consciously per feature — consistency vs availability, freshness vs speed — instead of chasing all three blindly."
