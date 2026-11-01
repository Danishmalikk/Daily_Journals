# System Design Notes — Circuit Breaker Pattern (Resilience4j: States, Fallback, Retry, Timeout, Bulkhead)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem — cascading failure

*Definition:* A **cascading failure** is when one slow or dead downstream service drags down every service that calls it, because callers keep waiting on it and run out of threads/connections.

Real life: One cashier at a supermarket freezes. The queue behind them grows, people block the aisle, and soon the whole store jams — even the cashiers who were fine.

```
User ─► Order Service ─► Payment Service ─► Razorpay API (slow: 30s per call 🐢)
            │                  │
   threads waiting...   threads waiting...
   pool exhausted ❌    pool exhausted ❌
   → /orders, /cart, /products ALL time out, even ones not using payments
```

- Slow is **worse** than down: a fast error frees the thread; a 30s hang holds it.
- With 200 Tomcat threads and a 30s hang, ~7 requests/sec is enough to freeze the service.

**Interview one-liner:** Without protection, one slow dependency exhausts the caller's threads and connections, and the failure cascades up the call chain — resilience patterns stop that.

---

## 2. Circuit Breaker — the three states

*Definition:* A **circuit breaker** wraps calls to a dependency, tracks failures, and when the failure rate crosses a threshold it **opens** — failing fast **without calling** the dependency — then periodically **tests** whether it has recovered.

Real life: The electrical MCB in your house. Too much current → it trips (open) to protect the wiring. After fixing it, you flip it back and see if it holds.

```
              failure rate ≥ threshold
   ┌────────┐ ─────────────────────────► ┌────────┐
   │ CLOSED │                            │  OPEN  │  calls rejected instantly
   │(normal)│ ◄──────┐                   │(fail   │  (CallNotPermittedException)
   └────────┘        │                   │ fast)  │
       ▲     trial calls OK              └────────┘
       │             │                       │ waitDurationInOpenState (e.g. 10s)
       │       ┌───────────┐                 ▼
       └───────│ HALF_OPEN │ ◄───────────────┘
               │ few trial │ ── trial calls fail ──► back to OPEN
               │  calls    │
               └───────────┘
```

| State | Calls go through? | What it does |
|-------|-------------------|--------------|
| **CLOSED** | Yes | Records success/failure in a sliding window |
| **OPEN** | No — fail fast | Waits, gives the dependency time to recover |
| **HALF_OPEN** | Only N trial calls | Decides: recovered → CLOSED, still broken → OPEN |

Memory trick: **"Closed = Current flows, Open = Off, Half-open = Have a peek."** (Like a real circuit: closed circuit means electricity flows.)

> ⚠️ Classic mix-up: **OPEN means blocked**, not "open for business". Say it carefully in interviews.

**Interview one-liner:** A circuit breaker moves CLOSED → OPEN when the failure rate crosses a threshold, fails fast while OPEN, then lets a few trial calls through in HALF_OPEN to decide whether to close again.

---

## 3. Resilience4j configuration (Spring Boot)

*Definition:* **Resilience4j** is the lightweight Java fault-tolerance library (successor to Netflix **Hystrix**, which has been in maintenance mode since 2018). It provides CircuitBreaker, Retry, TimeLimiter, Bulkhead and RateLimiter as composable decorators.

```yaml
# application.yml  (dependency: resilience4j-spring-boot3 + spring-boot-starter-aop)
resilience4j:
  circuitbreaker:
    instances:
      razorpay:
        slidingWindowType: COUNT_BASED       # or TIME_BASED (last N seconds)
        slidingWindowSize: 20                # look at last 20 calls
        minimumNumberOfCalls: 10             # don't judge on 2 calls
        failureRateThreshold: 50             # % failures to OPEN
        slowCallDurationThreshold: 2s        # a call > 2s counts as "slow"
        slowCallRateThreshold: 60            # % slow calls also OPENs it
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        registerHealthIndicator: true        # shows in /actuator/health
        ignoreExceptions:
          - com.shop.payment.InvalidCardException   # business error ≠ outage
```

```java
@Service
class PaymentClient {

    @CircuitBreaker(name = "razorpay", fallbackMethod = "createOrderFallback")
    public RazorpayOrder createOrder(long amountPaise) {
        return razorpayHttp.createOrder(amountPaise);   // remote call
    }

    // same params + Throwable at the end, same return type
    private RazorpayOrder createOrderFallback(long amountPaise, Throwable ex) {
        throw new PaymentTemporarilyUnavailable("Payments are busy, retry in a minute", ex);
    }
}
```

