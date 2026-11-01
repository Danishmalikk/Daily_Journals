# System Design Notes — REST API Design Best Practices (resources, verbs, status codes, idempotency, pagination, errors)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Resources, not actions

*Definition:* **REST** (Representational State Transfer) models the API as **resources** (nouns) identified by URLs, manipulated with standard **HTTP methods** (verbs). The URL says *what*, the method says *what to do*.

Real life: A library. Books are resources on shelves (`/books/42`). You don't need a different door for "borrow", "return", "inspect" — you use the same shelf with different actions.

```
❌ Action-style (RPC in disguise)       ✅ Resource-style
POST /createOrder                       POST   /orders
GET  /getOrder?id=42                    GET    /orders/42
POST /updateOrderAddress                PATCH  /orders/42
POST /deleteOrder                       DELETE /orders/42
GET  /getOrdersOfUser?userId=7          GET    /users/7/orders
```

### Naming rules
- **Plural nouns**: `/orders`, `/products`, not `/order`.
- **lowercase, kebab-case**: `/shipping-labels`, not `/shippingLabels`.
- **Nest only one level** for ownership: `/users/7/orders` ✅, `/users/7/orders/42/items/3/reviews` ❌ — prefer `/order-items/3/reviews`.
- For real actions that aren't CRUD, use a sub-resource: `POST /orders/42/cancel` or `POST /payments/99/refunds`.

**Interview one-liner:** REST URLs are plural nouns for resources; HTTP methods carry the action — keep verbs out of the path except for genuine non-CRUD actions modeled as sub-resources.

---

## 2. HTTP methods and their guarantees

*Definition:* **Safe** = doesn't change server state. **Idempotent** = calling it once or N times leaves the server in the **same state**.

| Method | Use for | Safe | Idempotent | Typical success code |
|--------|---------|------|------------|----------------------|
| GET | Read a resource/collection | ✅ | ✅ | 200 |
| POST | Create / trigger an action | ❌ | ❌ | 201 (+ `Location` header) / 202 |
| PUT | Replace whole resource | ❌ | ✅ | 200 / 204 |
| PATCH | Partial update | ❌ | ❌ (not guaranteed) | 200 / 204 |
| DELETE | Remove resource | ❌ | ✅ | 204 (or 404 on repeat — state is still "gone") |

Memory trick: **"GPD are repeat-safe"** → **G**ET, **P**UT, **D**ELETE are idempotent; POST and PATCH aren't by default.

> ⚠️ Idempotent is about **server state**, not the response. A second `DELETE /orders/42` may return 404 instead of 204 — that's fine, the order is still deleted.

**Interview one-liner:** GET is safe; GET, PUT and DELETE are idempotent; POST is neither — so POST needs extra protection against retries.

---

## 3. Status codes that actually matter

| Code | Meaning | When I use it |
|------|---------|---------------|
| 200 OK | Success with body | GET, PUT/PATCH returning resource |
| 201 Created | New resource created | POST /orders (+ `Location: /orders/42`) |
| 202 Accepted | Accepted, processing async | Bulk import, report generation queued in Kafka |
| 204 No Content | Success, no body | DELETE |
| 400 Bad Request | Malformed / validation failed | Missing `quantity`, bad JSON |
| 401 Unauthorized | Not authenticated | No/expired JWT |
| 403 Forbidden | Authenticated but not allowed | Customer hitting admin route |
| 404 Not Found | Resource doesn't exist | `/orders/999999` |
| 409 Conflict | State conflict | Duplicate email, order already shipped |
| 422 Unprocessable Entity | Valid syntax, business rule fails | Coupon expired |
| 429 Too Many Requests | Rate limited | + `Retry-After` header |
| 500 / 502 / 503 / 504 | Server / upstream / unavailable / timeout | Bug, Razorpay down, deploy, slow upstream |

Memory trick: **4xx = "you (client) messed up", 5xx = "we (server) messed up".** 401 = "who are you?", 403 = "I know you, but no."

> ⚠️ Never return `200 OK` with `{ "success": false }` for errors. Monitoring (Prometheus `http_requests_total{status=~"5.."}`), retries and caches all rely on the real status code.

**Interview one-liner:** Use status codes honestly — 2xx success, 4xx client errors, 5xx server errors — because clients, proxies, retries and dashboards all depend on them.

---

## 4. Idempotency keys — making POST safe to retry

*Definition:* An **idempotency key** is a unique client-generated ID sent with a non-idempotent request; the server stores the result for that key and returns the **same result** on retries instead of doing the work twice.

Real life: A cheque number. If the bank sees cheque #1042 twice, it pays once.

```
Client ── POST /payments  Idempotency-Key: 7f3c-... ──> API
                                          │
                     Redis: SET idem:7f3c NX EX 86400 ?
                     ├── new key   → process charge, store response under key
                     └── seen key  → return stored response (no second charge)
```

```ts
// NestJS sketch
const key = req.headers['idempotency-key'];
const cached = await redis.get(`idem:${key}`);
if (cached) return JSON.parse(cached);                  // replay

const locked = await redis.set(`idem:${key}:lock`, '1', 'EX', 30, 'NX');
if (!locked) throw new ConflictException('Request in progress');

const result = await this.payments.charge(dto);         // the real work
await redis.set(`idem:${key}`, JSON.stringify(result), 'EX', 86400);
return result;
```

> ⚠️ Classic failure: the user double-clicks "Pay", or the mobile app retries on a timeout → **two charges**. Stripe and Razorpay both use this pattern; make your own order/payment APIs do it too. Also enforce a **unique DB constraint** (e.g. on `razorpay_payment_id`) as the last line of defense.

