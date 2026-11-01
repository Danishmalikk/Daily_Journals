# System Design Notes — Event-Driven Architecture (Kafka vs RabbitMQ, pub/sub, delivery guarantees, outbox, DLQ)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is Event-Driven Architecture?

*Definition:* **Event-Driven Architecture (EDA)** is a style where services communicate by **publishing events** ("something happened") to a broker, and other services **react** to those events asynchronously — instead of calling each other directly and waiting.

Real life: A WhatsApp group. You post "Order #42 placed" once. Billing, warehouse and the delivery guy each read it and act on their own — you don't phone each of them and wait on the line.

```
❌ Synchronous chain (order-svc waits on everyone)
order-svc ──HTTP──> payment-svc ──HTTP──> inventory-svc ──HTTP──> email-svc
   (if email-svc is slow, checkout is slow; if it's down, checkout fails)

✅ Event-driven
                         ┌──> inventory-svc  (reserve stock)
order-svc ── OrderPlaced ─┼──> email-svc      (send confirmation)
          (to Kafka)     ├──> shipping-svc   (create ShipRocket shipment)
                         └──> analytics-svc  (update dashboards)
```

**Key Points:**
- **Loose coupling**: the producer doesn't know who consumes. Adding analytics = new consumer, zero change in order-svc.
- **Resilience**: if email-svc is down, events wait in the broker; it catches up later.
- **Scalability**: absorb spikes (Big Billion Day sale) — the queue buffers, consumers drain at their pace.
- Cost: **eventual consistency**, harder debugging, need for idempotency and monitoring consumer lag.

> Event = a **fact in the past tense** (`OrderPlaced`, `PaymentCaptured`). Command = a **request** (`ReserveStock`). Events are broadcast; commands go to one handler.

**Interview one-liner:** EDA decouples services by publishing past-tense events to a broker that consumers react to asynchronously — gaining resilience and scalability at the cost of eventual consistency.

---

## 2. Queues vs Pub/Sub

*Definition:* In a **queue (point-to-point)** each message is processed by **one** consumer. In **pub/sub** each message is delivered to **every subscriber** (group).

```
Queue (work distribution)               Pub/Sub (fan-out)
            ┌─> worker A (msg 1)                     ┌─> email-svc     (all msgs)
producer ───┼─> worker B (msg 2)       producer ─────┼─> inventory-svc (all msgs)
            └─> worker C (msg 3)                     └─> analytics-svc (all msgs)
```

| | Queue | Pub/Sub |
|--|-------|---------|
| Who gets a message | One consumer | Every subscriber |
| Use for | Background jobs, load-spreading (resize images, send invoices) | Broadcasting domain events |
| Tools | RabbitMQ queue, AWS SQS, BullMQ (Redis) | Kafka topics, RabbitMQ fanout/topic exchange, AWS SNS, Redis Pub/Sub |

> Kafka does **both**: consumers in the **same consumer group** share partitions (queue behavior); **different groups** each get every message (pub/sub).

**Interview one-liner:** A queue gives each message to one worker for load-sharing; pub/sub gives every message to every subscriber for fan-out — Kafka consumer groups give you both.

---

## 3. Kafka vs RabbitMQ

*Definition:* **Kafka** is a distributed, partitioned, **append-only log** — messages stay for a retention period and consumers track their own offset. **RabbitMQ** is a **message broker** — it routes messages through exchanges into queues and deletes them once acknowledged.

