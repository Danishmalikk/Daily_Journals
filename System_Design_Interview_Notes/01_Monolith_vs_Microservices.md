# System Design Notes — Monolith vs Microservices (trade-offs, modular monolith, when to switch, strangler fig)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Monolith?

*Definition:* A **monolith** is an application where all features (users, products, orders, payments, shipping) live in **one codebase**, are built into **one deployable unit**, and usually share **one database**.

Real life: A single big restaurant kitchen. One team, one space, one fridge. Easy to coordinate, but if the kitchen catches fire, nobody eats.

```
            ┌──────────────────────────────────┐
 Client --> │  NestJS / Spring Boot app        │
            │  ├── UsersModule                 │
            │  ├── ProductsModule              │
            │  ├── OrdersModule                │
            │  ├── PaymentsModule (Razorpay)   │
            │  └── ShippingModule (ShipRocket) │
            └───────────────┬──────────────────┘
                            │
                     ┌──────▼──────┐
                     │ PostgreSQL  │  (one shared DB)
                     └─────────────┘
```

**Key Points:**
- One build, one deploy (`docker build` → one image), one process to monitor.
- In-process function calls → fast, no network failures between features.
- A single DB transaction can cover order + payment + stock (ACID for free).
- Pain grows with size: slow builds, risky deploys, one bad memory leak takes everything down.

> ⚠️ "Monolith" is **not** a dirty word. Shopify, Basecamp and Stack Overflow run (modular) monoliths at huge scale.

**Interview one-liner:** A monolith is one codebase, one deployable, usually one DB — simple to build and operate, but it gets harder to scale teams and deploys as it grows.

---

## 2. What are Microservices?

*Definition:* **Microservices** split the system into **small, independently deployable services**, each owning **one business capability** and **its own data**, talking over the network (REST, gRPC, or events via Kafka).

Real life: A food court. Each stall (pizza, biryani, juice) has its own staff, kitchen and menu. One stall closing doesn't shut the food court — but now you need signage, shared seating and a payment counter.

```
                  ┌──────────────┐
 Client --> Nginx/│ API Gateway  │
                  └──┬────┬────┬─┘
          ┌──────────┘    │    └───────────┐
   ┌──────▼─────┐  ┌──────▼─────┐  ┌───────▼──────┐
   │ order-svc  │  │ payment-svc│  │ shipping-svc │
   │ (NestJS)   │  │ (Spring)   │  │ (NestJS)     │
   └──────┬─────┘  └──────┬─────┘  └───────┬──────┘
      [orders DB]    [payments DB]    [shipments DB]
          └──────── Kafka (OrderPlaced, PaymentCaptured) ─────┘
```

**Key Points:**
- **Database per service** — no other service reads your tables directly.
- Each service can scale, deploy and pick its tech independently.
- Requires: service discovery, API gateway, centralized logging, tracing, CI/CD per service.
- Cross-service consistency needs **Sagas / eventual consistency**, not one DB transaction.

**Interview one-liner:** Microservices are small, independently deployable services that each own one business capability and its data, communicating over the network.

---

## 3. Trade-offs — side by side

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Deploy | One unit, all-or-nothing | Per service, independent |
| Scaling | Scale whole app (copy N times) | Scale only the hot service |
| Data consistency | Easy — local ACID transactions | Hard — sagas, eventual consistency |
| Latency | In-process calls (ns) | Network calls (ms) + retries |
| Failure | One bug can crash everything | Isolated, but partial failures everywhere |
| Debugging | One log, one stack trace | Distributed tracing (Jaeger, OpenTelemetry) |
| Team scaling | Merge conflicts past ~20-30 devs | Teams own services end-to-end |
| Tech stack | One language/framework | Polyglot possible |
| Ops cost | Low — one pipeline, one dashboard | High — K8s, service mesh, many pipelines |
| Testing | Simple end-to-end | Contract tests (Pact), many environments |

Memory trick: Microservices trade **code complexity** for **operational complexity**. You don't remove complexity — you **move it to the network**.

> ⚠️ **Distributed monolith** = the worst of both: services that must be deployed together, share one DB, or call each other synchronously in long chains. If `order-svc` can't deploy without `payment-svc`, you have a distributed monolith.

**Interview one-liner:** Monoliths win on simplicity, consistency and latency; microservices win on independent deploys, targeted scaling and team autonomy — at the price of network failures and operational overhead.

---

## 4. The Modular Monolith — the middle ground

*Definition:* A **modular monolith** is a single deployable app internally split into **strictly bounded modules** with clear public interfaces and no reaching into each other's internals or tables.

Real life: One building, but each department has its own locked floor and only talks through the reception desk.

```ts
// ❌ OrdersService reaching straight into another module's repository
constructor(@InjectRepository(Payment) private payments: Repository<Payment>) {}

// ✅ Go through the module's public API (exported service only)
@Module({ providers: [PaymentsService], exports: [PaymentsService] })
export class PaymentsModule {}

constructor(private readonly payments: PaymentsService) {}   // in OrdersService
```

