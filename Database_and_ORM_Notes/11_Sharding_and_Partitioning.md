# Database Notes — Database Sharding & Partitioning

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem: one table/DB gets too big

When a table grows to billions of rows, or one database server can't handle the load, queries slow down and the server runs out of resources. The solution: **split the data into smaller pieces**. Two ways to split:

- **Partitioning** → split a table into pieces **within the same database/server**.
- **Sharding** → split data across **multiple separate database servers**.

Memory trick: **Partitioning = split inside one house into rooms. Sharding = move into several separate houses.**

---

## 2. Partitioning

*Definition:* **Partitioning** breaks one large table into smaller sub-tables called **partitions**, based on a column's value — but they all still live in the **same database**. The database routes queries to the right partition automatically, so you scan less data.

```
   orders (huge table)  →  partitioned by year
   ┌───────────────┬───────────────┬───────────────┐
   │ orders_2024   │ orders_2025   │ orders_2026   │   (same DB server)
   └───────────────┴───────────────┴───────────────┘
   Query for 2026 → DB only scans orders_2026 (partition pruning)
```

### Types of partitioning
| Type | Splits by | Example |
|------|-----------|---------|
| **Range** | Value ranges | By date: 2024, 2025, 2026 |
| **List** | Discrete values | By region: 'US', 'EU', 'ASIA' |
| **Hash** | Hash of a column | Even spread across N partitions |
| **Key** | DB-managed hash | Like hash, DB picks the function |

```sql
-- MySQL range partitioning example
CREATE TABLE orders (id INT, created DATE, amount DECIMAL)
PARTITION BY RANGE (YEAR(created)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027)
);
```

> The big win is **partition pruning**: a query filtered on the partition column only scans the relevant partition(s), not the whole table.

**Interview one-liner:** Partitioning splits one big table into smaller partitions (by range, list, or hash) within the same database, so queries scan only relevant partitions (partition pruning).

---

## 3. Vertical vs Horizontal partitioning

*Definition:*
- **Horizontal partitioning** → split by **rows** (each partition has the same columns, different rows). This is the common one (and what sharding builds on).
- **Vertical partitioning** → split by **columns** (move rarely-used or large columns into a separate table).

```
Horizontal (by rows):           Vertical (by columns):
users_A: rows 1–1M              users_core:  id, name, email
users_B: rows 1M–2M            users_extra: id, bio, profile_pic (big/rare)
```

**Interview one-liner:** Horizontal partitioning splits rows (same schema, different rows); vertical partitioning splits columns (separate hot vs cold columns).

---

## 4. Sharding

*Definition:* **Sharding** is **horizontal partitioning across multiple database servers**. Each server (a **shard**) holds a subset of the data. Together the shards form the full dataset. This spreads both storage and load across machines — the main way to scale writes beyond one server.

```
                       app
                        │  picks shard by a "shard key" (e.g. user_id)
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐     ┌─────────┐     ┌─────────┐
   │ Shard 1 │     │ Shard 2 │     │ Shard 3 │   ← separate DB servers
   │ users   │     │ users   │     │ users   │
   │ 1–1M    │     │ 1M–2M   │     │ 2M–3M   │
   └─────────┘     └─────────┘     └─────────┘
```

*Definition:* The **shard key** is the column used to decide which shard a row lives on (e.g. `user_id`). Choosing it well is the hardest, most important sharding decision.

### Sharding strategies
| Strategy | How shard is chosen | Watch out for |
|----------|--------------------|---------------|
| **Range-based** | By key ranges (users 1–1M → shard 1) | Uneven load ("hot" shards) |
| **Hash-based** | `hash(key) % N` → shard | Even spread, but resharding is hard |
| **Directory-based** | A lookup table maps key → shard | Flexible, but the lookup is a bottleneck |

**Interview one-liner:** Sharding spreads data across multiple DB servers (shards), each holding part of the data, chosen by a shard key — it scales writes and storage beyond a single machine.

---

## 5. Partitioning vs Sharding (the money table)

| | Partitioning | Sharding |
|--|--------------|----------|
| Splits across | One database/server | Multiple servers |
| Managed by | The database itself | Often the application / middleware |
| Main goal | Faster queries on big tables | Scale beyond one server's limits |
| Complexity | Lower | Much higher |
| Cross-piece queries | Easy (same DB) | Hard (across servers) |

```
Partitioning = scale-UP a table inside one server
Sharding     = scale-OUT data across many servers
```

---

## 6. The hard parts of sharding (interview depth)

Sharding solves scale but **adds serious complexity**:
- **Cross-shard queries / joins** → data on different servers is hard to join; often you avoid them or aggregate in the app.
- **Cross-shard transactions** → ACID across servers is very hard (needs distributed transactions).
- **Rebalancing / resharding** → adding a shard means moving data; hash-based schemes make this painful (consistent hashing helps).
- **Hot shards** → a bad shard key concentrates load on one shard (e.g. sharding by country with one huge country).
- **Unique IDs** → auto-increment breaks across shards; need UUIDs or a global ID generator (e.g. Snowflake).

> **Golden interview point:** Shard **only when you must.** First exhaust indexing, query optimization, caching, read replicas, and partitioning. Sharding is a last resort because of its complexity.

**Interview one-liner:** Sharding's hard parts are cross-shard joins/transactions, resharding, hot shards, and global IDs — so use it only after indexing, caching, and read replicas aren't enough.

---

## 7. The scaling ladder (where these fit)

```
1. Optimize queries + indexes        (cheapest)
2. Add caching (Redis)
3. Add read replicas (scale reads)
4. Partition large tables
5. Shard across servers (scale writes) (most complex — last resort)
```

---

## Quick Revision Cheat Sheet

- **Partitioning** = split a big table into partitions **within one database** (range/list/hash) → partition pruning speeds queries.
- **Horizontal** = split rows; **Vertical** = split columns.
- **Sharding** = horizontal partitioning **across multiple servers** (shards) → scales writes & storage.
- **Shard key** decides which shard a row goes to; strategies: range, hash, directory.
- **Partitioning vs Sharding**: one server vs many; DB-managed vs app-managed; scale-up vs scale-out.
- **Sharding pain**: cross-shard joins/transactions, resharding, hot shards, global unique IDs.
- **Scaling ladder**: optimize → cache → read replicas → partition → shard (last resort).
