# System Design Notes — Saga Pattern (Choreography vs Orchestration, Compensating Transactions)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Why Do We Need a Saga?

*Definition:* A **Saga** is a way to run a business transaction that spans **multiple services** as a **sequence of local transactions**. Each service commits its own step in its own database, and if a later step fails, earlier steps are undone by running **compensating actions**.

Real life: Booking a trip — flight, hotel, cab. Each is booked separately with a different company. If the hotel is full, you don't "roll back" the airline's database — you **cancel** the flight booking. That cancel is the compensation.

In a monolith, one `@Transactional` / `BEGIN ... COMMIT` covers everything. In microservices, each service owns its DB (Order DB, Payment DB, Inventory DB), so there is **no single transaction** across them.

```
Monolith:   BEGIN → order + payment + stock → COMMIT        (one DB, ACID)

Microservices:
  Order Svc (Postgres)   Payment Svc (MySQL)   Inventory Svc (Postgres)
        T1 commit   →        T2 commit    →        T3 FAILS
        C1 cancel   ←        C2 refund    ←     (compensate backwards)
```

> ⚠️ A saga gives you **eventual consistency**, not ACID. Between steps, other users can see a half-done state (order "PENDING", payment captured, stock not yet reserved).

**Interview one-liner:** A saga splits a distributed transaction into a chain of local transactions, and undoes completed steps with compensating actions when a later step fails.

---

## 2. Compensating Transactions

*Definition:* A **compensating transaction** is a business-level "undo" for a step that already committed. It doesn't erase history — it **adds a new action** that semantically reverses the effect.

Real life: You can't un-send a bank transfer, but the bank can send a **refund**. Both entries stay in the statement.

