# System Design Notes — CAP Theorem (CP vs AP, PACELC, Consistency Models)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is the CAP Theorem?

*Definition:* The **CAP theorem** says that a distributed data store, **when a network partition happens**, can guarantee only one of: **Consistency** or **Availability**. Partition tolerance is not optional in real networks, so the real choice is **C or A during a partition**.

Real life: Two branches of a bank lose their phone line to each other. A customer wants to withdraw. The branch can either **refuse** until the line is back (Consistent) or **allow it** and risk the balance being wrong at the other branch (Available).

| Letter | Meaning (precise) |
|--------|-------------------|
| **C — Consistency** | Every read sees the **most recent write** (linearizability) — like a single copy of data |
| **A — Availability** | Every request to a **non-failed node** gets a (non-error) response |
| **P — Partition tolerance** | System keeps working even if messages between nodes are lost/delayed |

```
        Node A  ──── ✂ network partition ✂ ────  Node B
   (gets write x=2)                          (still has x=1)

   Client reads from B:
     CP choice → B returns ERROR / waits   (no stale data)
     AP choice → B returns x=1             (stale, but answers)
```

> ⚠️ The **C** in CAP is **not** the C in ACID. ACID's C = "data obeys rules/constraints". CAP's C = "all nodes show the latest value".

**Interview one-liner:** CAP says that during a network partition, a distributed system must choose between consistency (refuse or wait) and availability (answer, maybe with stale data).

---

## 2. Why "Pick 2 of 3" is Misleading

*Definition:* The popular "pick any 2" phrasing is wrong because **P is not a choice** — partitions *will* happen (switch fails, AWS AZ hiccup, GC pause looks like a dead node).

```
        ┌─────── C ───────┐
        │                 │
       CP                CA   ← only possible on a single node / no network
        │                 │       (e.g. one Postgres instance)
        P ────── AP ───── A

Real distributed systems: CP or AP.
```

- "CA" systems are really just **single-node** systems (or ones that give up when the network breaks).
- When there's **no partition**, a system can give both C and A — CAP only bites during failure.
- The choice can be **per operation**, not per system (e.g. Cassandra `QUORUM` vs `ONE`).

Memory trick: **"P is a fact, not a feature"** — so remember CAP as **"C or A, when P"**.

**Interview one-liner:** Partitions are unavoidable, so the real CAP trade-off is CP vs AP, and only during the partition — normal operation can be both consistent and available.

---

## 3. CP vs AP — real systems

| | **CP** (consistency first) | **AP** (availability first) |
|--|---------------------------|-----------------------------|
| During partition | Minority side rejects/blocks writes | All nodes keep serving, may diverge |
| Data | Never stale | May be stale; reconciled later |
| Examples | ZooKeeper, etcd, Consul (Raft), HBase, MongoDB (majority writes), Spanner, CockroachDB | Cassandra, DynamoDB (eventual reads), Couchbase, Riak, DNS, CDN caches |
| Good for | Payments, stock count, leader election, locks, config | Product catalog, likes, view counts, carts, feeds, session data |
| Cost | Errors/timeouts during failures | Conflict resolution (last-write-wins, vector clocks, CRDTs) |

### E-commerce mapping (Oceanevoke-style)
```
Payment status / wallet balance      → CP  (wrong answer = lost money)
Inventory for last 2 units           → CP  (avoid overselling)
Product catalog & reviews            → AP  (stale price for 2 sec is fine)
Shopping cart                        → AP  (Amazon Dynamo paper: always accept "add to cart")
Recommendation / "people also bought"→ AP
```

> Remember: a single PostgreSQL primary + async read replicas behaves like **AP for reads from replicas** (stale) and **CP for the primary** (if primary is unreachable, writes fail).

**Easy rule of thumb:** If a stale or conflicting answer costs **money or trust** → CP. If an error costs more than staleness → AP.

**Interview one-liner:** I choose CP for money, stock and locks where stale data is dangerous, and AP for catalog, carts and feeds where staying up matters more than perfect freshness.

---

## 4. PACELC — the more complete answer

*Definition:* **PACELC** extends CAP: **if Partition** → choose **A or C**; **Else** (normal operation) → choose **Latency or Consistency**.

Real life: Even with a perfect phone line, the bank branch can either call the other branch before every withdrawal (consistent but slow) or trust its local copy (fast but maybe stale).

```
  P ? ──yes──► A or C
   │
   no (Else)
   └────────► L or C     ← this is what you trade every single day
```

| System | If Partition | Else | Label |
|--------|--------------|------|-------|
| Cassandra / DynamoDB (default) | A | L | **PA/EL** |
| MongoDB (default) | C | L (replica reads) | PC/EL-ish (tunable) |
| Spanner / CockroachDB | C | C | **PC/EC** |
| etcd / ZooKeeper | C | C | **PC/EC** |
| PostgreSQL sync replication | C | C | PC/EC |
| PostgreSQL async replication | — | L | trades consistency for latency on replicas |

> ⚠️ Interviewers love PACELC because it shows you know CAP only covers failures; **latency vs consistency** is the everyday trade-off (sync replication adds a network round trip to every write).

**Interview one-liner:** PACELC says during a partition pick A or C, else pick latency or consistency — so even healthy systems trade speed for freshness every request.