> ⚠️ **Ignore business errors.** A `400 invalid card` is not the gateway being down. If you count it as failure, a burst of bad cards opens the breaker for everyone.

**Interview one-liner:** In Resilience4j you configure a sliding window (count or time based), a failure-rate and slow-call threshold, the open-state wait, and the number of half-open trial calls — and exclude business exceptions.

---

## 4. Fallbacks — what to do instead

*Definition:* A **fallback** is the alternative response returned when the call fails or the breaker is open.

| Fallback type | E-commerce example |
|---------------|--------------------|
| Cached / stale data | Product recommendations from Redis cache |
| Default value | Shipping ETA "3–5 days" when ShipRocket serviceability API is down |
| Degraded feature | Hide "recommended for you" widget |
| Queue for later | Accept order, put "create shipment" on Kafka, retry later |
| Fail fast with a clear error | Payment: "try again in a minute" (never fake success!) |

> ⚠️ Never fake success for money paths. A fallback that says "payment OK" when Razorpay timed out = real money bug. Payments should fail clearly or reconcile via webhook.

**Easy rule of thumb:** Reads → fall back to cache/default. Writes → queue for later or fail clearly.

**Interview one-liner:** Fallbacks provide a degraded but safe response — cached data, defaults, a hidden feature or a queued retry — but critical writes like payments should fail clearly, not pretend to succeed.

---

## 5. Retry — try again, carefully

*Definition:* **Retry** re-invokes a failed call a limited number of times, ideally with **exponential backoff + jitter**, for **transient** errors (timeouts, 503, connection reset).

Real life: A call drops — you redial after a moment, not 50 times per second.

```yaml
resilience4j:
  retry:
    instances:
      shiprocket:
        maxAttempts: 3                      # total attempts incl. the first
        waitDuration: 500ms
        enableExponentialBackoff: true      # 500ms → 1s → 2s
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException  # 4xx: don't retry
```

> ⚠️ **Retry storms**: 3 layers × 3 retries = 27 calls to an already-dying service. Retry at **one** layer, add **jitter** so clients don't retry in sync, and let the circuit breaker cut retries off.

> ⚠️ **Only retry idempotent operations.** Retrying `POST /payments` can charge twice. Use an **idempotency key** (Razorpay `receipt`/your own key) so a repeat is safe.

**Interview one-liner:** Retry only transient failures on idempotent operations, with capped attempts, exponential backoff and jitter — and at one layer only, to avoid retry storms.

---

## 6. Timeouts — the most important one

*Definition:* A **timeout** caps how long you wait for a response. Resilience4j's **TimeLimiter** does this for async calls (`CompletableFuture`).

```yaml
resilience4j:
  timelimiter:
    instances:
      razorpay:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

```java
@TimeLimiter(name = "razorpay")
@CircuitBreaker(name = "razorpay", fallbackMethod = "fallback")
public CompletableFuture<RazorpayOrder> createOrderAsync(long amount) {
    return CompletableFuture.supplyAsync(() -> razorpayHttp.createOrder(amount));
}
```

- ❌ Default HTTP clients often have **no** read timeout (or a very long one) → infinite hang.
- ✅ Always set **connect** (~1s) and **read** (~2–5s) timeouts on `RestTemplate`/`WebClient`/axios, and DB/Redis clients.
- Timeout budget: if the API gateway gives 10s, the inner calls must add up to less.

> ⚠️ `@TimeLimiter` only works on async return types (`CompletableFuture`, reactive types). For a blocking `RestTemplate`, set the timeout on the HTTP client itself.

**Interview one-liner:** Every remote call needs a timeout — set connect/read timeouts on the client and use TimeLimiter for async calls; without timeouts a circuit breaker reacts far too late.

---

## 7. Bulkhead — isolate the blast radius

*Definition:* A **bulkhead** limits how many concurrent calls can go to one dependency, so one slow dependency can't eat **all** threads.

Real life: A ship's hull is split into watertight compartments. One floods, the ship still floats.

```
Without bulkhead:  [ 200 shared threads ] → all stuck on ShipRocket → whole app dead
With bulkhead:     [ShipRocket: max 20] [Razorpay: max 30] [rest of app: free threads]
```

| Type | How | Use when |
|------|-----|----------|
| **SemaphoreBulkhead** | Caps concurrent calls on the caller's thread | Default, cheap, blocking code |
| **ThreadPoolBulkhead** | Separate thread pool + queue per dependency | Strong isolation, async calls |

```yaml
resilience4j:
  bulkhead:
    instances:
      shiprocket:
        maxConcurrentCalls: 20
        maxWaitDuration: 0          # reject immediately when full (BulkheadFullException)
