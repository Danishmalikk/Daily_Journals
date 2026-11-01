# System Design Notes — gRPC vs REST vs GraphQL (Protobuf, HTTP/2, streaming, over/under-fetching, N+1)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The big picture — three API styles

*Definition:* **REST**, **gRPC** and **GraphQL** are three ways for a client to talk to a server. REST exposes **resources** over HTTP, gRPC exposes **remote functions** with a binary contract, and GraphQL exposes **one typed graph** that clients query for exactly the fields they need.

Real life:
- **REST** = a restaurant menu: fixed dishes (`/orders/42`), you get the whole plate.
- **gRPC** = an intercom straight to the chef: fast, strict, short coded messages.
- **GraphQL** = a buffet with a waiter: you list exactly what you want on one plate.

```
REST:     GET /orders/42  → JSON (server decides shape)
gRPC:     OrderService.GetOrder({id:42})  → binary Protobuf over HTTP/2
GraphQL:  POST /graphql  { order(id:42){ total items{ name } } }  → JSON (client decides shape)
```

**Interview one-liner:** REST = resources over HTTP, gRPC = typed remote procedure calls over HTTP/2 + Protobuf, GraphQL = one endpoint where the client asks for exactly the data shape it wants.

---

## 2. REST — the default for public APIs

*Definition:* **REST** uses URLs for resources, HTTP methods for actions, and (usually) JSON payloads.

```
GET /orders/42           → { id, total, status, userId, ... }
GET /orders/42/items     → [ ... ]
GET /users/7             → { ... }        (3 round trips for one screen)
```

**Strengths:** universal (browsers, curl, every language), HTTP caching (CDN, `ETag`, `Cache-Control`), easy to debug, huge tooling (OpenAPI/Swagger, Postman).

**Weaknesses:**
- **Over-fetching** — you get 30 fields when the mobile card needs 3.
- **Under-fetching** — one screen needs several calls (order + items + user).
- JSON is text → bigger and slower to parse than binary.

**Interview one-liner:** REST is simple, cacheable and universal — the best default for public and browser-facing APIs — but suffers from over- and under-fetching.

---

## 3. gRPC — fast service-to-service calls

*Definition:* **gRPC** is Google's RPC framework: you define services and messages in a **`.proto`** file, generate client/server code, and calls travel as **binary Protobuf over HTTP/2**.

```proto
syntax = "proto3";
package inventory;

service InventoryService {
  rpc ReserveStock (ReserveRequest) returns (ReserveReply);          // unary
  rpc WatchStock   (StockQuery)     returns (stream StockUpdate);     // server streaming
}

message ReserveRequest { string sku = 1; int32 quantity = 2; string order_id = 3; }
message ReserveReply   { bool reserved = 1; int32 remaining = 2; }
message StockQuery     { string sku = 1; }
message StockUpdate    { string sku = 1; int32 available = 2; }
```

```
order-svc (NestJS, @grpc/grpc-js) ──HTTP/2, binary──> inventory-svc (Spring Boot, grpc-java)
          generated TS stub                              generated Java base class
```

### 4 call types
| Type | Shape | Example |
|------|-------|---------|
| Unary | 1 request → 1 response | ReserveStock |
| Server streaming | 1 → many | Live stock updates |
| Client streaming | many → 1 | Upload batch of scans from warehouse |
| Bidirectional | many ↔ many | Chat, live tracking |

**Strengths:** compact binary (often several times smaller than JSON), HTTP/2 multiplexing, strict typed contract + codegen, built-in deadlines and streaming.

**Weaknesses:** browsers can't call it directly (need **gRPC-Web** + Envoy proxy), not human-readable (use `grpcurl`), no simple HTTP caching, L7 load balancing required (long-lived HTTP/2 connections defeat plain L4 balancers).

> ⚠️ Protobuf **field numbers are the contract**. Never reuse or change a field number; mark removed ones `reserved`. Renaming a field is safe on the wire, changing its number or type is not.

**Interview one-liner:** gRPC uses Protobuf contracts over HTTP/2 for fast, strongly-typed, streaming-capable calls — ideal for internal microservice traffic, awkward for browsers.

---

## 4. GraphQL — client-shaped queries

*Definition:* **GraphQL** exposes a typed **schema**; clients send a **query** to one endpoint (`/graphql`) and get back **exactly** the fields they asked for — no more, no less.

```graphql
# Schema
type Order { id: ID!  total: Float!  status: String!  items: [Item!]!  user: User! }
type Query  { order(id: ID!): Order }
type Mutation { cancelOrder(id: ID!): Order }

# Client query — one round trip for the whole screen
query {
  order(id: "42") {
    total
    items { name quantity }
    user  { name }
  }
}
```

**Strengths:** no over/under-fetching, one round trip per screen, strongly typed schema + introspection (GraphiQL, Apollo Studio), evolves without versions (add fields, `@deprecated` old ones). Great as a **BFF** (Backend-for-Frontend) aggregating many services.

**Weaknesses:**
- **N+1 problem** — resolving `user` for 50 orders fires 50 queries. Fix with **DataLoader** (batch + cache per request).
- HTTP caching is hard (everything is `POST /graphql`) → use persisted queries / Apollo client cache.
- Query cost attacks — deeply nested queries can melt the DB → **depth limits, complexity limits, persisted queries**.
- Errors often return `200` with an `errors` array → monitoring needs care.

