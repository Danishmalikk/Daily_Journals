# System Design Notes — Caching Strategies (Cache-aside, Write-through, Write-back, Eviction, Stampede)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is Caching & Where Does It Live?

*Definition:* **Caching** means keeping a copy of frequently used data in a **faster storage layer** (memory) so future requests don't hit the slow source (database, external API) every time.

Real life: You keep your water bottle on your desk instead of walking to the kitchen every time you're thirsty. The kitchen is the database; the desk is the cache.

```
Browser cache → CDN (CloudFront/Cloudflare) → Nginx proxy_cache → App in-memory (node-cache/Caffeine)
     → Distributed cache (Redis/Memcached) → Database (Postgres/MySQL)
   fastest, closest to user  ─────────────────────────────►  slowest, source of truth
```

| Layer | Example | Scope | Typical latency |
|-------|---------|-------|-----------------|
| Client | Browser, `Cache-Control` | One user | ~0 ms |
| CDN | CloudFront, Cloudflare | Global edge | 10-50 ms |
| Reverse proxy | Nginx `proxy_cache` | All users of that server | ~1 ms |
| In-process | Caffeine (Java), `lru-cache` (Node) | One app instance | < 0.1 ms |
| Distributed | Redis, Memcached, ElastiCache | All app instances | ~0.5-1 ms |
| Database | Postgres buffer cache | DB | 5-50 ms per query |

- **Cache hit** = found in cache. **Cache miss** = go to the source.
- **Hit ratio** = hits / (hits + misses). Aim for > 90% on hot data; track it in Grafana.

> ⚠️ "There are only two hard things in computer science: cache invalidation and naming things." Every cache is a **copy**, and copies go stale.

**Interview one-liner:** Caching stores hot data in a faster layer (CDN, in-process, Redis) to cut latency and DB load, at the cost of managing staleness.

---

## 2. Read Strategies — Cache-aside vs Read-through

*Definition:* **Cache-aside (lazy loading)** — the **application** checks the cache, and on a miss reads the DB and fills the cache itself. **Read-through** — the app talks only to the cache, and the **cache library/provider** loads from the DB on a miss.

```
Cache-aside:                               Read-through:
App ──get──► Redis                         App ──get──► Cache ──miss──► DB
 │  miss                                                 ▲   (cache loads it)
 └──► DB ──► App ──set──► Redis                          └── returns value
```

```typescript
// ✅ Cache-aside in NestJS with ioredis
async getProduct(id: string): Promise<Product> {
  const key = `product:${id}`;
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);                 // HIT

  const product = await this.productRepo.findOneByOrFail({ id }); // MISS → DB
  await this.redis.set(key, JSON.stringify(product), 'EX', 300);  // TTL 5 min
  return product;
}
```

| | Cache-aside | Read-through |
|--|-------------|--------------|
| Who loads on miss | Application code | Cache provider / library |
| Coupling | App knows both cache & DB | App sees only the cache |
| Cache down? | App falls back to DB | App may fail |
| Examples | Redis + manual code (most common) | Spring `@Cacheable`, Caffeine `LoadingCache`, DAX for DynamoDB |

**Easy rule of thumb:** Default to **cache-aside** with Redis — simple, resilient, only caches data that's actually requested.

**Interview one-liner:** Cache-aside means the app loads the DB on a miss and fills the cache; read-through hides that inside the cache layer — cache-aside is the most common default.

---

## 3. Write Strategies — Write-through, Write-back, Write-around

*Definition:*
- **Write-through** — write goes to the **cache and DB together** (synchronously) before returning.
- **Write-back (write-behind)** — write goes to the **cache only**; the cache flushes to the DB **later, asynchronously**, often in batches.
- **Write-around** — write goes **directly to the DB**, skipping the cache; the cache is filled only on a later read (usually paired with cache-aside + invalidation).

```
Write-through:  App ──► Cache ──► DB  ──► ack      (both updated before ack)
Write-back:     App ──► Cache ──► ack ... later ──► DB (batched)
Write-around:   App ──────────────► DB ──► ack     (cache entry deleted / not written)
```

