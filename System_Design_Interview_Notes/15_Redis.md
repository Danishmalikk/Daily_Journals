# System Design Notes — Redis (Data Structures, TTL, Persistence, Pub/Sub, Distributed Lock)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is Redis?

*Definition:* **Redis** (REmote DIctionary Server) is an **in-memory key-value data store** that supports rich data structures. It's used as a cache, session store, message broker, rate limiter, leaderboard and lock service.

Real life: A whiteboard next to your desk — super fast to read and write, but if the power goes out you need a photo (persistence) to restore it.

**Why Redis is fast:**
- Data lives in **RAM** (sub-millisecond reads/writes).
- **Single-threaded command execution** → no locks, no context switching; each command is **atomic**. (Redis 6+ uses extra threads for network I/O only.)
- Event loop with I/O multiplexing (epoll) handles thousands of connections.
- Simple, efficient encodings (listpack, intset, skiplist).

```
NestJS / Spring Boot instances ──► Redis (primary) ──async replication──► replicas
                                    │
                                    ├─ cache (product:42)
                                    ├─ sessions (sess:abc)
                                    ├─ rate limits (rl:ip:1.2.3.4)
                                    └─ locks (lock:order:123)
```

> ⚠️ Single-threaded means **one slow command blocks everyone**. Never run `KEYS *` in production (use `SCAN`), and avoid huge `LRANGE 0 -1` / `HGETALL` on giant keys.

**Interview one-liner:** Redis is an in-memory, single-threaded data-structure store — fast because it's RAM-based with atomic commands, and used for caching, sessions, rate limiting, queues and locks.

---

## 2. Data Structures & What Each Is For

| Type | Think of it as | Key commands | Real use case |
|------|----------------|--------------|---------------|
| **String** | Value / counter / blob (≤ 512 MB) | `SET`, `GET`, `INCR`, `SETEX` | Cache JSON, page views, OTP |
| **Hash** | Object with fields | `HSET`, `HGET`, `HINCRBY` | User profile, cart (`cart:u1 → sku:qty`) |
| **List** | Linked list / queue | `LPUSH`, `RPOP`, `BLPOP`, `LRANGE` | Simple job queue, recent activity |
| **Set** | Unique unordered items | `SADD`, `SISMEMBER`, `SINTER` | Tags, unique visitors, "already processed" IDs |
| **Sorted Set (ZSET)** | Unique items + score, ordered | `ZADD`, `ZRANGE`, `ZREVRANGE`, `ZINCRBY` | Leaderboards, sliding-window rate limit, delayed jobs |
| **Bitmap** | Bits in a string | `SETBIT`, `BITCOUNT` | Daily active users, feature flags per user |
| **HyperLogLog** | Approx. unique count (~0.81% error, 12 KB) | `PFADD`, `PFCOUNT` | Unique visitors per page |
| **Geo** | Lat/long in a ZSET | `GEOADD`, `GEOSEARCH` | Nearby stores / delivery partners |
| **Stream** | Append-only log with consumer groups | `XADD`, `XREADGROUP`, `XACK` | Event queue, order events |

```bash
# Cart as a Hash
HSET cart:user:7 sku:TSHIRT-M 2 sku:MUG 1
HINCRBY cart:user:7 sku:MUG 1          # mug qty → 2
HGETALL cart:user:7

# Leaderboard as a Sorted Set
ZINCRBY leaderboard 50 "danish"
ZREVRANGE leaderboard 0 9 WITHSCORES   # top 10
```

Memory trick: **"SHLSZ + BHGS"** → **S**tring, **H**ash, **L**ist, **S**et, **Z**set — plus **B**itmap, **H**yperLogLog, **G**eo, **S**tream.

**Interview one-liner:** Redis isn't just strings — Hashes for objects, Lists for queues, Sets for uniqueness, Sorted Sets for rankings and time windows, plus Bitmaps, HyperLogLog, Geo and Streams.

