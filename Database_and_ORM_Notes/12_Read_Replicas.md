# Database Notes — Read Replicas

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem: too many reads

Most apps read data **far more** than they write (think: 100 reads per 1 write — browsing products, viewing profiles). If one database server handles *all* reads **and** writes, it becomes the bottleneck.

**The fix:** make **copies** of the database that handle reads, freeing the main server to focus on writes.

*Definition:* A **read replica** is a **copy** of your primary database that stays in sync with it and serves **read-only** queries. Writes go to the primary; reads are spread across the replicas — so the system handles far more total traffic.

Real life: One author (the primary) writes a book; printers make many copies (replicas) so thousands can read at once without bothering the author.

---

## 2. Primary–Replica architecture

```
                 WRITES ──► ┌───────────┐
                            │  PRIMARY  │  (handles all writes)
                            └─────┬─────┘
                  replication     │ (copies changes out)
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
          ┌──────────┐     ┌──────────┐     ┌──────────┐
          │ Replica1 │     │ Replica2 │     │ Replica3 │  (handle reads)
          └──────────┘     └──────────┘     └──────────┘
                ▲                 ▲                 ▲
                └──────── READS spread across ──────┘
```

- **Primary** (a.k.a. master/leader): accepts **writes** (INSERT/UPDATE/DELETE) and also can read.
- **Replicas** (a.k.a. read-replicas/followers): **read-only** copies; the primary streams its changes to them.

> Terminology note: the modern terms are **primary/replica** (older docs say master/slave).

**Interview one-liner:** A read replica is a synced read-only copy of the primary; writes go to the primary, reads are distributed to replicas to scale read-heavy workloads.

---

## 3. How replication works (and the key gotcha)

*Definition:* **Replication** is the process of copying changes from the primary to the replicas so they stay up to date. The primary records its changes (in a log — MySQL's **binlog**, Postgres's **WAL**) and ships them to replicas, which replay them.

### The crucial concept: replication lag
*Definition:* **Replication lag** is the small delay between a write hitting the primary and that change appearing on a replica. During this gap, a replica may return **slightly stale (old) data**.

```
1. User updates profile → PRIMARY (done instantly)
2. Replica hasn't received the change yet (lag of e.g. 50ms–seconds)
3. User immediately reads from a REPLICA → sees OLD profile  😬
```

This makes replicas **eventually consistent** — they catch up, but not instantly.

> **The classic interview problem ("read-your-own-writes"):** a user saves data then immediately reads stale data from a lagging replica. Fixes: route that user's reads to the **primary** for a short window, or read from primary right after a write.

**Interview one-liner:** Replication copies primary changes to replicas with a small **lag**, so replicas can serve stale data (eventual consistency) — watch the read-your-own-writes problem.

---

## 4. Sync vs Async replication

| | Asynchronous (common) | Synchronous |
|--|------------------------|-------------|
| Primary waits for replicas? | ❌ No — commits immediately | ✅ Yes — waits for replica confirm |
| Speed | Fast writes | Slower writes |
| Lag / stale reads | Possible | None (or minimal) |
| Risk | Small data loss if primary dies before replica syncs | Safer, but slower |

Most setups use **async** (speed) and accept small lag. Some critical systems use **semi-sync** (wait for at least one replica) as a balance.

---

## 5. Read/write splitting (how the app uses replicas)

The app must send writes to the primary and reads to replicas. This **read/write splitting** can be done by:
- The application / ORM (route based on query type).
- A middleware/proxy (e.g. ProxySQL, PgBouncer, AWS RDS Proxy).
- Framework support.

In Spring you can configure **multiple datasources** and route by transaction type:
```java
@Transactional(readOnly = true)   // hint → route to a read replica
public List<Product> getProducts() { ... }

@Transactional                    // writes → primary
public void createProduct(Product p) { ... }
```
A common pattern uses `AbstractRoutingDataSource` to pick primary vs replica based on whether the transaction is read-only.

**Interview one-liner:** Read/write splitting routes writes to the primary and reads to replicas — done in the app (e.g. `@Transactional(readOnly=true)`) or via a proxy.

---

## 6. Replicas vs Sharding (don't confuse them!)

A very common interview distinction:

| | Read Replicas | Sharding |
|--|---------------|----------|
| What's copied/split | **Full copy** of all data on each replica | Data **split** into pieces (each shard has part) |
| Scales | **Reads** | **Reads + writes + storage** |
| Each node has | Everything (a duplicate) | Only its slice |
| Complexity | Lower | Much higher |
| Helps with writes? | ❌ No (writes still one primary) | ✅ Yes |

```
Replicas  = same data, many copies   → scale READS
Sharding  = different data per node   → scale WRITES + storage
```

> Key point: **read replicas don't scale writes** (all writes still hit the single primary). For write scaling, you need sharding. They're often used **together**.

**Interview one-liner:** Read replicas are full copies that scale reads (writes still hit one primary); sharding splits data to scale writes and storage. Replicas duplicate, shards divide.

---

## 7. Benefits & costs

| 👍 Benefits | 👎 Costs |
|------------|---------|
| Scale read-heavy traffic | Replication lag → stale reads |
| Offload reporting/analytics to a replica | Extra servers = more cost |
| **High availability** — promote a replica if primary dies (failover) | Doesn't scale writes |
| Geographic replicas → lower latency for distant users | Eventual consistency complexity |

> Bonus use: replicas double as **failover** — if the primary crashes, a replica can be **promoted** to become the new primary (high availability/disaster recovery).

---

## Quick Revision Cheat Sheet

- **Read replica** = synced read-only **copy** of the primary; writes → primary, reads → replicas.
- Scales **read-heavy** workloads (most apps read >> write).
- **Replication** ships primary's changes (binlog/WAL) to replicas; **replication lag** → stale reads (eventual consistency).
- **Read-your-own-writes** problem: read from primary right after a write to avoid stale data.
- **Async** (fast, small lag) vs **sync** (safe, slower) replication.
- **Read/write splitting**: `@Transactional(readOnly=true)` → replica; writes → primary; or via a proxy.
- **Replicas ≠ Sharding**: replicas = full copies (scale reads); sharding = split data (scale writes). Replicas don't scale writes.
- Replicas also enable **failover** (promote a replica = high availability).