Real life: Kafka = a **newspaper archive** (anyone can read any past edition from any page). RabbitMQ = a **post office** (letter delivered to the right box, then it's gone).

```
Kafka topic "orders" (3 partitions, key = orderId)
P0: [o1][o4][o7] ...  ← consumer group "email"     offset=2
P1: [o2][o5][o8] ...  ← consumer group "inventory" offset=3
P2: [o3][o6][o9] ...      (each group tracks its own position; messages kept 7 days)

RabbitMQ
producer ─> exchange (topic: "order.*") ─┬─> queue email.q     ─> consumer (ack → deleted)
                                         └─> queue inventory.q ─> consumer
```

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| Model | Distributed commit log | Smart broker, queues |
| Message after consume | **Kept** (retention, replayable) | **Deleted** after ack |
| Ordering | Per partition (same key → same partition) | Per queue (single consumer) |
| Throughput | Very high (millions/sec, batching, sequential disk) | High (tens of thousands/sec per node) |
| Routing | Simple (topic + key) | Rich (direct, topic, fanout, headers exchanges) |
| Consumer model | Pull, consumer tracks offset | Push, broker tracks delivery |
| Replay history | ✅ Reset offset | ❌ (except Streams feature) |
| Per-message features | Basic | Priorities, TTL, delayed messages (plugin), per-message ack |
| Best for | Event streaming, event sourcing, analytics, CDC, log aggregation | Task queues, RPC-ish workflows, complex routing |
| Managed options | AWS MSK, Confluent Cloud, Redpanda | Amazon MQ, CloudAMQP |

Memory trick: **Kafka = "Keep it"** (log, replay). **Rabbit = "Route it, then Remove it"**.

**Easy rule of thumb:** Need replay, very high throughput or multiple teams consuming the same event stream → **Kafka**. Need job queues, flexible routing, priorities or delays with modest volume → **RabbitMQ** (or SQS / BullMQ for simple jobs).

**Interview one-liner:** Kafka is a durable, replayable partitioned log built for high-throughput event streaming; RabbitMQ is a flexible routing broker that deletes messages after acknowledgment — ideal for task queues.

---

## 4. Delivery guarantees

*Definition:* A **delivery guarantee** says how many times a consumer may see a message when things fail (crashes, network blips, retries).

| Guarantee | Meaning | How it happens | Risk |
|-----------|---------|----------------|------|
| **At-most-once** | 0 or 1 time | Commit offset/ack **before** processing | Lost messages |
| **At-least-once** | 1 or more times | Process **then** ack/commit | Duplicates |
| **Exactly-once** | Effectively 1 time | Kafka idempotent producer + transactions (read-process-write within Kafka) | Complex; only end-to-end inside Kafka |

```
At-least-once (the common default):
consume msg ──> process (charge, update DB) ──> commit offset
                          │
             crash here ──┘  → on restart the message is redelivered → DUPLICATE
```

> ⚠️ "Exactly-once" across Kafka **and** your Postgres DB or Razorpay API **does not exist for free**. Real systems do **at-least-once delivery + idempotent consumers** = exactly-once **effect**.

**Interview one-liner:** Brokers give at-most-once or at-least-once in practice; we pick at-least-once and make consumers idempotent to get an exactly-once effect.

---

## 5. Idempotent consumers

*Definition:* An **idempotent consumer** produces the same result whether it processes a message **once or many times**.

```ts
// NestJS Kafka consumer — dedupe by eventId inside the same DB transaction
@EventPattern('payment.captured')
async onPaymentCaptured(@Payload() evt: PaymentCaptured) {
  await this.dataSource.transaction(async (tx) => {
    const inserted = await tx.query(
      `INSERT INTO processed_events(event_id) VALUES ($1)
       ON CONFLICT DO NOTHING RETURNING event_id`,
      [evt.eventId],
    );
    if (inserted.length === 0) return;              // already handled → skip
    await tx.update(Order, { id: evt.orderId }, { status: 'PAID' });
  });
}
```

**Techniques:**
- **Processed-events table** with a unique `event_id` (above) — dedupe + business update in one DB transaction.
- **Natural idempotency**: `UPDATE orders SET status='PAID' WHERE id=42` is safe to repeat; `balance = balance + 100` is not.
- **Unique constraints**: e.g. unique `razorpay_payment_id` on the payments table.
- **Version/state checks**: only move `PENDING → PAID`, ignore if already `PAID`.

**Interview one-liner:** Make consumers idempotent with a unique event ID stored in the same transaction as the business change, so redelivered messages are harmless.

---

## 6. Transactional Outbox — no lost or phantom events

*Definition:* The **outbox pattern** solves the **dual-write problem**: instead of writing to the DB *and* publishing to Kafka separately (one can fail), you write the business row **and** an outbox row in **one DB transaction**, then a separate relay publishes outbox rows to the broker.

```
❌ Dual write
BEGIN; INSERT order; COMMIT;   ──>  kafka.send(OrderPlaced)   ← crash here = order exists, no event
                                                               (or send first + DB rollback = phantom event)
✅ Outbox
BEGIN;
  INSERT INTO orders ...;
  INSERT INTO outbox(id, topic, payload) VALUES (...);   ← same transaction, atomic
COMMIT;
          │
  Relay (poller or Debezium CDC reading Postgres WAL) ──> Kafka "orders" ──> consumers
          └── marks outbox row as sent
```

**Key Points:**
- Relay options: polling (`SELECT ... WHERE sent=false FOR UPDATE SKIP LOCKED`) or **Debezium** CDC (lower latency, no polling load).
- Relay may publish twice on crash → consumers must still be **idempotent** (section 5).
- Pair with the **inbox pattern** (processed-events table) on the consumer side.

> ⚠️ Classic interview trap: *"Order saved, then Kafka publish failed — what now?"* Answer: outbox. Don't say "retry in a try/catch" — the process can crash between the two writes.

**Interview one-liner:** The outbox pattern writes the event to an outbox table in the same DB transaction as the business change, and a relay (poller or Debezium) publishes it — guaranteeing no lost or phantom events.

---

## 7. Dead Letter Queue (DLQ) & retries

*Definition:* A **Dead Letter Queue** is a separate queue/topic where messages go after they **fail processing repeatedly**, so one bad message (a **poison pill**) doesn't block the whole stream.

```
orders ──> consumer ──fail──> orders.retry.1m ──fail──> orders.retry.10m ──fail──> orders.DLQ
                                                                                    │
                                                         alert (Grafana) + inspect + fix + replay
```

**Key Points:**
- Retry **transient** errors (timeouts, Razorpay 503) with exponential backoff; send **permanent** errors (invalid JSON, schema mismatch) straight to DLQ.
- RabbitMQ: native dead-letter exchanges (`x-dead-letter-exchange`) + TTL for delayed retry.
- Kafka: no built-in DLQ in the broker — use retry/DLT topics (Spring Kafka `@RetryableTopic` + `DefaultErrorHandler`, Kafka Connect DLQ config).
- Store error reason + original headers with the DLQ message; build a **replay** tool.
- Monitor **DLQ size** and **consumer lag** (Burrow, kafka-exporter → Prometheus/Grafana).

> ⚠️ In Kafka, a consumer stuck retrying one message **blocks its whole partition** (ordering!). Move it aside to a retry topic instead of retrying forever in place.

**Interview one-liner:** Retry transient failures with backoff, park repeated failures in a DLQ so poison pills don't block the stream, and alert on DLQ growth with a replay path.

---

## 8. Trade-offs & what interviewers probe

| Benefit | Cost |
|---------|------|
| Loose coupling, easy to add consumers | Eventual consistency — UI may show "processing" |
| Absorbs traffic spikes | Harder debugging — need correlation IDs + tracing (OpenTelemetry) |
| Fault isolation | Duplicates & ordering issues — idempotency, partition keys |
| Replay/audit (Kafka) | Schema evolution — Schema Registry (Avro/Protobuf) |

| Question | Strong answer |
|----------|---------------|
| "How do you keep order for one customer's events?" | Same partition key (`orderId`/`userId`) → same partition → ordered |
| "Consumer is slow — lag growing?" | Add consumers up to partition count, batch, scale out; partitions cap parallelism |
| "Sync or async for payment?" | Sync for user-facing auth/redirect (Razorpay checkout), async events for downstream (invoice, shipping) |
| "How do you evolve event schemas?" | Backward-compatible changes + Schema Registry compatibility checks |

**Easy rule of thumb:** Use sync calls when the caller **needs the answer now**; use events when the caller just needs to **announce something happened**.

---

## Quick Revision Cheat Sheet

- **EDA**: services publish past-tense events; consumers react async → loose coupling, resilience, eventual consistency.
- **Queue** = one consumer per message (jobs); **Pub/Sub** = every subscriber gets it (fan-out); Kafka groups do both.
- **Kafka** = partitioned retained log, replay, huge throughput, order per partition; **RabbitMQ** = routing broker, deletes on ack, task queues.
- **Delivery**: at-most-once (may lose), at-least-once (may duplicate, the default), exactly-once (only inside Kafka transactions).
- **Idempotent consumer**: unique event ID in a processed-events table, same transaction as the update.
- **Outbox**: business row + event row in one DB transaction, relay/Debezium publishes → no dual-write loss.
- **DLQ**: retry transient errors with backoff, park poison pills, alert + replay; Kafka uses retry/DLT topics.
- **Ordering**: same partition key → ordered; partitions cap consumer parallelism.
