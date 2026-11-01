# System Design Notes — API Versioning Strategies (URI, header, query param, media type, compatibility, deprecation)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Why version an API at all?

*Definition:* **API versioning** is a way to change an API's contract **without breaking existing clients** — old clients keep using the old version while new clients move to the new one.

Real life: A phone charger standard. When USB-C arrives, shops still sell micro-USB cables for a few years so nobody's old phone suddenly stops charging.

```
Mobile app v3.1 (in Play Store, can't force update) ──> /v1/orders  ✅ still works
New web checkout                                     ──> /v2/orders  ✅ new shape
```

**Key Points:**
- You **don't control** when clients upgrade — mobile apps, partner integrations (ShipRocket webhooks, B2B resellers) lag months.
- Public/partner APIs need versioning most; internal APIs between services you own can often evolve with coordinated deploys + contract tests.
- Goal: **evolve without breaking**, and version only when you truly must break.

**Interview one-liner:** Versioning lets you change an API contract while existing clients — which you can't force to upgrade — keep working.

---

## 2. Breaking vs non-breaking changes

*Definition:* A **breaking change** makes a previously valid client request or response handling fail. A **non-breaking (backward-compatible)** change doesn't.

| Change | Breaking? |
|--------|-----------|
| Add a new optional response field | ✅ Non-breaking (if clients ignore unknown fields) |
| Add a new endpoint | ✅ Non-breaking |
| Add a new **optional** request param | ✅ Non-breaking |
| Remove or rename a field | ❌ Breaking |
| Change a field's type (`price: 499` → `price: "499.00"`) | ❌ Breaking |
| Make an optional request field **required** | ❌ Breaking |
| Change status code semantics / error format | ❌ Breaking |
| Add a new **enum value** in a response | ⚠️ Often breaking (strict client `switch` fails) |
| Change default sort/page size | ⚠️ Behavioral break |

Memory trick: **"Add = safe, Remove/Rename/Retype = new version."**

> ⚠️ **Tolerant Reader / Postel's Law:** clients should ignore unknown fields. In Java, Jackson fails on unknown properties by default (`FAIL_ON_UNKNOWN_PROPERTIES` — Spring Boot's auto-configured mapper disables it, plain `ObjectMapper` doesn't). Make your clients tolerant so additive changes never break them.

**Interview one-liner:** Additive changes (new optional fields/endpoints) are safe; removing, renaming or retyping fields — or tightening requirements — is breaking and needs a new version.

---

## 3. The four versioning strategies

### A. URI path versioning
```
GET /v1/orders/42
GET /v2/orders/42
```

### B. Query parameter
```
GET /orders/42?version=2
```

### C. Custom header
```
GET /orders/42
X-API-Version: 2            (or Api-Version: 2026-10-01 — date-based, Stripe style)
```

### D. Media type (content negotiation)
```
GET /orders/42
Accept: application/vnd.oceanevoke.order.v2+json
```

| Strategy | Visibility | Caching | Ease (browser/curl) | Purity | Used by |
|----------|-----------|---------|---------------------|--------|---------|
| URI `/v1/` | High — in every log & URL | Easy — different URL | Easiest | "Not RESTful" (URL should = resource) | Twitter/X, Google, most public APIs |
| Query param | Medium | OK (part of URL) | Easy | Meh — mixes with filters | Some AWS/Azure APIs (`api-version=`) |
| Header | Low — hidden | Needs `Vary: X-API-Version` | Need tooling | Clean URLs | Stripe (`Stripe-Version`), Azure |
| Media type | Low | Needs `Vary: Accept` | Hardest | Most "RESTful" | GitHub (`application/vnd.github+json`) |

Memory trick: **"PQHM"** → **P**ath, **Q**uery, **H**eader, **M**edia type — ordered from **most visible** to **most hidden**.

**Easy rule of thumb:** Default to **URI versioning** — it's obvious, cache-friendly, easy to route in Nginx/API Gateway, and every interviewer accepts it. Pick header/date versioning if you need fine-grained, per-client evolution like Stripe.

**Interview one-liner:** Four options — path, query, header, media type; path versioning is the pragmatic default for visibility and routing, header/media type keep URLs clean at the cost of discoverability and caching complexity.

---

## 4. Implementing it (NestJS + Spring Boot)

```ts
// NestJS — built-in versioning
app.enableVersioning({ type: VersioningType.URI });   // or HEADER / MEDIA_TYPE

@Controller({ path: 'orders', version: '1' })
export class OrdersV1Controller { @Get(':id') find() { /* old shape */ } }

@Controller({ path: 'orders', version: '2' })
export class OrdersV2Controller { @Get(':id') find() { /* new shape */ } }
// → GET /v1/orders/42  and  GET /v2/orders/42
```

