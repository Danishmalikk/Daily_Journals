# System Design Notes — CQRS & Event Sourcing (Commands, Queries, Projections, Event Store, Snapshots)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem — one model for reads AND writes

*Definition:* In a classic **CRUD** app, the **same model** (same tables, same entity, same service) handles both **writes** (place order, cancel order) and **reads** (order history page, admin dashboard).

Real life: One person in a shop takes orders, packs parcels, AND answers "where's my parcel?" phone calls. Fine when small, chaos when busy.

```
             ┌───────────────────────────┐
 POST /order │                           │
 ───────────►│  OrderService + Order     │──► orders table (Postgres)
 GET /orders │  entity (TypeORM / JPA)   │◄── same table, 6 JOINs
 ───────────►│                           │
             └───────────────────────────┘
```

Where it bites (e-commerce, Oceanevoke-style):
- Writes need **normalized** tables + strict validation + transactions.
- Reads want **denormalized**, pre-joined data (order + items + shipment status + payment status) — fast.
- Reads usually outnumber writes **10:1 to 100:1**, but you can't scale them separately.

**Interview one-liner:** CRUD uses one model for reads and writes; that works until read and write needs diverge — that tension is what CQRS solves.

---

## 2. CQRS — split Commands from Queries

*Definition:* **CQRS (Command Query Responsibility Segregation)** splits the system into a **write side** (Commands that change state) and a **read side** (Queries that only return data), each with its **own model** — and optionally its own database.

Real life: A restaurant — waiters take orders (commands) into the kitchen; a separate display board shows "order ready" (query). The board isn't the kitchen.

```
            Commands (change state)              Queries (read only)
  POST /orders ──► CommandHandler                GET /orders/me ──► QueryHandler
                     │ validate + business rules                      │
                     ▼                                                ▼
              Write DB (Postgres, normalized) ──events──► Read DB (Postgres view /
                                                         Elasticsearch / Redis / Mongo)
```

| Aspect | Command side | Query side |
|--------|--------------|------------|
| Purpose | Change state | Return data |
| Returns | Ack / id (no data) | DTOs shaped for the screen |
| Model | Rich domain, invariants | Flat, denormalized |
| Storage | Normalized RDBMS | Whatever reads fastest (ES, Redis, read replica) |
| Scaling | Few instances, consistency | Many instances, cache-heavy |

NestJS has this built in via `@nestjs/cqrs`:

```ts
// command side
export class PlaceOrderCommand { constructor(public userId: string, public items: Item[]) {} }

@CommandHandler(PlaceOrderCommand)
export class PlaceOrderHandler implements ICommandHandler<PlaceOrderCommand> {
  constructor(private repo: OrderRepository, private eventBus: EventBus) {}
  async execute(cmd: PlaceOrderCommand) {
    const order = Order.create(cmd.userId, cmd.items);   // business rules here
    await this.repo.save(order);
    this.eventBus.publish(new OrderPlacedEvent(order.id, order.total));
    return order.id;                                     // id only, not the view
  }
}

// query side
@QueryHandler(GetMyOrdersQuery)
export class GetMyOrdersHandler implements IQueryHandler<GetMyOrdersQuery> {
  constructor(private readRepo: OrderSummaryReadRepo) {}
  execute(q: GetMyOrdersQuery) { return this.readRepo.findByUser(q.userId); } // no JOINs
}
```

> ⚠️ CQRS does **not** require two databases. Level 1 = separate classes/handlers on the same DB. Level 2 = read replica or materialized view. Level 3 = separate read store fed by events. Start at level 1.

Memory trick: **"Commands Change, Queries Quietly read."**

**Interview one-liner:** CQRS separates the write model (commands, business rules) from the read model (queries, denormalized views) so each can be optimized and scaled independently — it can be as simple as separate handlers on one DB.

---

## 3. Read models, projections & eventual consistency

*Definition:* A **projection** is a component that listens to events from the write side and **builds/updates a read model** (a table or index shaped exactly for one screen).

Real life: A newspaper — reporters write raw stories (events); editors "project" them into the front page, sports page, city page.

```
OrderPlaced ──┐
PaymentCaptured (Razorpay webhook) ──┼──► Kafka topic "order-events" ──► Projector
ShipmentCreated (ShipRocket) ──┘                                     │
                                                                     ▼
                                              order_summary (one row per order:
                                              status, total, awb_no, paid_at)
```

- One event stream can feed **many** read models (customer "My Orders", admin dashboard, Elasticsearch search, analytics in a warehouse).
- Read model is **eventually consistent** — it lags the write by ms to seconds.

> ⚠️ The classic gotcha: user places order, gets redirected to "My Orders" — and it's not there yet. Fixes: return the created data in the command response, **read-your-own-writes** (read from write DB for that user for a few seconds), or show "processing…" with polling/WebSocket.

