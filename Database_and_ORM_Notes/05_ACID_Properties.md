# Database Notes — ACID Properties

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is ACID?

*Definition:* **ACID** is a set of **4 guarantees** a database makes about transactions, so your data stays correct and reliable even during crashes, errors, or many users at once. It stands for **A**tomicity, **C**onsistency, **I**solation, **D**urability.

Memory trick: **ACID** = "A Clean Internal Database" → **A**tomicity, **C**onsistency, **I**solation, **D**urability.

Real-life anchor: a **bank transfer** of ₹100 from Alice to Bob explains all four.

```
   A — all-or-nothing (both debit & credit, or neither)
   C — rules stay valid (total money unchanged)
   I — concurrent transfers don't interfere
   D — once done, it survives a power cut
```

---

## 2. A — Atomicity (all or nothing)

*Definition:* **Atomicity** guarantees that a transaction is **indivisible** — either *every* operation in it succeeds, or *none* of them do. If any step fails, the whole transaction is rolled back as if it never happened.

```
Transfer ₹100:
   Step 1: debit Alice  -100   ✅
   Step 2: credit Bob   +100   ❌ fails (crash)
   → Step 1 is ROLLED BACK. Alice keeps her ₹100.
```
Without atomicity, Alice would lose ₹100 that Bob never received.

**Interview one-liner:** Atomicity = all steps of a transaction succeed together or all are undone — no partial changes.

---

## 3. C — Consistency (rules stay valid)

*Definition:* **Consistency** guarantees that a transaction takes the database from one **valid state** to another **valid state**, never breaking the database's rules (constraints, foreign keys, data types). If a transaction would violate a rule, it's rejected.

```
Rule: total money in system = constant.
Before transfer: Alice ₹500 + Bob ₹300 = ₹800
After transfer:  Alice ₹400 + Bob ₹400 = ₹800   ✅ still valid

A transaction that tried to make a balance negative (if a rule forbids it)
→ rejected, DB stays consistent.
```

> Don't confuse with the "C" in **CAP theorem** (a different consistency about distributed copies). ACID's consistency = "all DB rules/constraints hold."

**Interview one-liner:** Consistency = every transaction leaves the database in a valid state that obeys all its rules and constraints.

---

## 4. I — Isolation (transactions don't interfere)

*Definition:* **Isolation** guarantees that **concurrent** transactions don't step on each other — each runs as if it were the only one. The intermediate, half-done state of one transaction is hidden from others.

```
Two transfers at the same time:
   T1: Alice → Bob
   T2: Alice → Carol
Isolation ensures they don't read each other's half-finished data
and corrupt Alice's balance.
```

Isolation has **levels** (READ_UNCOMMITTED → SERIALIZABLE) that trade safety for speed — covered fully in the Transactions file. Higher isolation = fewer anomalies (dirty/non-repeatable/phantom reads) but slower.

**Interview one-liner:** Isolation = concurrent transactions behave as if run one at a time; the degree is set by isolation levels that balance correctness vs performance.

---

## 5. D — Durability (survives crashes)

*Definition:* **Durability** guarantees that once a transaction is **committed**, its changes are **permanent** — they survive crashes, power failures, or restarts. The data is safely written to non-volatile storage (disk).

```
Transfer committed → "Success" shown to user
   → power goes out 1 second later
   → on restart, the transfer is STILL there.
```

How databases achieve it: **write-ahead logging (WAL)** — changes are written to a durable log *before* being applied, so on restart the DB can replay/recover committed work.

**Interview one-liner:** Durability = once committed, changes persist permanently even after a crash, typically via write-ahead logging to disk.

---

## 6. All four together (the money table)

| Letter | Property | Guarantees | Bank example |
|--------|----------|------------|--------------|
| **A** | Atomicity | All-or-nothing | Debit & credit both happen, or neither |
| **C** | Consistency | Valid state → valid state | Total money stays ₹800 |
| **I** | Isolation | No interference between txns | Parallel transfers don't corrupt balance |
| **D** | Durability | Committed = permanent | Survives a power cut |

---

## 7. Why it matters & where it's challenged

- **Relational databases (MySQL, PostgreSQL)** are built around strong ACID guarantees → great for money, orders, anything needing correctness.
- **Many NoSQL databases** relax some ACID guarantees (especially Isolation/Consistency) for **speed and scale**, favoring **BASE** instead:

*Definition:* **BASE** = **B**asically **A**vailable, **S**oft state, **E**ventually consistent — the opposite trade-off: prioritize availability & scale, accept that data becomes consistent *eventually* rather than instantly.

| ACID | BASE |
|------|------|
| Strong consistency now | Eventual consistency |
| Reliability first | Availability & scale first |
| SQL databases | Many NoSQL databases |

> Modern reality: many NoSQL systems (e.g. MongoDB) now offer ACID transactions too — the line has blurred. (See the NoSQL file.)

**Interview one-liner:** ACID (SQL) prioritizes correctness; BASE (many NoSQL) prioritizes availability and scale with eventual consistency — pick based on whether correctness or scale matters more.

---

## Quick Revision Cheat Sheet

- **ACID** = Atomicity, Consistency, Isolation, Durability — guarantees of reliable transactions.
- **Atomicity**: all-or-nothing (rollback on any failure).
- **Consistency**: every transaction leaves a valid state (rules/constraints hold).
- **Isolation**: concurrent transactions don't interfere (levels trade safety vs speed).
- **Durability**: committed data survives crashes (via write-ahead logging).
- **Bank transfer** is the go-to example for all four.
- **BASE** (many NoSQL) = Basically Available, Soft state, Eventually consistent — scale over strict consistency.
