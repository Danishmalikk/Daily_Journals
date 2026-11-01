# System Design Notes — Rate Limiting & Throttling (Token Bucket, Leaky Bucket, Fixed/Sliding Window, Redis, Nginx, 429)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is rate limiting (and how is throttling different)?

*Definition:* **Rate limiting** caps how many requests a client (user, IP, API key) can make in a time period — e.g. **100 requests/minute** — and rejects the extras with **HTTP 429 Too Many Requests**.

*Definition:* **Throttling** is the softer cousin: instead of (or before) rejecting, it **slows down / queues / degrades** requests so they're served at a controlled pace.

Real life: A nightclub bouncer lets 10 people in per minute (rate limit — the rest are turned away). A toll booth with one lane lets everyone through, just slower (throttling).

| | Rate limiting | Throttling |
|---|---------------|-----------|
| Over the limit | Reject (429) | Delay / queue / reduce quality |
| Goal | Protect + enforce quota | Smooth traffic |
| Example | Login: 5 attempts/min per IP | Nginx `burst` queue, slowing bulk exports |

Why we need it:
- **Abuse protection** — brute-force login/OTP, scraping product prices, credential stuffing.
- **Fairness** — one noisy client shouldn't starve others.
- **Cost / downstream protection** — don't blow through ShipRocket/SMS/Razorpay API quotas.
- **Monetization** — API plans (free 1k/day, pro 100k/day).

**Interview one-liner:** Rate limiting rejects requests beyond a quota with 429; throttling slows or queues them — both protect the system, enforce fairness and control cost.

---

## 2. Fixed Window Counter

*Definition:* **Fixed window** counts requests in fixed time buckets (e.g. each calendar minute); if the count exceeds the limit, reject until the next window.

```
limit = 5 / minute
12:00:00 ─────────── 12:01:00 ─────────── 12:02:00
 [ 1 2 3 4 5 ✗ ✗ ]     [ 1 2 3 ... ]       counter resets at boundary
```

```ts
// Redis: one key per user per minute
const key = `rl:${userId}:${Math.floor(Date.now() / 60000)}`;
const count = await redis.incr(key);
if (count === 1) await redis.expire(key, 60);
if (count > 100) throw new HttpException('Too Many Requests', 429);
```

> ⚠️ **Boundary burst problem:** limit 100/min → a client sends 100 at 12:00:59 and 100 at 12:01:00 = **200 requests in 2 seconds**, all allowed.

**Interview one-liner:** Fixed window is a simple per-window counter (Redis `INCR` + `EXPIRE`) — cheap, but allows up to 2× the limit at window boundaries.

---

## 3. Sliding Window Log & Sliding Window Counter

*Definition:* **Sliding window log** stores the **timestamp of every request** and counts those within the last N seconds — exact, but memory-heavy.

```ts
// Redis sorted set: score = timestamp
const now = Date.now(), windowMs = 60_000;
const res = await redis.multi()
  .zremrangebyscore(key, 0, now - windowMs)    // drop old entries
  .zadd(key, now, `${now}-${Math.random()}`)   // add this request (unique member)
  .zcard(key)                                  // count in window
  .pexpire(key, windowMs)
  .exec();
const count = res[2][1] as number;
if (count > 100) reject429();
```

*Definition:* **Sliding window counter** approximates the log using just **two counters** (current + previous window), weighted by how far you are into the current window.

```
estimated = prev_count × (1 − elapsed_in_current / window) + curr_count

window = 60s, now 15s into current minute
prev = 80, curr = 30  →  80 × 0.75 + 30 = 90  → under 100 ✅
```

| | Sliding log | Sliding counter |
|---|-------------|-----------------|
| Accuracy | Exact | Approximate (very close in practice) |
| Memory per client | O(requests in window) | O(1) — 2 numbers |
| Used by | Small limits (login 5/min) | Large-scale APIs (Cloudflare uses this idea) |

**Interview one-liner:** Sliding log keeps every timestamp (exact, memory-heavy, Redis ZSET); sliding counter weights the previous window's count to smooth the boundary burst with O(1) memory.

---

## 4. Token Bucket — the industry favourite

*Definition:* **Token bucket**: a bucket holds up to **capacity** tokens and refills at a **fixed rate**. Each request takes a token; no token → reject. It allows **short bursts** up to capacity while enforcing the average rate.

Real life: A metro card with a top-up every second. You can save up and spend a few at once (burst), but long-term you can't spend faster than it refills.