```ts
// NestJS + DataLoader — batch 50 user lookups into one SQL IN (...)
const userLoader = new DataLoader<number, User>(async (ids) => {
  const users = await userRepo.findBy({ id: In([...ids]) });
  return ids.map((id) => users.find((u) => u.id === id)!);
});
```

**Interview one-liner:** GraphQL lets clients fetch exactly what a screen needs in one request from a typed schema, but you must handle N+1 (DataLoader), caching and query-cost limits yourself.

---

## 5. Side-by-side comparison

| Feature | REST | gRPC | GraphQL |
|---------|------|------|---------|
| Contract | OpenAPI (optional) | `.proto` (mandatory) | Schema SDL (mandatory) |
| Payload | JSON (text) | Protobuf (binary) | JSON (text) |
| Transport | HTTP/1.1 or 2 | HTTP/2 | HTTP (usually POST) |
| Endpoints | Many (per resource) | Methods on services | One (`/graphql`) |
| Who shapes response | Server | Server (fixed messages) | Client |
| Streaming | SSE / WebSockets bolt-on | Native (4 modes) | Subscriptions (WebSocket) |
| HTTP caching | Excellent | None | Hard |
| Browser support | Native | Needs gRPC-Web proxy | Native |
| Performance | Good | Best | Good (resolver overhead) |
| Learning curve | Low | Medium | Medium-high |
| Versioning | `/v1`, `/v2` | Field numbers, new packages | Add fields, `@deprecated` |
| Tools | Swagger, Postman | protoc, buf, grpcurl | Apollo, GraphiQL, Hasura |

Memory trick: **"REST for the world, gRPC for the backend, GraphQL for the frontend."**

---

## 6. When to use which

| Scenario | Pick | Why |
|----------|------|-----|
| Public API for partners / webhooks (like Razorpay, ShipRocket) | **REST** | Universal, easy to document and cache |
| Internal service-to-service, high QPS, low latency (order → inventory) | **gRPC** | Binary, typed, HTTP/2, deadlines |
| Real-time streaming between services (live tracking, telemetry) | **gRPC** streaming | Native bidirectional streams |
| Mobile + web apps with many different screens | **GraphQL** (as BFF) | Each screen fetches exactly its data in one trip |
| Aggregating 5 microservices into one client view | **GraphQL** gateway (Apollo Federation) | One graph over many services |
| Simple CRUD admin, small team | **REST** | Least moving parts |
| Heavily cached public content (product catalog via CDN) | **REST** | GET + CDN caching just works |

```
               ┌─── GraphQL BFF (web/mobile) ───┐
 Browser/App ──┤                                ├──gRPC──> order-svc ──gRPC──> inventory-svc
               └─── REST public API (partners) ─┘           └── Kafka events ──> shipping-svc
```

> ⚠️ It's **not either/or**. Real systems mix them: REST/GraphQL at the edge, gRPC inside, Kafka for async. Saying "we'd use all three, each where it fits" is a strong interview answer.

**Easy rule of thumb:** Default to REST. Reach for gRPC when internal latency/throughput or streaming matters; reach for GraphQL when many different clients need flexible, aggregated data.

**Interview one-liner:** REST for public and cacheable APIs, gRPC for fast typed internal calls and streaming, GraphQL for flexible client-driven data aggregation — most mature systems use a mix.

---

## 7. What interviewers probe

| Question | Strong answer |
|----------|---------------|
| "Why is gRPC faster?" | Binary Protobuf (smaller, faster to parse) + HTTP/2 multiplexing over one persistent connection + no JSON reflection |
| "How do you solve N+1 in GraphQL?" | DataLoader batching per request; also query planning in federation |
| "How do you secure GraphQL?" | Depth/complexity limits, persisted queries, field-level auth, rate limit by cost |
| "How do you load-balance gRPC?" | L7 balancer (Envoy, Linkerd, Istio, Nginx `grpc_pass`) or client-side LB — L4 sticks all calls to one pod |
| "Can a browser call gRPC?" | Not directly — gRPC-Web or Connect via Envoy proxy |
| "Does GraphQL replace REST?" | No — it's a query layer; it often sits on top of REST/gRPC services |

---

## Quick Revision Cheat Sheet

- **REST**: resources + HTTP verbs + JSON; universal, cacheable; suffers over/under-fetching.
- **gRPC**: `.proto` contract, binary Protobuf over HTTP/2, codegen, 4 streaming modes; best for internal service calls; browsers need gRPC-Web.
- **Protobuf rule**: never reuse/change field numbers — mark removed ones `reserved`.
- **GraphQL**: one endpoint, typed schema, client picks fields; great BFF/aggregator.
- **GraphQL pitfalls**: N+1 (DataLoader), hard HTTP caching, query cost attacks (depth/complexity limits), 200-with-errors.
- **Memory trick**: REST for the world, gRPC for the backend, GraphQL for the frontend.
- **Default**: REST; gRPC for latency/streaming internally; GraphQL for many clients needing flexible data.
- **Real systems mix all three** + Kafka for async.
