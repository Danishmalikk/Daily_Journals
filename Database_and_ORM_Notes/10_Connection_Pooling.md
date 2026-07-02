# Database Notes — Connection Pooling (HikariCP internals)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Why connection pooling exists

Opening a database connection is **expensive**: TCP handshake, authentication, session setup — it can take tens of milliseconds. If every request opened a fresh connection, your app would crawl under load.

*Definition:* **Connection pooling** keeps a set of **already-open** database connections ready in a "pool." When your app needs one, it **borrows** an existing connection, uses it, and **returns** it to the pool (instead of closing it). No repeated open/close cost.

Real life: A library of pre-printed books you borrow and return — versus printing a brand-new book every time someone wants to read. The pool reuses connections.

```
Without pool:  open → query → close   (open/close every time = slow)
With pool:     borrow → query → return (reuse open connections = fast)
```

**Interview one-liner:** Connection pooling reuses a set of pre-opened DB connections — apps borrow and return them — avoiding the heavy cost of opening a new connection per request.

---

## 2. How a pool works

```
   ┌─────────────── Connection Pool ───────────────┐
   │  [conn1] [conn2] [conn3] [conn4] [conn5]       │  ← idle, ready
   └────────────────────────────────────────────────┘
        ↑ borrow                      ↓ return
   App request  → uses conn2 → done → puts conn2 back

   If all are busy → new request WAITS (up to a timeout) for one to free up,
   or the pool opens a new connection (until it hits the max size).
```

- App asks the pool for a connection → gets an idle one.
- Uses it for the query, then **returns** it (calling `connection.close()` actually returns it to the pool, not really closing it!).
- If none free and pool is at max → request waits up to `connectionTimeout`, then errors.

> Key insight for interviews: with a pool, **`close()` doesn't close** the physical connection — it hands it back to the pool. That's why pooling is transparent to your code.

---

## 3. HikariCP — the default in Spring Boot

*Definition:* **HikariCP** is a very fast, lightweight connection pool library — the **default** in Spring Boot. "Hikari" means "light" in Japanese; it's known for being the fastest pool with the smallest overhead.

Why it's fast (the "internals" interviewers ask about):
- **Lightweight & lock-minimized** code paths.
- A custom **`FastList`** instead of `ArrayList` (skips range-checks).
- A **`ConcurrentBag`** structure for lock-free connection handoff between threads.
- Avoids unnecessary work on borrow/return; bytecode kept tiny.

> You rarely configure HikariCP manually — Spring Boot auto-configures it. You just tune its settings.

**Interview one-liner:** HikariCP is Spring Boot's default connection pool — fast and lightweight thanks to lock-free structures (ConcurrentBag, FastList) and minimal overhead on borrow/return.

---

## 4. The key settings (must-know)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # max connections in the pool
      minimum-idle: 10             # min idle connections kept ready
      connection-timeout: 30000    # ms to wait for a free connection before erroring
      idle-timeout: 600000         # ms before an idle connection is removed
      max-lifetime: 1800000        # ms max life of a connection, then recycled
```

| Setting | Meaning | 
|---------|---------|
| **maximum-pool-size** | Max total connections (the most important one) |
| **minimum-idle** | Connections kept ready even when idle |
| **connection-timeout** | How long a thread waits for a connection before failing |
| **idle-timeout** | Remove idle connections after this long |
| **max-lifetime** | Retire & replace a connection after this long (avoids stale connections) |

> **`max-lifetime` should be a bit less than the DB's own timeout** (e.g. MySQL `wait_timeout`), so Hikari retires connections *before* the database kills them — preventing "connection is closed" errors. Classic gotcha.

---

## 5. Sizing the pool (the counter-intuitive part)

Beginners think "more connections = faster." **Wrong.** Too many connections overwhelm the database (each uses CPU/memory and they fight over locks).

> HikariCP's own guidance: a **small** pool is usually better. A common starting formula:
```
pool size ≈ (CPU cores × 2) + effective_spindle_count
```
For many apps, a pool of **10** handily serves thousands of users, because each query is held only briefly. Bigger isn't better — measure under load.

**Easy rule of thumb:** Start small (e.g. 10), monitor for waiting threads, and increase only if needed. An oversized pool can *hurt* DB performance.

**Interview one-liner:** Bigger pools aren't faster — connections are held briefly, so a small pool (often ~10) serves heavy load; oversizing overwhelms the database.

---

## 6. Signs of pool problems (real-world)

| Symptom | Likely cause |
|---------|--------------|
| `Connection is not available, request timed out` | Pool exhausted — too few connections, or connections not returned (leak) |
| Connections slowly disappearing | **Connection leak** — code borrowed but never returned (missing `close()`/try-with-resources) |
| Random "connection closed" errors | `max-lifetime` ≥ DB timeout — connections going stale |

Leak protection:
```yaml
spring.datasource.hikari.leak-detection-threshold: 30000   # warn if held >30s
```

> Connection pooling pairs with `@Transactional`: a transaction borrows one connection for its whole duration. Long transactions hold connections longer → can exhaust the pool. Keep transactions short.

---

## 7. Other pools (awareness)

| Pool | Note |
|------|------|
| **HikariCP** | Default in Spring Boot; fastest, lightest |
| **Tomcat JDBC Pool** | Older Spring Boot default (pre-2.0) |
| **Apache DBCP2** | Mature, feature-rich, heavier |
| **C3P0** | Older, legacy |

---

## Quick Revision Cheat Sheet

- **Connection pooling** = reuse pre-opened DB connections (borrow → use → return); avoids costly open/close per request.
- With a pool, **`close()` returns** the connection to the pool, doesn't really close it.
- **HikariCP** = Spring Boot's default; fast/light via lock-free **ConcurrentBag** + **FastList**.
- Key settings: **maximum-pool-size** (most important), minimum-idle, connection-timeout, idle-timeout, **max-lifetime**.
- Set **max-lifetime < DB wait_timeout** to avoid stale "connection closed" errors.
- **Small pools win** — ~10 often serves thousands; oversizing hurts the DB.
- Watch for **connection leaks** (use try-with-resources; `leak-detection-threshold`).
- Keep **transactions short** — they hold a connection the whole time.