---

## 5. Consistency Models (strong → weak)

*Definition:* A **consistency model** is the contract that tells clients **what values a read is allowed to return** after writes.

| Model | Guarantee | Example |
|-------|-----------|---------|
| **Linearizable (strong)** | Reads always see the latest committed write, as if one copy | etcd, Spanner, single Postgres primary |
| **Sequential** | Everyone sees operations in the same order (not necessarily real-time) | Some replicated logs |
| **Causal** | If A caused B, everyone sees A before B | Comment then reply always in order; MongoDB causal sessions |
| **Read-your-writes** | You always see your own updates | Update profile → refresh shows new name |
| **Monotonic reads** | You never see data go "back in time" | Don't show 5 likes, then 3 |
| **Eventual** | If writes stop, replicas converge eventually | DNS, Cassandra `ONE`, S3 cross-region replication |

```
Strong ───────────────────────────────────────────► Eventual
Linearizable > Sequential > Causal > Session (RYW, monotonic) > Eventual
more correct, slower, less available     faster, more available, staler
```

### Practical fix — read-your-writes with read replicas
```typescript
// NestJS + TypeORM replication: after a write, read from primary for this user
const user = await this.dataSource
  .createQueryRunner('master')    // force primary instead of a lagging replica
  .manager.findOne(User, { where: { id } });
```

Memory trick: **"Lin-Seq-Cause-Session-Eventual"** → "**L**ittle **S**heep **C**an **S**leep **E**verywhere".

**Interview one-liner:** Consistency models range from linearizable (always latest) to eventual (converges later); most apps need strong consistency for a few things and session guarantees like read-your-writes for the rest.

---

## 6. Tuning Consistency — Quorums

*Definition:* In leaderless stores (Cassandra, DynamoDB-style), with **N** replicas, a write waits for **W** acks and a read asks **R** replicas. If **R + W > N**, read and write sets overlap, so a read sees the latest write.

```
N = 3 replicas
W = 2, R = 2  → 2 + 2 > 3  ✅ strong-ish (overlap guaranteed)
W = 1, R = 1  → 1 + 1 ≤ 3  ⚠️ fast but can read stale
```

```sql
-- Cassandra CQL (cqlsh)
CONSISTENCY QUORUM;   -- per session/query: ONE, QUORUM, ALL, LOCAL_QUORUM
SELECT * FROM orders WHERE order_id = 'ord-123';
```

| Setting | Latency | Consistency | Availability |
|---------|---------|-------------|--------------|
| W=1, R=1 | Lowest | Weakest | Highest |
| QUORUM/QUORUM | Medium | Strong-ish | Survives 1 of 3 down |
| W=ALL | Highest | Strong reads with R=1 | Any node down → writes fail |

> ⚠️ R + W > N is necessary but edge cases (sloppy quorums, hinted handoff, clock-based last-write-wins) can still produce anomalies. Say "strong-ish", not "linearizable".

**Interview one-liner:** With N replicas, choosing R + W > N makes reads overlap the latest write, letting you tune each query between latency and consistency.

---

## 7. What Interviewers Probe

- **"Is MySQL/Postgres CP or AP?"** → A single node isn't distributed (CAP doesn't really apply). With replication: primary writes are consistent; async replica reads are eventually consistent; failover setups (Patroni, RDS Multi-AZ) lean CP.
- **"Is Redis CP or AP?"** → Redis replication is async, so Redis Cluster/Sentinel can lose acknowledged writes on failover → closer to **AP**. Don't build money logic on it alone.
- **"Is Kafka CP or AP?"** → Configurable: `acks=all` + `min.insync.replicas=2` + `unclean.leader.election.enable=false` → CP-leaning; relaxing them → AP-leaning.
- **"Design a like counter vs a bank balance."** → Likes: AP, eventual, Redis INCR/Cassandra counters. Balance: CP, single Postgres primary, `SELECT ... FOR UPDATE`.
- **"How do AP systems resolve conflicts?"** → Last-write-wins (timestamps), vector clocks, CRDTs, app-level merge (union of cart items).

**Interview one-liner:** I answer CAP questions per feature, not per database — and I mention PACELC, replication mode, and tunable consistency to show the real trade-offs.

---

## Quick Revision Cheat Sheet

- **CAP** = during a partition, choose Consistency (latest or error) or Availability (answer, maybe stale).
- **P is mandatory** in distributed systems → real choice is **CP vs AP**; "CA" = single node.
- **CAP C ≠ ACID C** — CAP C means linearizable reads; ACID C means valid data.
- **CP**: etcd, ZooKeeper, Consul, Spanner, CockroachDB → payments, stock, locks.
- **AP**: Cassandra, DynamoDB, DNS, CDNs → catalog, carts, feeds, counters.
- **PACELC** = if Partition A/C, Else Latency/C; Cassandra PA/EL, Spanner PC/EC.
- **Consistency models**: linearizable > sequential > causal > session (read-your-writes, monotonic) > eventual.
- **Quorum**: R + W > N → reads overlap latest write; tune per query (ONE, QUORUM, ALL).
- **Interview angle**: decide per feature; Redis ≈ AP, Kafka tunable, Postgres replicas eventual.