| Strategy | Write latency | Read after write | Data-loss risk | Best for |
|----------|---------------|------------------|----------------|----------|
| **Write-through** | Higher (2 writes) | Always fresh in cache | Low | Read-heavy data that's read right after write (user profile) |
| **Write-back** | Lowest | Fresh in cache | **High** if cache dies before flush | Write-heavy counters, view counts, metrics, IoT |
| **Write-around** | DB-only | First read is a miss | Low | Write-once, read-rarely data (logs, order history) |

Memory trick: **Through = Together**, **Back = Batch later**, **Around = Avoid the cache**.

> ⚠️ Write-back with Redis means Redis holds the only copy for a while. Use it only for data you can afford to lose (view counts), or enable AOF and flush often.

**Interview one-liner:** Write-through updates cache and DB together (safe, slower writes), write-back writes cache first and flushes later (fast, risky), and write-around skips the cache for data rarely re-read.

---

## 4. Invalidation — keeping the cache honest

*Definition:* **Cache invalidation** is how you remove or update stale entries when the source data changes.

| Technique | How | Trade-off |
|-----------|-----|-----------|
| **TTL (expiry)** | `SET key val EX 300` | Simple; stale for up to TTL |
| **Delete on write** | Update DB → `DEL product:42` | Fresh; next read is a miss |
| **Update on write** | Update DB → `SET product:42 new` | Race-prone with concurrent writers |
| **Event-based** | DB change → Kafka / Debezium CDC → consumers `DEL` keys | Works across services |
| **Versioned keys** | `product:42:v7` — bump version on change | No delete needed; old keys expire |

```typescript
// ✅ Update DB first, THEN delete cache (not set)
await this.productRepo.update(id, dto);
await this.redis.del(`product:${id}`);
```

> ⚠️ Prefer **"update DB, then delete cache"** over "update cache". Two concurrent writers doing `SET` can leave the older value in the cache. Even delete has a tiny race (a slow reader re-populates old data) — a short TTL as a backstop fixes it. Some teams add a **delayed second delete** (~500 ms later).

**Easy rule of thumb:** **Always set a TTL**, even when you invalidate explicitly. TTL is the safety net for bugs.

**Interview one-liner:** I invalidate by updating the DB and then deleting the cache key, always with a TTL as a backstop, and use CDC/events when other services own the data.

---

## 5. Eviction Policies — LRU, LFU & friends

*Definition:* **Eviction** decides which entry to throw out when the cache is **full**. (Expiry = time-based removal; eviction = memory-pressure removal.)

| Policy | Evicts | Good for | Weakness |
|--------|--------|----------|----------|
| **LRU** (Least Recently Used) | Item not accessed for longest | General workloads | One big scan can flush hot items |
| **LFU** (Least Frequently Used) | Item with fewest accesses | Stable hot set (top products) | Old popular items linger |
| **FIFO** | Oldest inserted | Simple queues | Ignores popularity |
| **TTL-based / volatile** | Soonest to expire | Mixed TTL data | Needs TTLs |
| **Random** | Any item | Cheap | Unpredictable |

```
LRU (capacity 3):  access A B C → [C B A]   access A → [A C B]   insert D → evict B → [D A C]
```

```conf
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru     # or allkeys-lfu, volatile-lru, volatile-ttl, noeviction
```

Memory trick: **LRU = "Last seen long ago"**, **LFU = "Least popular"**.

> Remember: Redis' default `maxmemory-policy` is **noeviction** — when full, writes fail with OOM errors. For a pure cache, set `allkeys-lru` or `allkeys-lfu`. Redis LRU/LFU are **approximated** (sampling), not exact.

**Interview one-liner:** Eviction kicks in when memory is full; LRU drops the least recently used, LFU the least frequently used — for Redis as a cache I set `allkeys-lru` or `allkeys-lfu`.

---

## 6. Cache Failure Modes — Stampede, Penetration, Avalanche

*Definition:* Three classic ways a cache fails and the DB gets crushed.

