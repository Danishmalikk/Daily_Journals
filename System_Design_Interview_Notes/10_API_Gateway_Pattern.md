# System Design Notes — API Gateway Pattern (Routing, Auth, Aggregation, BFF, Spring Cloud Gateway, Kong, AWS, Nginx)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem — clients talking to every service directly

*Definition:* Without a gateway, every client (web, Android, iOS, partner) must know the address of **every** microservice and re-implement auth, retries and rate limiting against each one.

Real life: A hotel without a reception desk — guests wander to the kitchen for food, laundry room for clothes, manager's office for billing. Nobody checks who they are.

```
Without a gateway:                         With a gateway:
Mobile ──► user-svc                        Mobile ─┐
Mobile ──► product-svc                     Web    ─┼─► API Gateway ─┬─► user-svc
Mobile ──► order-svc                       Partner┘   (one door)    ├─► product-svc
Web    ──► user-svc, order-svc ...                                  ├─► order-svc
  • N URLs, N auth checks, CORS everywhere                          └─► payment-svc
  • internal services exposed publicly ❌
  • a mobile screen = 5 round trips over 4G 🐢
```

**Interview one-liner:** Without a gateway, clients couple to internal service layout, every service repeats auth/rate limiting, and mobile clients pay for many round trips — the gateway gives one controlled entry point.

---

## 2. What is an API Gateway? (responsibilities)

*Definition:* An **API Gateway** is a single entry point that sits in front of your services and handles **cross-cutting concerns** — routing, authentication, rate limiting, TLS, logging — so services can focus on business logic.

Memory trick: **"RAT-CLAMP"** → **R**outing, **A**uth, **T**LS termination, **C**aching, **L**ogging/metrics, **A**ggregation, **M**odification (headers/protocol), **P**olicing (rate limits, quotas).

| Responsibility | Example in an e-commerce stack |
|----------------|--------------------------------|
| **Routing** | `/api/orders/**` → order-svc, `/api/products/**` → product-svc |
| **Authentication** | Validate JWT once; forward `X-User-Id` to services |
| **Rate limiting / quotas** | 100 req/min per user, 5/min on `/otp` |
| **TLS termination** | HTTPS at the edge, HTTP/mTLS inside |
| **Request/response transform** | Add headers, strip internal fields, REST ↔ gRPC |
| **Aggregation** | One `/home` call → products + offers + cart count |
| **Caching** | Cache `GET /products` for 60s |
| **Observability** | Correlation ID (`X-Request-Id`), access logs, Prometheus metrics |
| **Resilience** | Timeouts, circuit breaker, fallback per route |
| **Canary / versioning** | 5% of traffic to `order-svc:v2`, `/v1` vs `/v2` routing |

> ⚠️ The gateway should be **smart about traffic, dumb about business**. No order-total calculation or discount rules in the gateway — that becomes a new monolith.

**Interview one-liner:** An API gateway is the single entry point that handles routing, auth, rate limiting, TLS, transformation, aggregation and observability for all backend services — without containing business logic.

---

## 3. Routing & request aggregation

*Definition:* **Routing** maps an incoming path/host/header to a backend service. **Aggregation (API composition)** is when the gateway calls several services **in parallel** and merges the results into one response.

```
GET /api/mobile/order/42  (1 call from phone)
          │
     API Gateway  ──parallel──► order-svc     /orders/42
                  ──parallel──► shipping-svc  /shipments?order=42   (ShipRocket AWB, status)
                  ──parallel──► payment-svc   /payments?order=42    (Razorpay status)
          │
          ▼  merged JSON  { order, shipment, payment }   ~1 round trip instead of 3
```

Spring Cloud Gateway routing (reactive, Netty-based):

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service          # lb:// = resolve via service discovery
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1                # /api/orders/42 → /orders/42
            - name: CircuitBreaker
              args:
                name: ordersCB
                fallbackUri: forward:/fallback/orders
            - name: RequestRateLimiter     # Redis token bucket
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
```

> ⚠️ Newer Spring Cloud Gateway releases (2025.x) moved properties under `spring.cloud.gateway.server.webflux.*`; the older `spring.cloud.gateway.routes` form above is what most tutorials and interviews use.

> ⚠️ Aggregation latency = the **slowest** downstream. Use per-call timeouts and return partial data (e.g. order without live shipment status) rather than failing the whole page.

**Interview one-liner:** The gateway routes by path/host/header (often resolving services through discovery with `lb://`) and can aggregate several backend calls in parallel into one client response, bounded by per-call timeouts.

---

## 4. Auth and security at the gateway