---

## 3. TTL, Expiry & Eviction

*Definition:* **TTL (time to live)** makes a key auto-delete after a set time. **Eviction** removes keys when Redis hits `maxmemory`.

```bash
SET otp:9876543210 482913 EX 300 NX   # OTP valid 5 min, only if not already set
EXPIRE session:abc 1800               # 30 min
TTL session:abc                       # seconds left (-1 = no TTL, -2 = key missing)
PERSIST session:abc                   # remove TTL
```

**How expiry works internally:**
- **Lazy (passive)** — key checked & deleted when accessed.
- **Active** — Redis periodically samples keys with TTLs and deletes expired ones.

| `maxmemory-policy` | Evicts from | When to use |
|--------------------|-------------|-------------|
| `noeviction` (default) | Nothing → writes return OOM error | Redis as primary store / queue |
| `allkeys-lru` | Any key, least recently used | Pure cache (most common) |
| `allkeys-lfu` | Any key, least frequently used | Cache with stable hot set |
| `volatile-lru` / `volatile-lfu` | Only keys with TTL | Mixed cache + persistent keys |
| `volatile-ttl` | Keys closest to expiry | TTL-driven data |
| `allkeys-random` / `volatile-random` | Random | Rarely |

> ⚠️ `SET key value` **without** `EX/KEEPTTL` **removes** any existing TTL on that key. A cache overwrite can silently turn into a forever key.

**Interview one-liner:** TTL auto-expires keys (lazily on access and actively by sampling), while eviction policy decides what to drop at maxmemory — I use `allkeys-lru` for caches and `noeviction` for queues.

---

## 4. Persistence — RDB vs AOF

*Definition:* **RDB** takes point-in-time **snapshots** of memory to disk. **AOF (Append Only File)** logs **every write command** so it can be replayed on restart.

```
RDB:  memory ──fork()──► child writes dump.rdb every N min    (compact, fast restart)
AOF:  every write ──► appendonly.aof (fsync: always / everysec / no)  (durable, bigger)
```

| | **RDB** | **AOF** |
|--|---------|---------|
| What's stored | Snapshot of data | Log of write commands |
| Data loss on crash | Minutes (since last snapshot) | ~1 sec with `everysec` |
| File size | Small, compact | Larger (rewritten/compacted by `BGREWRITEAOF`) |
| Restart speed | Fast | Slower (replay) |
| Good for | Backups, DR, fast restarts | Durability |

```conf
# redis.conf — common production combo
save 900 1 300 10 60 10000     # RDB: snapshot if 1 change in 900s, 10 in 300s, ...
appendonly yes
appendfsync everysec           # good balance: ≤ ~1 sec of loss
aof-use-rdb-preamble yes       # hybrid: RDB base + AOF tail (Redis 4+)
```