**Key Points:**
- Each module owns its own tables/schema (e.g. Postgres schemas `orders.*`, `payments.*`).
- Enforce boundaries with tooling: Nx module boundaries / eslint rules (Node), **ArchUnit** or **Spring Modulith** (Java).
- Later, a well-bounded module can be **extracted** into a service with little rework.

**Easy rule of thumb:** Start with a modular monolith. If a module's boundary is clean, extracting it later is cheap; if it's messy, microservices would have been a disaster anyway.

**Interview one-liner:** A modular monolith keeps one deployable but enforces service-like boundaries inside it — most of the design benefits of microservices without the network tax.

---

## 5. When to switch (and when NOT to)

### Signals it's time to split
| Signal | Example |
|--------|---------|
| Deploy contention | 5 teams queue for one release train; every deploy needs full regression |
| Uneven scaling | Product search needs 20 pods during a sale, admin panel needs 1 |
| Fault isolation | A PDF-invoice memory leak keeps crashing checkout |
| Different change rates | Pricing changes daily, auth changes yearly |
| Different tech needs | ML recommendations in Python, core in Spring Boot |
| Team size | Conway's Law — org already split into domain teams |

### Signals to stay a monolith
- Small team (< ~10-15 devs) or early startup still finding product-market fit.
- Domain boundaries still changing every month.
- No CI/CD, no centralized logging/monitoring, no container platform yet.

> ⚠️ Interviewers love: *"Would you start a new startup with microservices?"* Good answer: **No** — start with a modular monolith, measure, and extract services when a concrete pain (scaling, deploys, team size) shows up. Cite Martin Fowler's **"MonolithFirst"**.

**Easy rule of thumb:** Split for **organizational** or **scaling** reasons you can name with numbers — never because it's trendy.

**Interview one-liner:** Switch when you have measurable pain — deploy contention, uneven scaling, fault isolation or team growth — and the platform maturity (CI/CD, observability) to run many services.

---

## 6. Strangler Fig Pattern — migrating safely

*Definition:* The **Strangler Fig pattern** migrates a monolith **incrementally**: put a routing layer in front, rebuild one capability at a time as a new service, route its traffic there, and retire the old code — until the monolith is gone (or small).

Real life: A strangler fig vine grows around an old tree, slowly replacing it, until the old tree is gone but the shape stands.

```
Phase 1:  Client ─> Nginx ─────────────────────────> Monolith (all routes)

Phase 2:  Client ─> Nginx ─┬─ /api/shipping/* ─────> shipping-svc (new)
                           └─ everything else ─────> Monolith

Phase 3:  Client ─> Nginx ─┬─ /api/shipping/* ─────> shipping-svc
                           ├─ /api/payments/* ─────> payment-svc
                           └─ rest ────────────────> Monolith (shrinking)
```

```nginx
# Nginx as the "strangler" facade
location /api/shipping/ { proxy_pass http://shipping-svc:3001; }
location /api/          { proxy_pass http://monolith:3000; }
```

**Steps:**
1. Add a facade (Nginx, AWS API Gateway, Kong) in front of the monolith.
2. Pick a **low-risk, well-bounded** capability first (e.g. ShipRocket shipping, notifications).
3. Sync data: the new service gets its own DB, fed via events or CDC (**Debezium**) during transition.
4. Shift traffic gradually (feature flag / canary %), compare results, then delete old code.

> ⚠️ Never do a **big-bang rewrite**. It freezes features for months and usually fails. Strangler fig lets you ship value while migrating and roll back by flipping a route.

**Interview one-liner:** Strangler fig migrates a monolith piece by piece behind a routing facade, so you can move traffic gradually and roll back instantly — no big-bang rewrite.

---

## 7. What interviewers probe

| Question | What they want to hear |
|----------|------------------------|
| "How do services share data?" | They don't share DBs — APIs or events; read models via events/CDC |
| "How do you keep data consistent?" | Saga (choreography/orchestration) + outbox, idempotent consumers |
| "How do you find bugs across services?" | Correlation IDs, OpenTelemetry tracing, centralized logs (ELK/Loki) |
| "How big should a service be?" | One bounded context (DDD), owned by one team — not "100 lines" |
| "What breaks first?" | Sync call chains → cascading failure → need timeouts, circuit breakers |

**Interview one-liner:** Show you know the hidden costs — data ownership, distributed consistency, observability and cascading failures — not just "microservices scale better."

---

## Quick Revision Cheat Sheet

- **Monolith**: one codebase, one deployable, one DB — simple, fast in-process calls, easy ACID.
- **Microservices**: independent services, each owns one capability + its own DB, talk over network.
- **Trade-off**: microservices move complexity from code into the network and operations.
- **Distributed monolith**: shared DB / lockstep deploys — worst of both worlds, avoid.
- **Modular monolith**: one deployable, strict internal boundaries — the recommended starting point.
- **When to switch**: measurable deploy contention, uneven scaling, fault isolation, team growth + mature CI/CD and observability.
- **Strangler fig**: facade in front, extract one capability at a time, shift traffic, retire old code — never big-bang.
- **Interview angle**: data ownership, sagas, tracing, service size = bounded context, cascading failures.