```
refill: 10 tokens/sec  ──►  ┌──────────────┐  capacity = 20
                            │ ● ● ● ● ● ●  │
                            └──────┬───────┘
          request ─► take 1 token ─┘   empty? → 429
```

Implementation trick — no background refill timer; compute lazily on each request:
```
tokens = min(capacity, tokens + (now − lastRefill) × rate)
if tokens ≥ 1 → allow, tokens -= 1  else → reject
```

Used by: **AWS API Gateway** (rate + burst), **Stripe**, Guava `RateLimiter` (a smooth variant), **Bucket4j** (Java), Resilience4j RateLimiter (a fixed-refresh-period variant).

**Interview one-liner:** Token bucket refills tokens at a fixed rate up to a capacity; each request spends one — it enforces an average rate while allowing controlled bursts, which is why most APIs use it.

---

## 5. Leaky Bucket — smooth constant output

*Definition:* **Leaky bucket**: requests go into a **queue (bucket)** of fixed size and are processed ("leak") at a **constant rate**. If the bucket is full, new requests are dropped.

Real life: A funnel — pour water in as fast as you like, it drips out at one steady speed; overflow spills.

```
bursty input ─► ┌─────────┐
  ▓▓▓  ▓ ▓▓▓▓   │ queue   │ full? → drop (429/503)
                │ ▓▓▓▓▓   │
                └────┬────┘
                     ▼  constant rate: 10 req/s → backend
```

| | Token bucket | Leaky bucket |
|---|--------------|--------------|
| Bursts | **Allowed** (up to capacity) | **Smoothed** into constant rate |
| Output rate | Variable | Constant |
| Extra latency | None | Queued requests wait |
| Good for | Public APIs, user-facing | Protecting a fragile backend, outbound calls to a quota'd API |
| Example | AWS API Gateway | **Nginx `limit_req`** |

Memory trick: **Token = "Take & burst"**, **Leaky = "Level & steady"**.

**Interview one-liner:** Leaky bucket queues requests and drains them at a constant rate (smooth output, extra latency); token bucket lets bursts through up to capacity — Nginx uses leaky bucket, AWS API Gateway uses token bucket.

---

## 6. Distributed rate limiting with Redis

*Definition:* A **distributed rate limiter** keeps counters in a **shared store** (Redis) so the limit holds across **all** app instances, not per instance.

```
            ┌─► NestJS pod 1 ─┐
Client ─► LB┼─► NestJS pod 2 ─┼─► Redis (shared counters, atomic Lua)
            └─► NestJS pod 3 ─┘
In-memory limit of 100/min × 3 pods = 300/min ❌   Redis = 100/min total ✅
```

Token bucket as an **atomic Lua script** (read-modify-write in one step, no race):

```lua
-- KEYS[1]=bucket  ARGV: capacity, refillPerSec, nowMs, cost
local capacity = tonumber(ARGV[1])
local rate     = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local cost     = tonumber(ARGV[4])
local d = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(d[1]) or capacity
local ts     = tonumber(d[2]) or now
tokens = math.min(capacity, tokens + (now - ts) / 1000 * rate)
local allowed = 0
if tokens >= cost then tokens = tokens - cost; allowed = 1 end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(capacity / rate * 1000))
return { allowed, math.floor(tokens) }
```

```ts
const [allowed, remaining] = await redis.eval(
  luaScript, 1, `tb:${apiKey}`, 20, 10, Date.now(), 1) as [number, number];
```

> ⚠️ **Race condition:** `GET` then `SET` from app code lets two pods both read "99" and both allow → use `INCR` (atomic) or a Lua script.
> ⚠️ **Redis down?** Decide **fail-open** (allow traffic, log it — usual for general APIs) vs **fail-closed** (block — for login/OTP). Add a short timeout on the Redis call.
> ⚠️ Clock skew across pods: pass Redis `TIME` instead of app time if it matters.

Tools: `@nestjs/throttler` + Redis storage (`@nest-lab/throttler-storage-redis`), `rate-limiter-flexible` (Node), **Bucket4j** + Redis/Hazelcast (Java), Spring Cloud Gateway `RequestRateLimiter` (Redis token bucket built in).

**Interview one-liner:** For multiple instances, keep limiter state in Redis and update it atomically (INCR or a Lua script); decide fail-open vs fail-closed for when Redis is unavailable.

---

## 7. In practice — Nginx, NestJS, and the 429 response

**Nginx (edge, per IP — leaky bucket):**