*Definition:* **Edge authentication** means the gateway validates the token (JWT signature, expiry, audience) **once**, rejects bad requests with 401/403, and forwards trusted identity to services.

```
Client ──Bearer JWT──► Gateway ──verify signature (JWKS), exp, aud──► ✅
                              └─ adds X-User-Id: 123, X-Roles: customer
                                 strips the client-sent X-User-Id first! ──► order-svc
```

```nginx
# Nginx: delegate auth to an auth service (auth_request module)
location /api/ {
    auth_request /_auth;                              # 2xx = allow, 401/403 = deny
    auth_request_set $user_id $upstream_http_x_user_id;
    proxy_set_header X-User-Id $user_id;              # overwrites any client value
    proxy_pass http://backend;
}
location = /_auth {
    internal;
    proxy_pass http://auth-service/verify;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
}
```

> ⚠️ **Header spoofing:** if the gateway forwards `X-User-Id` but services are also reachable directly, anyone can send `X-User-Id: 1`. Keep services on a private network/security group and always overwrite identity headers at the gateway.

> ⚠️ **Zero trust:** gateway auth is coarse ("is this a valid user?"). Services still enforce **authorization** ("does user 123 own order 42?").

Webhooks (Razorpay, ShipRocket) also enter through the gateway, but are authenticated by **signature verification** (`X-Razorpay-Signature` HMAC) in the service, not JWT.

**Interview one-liner:** Validate tokens once at the gateway and forward a trusted identity header, but keep services private, overwrite client-sent identity headers, and still do fine-grained authorization inside each service.

---

## 5. Market tools — which gateway?

| Tool | Type | Strengths | Watch out |
|------|------|-----------|-----------|
| **Nginx / OpenResty** | Reverse proxy (config-driven) | Super fast, you already run it, `limit_req`, `auth_request` | No dashboard/plugins ecosystem; logic in config/Lua |
| **Spring Cloud Gateway** | Library (Java, reactive) | Native with Eureka, Resilience4j, Redis rate limiter; code-level filters | JVM service you must operate; Java-centric |
| **Kong** (on Nginx/OpenResty) | Standalone gateway | Plugin ecosystem (JWT, OAuth2, rate-limiting, Prometheus), DB-less YAML, Kong Konnect | Advanced features in Enterprise |
| **AWS API Gateway** | Fully managed | No servers, usage plans + API keys, Cognito/Lambda authorizers, WAF, throttling | Per-request cost at scale, ~29s default integration timeout, vendor lock-in |
| **Envoy / Istio ingress, Traefik, Apigee, Azure APIM, Tyk, KrakenD** | Various | Cloud-native/L7, enterprise API management, K8s-native | Pick per platform |

Kong DB-less example:

```yaml
_format_version: "3.0"
services:
  - name: order-service
    url: http://order-service:3000
    routes:
      - name: orders
        paths: [/api/orders]
    plugins:
      - name: jwt
      - name: rate-limiting
        config: { minute: 100, policy: local }
```

**Easy rule of thumb:** Small stack on EC2/Docker → **Nginx**. Spring Cloud microservices → **Spring Cloud Gateway**. Polyglot + want plugins → **Kong**. Serverless/Lambda or "no ops" → **AWS API Gateway** (HTTP API is cheaper than REST API).

**Interview one-liner:** Nginx for simple fast proxying, Spring Cloud Gateway for Spring ecosystems, Kong for a plugin-driven polyglot gateway, AWS API Gateway when you want it fully managed — chosen by platform, team skills and cost.

---

## 6. BFF — Backend for Frontend

*Definition:* **BFF (Backend for Frontend)** means a **separate gateway/API layer per client type** (web BFF, mobile BFF, partner API), each shaping data exactly for that client.

Real life: One restaurant kitchen, but a separate menu for dine-in, delivery app and kids — same food, different packaging.

```
 Web app     ──► Web BFF (NestJS)     ─┐    rich payloads, SSR-friendly
 Mobile app  ──► Mobile BFF (NestJS)  ─┼──► order-svc, product-svc, user-svc ...
 Partner API ──► Public API gateway   ─┘    stable versioned contract, API keys
```

| | Single general gateway | BFF per client |
|---|------------------------|----------------|
| Payload | One-size-fits-all (over/under-fetching) | Tailored per screen |
| Ownership | Platform team, bottleneck | Each frontend team owns its BFF |
| Release speed | Every client change waits on one team | Mobile changes don't break web |
| Cost | One thing to run | More services, possible duplicated logic |