```

**Interview one-liner:** A bulkhead caps concurrency per dependency (semaphore or dedicated thread pool) so one slow integration can only take its own slice of resources, not the whole service.

---

## 8. Putting it together — order, Node.js, monitoring

Resilience4j Spring Boot default aspect order (outer → inner):

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( yourCall ) ) ) ) )
```
- Retry is outermost → each retry attempt is recorded by the breaker; once OPEN, retries fail instantly.

NestJS / Node equivalent — **opossum**:

```ts
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(callShiprocketRates, {
  timeout: 3000,                  // ms → counts as failure
  errorThresholdPercentage: 50,   // open at 50% failures
  resetTimeout: 10000,            // OPEN → HALF_OPEN after 10s
  volumeThreshold: 10,            // min calls before judging
});
breaker.fallback(() => ({ etaDays: '3-5', source: 'default' }));
breaker.on('open', () => logger.warn('ShipRocket breaker OPEN'));

const rates = await breaker.fire(pincode);
```

Monitoring: Resilience4j exports Micrometer metrics (`resilience4j_circuitbreaker_state`, `..._failure_rate`, `..._calls_seconds`) → **Prometheus** → **Grafana** alert "breaker OPEN > 1 min". Service meshes (**Istio**/**Envoy** outlier detection) can do breaking at the network layer instead of in code.

**Interview one-liner:** Combine them as Retry → CircuitBreaker → TimeLimiter → Bulkhead around the call, use opossum in Node, and alert on breaker state via Prometheus/Grafana.

---

## 9. When to use which + what interviewers probe

| Problem | Pattern |
|---------|---------|
| Call can hang | **Timeout** (always) |
| Brief network blip | **Retry** with backoff + jitter |
| Dependency down for minutes | **Circuit breaker** + fallback |
| One dependency eating all threads | **Bulkhead** |
| Protect a dependency from you | **Rate limiter** (client-side) |

Common interview questions:
1. *"What's HALF_OPEN for?"* → safely probe recovery with a few calls instead of flooding a recovering service.
2. *"Retry vs circuit breaker?"* → retry handles short blips; breaker stops calling during long outages. Use both, breaker wraps the call, retry outside.
3. *"Where do you put it — gateway or service?"* → per dependency, at the caller (each service's outbound client), optionally also in the gateway/mesh.
4. *"What counts as failure?"* → 5xx, timeouts, connection errors, slow calls — not 4xx business errors.
5. *"Hystrix?"* → in maintenance mode; Resilience4j is the current choice in Spring Cloud (`spring-cloud-starter-circuitbreaker-resilience4j`).

**Easy rule of thumb:** Timeout everything, retry rarely, break on sustained failure, isolate with bulkheads.

**Interview one-liner:** Timeouts are mandatory, retries for transient blips, circuit breakers for sustained outages, bulkheads for isolation — together they stop one bad dependency from taking down the system.

---

## Quick Revision Cheat Sheet

- **Cascading failure**: slow dependency exhausts caller threads → whole chain dies; slow is worse than down.
- **Circuit breaker**: CLOSED (normal) → OPEN (fail fast, `CallNotPermittedException`) → HALF_OPEN (trial calls) → CLOSED/OPEN.
- **Resilience4j config**: sliding window (count/time), `failureRateThreshold`, slow-call threshold, `waitDurationInOpenState`, half-open calls; ignore 4xx business errors.
- **Fallback**: cache/default/degrade/queue for reads & async; payments fail clearly, never fake success.
- **Retry**: transient errors only, idempotent ops only, backoff + jitter, one layer only (avoid retry storms).
- **Timeout**: always set connect/read timeouts; `@TimeLimiter` needs `CompletableFuture`.
- **Bulkhead**: cap concurrency per dependency — semaphore (cheap) or thread pool (strong isolation).
- **Order**: Retry(CircuitBreaker(RateLimiter(TimeLimiter(Bulkhead(call))))); Node → opossum; metrics → Prometheus/Grafana.
- **Choose**: timeout always, retry blips, breaker outages, bulkhead isolation; Hystrix is legacy.