**Key points:**
- Projectors must be **idempotent** — Kafka is at-least-once, so the same event can arrive twice. Track `last_processed_event_id` / version per aggregate.
- Order matters per aggregate → use `orderId` as the Kafka partition key.
- Read models are disposable — you can drop and rebuild them (that's the superpower with Event Sourcing).

**Interview one-liner:** Projections consume events and build denormalized read models; they're eventually consistent, so handle stale reads and make projectors idempotent.

---

## 4. Event Sourcing — store what happened, not just the current state

*Definition:* **Event Sourcing** stores every state change as an **immutable, append-only event** (`OrderPlaced`, `ItemAdded`, `OrderPaid`). The current state is **derived by replaying** those events — you never `UPDATE` the row in place.

Real life: A bank passbook. The bank doesn't just store "balance = ₹5,000"; it stores every deposit and withdrawal. Balance = sum of all entries. Your Git history is event sourcing too.

```
CRUD (state only):            Event Sourcing (log of facts):
orders                        order_events (append-only)
┌────┬─────────┐              ┌─────┬──────────┬───┬──────────────────────────┐
│ id │ status  │              │ seq │ order_id │ v │ event                    │
├────┼─────────┤              ├─────┼──────────┼───┼──────────────────────────┤
│ 42 │ SHIPPED │              │ 101 │ 42       │ 1 │ OrderPlaced{total:1499}  │
└────┴─────────┘              │ 102 │ 42       │ 2 │ PaymentCaptured{rzp_..}  │
  "how did it get here?" 🤷   │ 103 │ 42       │ 3 │ AddressChanged{...}      │
                              │ 104 │ 42       │ 4 │ OrderShipped{awb:..}     │
                              └─────┴──────────┴───┴──────────────────────────┘
```

Rebuilding state = a fold over events:

```java
public Order rehydrate(List<DomainEvent> events) {
    Order o = new Order();
    for (DomainEvent e : events) {
        o.apply(e);              // apply() only mutates state, NO side effects
    }
    return o;
}
```

Appending with **optimistic concurrency** (expected version) — plain Postgres works fine:

```sql
-- UNIQUE(order_id, version) makes concurrent writers collide safely
INSERT INTO order_events (order_id, version, type, payload)
VALUES (42, 5, 'OrderCancelled', '{"reason":"customer"}');
-- if another writer already inserted version 5 → unique violation → retry/reject
```

| | CRUD | Event Sourcing |
|---|------|----------------|
| Stores | Latest state | Full history of changes |
| Updates | `UPDATE` in place | Append only |
| Audit trail | Extra work | Free, built-in |
| Time travel ("state on 1 Oct?") | No | Yes, replay up to a point |
| Query current state | Simple `SELECT` | Need projections |
| Complexity | Low | High |

> ⚠️ Event Sourcing ≠ Event-Driven Architecture. EDA = services talk via events. ES = a service **persists its own state** as events. You can do either without the other.

**Interview one-liner:** Event Sourcing persists every change as an immutable event and rebuilds state by replaying them — giving a free audit log and time travel, at the cost of needing projections for queries.

---

## 5. Snapshots, replay & rebuilding projections

*Definition:* A **snapshot** is a saved copy of an aggregate's state at version N, so you load the snapshot + only the events **after** N instead of replaying thousands.

Real life: A game save point — you don't replay the whole game from level 1 every time.

```
events: v1 v2 ... v500 [SNAPSHOT@500] v501 v502 v503
load  = snapshot@500 + apply(v501..v503)   ← 3 events instead of 503
```

- Take snapshots every N events (e.g. 100) or on a timer; snapshots are an optimization — you can always delete them.
- **Replay** = reset a projection's offset to 0 and re-consume → brand-new read model (e.g. you add a "revenue by city" dashboard 6 months later and backfill it from history).

> ⚠️ During replay, projectors must **not** trigger side effects (send email, call Razorpay refund API). Keep side effects in separate handlers that are disabled in replay mode.

**Interview one-liner:** Snapshots cap rehydration cost by saving state every N events; replaying the event log rebuilds or creates new read models — just never re-fire side effects during replay.

---

## 6. CQRS + Event Sourcing together — the full picture

They pair naturally: Event Sourcing makes the write side an event log, which is awful to query — so CQRS projections give you queryable views.

```
 Client ─POST─► API (NestJS / Spring Boot)
                  │ Command
                  ▼
           Command Handler ── load events ──► Event Store
                  │ validate on aggregate      (Postgres table / EventStoreDB /
                  │ append new events           Axon Server)
                  ▼
           Outbox / CDC (Debezium) ──► Kafka "order-events"
                                          │
                     ┌────────────────────┼─────────────────────┐
                     ▼                    ▼                     ▼
             order_summary (PG)   Elasticsearch (search)   Redis (live status)
                     ▲
 Client ─GET──► Query Handler
```

Industry tools: **Axon Framework** (Java/Spring), **EventStoreDB (now Kurrent)**, **Marten** (.NET on Postgres), **@nestjs/cqrs**, Kafka/Debezium for transport.

> ⚠️ **Dual-write problem:** "save to DB, then publish to Kafka" can crash in between → event lost. Use the **Transactional Outbox** (write event in same DB transaction, relay later) or CDC with Debezium.

**Interview one-liner:** With CQRS + ES, commands append events to an event store, those events are published (via outbox/CDC) to Kafka, and projectors build query-optimized read models.

---

## 7. Event design & versioning gotchas (what interviewers probe)

*Definition:* **Event versioning (upcasting)** is how you handle old events when the event's shape changes — because stored events are immutable forever.

```ts
// v1 stored in 2025:  { type: 'OrderPlaced', total: 1499 }
// v2 today:           { type: 'OrderPlaced', total: { amount: 1499, currency: 'INR' } }
function upcast(e: any) {
  if (e.type === 'OrderPlaced' && typeof e.total === 'number') {
    return { ...e, total: { amount: e.total, currency: 'INR' } };  // v1 → v2 on read
  }
  return e;
}
```

**Rules:**
- ❌ `OrderUpdated{...everything}` — meaningless. ✅ `OrderAddressChanged`, `OrderCancelled` — past tense, business intent.
- Only **add** optional fields; never rename/remove — or upcast.
- **GDPR "right to be forgotten"** vs immutable log → **crypto-shredding** (encrypt PII per user, delete the key).
- Events are facts in the past tense; commands are requests (`CancelOrder` can be rejected, `OrderCancelled` cannot).

Common interview questions:
1. *"How do you query current state with ES?"* → projections / read models.
2. *"How do you handle eventual consistency on the UI?"* → read-your-writes, optimistic UI, polling.
3. *"Event got published twice?"* → idempotent consumers, dedupe by event id.
4. *"How do you avoid lost events?"* → outbox / CDC.
5. *"Schema change?"* → versioned events + upcasters, schema registry (Avro/Protobuf).

**Interview one-liner:** Events are immutable past-tense facts, so you evolve them with additive changes and upcasters, handle GDPR with crypto-shredding, and keep consumers idempotent.

---

## 8. When to use which — trade-offs

| Situation | Choice |
|-----------|--------|
| Simple CRUD admin panel, blog, small shop | **Plain CRUD** — don't over-engineer |
| Read traffic >> write, heavy dashboards/search | **CQRS** (read replica or separate read store) |
| Need full audit trail (payments, ledgers, inventory) | **Event Sourcing** (+ CQRS for queries) |
| Complex domain with many state transitions (order lifecycle) | **CQRS + ES** for that bounded context only |
| Team new to distributed systems | CQRS level 1 at most |

| Pros | Cons |
|------|------|
| Independent read/write scaling | Eventual consistency to explain to product/UI |
| Read models tuned per screen | More moving parts (Kafka, projectors, outbox) |
| Full history, audit, time travel (ES) | Event versioning forever |
| Rebuild/add new views from history | Harder debugging, steeper learning curve |

> ⚠️ Apply CQRS/ES **per bounded context**, not to the whole system. Payments ledger: yes. User profile settings: no.

**Easy rule of thumb:** If you can't name the business question that needs history or separate read scaling, use CRUD.

**Interview one-liner:** Use CQRS when reads and writes have very different shapes or load, add Event Sourcing when history/audit is a business requirement — and apply both only to the contexts that need them.

---

## Quick Revision Cheat Sheet

- **CRUD problem**: one model for reads + writes → can't optimize or scale them separately.
- **CQRS**: Commands change state (return id/ack), Queries read denormalized views; separate DBs optional.
- **Projections**: consume events → build read models; eventually consistent; must be idempotent; partition by aggregate id.
- **Event Sourcing**: store immutable append-only events; state = replay; optimistic concurrency via `UNIQUE(aggregate_id, version)`.
- **ES ≠ EDA**: ES is a persistence pattern; EDA is a communication style.
- **Snapshots**: save state every N events to speed rehydration; replay rebuilds read models — no side effects on replay.
- **CQRS + ES**: command → event store → outbox/CDC → Kafka → projectors → read stores (PG/ES/Redis).
- **Versioning**: past-tense intent events, additive changes, upcasters, crypto-shredding for GDPR.
- **When**: CQRS for read/write asymmetry; ES for audit/history domains; apply per bounded context, default to CRUD.