| Step (T) | Compensation (C) |
|----------|------------------|
| Create order (PENDING) | Mark order CANCELLED |
| Capture payment (Razorpay) | Refund payment (Razorpay Refund API) |
| Reserve stock | Release stock |
| Create shipment (ShipRocket) | Cancel shipment |
| Send "order confirmed" email | Send "order cancelled" email (can't un-send!) |

### Rules for compensations
- Must be **idempotent** — the same compensation may be retried or delivered twice.
- Must be **retryable until it succeeds** — a failed compensation needs retries + alerting, not silence.
- Some steps are **not compensatable** (email sent, item physically shipped) → put them **last** (the "pivot" step and after).

> ⚠️ Order your steps so the risky/irreversible ones come last. Send the confirmation email only after payment and stock are both done.

**Interview one-liner:** Compensations are semantic undos (refund, release, cancel) — they must be idempotent and retryable, and irreversible steps should come at the end of the saga.

---

## 3. Choreography — services react to events

*Definition:* In **choreography**, there is **no central coordinator**. Each service listens for events, does its local work, and publishes the next event. The flow emerges from who subscribes to what.

Real life: A dance troupe without a conductor — each dancer knows "when she spins, I jump."

```
          OrderCreated            PaymentCompleted          StockReserved
Order Svc ───────────► Kafka ───► Payment Svc ───► Kafka ───► Inventory Svc ───► Kafka ───► Order Svc
                                                                                         (mark CONFIRMED)

Failure path:
Inventory Svc ──StockReservationFailed──► Payment Svc (refund) ──PaymentRefunded──► Order Svc (CANCELLED)
```

```typescript
// NestJS + Kafka — Payment service reacting to OrderCreated
@EventPattern('order.created')
async onOrderCreated(@Payload() evt: OrderCreatedEvent) {
  if (await this.payments.existsForOrder(evt.orderId)) return; // idempotency
  const ok = await this.razorpay.capture(evt.paymentId, evt.amount);
  this.kafka.emit(ok ? 'payment.completed' : 'payment.failed', { orderId: evt.orderId });
}
```

**Pros:** loose coupling, no single point of failure, easy for 2-4 steps.
**Cons:** flow is scattered across services, hard to see "where is order #123?", risk of cyclic event dependencies.

**Interview one-liner:** Choreography = services publish and react to events with no coordinator; simple and decoupled, but the overall flow becomes hard to track as steps grow.

---

## 4. Orchestration — a central brain drives the steps

*Definition:* In **orchestration**, a **saga orchestrator** (a service or workflow engine) tells each participant what to do, waits for the reply, and decides the next step or the compensations.

Real life: An orchestra conductor — musicians don't watch each other, they watch the conductor.

```
                    ┌──────────────────────────┐
                    │  Order Saga Orchestrator │  (stores saga state in DB)
                    └──────────────────────────┘
          1. ReservePayment │      ▲ PaymentOK
                            ▼      │
                       Payment Svc
          2. ReserveStock   │      ▲ StockFailed ✖
                            ▼      │
                       Inventory Svc
          3. (on failure) RefundPayment → Payment Svc
          4. CancelOrder → Order Svc
```

```java
// Pseudo-flow of an orchestrator (Spring Boot)
public void handle(OrderSaga saga) {
    switch (saga.getState()) {
        case STARTED          -> send("payment.cmd", new ChargePayment(saga.orderId()));
        case PAYMENT_DONE     -> send("inventory.cmd", new ReserveStock(saga.orderId()));
        case STOCK_FAILED     -> send("payment.cmd", new RefundPayment(saga.orderId()));
        case PAYMENT_REFUNDED -> send("order.cmd", new CancelOrder(saga.orderId()));
        case STOCK_RESERVED   -> send("order.cmd", new ConfirmOrder(saga.orderId()));
    }
    sagaRepo.save(saga); // persist state so a restart can resume
}
```

**Tools in industry:** Temporal, Camunda / Zeebe, AWS Step Functions, Netflix Conductor, Axon Framework, Eventuate Tram Sagas.

**Interview one-liner:** Orchestration uses a central coordinator that sends commands and tracks saga state — easier to understand, monitor and change, at the cost of an extra component that holds the logic.

---

## 5. Choreography vs Orchestration — when to use which

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Control | Decentralized (events) | Central orchestrator (commands) |
| Coupling | Very loose | Participants coupled to orchestrator API |
| Visibility | Hard — trace across services | Easy — saga state in one place |
| Adding a step | Touch several services | Change the orchestrator |
| Best for | 2-4 simple steps | 5+ steps, branching, timeouts, retries |
| Failure handling | Each service handles its own | Central, explicit |
| Risk | Event spaghetti, cycles | Orchestrator becomes a "god service" |
| Tools | Kafka, RabbitMQ, SNS/SQS | Temporal, Camunda, Step Functions |

Memory trick: **Choreo = "Chat"** (everyone gossips via events), **Orchestra = "Order"** (one boss gives orders).

**Easy rule of thumb:** Few steps, stable flow → **choreography**. Long, branching, business-critical flow (checkout, loan approval) → **orchestration**.

**Interview one-liner:** I pick choreography for short, stable flows and orchestration once the flow has many steps, timeouts or branches and I need one place to see and control saga state.

---

## 6. E-commerce Example — Checkout Saga (end to end)

The checkout on an Oceanevoke-style store: Order → Payment (Razorpay) → Inventory → Shipping (ShipRocket).

```
Step  Service       Action (T)                 Compensation (C)
 1    Order         create order PENDING       cancel order
 2    Payment       capture via Razorpay       refund via Razorpay
 3    Inventory     reserve SKU qty            release SKU qty
 4    Shipping      create ShipRocket order    cancel ShipRocket order   ← pivot-ish
 5    Notification  send confirmation          (none — irreversible, so last)

Happy path : T1 → T2 → T3 → T4 → T5 → order CONFIRMED
Fail at T3 : T1 → T2 → T3✖ → C2 (refund) → C1 (cancel) → order CANCELLED
```

### Practical details that make it actually work
- **Saga ID = orderId** — carried in every event/command for tracing and idempotency.
- **Outbox pattern** — write the DB change + the event in the same local transaction, then relay to Kafka (Debezium CDC or a poller). Avoids "DB committed but event lost."
- **Idempotent consumers** — store processed message IDs; Kafka is at-least-once.
- **Timeouts** — if payment webhook never arrives in 15 min, orchestrator times out and compensates.
- **Semantic locks** — order status `PENDING` tells other flows "don't touch this yet."

> ⚠️ Razorpay payments are often confirmed by **webhook**, which can arrive late or twice. Verify the signature and treat it as an idempotent event keyed by `razorpay_payment_id`.

**Interview one-liner:** In checkout, I'd orchestrate order → payment → stock → shipping with an outbox for reliable events, idempotent handlers keyed by orderId, and refunds/releases as compensations.

---

## 7. Saga Pitfalls & What Interviewers Probe

*Definition:* Sagas lack the **I** in ACID (**isolation**), so concurrent sagas can interfere — these are called **saga anomalies**.

| Anomaly | Example | Countermeasure |
|---------|---------|----------------|
| Lost update | Saga A cancels an order Saga B just updated | Semantic lock (status PENDING), version checks |
| Dirty read | Customer sees "paid" before stock fails and refund happens | Show "processing" state, read your own status |
| Fuzzy read | Inventory read twice gives different counts | Re-read values, commutative updates (`qty = qty - 1`) |

### Common interview questions
- **"What if a compensation fails?"** → Retry with backoff, it must be idempotent; after N tries → dead-letter queue + alert + manual fix.
- **"What if the orchestrator crashes?"** → Saga state is persisted; on restart it resumes from the last state (Temporal does this via event history).
- **"Saga vs 2PC?"** → 2PC locks resources across services and blocks on coordinator failure; saga never holds cross-service locks, trades isolation for availability.
- **"How do you debug a saga?"** → Correlation ID (orderId) in logs, distributed tracing (OpenTelemetry/Jaeger), saga state table, Grafana dashboard for stuck sagas.

> ⚠️ "Just retry" is not a plan. Every step needs a decision: retry, compensate, or escalate to a human.

**Interview one-liner:** Sagas lack isolation, so I use semantic locks, idempotency, persisted saga state, and retries with a dead-letter queue for failing compensations.

---

## Quick Revision Cheat Sheet

- **Saga** = distributed transaction as a chain of local transactions + compensations; eventual consistency, not ACID.
- **Compensating transaction** = semantic undo (refund, release, cancel); idempotent, retryable; irreversible steps go last.
- **Choreography** = services react to events, no coordinator; decoupled but hard to trace.
- **Orchestration** = central orchestrator sends commands and stores saga state; clear and controllable, extra component.
- **Choose**: few stable steps → choreography; long/branching/timeouts → orchestration (Temporal, Camunda, Step Functions).
- **Checkout saga**: order → Razorpay payment → stock → ShipRocket → email; fail → refund + cancel.
- **Make it reliable**: outbox pattern, idempotent consumers, saga ID = orderId, timeouts.
- **Pitfalls**: no isolation → semantic locks, versioning; failed compensation → retry + DLQ + alert.
