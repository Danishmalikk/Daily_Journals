# System Design Notes — Distributed Transactions (2PC, 3PC, Saga, Outbox, Idempotency)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Distributed Transaction?

*Definition:* A **distributed transaction** is a single business operation that must update data in **two or more independent systems** (different databases, services, or a DB + a message broker) and still behave as **all-or-nothing**.

Real life: Buying a house — the bank releases money, the registry transfers ownership, the seller hands over keys. Three parties, one deal. Nobody wants "money gone, house not transferred."

```
Local transaction:     App ──► one Postgres  (BEGIN ... COMMIT — easy, ACID)

Distributed:           App ──► Order DB (Postgres)
                           ──► Payment DB (MySQL)
                           ──► Kafka (publish OrderPlaced)
                       All must agree — but there's no shared BEGIN/COMMIT.
```

### The classic hidden case — "dual write"
```typescript
// ❌ Looks innocent in NestJS, but it's a distributed transaction
await this.orderRepo.save(order);              // 1. DB commit succeeds
await this.kafka.emit('order.placed', order);  // 2. App crashes here → event lost forever
```

> ⚠️ Any time you write to a DB **and** send to Kafka/RabbitMQ/an HTTP API in one flow, you have a distributed transaction — even inside one service.

**Interview one-liner:** A distributed transaction spans multiple independent resources that can each fail separately, so you can't rely on one database's COMMIT to keep them consistent.

---

## 2. Two-Phase Commit (2PC)

*Definition:* **2PC** is a protocol where a **coordinator** asks every participant to **prepare** (vote), and only if **all vote YES** does it tell everyone to **commit**; otherwise everyone aborts.

Real life: A wedding — the priest asks both "Do you?" (prepare). Only if both say "I do" is the marriage declared (commit).

```
           Coordinator (Transaction Manager)
             │                         │
Phase 1:  PREPARE?                  PREPARE?
             ▼                         ▼
        Order DB: YES            Payment DB: YES   (both lock rows, write to log)
             │                         │
Phase 2:  COMMIT                    COMMIT
             ▼                         ▼
           done                       done

If any says NO (or times out) → coordinator sends ROLLBACK to all.
```

**Where it exists:** XA transactions (JTA in Java with Atomikos / Narayana), PostgreSQL `PREPARE TRANSACTION`, MySQL XA, Microsoft DTC.

### Why 2PC is avoided in microservices
| Problem | What happens |
|---------|--------------|
| **Blocking** | If the coordinator dies after PREPARE, participants hold locks and wait — possibly forever |
| **Latency** | 2 network round trips + locks held for the whole duration |
| **Availability** | One slow/down participant blocks everyone (fails "A" in CAP) |
| **Support** | Kafka, Redis, most REST APIs, Razorpay don't speak XA |
| **Coupling** | All services must use the same transaction manager |

> ⚠️ 2PC guarantees atomicity but **not liveness** — it can block. That's the key point interviewers look for.

**Interview one-liner:** 2PC is prepare-then-commit via a coordinator; it gives atomicity but is blocking, slow and poorly supported, so microservices usually avoid it.

---

## 3. Three-Phase Commit (3PC)

*Definition:* **3PC** adds a middle **pre-commit** phase (CanCommit → PreCommit → DoCommit) plus timeouts, so participants can make progress if the coordinator crashes.

```
Phase 1: CanCommit?   → all YES
Phase 2: PreCommit    → all ACK  ("I know everyone agreed")
Phase 3: DoCommit     → commit
Timeout in phase 3 → participant can safely commit on its own
```

| | 2PC | 3PC |
|--|-----|-----|
| Phases | 2 | 3 |
| Blocking on coordinator crash | Yes | Reduced (non-blocking under its assumptions) |
| Handles network partitions | No | **No** — can become inconsistent |
| Used in practice | Yes (XA) | Rarely |

> Remember: 3PC assumes a network with bounded delays and no partitions. Real networks partition, so 3PC is mostly a textbook answer. Real systems use **consensus** (Raft/Paxos — etcd, Spanner, CockroachDB) instead.

**Interview one-liner:** 3PC adds a pre-commit phase to avoid 2PC's blocking, but it breaks under network partitions, so it's rarely used — consensus protocols like Raft replaced it.

---

## 4. The Microservice Answer — Saga + Eventual Consistency

*Definition:* **Eventual consistency** means data across services may be temporarily out of sync, but if no new updates happen, all copies **converge** to the correct state.

Instead of locking everything (2PC), each service commits locally and the system fixes itself forward or backward with a **saga** (see `11_Saga_Pattern.md`).

```
2PC:   lock A + lock B → commit both          (strong, blocking)
Saga:  commit A → event → commit B ✖ → compensate A   (eventual, non-blocking)
```

| Approach | Consistency | Availability | Complexity | Use when |
|----------|-------------|--------------|------------|----------|
| Single DB transaction | Strong | High | Low | Data fits in one DB (best option!) |
| 2PC / XA | Strong | Low | Medium | Few DBs, same TM, low throughput, legacy enterprise |
| Saga | Eventual | High | High | Microservices, long-running flows |
| Outbox + events | Eventual | High | Medium | Reliable DB write + message publish |