**Interview one-liner:** For POSTs that must not repeat (payments, orders), accept an Idempotency-Key, store the first result, and replay it on retries — backed by a unique DB constraint.

---

## 5. Pagination, filtering, sorting

*Definition:* **Pagination** returns a large collection in chunks so neither the DB nor the client chokes.

| Style | Request | Pros | Cons |
|-------|---------|------|------|
| Offset | `?page=3&limit=20` → `OFFSET 40 LIMIT 20` | Simple, jump to any page | Slow on deep pages (DB scans skipped rows); duplicates/skips when data changes |
| Cursor / keyset | `?limit=20&cursor=eyJpZCI6MTIzfQ` → `WHERE id < 123 ORDER BY id DESC LIMIT 20` | Fast at any depth (uses index), stable | No "jump to page 50" |

```
GET /orders?status=SHIPPED&createdAfter=2026-10-01&sort=-createdAt&limit=20&cursor=abc

{
  "data": [ ... 20 orders ... ],
  "page": { "nextCursor": "def", "hasMore": true }
}
```

**Key Points:**
- Always enforce a **max `limit`** (e.g. 100) — never let `?limit=100000` through.
- Filtering: plain query params (`?status=PAID`); sorting: `sort=-createdAt` (minus = desc).
- Sparse fields to cut payload: `?fields=id,total,status`.

**Easy rule of thumb:** Admin tables with page numbers → offset. Infinite scroll, feeds, large tables, exports → cursor.

**Interview one-liner:** Paginate every collection with an enforced max limit; prefer cursor (keyset) pagination for large or fast-changing data because offset gets slow and inconsistent at depth.

---

## 6. Consistent error format

*Definition:* A **standard error body** lets every client handle errors with one piece of code. **RFC 9457 (Problem Details, formerly RFC 7807)** is the industry standard.

```json
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.oceanevoke.com/errors/coupon-expired",
  "title": "Coupon expired",
  "status": 422,
  "detail": "Coupon DIWALI50 expired on 2026-10-25",
  "instance": "/orders/42",
  "traceId": "9f2a1c..."
}
```

**Key Points:**
- Same shape everywhere — in NestJS use a global `ExceptionFilter`; in Spring use `@RestControllerAdvice` (Spring 6 has built-in `ProblemDetail`).
- Include a **traceId / correlation ID** so support can find the log line in Grafana/Loki.
- ❌ Never leak stack traces, SQL or internal hostnames in the response.
- Validation errors: list each field — `"errors": [{ "field": "quantity", "message": "must be > 0" }]`.

**Interview one-liner:** Return one consistent error shape (RFC 9457 Problem Details) with a real status code, a machine-readable type, a human message and a trace ID — never stack traces.

---

## 7. Other best practices interviewers check

| Practice | Why |
|----------|-----|
| **HTTPS only**, auth via `Authorization: Bearer <JWT>` | Never tokens in query strings (they land in Nginx logs) |
| **Versioning** (`/v1/...`) | Evolve without breaking clients (see next file) |
| **Rate limiting** + `429` + `Retry-After` | Protect from abuse/bursts |
| **Caching**: `ETag` / `If-None-Match` → `304`, `Cache-Control` | Saves bandwidth, enables CDN caching |
| **Optimistic concurrency**: `If-Match: <etag>` → `412` on mismatch | Prevents lost updates on PUT/PATCH |
| **Docs**: OpenAPI/Swagger (`@nestjs/swagger`, springdoc) | Contract-first, client generation |
| **Long tasks** → `202` + status URL | `POST /reports` → `202`, poll `GET /reports/77` |

---

## 8. HATEOAS (briefly)

*Definition:* **HATEOAS** (Hypermedia As The Engine Of Application State) means responses include **links** to the next possible actions, so clients navigate the API like a website.

```json
{
  "id": 42, "status": "PAID",
  "_links": {
    "self":   { "href": "/orders/42" },
    "cancel": { "href": "/orders/42/cancel", "method": "POST" },
    "ship":   { "href": "/orders/42/shipments", "method": "POST" }
  }
}
```

> It's Level 3 of the **Richardson Maturity Model** (L0 = one endpoint/RPC, L1 = resources, L2 = HTTP verbs + status codes, L3 = hypermedia). Most real-world "REST" APIs stop at Level 2 — that's fine to admit in an interview. Spring HATEOAS supports it if you need it.

**Interview one-liner:** HATEOAS adds links to responses so clients discover allowed actions; it's REST maturity Level 3, but most production APIs pragmatically stop at Level 2.

---

## Quick Revision Cheat Sheet

- **Resources**: plural nouns (`/orders/42`), verbs come from HTTP methods; non-CRUD actions as sub-resources (`/orders/42/cancel`).
- **Methods**: GET safe; GET/PUT/DELETE idempotent; POST/PATCH not.
- **Status codes**: 201 + Location on create, 204 on delete, 4xx client fault, 5xx server fault; never 200 for errors.
- **Idempotency key**: store first result per key (Redis + unique DB constraint) → retries don't double-charge.
- **Pagination**: always cap `limit`; offset for simple admin pages, cursor/keyset for large or changing data.
- **Errors**: one shape — RFC 9457 Problem Details + traceId, no stack traces.
- **Extras**: HTTPS, Bearer JWT, versioning, rate limits (429), ETag caching, If-Match concurrency, OpenAPI docs, 202 for async.
- **HATEOAS**: links in responses, Richardson Level 3 — most APIs stop at Level 2.