```nginx
http {
  limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;   # 10m ≈ 160k IPs
  limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

  server {
    location /api/        { limit_req zone=api burst=20 nodelay; }
    location /api/login   { limit_req zone=login burst=3; }
    limit_req_status 429;          # default is 503!
  }
}
```
- `burst=20` = queue up to 20 extra; `nodelay` = serve the burst immediately instead of spacing them out.

**NestJS (app, per user/route):**

```ts
@Module({
  imports: [ThrottlerModule.forRoot([{ name: 'default', ttl: 60_000, limit: 100 }])], // ttl in ms (v5+)
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule {}

@Throttle({ default: { limit: 5, ttl: 60_000 } })   // stricter on OTP
@Post('otp/send') sendOtp() { /* ... */ }
```

**A good 429 response:**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1793520000
Content-Type: application/json

{ "error": "rate_limited", "message": "Try again in 30 seconds" }
```
- `Retry-After` is standard (RFC 9110); `X-RateLimit-*` are de-facto (GitHub, Twitter); IETF is standardizing `RateLimit` / `RateLimit-Policy` headers.

> ⚠️ Behind a load balancer/Cloudflare, `$binary_remote_addr` is the **proxy IP** → everyone shares one bucket. Use `real_ip_header X-Forwarded-For` + `set_real_ip_from <trusted proxy>` (or `CF-Connecting-IP`).

**Interview one-liner:** Coarse per-IP limits at the edge (Nginx `limit_req`, Cloudflare/AWS WAF), fine-grained per-user/API-key limits in the app or gateway, and always return 429 with `Retry-After`.

---

## 8. Choosing an algorithm + what interviewers probe

| Algorithm | Bursts | Memory | Accuracy | Pick when |
|-----------|--------|--------|----------|-----------|
| Fixed window | 2× at edges | O(1) | Low | Simple quotas (daily API limit) |
| Sliding log | No | O(N) | Exact | Small strict limits (login, OTP) |
| Sliding counter | Smoothed | O(1) | High | Large-scale general APIs |
| Token bucket | Yes, capped | O(1) | High | Public APIs needing burst tolerance (default) |
| Leaky bucket | No (queued) | O(queue) | High | Steady outbound to fragile/quota'd services |

**What to limit by:** IP (anonymous), user id (logged in), API key/tenant (B2B plans), route (login vs product listing), and cost (a heavy report = 10 tokens).

Common interview questions ("design a rate limiter"):
1. Where? → client SDK, edge (CDN/WAF/Nginx), API gateway, service. Usually **edge + gateway**.
2. Which algorithm and why? → token bucket for bursts, sliding window counter for accuracy at scale.
3. Distributed? → Redis + Lua, key = `rl:{tenant}:{route}`, TTL on keys.
4. Latency? → one Redis round trip (~1ms); local pre-check cache for very hot paths.
5. Failure mode? → fail-open vs fail-closed; timeouts on Redis.
6. Client experience? → 429 + `Retry-After`, client exponential backoff.

**Easy rule of thumb:** Token bucket by default, sliding log for login/OTP, leaky bucket when calling someone else's quota'd API.

**Interview one-liner:** Choose by burst tolerance and memory: token bucket for most APIs, sliding windows for accuracy, leaky bucket for steady output — enforce in Redis at the edge/gateway and tell clients when to retry.

---

## Quick Revision Cheat Sheet

- **Rate limiting** rejects with **429**; **throttling** slows/queues; both protect, ensure fairness, control cost.
- **Fixed window**: `INCR`+`EXPIRE` per window; cheap but 2× burst at boundaries.
- **Sliding log**: ZSET of timestamps, exact, O(N) memory; **sliding counter**: weighted prev+curr, O(1).
- **Token bucket**: refill rate + capacity → average rate with bursts; AWS API Gateway, Stripe, Bucket4j.
- **Leaky bucket**: queue drained at constant rate → smooth output, adds latency; Nginx `limit_req`.
- **Distributed**: shared Redis, atomic Lua/INCR, avoid GET-then-SET races; choose fail-open vs fail-closed.
- **Practice**: Nginx `limit_req_zone` + `burst`/`nodelay` + `limit_req_status 429`; NestJS `ThrottlerGuard` (+ Redis storage); real client IP behind proxies.
- **Response**: 429 + `Retry-After` + `X-RateLimit-Limit/Remaining/Reset`.
- **Design**: limit by IP/user/key/route/cost; edge + gateway; token bucket default.