| Problem | What happens | Fixes |
|---------|--------------|-------|
| **Stampede** (thundering herd / dogpile) | One **hot key** expires → 1,000 requests miss at once → all hit DB | Mutex/lock on rebuild (`SET lock NX PX`), single-flight, early probabilistic refresh, stale-while-revalidate |
| **Penetration** | Requests for keys that **don't exist** (`product:-1`, bots) always miss → always hit DB | Cache **null** results with short TTL, **Bloom filter** in front, input validation |
| **Avalanche** | **Many keys expire at the same time** or Redis goes down → DB flooded | **TTL jitter** (`300 + random(0..60)`), Redis replication/Sentinel, circuit breaker + rate limit to DB, warm-up |

```
Stampede:     hot key expires → 💥 1000 misses → Postgres CPU 100%
Penetration:  GET /product/999999 (doesn't exist) × 10k → every one hits DB
Avalanche:    cache warmed at 10:00 with TTL 1h → 11:00 everything expires at once
```

```typescript
// ✅ Stampede guard: only one request rebuilds, others wait briefly / serve stale
const lock = await this.redis.set(`lock:${key}`, '1', 'PX', 5000, 'NX');
if (lock === 'OK') {
  const fresh = await this.loadFromDb(id);
  await this.redis.set(key, JSON.stringify(fresh), 'EX', 300 + Math.floor(Math.random() * 60)); // jitter
  await this.redis.del(`lock:${key}`); // simplified — production: token + Lua compare-and-delete (see 15_Redis.md)
  return fresh;
}
await sleep(50);              // someone else is rebuilding
return this.getProduct(id);   // retry (cap retries in real code)
```

Memory trick: **S-P-A** → **S**ingle hot key, **P**hantom keys, **A**ll at once.

**Interview one-liner:** Stampede = one hot key expires (fix with lock/single-flight), penetration = missing keys always miss (cache nulls, Bloom filter), avalanche = many keys expire together (TTL jitter, HA Redis).

---

## 7. When to Cache (and when not) — interview angle

### Good candidates
- Read-heavy, rarely changing: product catalog, categories, config, feature flags.
- Expensive computations: dashboards, aggregates, recommendation results.
- External API responses: ShipRocket serviceability/rates, currency rates (respect their TTLs).
- Sessions, rate-limit counters, OTPs.

### Bad candidates
- Highly volatile data needing strong consistency: wallet balance, final stock at checkout, payment status.
- Data read once (write-around instead).
- Per-user data with huge cardinality and low reuse.

| Scenario | Strategy |
|----------|----------|
| Product page, 95% reads | Cache-aside + TTL + delete on update |
| User profile read right after update | Write-through |
| Page view counters | Write-back (Redis `INCR`, flush to DB every minute) |
| Order history / audit logs | Write-around |
| Hot flash-sale item | Cache-aside + lock + TTL jitter + local in-process cache |

### What interviewers probe
- "How do you keep cache and DB consistent?" → update DB then delete key + TTL; CDC for cross-service.
- "What happens when Redis goes down?" → fall back to DB with circuit breaker/rate limit; Redis Sentinel/Cluster or ElastiCache Multi-AZ.
- "How do you measure it?" → hit ratio, p95 latency, evictions (`INFO stats`: `keyspace_hits`, `keyspace_misses`, `evicted_keys`) in Prometheus via `redis_exporter`.

**Interview one-liner:** I cache read-heavy, tolerate-stale data with cache-aside + TTL, avoid caching money and final stock, and monitor hit ratio and evictions to prove the cache is earning its keep.

---

## Quick Revision Cheat Sheet

- **Cache layers**: browser → CDN → Nginx → in-process → Redis → DB; hit ratio is the key metric.
- **Cache-aside**: app checks cache, loads DB on miss, fills cache — the default.
- **Read-through**: cache provider loads DB on miss (`@Cacheable`, LoadingCache).
- **Write-through**: cache + DB together; fresh, slower writes.
- **Write-back**: cache first, DB later in batches; fast, can lose data.
- **Write-around**: DB only; cache filled on later read.
- **Invalidation**: update DB → delete key; always TTL; CDC/events across services; versioned keys.
- **Eviction**: LRU (recent), LFU (frequent); Redis default `noeviction` → set `allkeys-lru/lfu`.
- **Stampede** → lock/single-flight; **Penetration** → cache nulls + Bloom filter; **Avalanche** → TTL jitter + HA.
- **Don't cache** balances, final stock, payment status without strong guarantees.