> ⚠️ Common combo: a thin **edge gateway** (TLS, WAF, rate limiting — e.g. AWS API Gateway/Nginx) in front of **BFFs** (aggregation, client shaping). **GraphQL** (Apollo Federation) is often used as a BFF alternative.

**Interview one-liner:** BFF gives each client type its own backend layer to tailor responses and let frontend teams move independently — often behind a thin edge gateway for TLS, WAF and rate limiting.

---

## 7. API Gateway vs Load Balancer vs Reverse Proxy vs Service Mesh

| | Reverse proxy | Load balancer | API gateway | Service mesh |
|---|---------------|---------------|-------------|--------------|
| Main job | Forward requests, hide backends | Spread traffic across instances | Manage APIs for external clients | Service-to-service traffic |
| Layer | L7 | L4 (NLB) or L7 (ALB) | L7, API-aware | L4/L7 sidecars |
| Traffic | North-south | North-south / internal | **North-south** (into the system) | **East-west** (between services) |
| Auth / quotas / API keys | Basic | No | **Yes** | mTLS, policies |
| Examples | Nginx, HAProxy | AWS ALB/NLB | Kong, SCG, AWS API GW | Istio, Linkerd, Consul Connect |

Memory trick: **Gateway guards the front door (north-south); mesh manages the hallways (east-west).**

> They stack: `Client → CDN/WAF → ALB → API Gateway (2+ instances) → services (+ mesh sidecars)`.

**Interview one-liner:** A load balancer spreads traffic, a reverse proxy forwards it, an API gateway adds API management (auth, quotas, routing, aggregation) for external traffic, and a service mesh handles internal east-west traffic.

---

## 8. Trade-offs, pitfalls & what interviewers probe

| Pros | Cons |
|------|------|
| One entry point, hides internal topology | **Single point of failure** if not HA |
| Cross-cutting concerns in one place | Extra hop → +1–5 ms latency |
| Smaller, cleaner services | Can become a bottleneck / "god gateway" |
| Easier client evolution, versioning, canaries | Another component to deploy, monitor, scale |

Rules:
- Run **2+ instances** behind a load balancer (or use managed); it must be **stateless** (rate-limit state in Redis).
- Keep **business logic out**; push it into services or BFFs.
- Set **timeouts lower than the client's** and a per-route circuit breaker.
- Generate a **correlation ID** at the gateway and propagate it (OpenTelemetry traces → Grafana Tempo/Jaeger).
- Monitor p95 latency and 4xx/5xx per route in Prometheus/Grafana — the gateway is the best place to see the whole system's health.

Common interview questions:
1. *"Gateway is a SPOF — how do you fix it?"* → multiple stateless instances, health-checked behind an LB, multi-AZ, or managed service.
2. *"Where does auth happen?"* → authentication at the gateway, authorization in services.
3. *"Gateway vs service discovery?"* → gateway is the external entry; it uses discovery (`lb://`) to find internal instances.
4. *"How do you version APIs at the gateway?"* → route `/v1` vs `/v2` or by header to different service versions.
5. *"When NOT to use one?"* → a monolith or 2 services behind Nginx; don't add Kong + BFFs on day one.

**Easy rule of thumb:** Monolith → Nginx reverse proxy. A handful of microservices → one gateway. Multiple client types with diverging needs → edge gateway + BFFs.

**Interview one-liner:** An API gateway simplifies clients and centralizes cross-cutting concerns, but must be stateless, highly available and free of business logic — otherwise it becomes the new bottleneck or monolith.

---

## Quick Revision Cheat Sheet

- **Problem**: clients calling every service directly → coupling, duplicated auth, exposed internals, many mobile round trips.
- **API gateway**: single entry point; **RAT-CLAMP** = Routing, Auth, TLS, Caching, Logging, Aggregation, Modification, Policing.
- **Routing/aggregation**: path/host/header → service (`lb://` via discovery); parallel fan-out bounded by the slowest call → timeouts + partial responses.
- **Auth**: validate JWT once, forward trusted `X-User-Id`, overwrite client headers, keep services private; authorization still in services.
- **Tools**: Nginx (simple, fast), Spring Cloud Gateway (Spring + Redis limiter + Resilience4j), Kong (plugins), AWS API Gateway (managed, usage plans, 29s default timeout).
- **BFF**: one backend layer per client type (web/mobile/partner), often behind a thin edge gateway; GraphQL is an alternative.
- **vs others**: LB spreads, reverse proxy forwards, gateway manages APIs (north-south), mesh handles east-west.
- **Trade-offs**: SPOF/latency/bottleneck → stateless HA instances, no business logic, correlation IDs, per-route metrics.