> Remember: `fork()` for RDB/AOF rewrite uses copy-on-write — on a write-heavy 20 GB instance, memory can spike. Leave headroom (don't run Redis at 95% RAM).

**Easy rule of thumb:** Pure cache → persistence optional (RDB or none). Sessions/queues/locks you care about → **AOF everysec + RDB**. Managed: AWS ElastiCache / MemoryDB (MemoryDB is durable via a multi-AZ transaction log).

**Interview one-liner:** RDB snapshots are compact and fast to restore but lose minutes of data; AOF logs every write and loses about a second with everysec — production usually runs both.

---

## 5. Pub/Sub vs Streams

*Definition:* **Pub/Sub** broadcasts a message to **whoever is subscribed right now** (fire-and-forget). **Streams** are a **persistent, append-only log** with IDs, consumer groups and acknowledgements — like a mini Kafka.

```
Pub/Sub:  PUBLISH orders "o-1" ──► [sub A ✅] [sub B ✅] [sub C offline ❌ lost forever]

Streams:  XADD orders * id o-1 ──► stream (stored)
            consumer group "billing": worker1, worker2  (each msg to ONE worker)
            XACK after processing; unacked → pending list → can be reclaimed (XAUTOCLAIM)
```

```bash
# Streams with a consumer group
XADD orders * orderId o-123 status PAID
XGROUP CREATE orders billing $ MKSTREAM
XREADGROUP GROUP billing worker-1 COUNT 10 BLOCK 5000 STREAMS orders >
XACK orders billing 1730450000000-0
```

| | **Pub/Sub** | **Streams** | **Kafka** |
|--|------------|-------------|-----------|
| Persistence | ❌ None | ✅ In Redis (memory + AOF/RDB) | ✅ Disk, long retention |
| Offline consumer | Misses messages | Catches up | Catches up |
| Acks / redelivery | ❌ | ✅ `XACK`, pending list | ✅ offsets |
| Consumer groups | ❌ (all get all) | ✅ | ✅ |
| Scale | Small | Medium (bounded by RAM) | Huge |
| Use for | Live notifications, cache invalidation broadcast, Socket.IO adapter | Light job/event queues | Event backbone, analytics, high throughput |

> ⚠️ Don't use Pub/Sub for anything you can't lose (orders, payments). Use Streams, Kafka, RabbitMQ or SQS. For Node job queues, **BullMQ** (on Redis) is the common choice.

**Interview one-liner:** Pub/Sub is fire-and-forget broadcast with no persistence; Streams add a stored log, consumer groups and acks — use Pub/Sub for live signals and Streams or Kafka for events you can't lose.

---

## 6. Distributed Lock — SET NX PX, Redlock, Redisson

*Definition:* A **distributed lock** ensures only **one process across many servers** does a critical task at a time (e.g. only one instance runs the nightly Razorpay reconciliation, or only one request processes order #123).

Real life: A single bathroom key at a café — whoever holds the key goes in; it has a timer so someone who walks off with it doesn't lock everyone out forever.

```bash
# Acquire: set only if Not eXists, auto-expire in 30s, value = unique token
SET lock:order:123 "a1b2-uuid" NX PX 30000     # → OK (got it) or nil (someone else has it)
```

```lua
-- Release safely: delete ONLY if I still own it (atomic via Lua)
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

```
❌ SETNX lock 1  then  EXPIRE lock 30   → crash between them = lock forever
❌ DEL lock (without token check)       → you may delete someone else's lock after yours expired
✅ SET key token NX PX ttl  +  Lua compare-and-delete
```

### Redlock
- Algorithm by Redis' creator for **N independent masters (usually 5)**: acquire the lock on a **majority (≥3)** within a time bound; lock is valid for TTL minus elapsed time.
- Protects against a single Redis node failing (a single-instance lock can be lost on async-replica failover).
- ⚠️ Criticized (Martin Kleppmann) — relies on bounded clock drift and pauses; a long GC pause can let two clients think they hold the lock. For **correctness-critical** locks, use **fencing tokens** (monotonic number checked by the storage) or a CP store (etcd, ZooKeeper, Postgres advisory locks).

### Redisson (Java) — production-ready locks
```java
RLock lock = redisson.getLock("lock:order:" + orderId);
if (lock.tryLock(5, TimeUnit.SECONDS)) {       // wait up to 5s; no leaseTime → watchdog
    try {
        processOrder(orderId);
    } finally {
        lock.unlock();
    }
}
// Watchdog auto-extends the lock (default 30s) while the thread is alive.
// Also: fair locks, read/write locks, semaphores. Node.js equivalent: `redlock` npm package.
```

| Option | Safety | Complexity | Use when |
|--------|--------|------------|----------|
| Single Redis `SET NX PX` | Efficiency-grade | Low | Avoid duplicate work (cron, cache rebuild) |
| Redlock (5 masters) | Better, still debated | High | Need tolerance to node failure |
| Redisson / `redlock` lib | Same as above + watchdog | Low (library) | Java / Node apps |
| etcd / ZooKeeper / Postgres advisory lock + fencing | Correctness-grade | Medium | Money, inventory, leader election |

**Interview one-liner:** I acquire a Redis lock with `SET key token NX PX ttl` and release with a Lua compare-and-delete; Redisson adds a watchdog, and for correctness-critical work I add fencing tokens or use etcd/ZooKeeper.

---

## 7. Real Use Cases & Scaling (what interviewers probe)

| Use case | Redis feature |
|----------|---------------|
| Product/page cache | String + TTL, `allkeys-lru` |
| Sessions (NestJS `express-session`, Spring Session) | String/Hash + TTL |
| Rate limiting | `INCR` + `EXPIRE` (fixed window) or ZSET (sliding window) |
| OTP / idempotency keys | `SET NX EX` |
| Leaderboard / trending products | Sorted Set |
| Job queues | BullMQ (Node), Streams, Lists |
| Real-time notifications / Socket.IO scaling | Pub/Sub |
| Distributed lock | `SET NX PX`, Redisson |

```typescript
// Fixed-window rate limit: 100 req/min per IP (NestJS guard)
const key = `rl:${ip}:${Math.floor(Date.now() / 60000)}`;
const count = await this.redis.incr(key);
if (count === 1) await this.redis.expire(key, 60);
if (count > 100) throw new HttpException('Too Many Requests', 429);
```

### High availability & scaling
| Setup | What it gives |
|-------|---------------|
| **Replication** | Primary + async replicas; read scaling |
| **Sentinel** | Monitoring + automatic failover for primary/replica |
| **Cluster** | Sharding over **16384 hash slots**, multiple primaries, built-in failover |
| **Managed** | AWS ElastiCache, MemoryDB, Redis Cloud, Upstash |

- **Multi-key ops in Cluster** need keys in the same slot → use **hash tags**: `{user:7}:cart`, `{user:7}:profile`.
- **Transactions**: `MULTI/EXEC` queues commands atomically (no rollback); `WATCH` for optimistic locking; **Lua scripts** for atomic read-modify-write.
- **Pipelining** batches commands to cut round trips.
- **Watch for**: big keys, hot keys, `KEYS *`, missing TTLs, memory fragmentation. Monitor with `redis_exporter` → Prometheus → Grafana (`used_memory`, hit ratio, `evicted_keys`, `connected_clients`).

> ⚠️ Replication is **async** — a failover can lose recently acknowledged writes. Redis is great for speed, but your source of truth for orders and payments stays in Postgres/MySQL.

**Interview one-liner:** I use Redis for caching, sessions, rate limiting, queues and locks; scale with Sentinel for HA or Cluster for sharding (16384 slots); and keep the real source of truth in the database because replication is async.

---

## Quick Revision Cheat Sheet

- **Redis** = in-memory, single-threaded (commands atomic), sub-ms; never `KEYS *` in prod, use `SCAN`.
- **Data structures**: String, Hash, List, Set, Sorted Set + Bitmap, HyperLogLog, Geo, Stream.
- **TTL**: `EX/PX`, `TTL`, `PERSIST`; expiry is lazy + active; plain `SET` clears TTL.
- **Eviction**: default `noeviction`; caches use `allkeys-lru` / `allkeys-lfu`.
- **Persistence**: RDB = snapshots (fast restart, lose minutes); AOF = write log (`everysec` ≈ 1s loss); use both.
- **Pub/Sub** = fire-and-forget broadcast; **Streams** = persisted log + consumer groups + `XACK`.
- **Lock**: `SET key token NX PX ttl` + Lua compare-and-delete; Redlock = majority of 5 masters (debated); Redisson = watchdog; fencing/etcd for correctness.
- **Scaling**: replication, Sentinel (failover), Cluster (16384 slots, hash tags); async replication → can lose writes.
- **Use cases**: cache, sessions, rate limiting, OTP/idempotency, leaderboards, BullMQ queues, Socket.IO.