```java
// Spring Boot — path versioning (Spring Framework 7 also adds first-class API versioning)
@RestController
@RequestMapping("/v2/orders")
class OrderV2Controller {
    @GetMapping("/{id}")
    OrderV2Dto get(@PathVariable Long id) { return mapper.toV2(service.find(id)); }
}

// header-based alternative
@GetMapping(value = "/orders/{id}", headers = "X-API-Version=2")
```

```nginx
# Or route versions to different deployments at the edge
location /v1/ { proxy_pass http://orders-v1:3000; }
location /v2/ { proxy_pass http://orders-v2:3000; }
```

> ⚠️ Don't copy the whole service per version. Keep **one domain/service layer** and version only the **controller + DTO mapping**. Duplicated business logic across v1/v2 drifts and causes bugs.

**Interview one-liner:** Version at the edge — controllers and DTOs — while sharing one domain layer underneath, so versions differ only in contract, not in business rules.

---

## 5. Backward compatibility techniques (avoid new versions)

*Definition:* **Backward compatibility** means new server code still works for old clients. The best versioning strategy is needing fewer versions.

**Techniques:**
- **Expand → migrate → contract**: add the new field alongside the old one, migrate clients, then remove the old one in the next major version.
- **Defaults**: new request fields are optional with sensible defaults.
- **Feature flags / opt-in**: `?include=shipments` instead of changing the default payload.
- **Consumer-driven contract tests** (**Pact**, Spring Cloud Contract) catch breaks in CI before deploy.
- **Schema registries** for events (Confluent Schema Registry with Avro/Protobuf compatibility modes) — same idea for Kafka messages.

```json
// Expand phase — both fields served during migration
{ "amount": 49900, "amountMoney": { "value": "499.00", "currency": "INR" } }
```

**Interview one-liner:** Prefer expand-migrate-contract, optional fields with defaults and contract tests so most changes never need a new version.

---

## 6. Deprecation & sunset — retiring old versions

*Definition:* **Deprecation** announces a version will be removed; **sunset** is the date it actually stops working.

```
Timeline:  v2 launched ──> v1 deprecated (headers + email) ──> 6-12 months ──> v1 sunset (410 Gone)
```

```
HTTP/1.1 200 OK
Deprecation: @1767225600                          (RFC 9745 — date it was deprecated)
Sunset: Thu, 30 Apr 2027 00:00:00 GMT             (RFC 8594)
Link: <https://docs.oceanevoke.com/migrate-v2>; rel="deprecation"
```

**Key Points:**
- **Measure before removing**: track traffic per version (Prometheus label `api_version`, Grafana panel) and per client/API key.
- Communicate: changelog, docs banner, emails to API-key owners, deprecation headers.
- Support policy: e.g. "N-1 versions supported, 12 months minimum".
- After sunset, return **410 Gone** with a link to the migration guide — not a silent 404.
- Brownouts: briefly disable v1 for an hour before sunset to flush out forgotten clients (GitHub does this).

> ⚠️ Every live version is **code you must maintain, test and secure**. Supporting 5 versions forever is a real cost — have a sunset policy from day one.

**Interview one-liner:** Deprecate with headers and communication, watch per-version traffic, give a fixed migration window, then sunset with 410 Gone.

---

## 7. What interviewers probe

| Question | Strong answer |
|----------|---------------|
| "Which strategy would you pick?" | URI for public APIs (visible, cache/route friendly); mention trade-offs of header/media type |
| "When do you bump the version?" | Only for breaking changes; additive changes ship in place |
| "How do you version mobile APIs?" | Long overlap windows + min-supported-app-version check + forced-update screen |
| "How do you version events/messages?" | Schema registry, backward-compatible Avro/Protobuf, version field in envelope |
| "Major vs minor in the URL?" | Only major (`/v2`), never `/v2.3.1` — minor/patch must be non-breaking |

---

## Quick Revision Cheat Sheet

- **Why version**: change contracts without breaking clients you can't force to upgrade.
- **Breaking**: remove/rename/retype fields, make optional → required, new enum values (often). **Non-breaking**: add optional fields/endpoints.
- **Strategies (PQHM)**: Path `/v1`, Query `?version=`, Header `X-API-Version`, Media type `Accept: vnd...v2+json`.
- **Default choice**: URI path — visible, cacheable, easy Nginx/gateway routing; header/date style (Stripe) for fine-grained evolution.
- **Implementation**: version controllers + DTOs only; share one domain layer.
- **Avoid versions**: expand-migrate-contract, defaults, tolerant readers, Pact contract tests, schema registry for events.
- **Deprecation**: `Deprecation` + `Sunset` headers, per-version metrics, migration window, then 410 Gone.
- **Only major versions in the URL**; minor changes must stay backward compatible.