**Easy rule of thumb:** First ask "can these tables live in **one** database?" If yes, use a local transaction. Distributed transactions are a cost you pay only when service boundaries force it.

**Interview one-liner:** In microservices I avoid 2PC and accept eventual consistency, using sagas for multi-service flows and keeping data that must be strongly consistent inside one service's database.

---

## 5. Transactional Outbox Pattern

*Definition:* The **outbox pattern** solves dual writes by saving the business change **and** the event into the **same database in one local transaction**. A separate relay process later reads the outbox table and publishes to the broker.

Real life: Instead of mailing a letter on your way out (and maybe forgetting), you drop it in your office's outbox tray. The mailroom reliably sends everything in the tray.

```
┌────────────── Order Service ──────────────┐
│  BEGIN                                     │
│    INSERT INTO orders ...                  │
│    INSERT INTO outbox (event_type, payload)│
│  COMMIT   ← atomic: both or neither        │
└────────────────────┬───────────────────────┘
                     │  relay: poller or CDC (Debezium reads WAL/binlog)
                     ▼
                   Kafka  ──►  Payment / Inventory / Notification
```

```typescript
// ✅ NestJS + TypeORM — one local transaction
await this.dataSource.transaction(async (em) => {
  await em.save(Order, order);
  await em.save(Outbox, {
    aggregateId: order.id,
    type: 'order.placed',
    payload: JSON.stringify(order),
  });
});
// A separate worker (or Debezium) publishes unsent outbox rows to Kafka.
```

- Relay is **at-least-once** → consumers may see duplicates → they must be idempotent.
- Mirror pattern on the consumer side: **Inbox** table (store processed message IDs).
- Tools: Debezium + Kafka Connect, Eventuate Tram, Spring Modulith events, AWS DynamoDB Streams.

**Interview one-liner:** The outbox pattern writes the entity and its event in one local transaction, then a relay (poller or Debezium CDC) publishes it, so a DB commit can never lose its event.

---

## 6. Idempotency — the safety net for retries

*Definition:* An operation is **idempotent** if doing it **once or many times gives the same result**. In distributed systems, messages and HTTP calls are retried, so handlers must be idempotent.

Real life: Pressing the elevator button 5 times still calls the elevator once.

```
Client ──POST /payments (Idempotency-Key: ord-123)──► API
          timeout... retry
Client ──POST /payments (Idempotency-Key: ord-123)──► API → "already processed" → returns same result
```

```sql
-- Consumer-side dedupe (inbox) in PostgreSQL
INSERT INTO processed_messages (message_id) VALUES ('evt-789')
ON CONFLICT (message_id) DO NOTHING;   -- 0 rows inserted → duplicate, skip
```

### Techniques
- **Idempotency key** header (Stripe, Razorpay order receipts) stored with the response.
- **Unique constraints** — e.g. `UNIQUE(order_id)` on payments.
- **Processed-message table** in the same transaction as the side effect.
- **Conditional updates** — `UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'`.
- Prefer absolute writes (`set status=PAID`) over relative ones (`balance = balance + 100`) unless deduped.

> ⚠️ Kafka "exactly-once" (idempotent producer + transactions) only covers Kafka-to-Kafka. The moment your consumer writes to Postgres or calls Razorpay, you're back to at-least-once + idempotent handlers.

**Interview one-liner:** Since delivery is at-least-once, I make every handler idempotent with idempotency keys, unique constraints, or a processed-messages table so retries are harmless.

---

## 7. What Interviewers Probe

| Question | Strong answer |
|----------|---------------|
| "How do you keep Order and Payment DBs consistent?" | Saga with compensations + outbox + idempotent consumers |
| "Why not 2PC?" | Blocking, lock-holding, low availability, brokers/APIs don't support XA |
| "DB saved but Kafka publish failed?" | Outbox pattern / CDC with Debezium |
| "Message processed twice?" | Idempotent consumer, inbox table, unique keys |
| "When IS strong consistency required?" | Money balances, stock that can't oversell → keep in one DB, use row locks / `SELECT ... FOR UPDATE` |
| "How do you detect stuck flows?" | Reconciliation jobs, saga state table, Grafana alerts on PENDING > 15 min |

> Remember: also mention **reconciliation** — a nightly job that compares your orders with Razorpay settlements catches anything all the clever patterns missed.

**Interview one-liner:** I avoid distributed transactions where I can, and where I can't, I combine sagas, outbox, idempotency and reconciliation to reach eventual consistency safely.

---

## Quick Revision Cheat Sheet

- **Distributed transaction** = one business op across multiple DBs/services/brokers; no shared COMMIT.
- **Dual write** (DB + Kafka) is a hidden distributed transaction → events can be lost.
- **2PC** = prepare/vote then commit via coordinator; atomic but blocking, slow, poor support (XA, JTA).
- **3PC** = adds pre-commit to reduce blocking; unsafe under partitions; rarely used — Raft/Paxos instead.
- **Eventual consistency** = replicas converge over time; the usual microservice trade-off.
- **Saga** = local transactions + compensations; preferred over 2PC in microservices.
- **Outbox** = entity + event in one local txn, relay via poller/Debezium → no lost events.
- **Idempotency** = same result on retries; keys, unique constraints, inbox table, conditional updates.
- **Best option** = keep strongly consistent data in one DB; add reconciliation jobs as a safety net.
