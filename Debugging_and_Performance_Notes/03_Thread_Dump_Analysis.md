# Debugging & Performance Notes — Thread Dump Analysis (jstack, jcmd Thread.print, Thread States, Deadlocks, High-CPU Threads, Pool Starvation, Virtual Threads)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a thread dump? (and how to take one)

*Definition:* A **thread dump** is a text snapshot of **every thread in the JVM** — its name, state, CPU time, full stack trace, and which locks it holds or waits for. It answers "**what is every thread doing right now?**" — the go-to tool for hangs, slow requests, deadlocks and high CPU.

Real life: Freezing a busy kitchen for one second and photographing every cook: who's chopping, who's waiting for the oven, who's stuck because two cooks each hold the knife the other needs.

```bash
jcmd 1 Thread.print -l > td-1.txt        # ✅ preferred; -l adds j.u.c lock (ownable synchronizer) info
jstack -l 1 > td-1.txt                   # classic equivalent
kill -3 1                                # SIGQUIT → dump goes to the JVM's STDOUT (= container logs)
kubectl exec order-svc-7d9 -- jcmd 1 Thread.print -l > td-1.txt
jcmd 1 Thread.dump_to_file -format=json /tmp/threads.json   # JDK 21+, includes virtual threads
```

| Method | Output goes to | Notes |
|--------|----------------|-------|
| `jcmd <pid> Thread.print -l` | Your terminal | Modern, recommended |
| `jstack -l <pid>` | Your terminal | Legacy but everywhere |
| `kill -3 <pid>` | JVM stdout → `kubectl logs` | Works without JDK tools in the image |
| VisualVM / JMC "Thread Dump" | GUI | Local/staging |
| Spring Boot `/actuator/threaddump` | HTTP (JSON or text) | Lock it down — never public |

> ⚠️ Thread dumps are cheap (short safepoint pause) — far safer than heap dumps. Taking several in prod is fine.

**Interview one-liner:** A thread dump is a snapshot of every thread's state, stack and locks; I take it with `jcmd <pid> Thread.print -l` or `kill -3` to diagnose hangs, deadlocks and CPU spikes.

---

## 2. Thread states — what each one really means

*Definition:* Every Java thread is in one of six `Thread.State` values; the state plus the **top frames** tells you what it's waiting on.

| State | Meaning | Typical stack top |
|-------|---------|-------------------|
| **NEW** | Created, not started | (rarely seen) |
| **RUNNABLE** | Running on CPU **or** in native I/O | Your code, regex, JSON; `socketRead`/`NioSocketImpl.read` |
| **BLOCKED** | Waiting to enter a `synchronized` block | `- waiting to lock <0x...>` |
| **WAITING** | Parked with no timeout | `Object.wait()`, `LockSupport.park`, `Future.get()` |
| **TIMED_WAITING** | Parked/sleeping with a timeout | `Thread.sleep`, `poll(30s)`, idle pool threads |
| **TERMINATED** | Finished | (not shown) |

```
   NEW ──start()──► RUNNABLE ◄──────────────┐
                     │   │   │              │ lock acquired / notified / timeout
     synchronized ───┘   │   └── sleep/poll(t) ──► TIMED_WAITING
       (contended)       │
          ▼              └── wait()/park()/join() ──► WAITING
       BLOCKED ──────────────────────────────────────┘
```

> ⚠️ **RUNNABLE ≠ using CPU.** A thread blocked in a socket read to ShipRocket or Postgres shows **RUNNABLE** (it's in native code). Check the stack top before blaming CPU.

Memory trick: **BLOCKED = "Bouncer at `synchronized` door"**, **WAITING = "Waiting for a call back"**, **TIMED_WAITING = "Waiting with an alarm set."**

**Interview one-liner:** BLOCKED means waiting for a `synchronized` monitor, WAITING/TIMED_WAITING means parked (without/with timeout), and RUNNABLE includes threads stuck in native socket I/O — so always read the stack, not just the state.

---

## 3. Reading a thread dump — anatomy of one entry

```
"http-nio-8080-exec-17" #57 [2391] daemon prio=5 os_prio=0 cpu=48213.55ms elapsed=3601.20s tid=0x00007f3a2c1e8000 nid=2391 runnable  [0x00007f39f4bfe000]
   java.lang.Thread.State: RUNNABLE
	at java.util.regex.Pattern$Loop.match(java.base@21.0.4/Pattern.java:5067)
	at java.util.regex.Matcher.matches(java.base@21.0.4/Matcher.java:729)
	at com.shop.coupon.CouponRule.match(CouponRule.java:41)
	at com.shop.coupon.CouponService.applyCoupons(CouponService.java:88)
	at com.shop.order.OrderController.checkout(OrderController.java:52)
	...
   Locked ownable synchronizers:
	- <0x00000006c1a2f0b8> (a java.util.concurrent.ThreadPoolExecutor$Worker)
```

| Field | Meaning |
|-------|---------|
| `"http-nio-8080-exec-17"` | Thread name → **name your pools** (`payment-worker-%d`) so dumps are readable |
| `cpu=48213.55ms elapsed=3601.20s` | CPU used so far vs lifetime — high CPU/elapsed ratio = busy thread |
| `nid` | **Native (OS) thread id** — matches `top -H` / `ps -L` |
| `- locked <0x...>` | Holds this monitor |
| `- waiting to lock <0x...>` | BLOCKED on this monitor |
| `- parking to wait for <0x...>` | Parked on a `j.u.c` lock/condition (ReentrantLock, queue, Hikari) |

> ⚠️ **Format changed across JDKs.** JDK 8–17 print `nid=0x957` in **hex**; JDK 19+ print the OS id in **decimal** (`nid=2391`, also in `[2391]` after `#57`). Know which you're reading before converting.

**Interview one-liner:** For each thread I read the name, state, `cpu=`, `nid`, and the top `com.shop.*` frame, then follow `locked` / `waiting to lock` / `parking to wait for` to see who holds what.

---

## 4. Deadlock detection

*Definition:* A **deadlock** is when two or more threads each hold a lock the other needs, so none can proceed — forever. The JVM detects monitor and `ReentrantLock` deadlocks and prints them at the **bottom** of the dump.

```
Found one Java-level deadlock:
=============================
"order-worker-1":
  waiting to lock monitor 0x00007f8b1c003f00 (object 0x00000007bf1a2c40, a com.shop.Inventory),
  which is held by "order-worker-2"
"order-worker-2":
  waiting to lock monitor 0x00007f8b1c006a00 (object 0x00000007bf1a2c60, a com.shop.Wallet),
  which is held by "order-worker-1"

Java stack information for the threads listed above:
===================================================
"order-worker-1":
	at com.shop.order.OrderService.reserve(OrderService.java:42)
	- waiting to lock <0x00000007bf1a2c40> (a com.shop.Inventory)
	- locked <0x00000007bf1a2c60> (a com.shop.Wallet)
Found 1 deadlock.
```

```java
// ❌ Thread A: wallet → inventory, Thread B: inventory → wallet  → deadlock
synchronized (wallet) { synchronized (inventory) { reserve(); } }

// ✅ Fix 1: global lock ordering (always wallet before inventory, or order by id)
// ✅ Fix 2: tryLock with timeout so a thread backs off instead of hanging
if (walletLock.tryLock(200, MILLISECONDS)) {
    try { if (invLock.tryLock(200, MILLISECONDS)) { try { reserve(); } finally { invLock.unlock(); } } }
    finally { walletLock.unlock(); }
}
```

| | **JVM deadlock** | **Database deadlock** |
|---|------------------|-----------------------|
| Detected by | `jstack`/`jcmd` ("Found one Java-level deadlock") / `ThreadMXBean.findDeadlockedThreads()` | Postgres/MySQL lock manager |
| Outcome | Threads hang forever | DB kills one tx: `ERROR: deadlock detected` / MySQL `1213` |
| Fix | Lock ordering, `tryLock`, smaller critical sections | Update rows in consistent order, short transactions, retry |

Memory trick: Deadlock needs **"MHNC"** (Coffman) — **M**utual exclusion, **H**old-and-wait, **N**o preemption, **C**ircular wait. Break any one (usually circular wait via ordering).

**Interview one-liner:** I detect deadlocks with a thread dump — the JVM prints "Found one Java-level deadlock" with the cycle — and fix them with consistent lock ordering or `tryLock` with timeouts.

---

## 5. Take 3 dumps, 5–10 seconds apart

*Definition:* One dump is a single frame; **three dumps 5–10s apart** show **motion**. A thread with the same stack in all three is **stuck**; a stack that changes is just busy.

```bash
for i in 1 2 3; do
  kubectl exec order-svc-7d9 -- jcmd 1 Thread.print -l > td-$i-$(date +%H%M%S).txt
  sleep 7
done
```

| Pattern across 3 dumps | Diagnosis |
|------------------------|-----------|
| Same thread, same `com.shop.*` frame, RUNNABLE | Infinite loop / very slow code (CPU) |
| Same thread, same socket read frame | Downstream hang — missing read timeout |
| Many threads BLOCKED on same `<0x...>` | Lock contention — one hot `synchronized` |
| All pool threads parked in Hikari `getConnection` | Connection pool starvation |
| Stacks change every time | Healthy but busy — look elsewhere (GC, DB) |

Tools that parse and compare: **fastthread.io**, **JStack Review**, **IBM TMDA**, IntelliJ "Analyze Stack Trace", JMC.

**Easy rule of thumb:** **"Same stack three times = stuck. Different stacks = busy."**

**Interview one-liner:** I take three thread dumps 5–10 seconds apart because a single dump can't distinguish a stuck thread from a busy one — identical stacks across all three mean stuck.

---

## 6. High CPU → find the exact thread (top -H + nid)

*Definition:* To find **which Java thread** burns CPU, list OS threads by CPU with `top -H`, then match the OS thread id to the dump's `nid`.

```bash
top -H -b -n 1 -p 1 | head -15          # inside the container (JVM is PID 1)
#   PID USER   %CPU  COMMAND
#  2391 app    98.7  http-nio-8080-e     ← OS thread id 2391
#    41 app    12.1  GC Thread#0

printf '%x\n' 2391                       # → 957  (needed for JDK ≤17 hex nid)
grep -A 20 'nid=0x957 ' td-1.txt         # JDK 8–17
grep -A 20 'nid=2391 '  td-1.txt         # JDK 19+ (decimal)
```

- Hot thread is `GC Thread#N` / `G1 Conc#N` → it's GC, not your code → check heap/allocation rate (`jstat -gcutil 1 1000`).
- Hot thread is `C2 CompilerThread` right after deploy → JIT warm-up, usually transient.
- Hot thread is `http-nio-*` stuck in `com.shop.*` → your code (regex, loop, JSON, `HashMap` misuse).
- Shortcut: `cpu=` per thread in `jcmd Thread.print`, or `asprof -e cpu -t` for a per-thread flame graph.

> ⚠️ `top -H` shows the CPU of a **moment**; take it 2–3 times alongside the thread dumps so the snapshots line up.

**Interview one-liner:** For high CPU I run `top -H -p <pid>` to get the hottest OS thread id, convert it to hex for JDK ≤17 (decimal on 19+), and grep that `nid` in the thread dump to see the exact stack.

---

## 7. Thread pool starvation — the most common "hang"

*Definition:* **Thread pool starvation** is when all threads of a pool (Tomcat's 200 `http-nio` threads, a `@Async` executor, HikariCP's connections) are busy **waiting** on something, so new work queues up and requests time out — even though CPU is low.

```
"http-nio-8080-exec-187" #212 daemon ... waiting on condition
   java.lang.Thread.State: TIMED_WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@21.0.4/Native Method)
	- parking to wait for <0x00000006c0f1a2b8> (a java.util.concurrent.SynchronousQueue$TransferStack)
	at java.util.concurrent.SynchronousQueue.poll(...)
	at com.zaxxer.hikari.util.ConcurrentBag.borrow(ConcurrentBag.java:...)
	at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:...)
	at com.shop.order.OrderService.place(OrderService.java:63)
→ 190 of 200 Tomcat threads look like this. Logs: "HikariPool-1 - Connection is not available, request timed out after 30000ms"
```

| Stack top across most threads | Root cause | Fix |
|-------------------------------|------------|-----|
| `HikariPool.getConnection` | DB pool (10 conns) exhausted — slow queries or long `@Transactional` holding conns during HTTP calls | Fix slow SQL, don't call ShipRocket inside a transaction, size pool sanely |
| `NioSocketImpl.read` → ShipRocket/Razorpay client | Downstream slow, **no read timeout** | Connect/read timeouts, circuit breaker, bulkhead |
| `CompletableFuture.get` / `Future.get` in same pool | Pool waits on tasks queued in **itself** (nested-submit deadlock) | Separate executors per stage, avoid blocking `get()` |
| `RedisTemplate` / Lettuce wait | Redis slow or pool exhausted | Timeouts, check `SLOWLOG`, pool size |

> ⚠️ `CompletableFuture.supplyAsync()` without an executor uses the shared **ForkJoinPool.commonPool** (size ≈ CPU cores − 1). Blocking I/O there starves every other async user in the JVM. Pass your own executor.

**Interview one-liner:** If CPU is low but requests hang, I look for pool starvation in the dump — e.g. all Tomcat threads parked in `HikariPool.getConnection` or stuck in a socket read with no timeout — and fix the slow dependency, timeouts, and pool boundaries.

---

## 8. Virtual threads (JDK 21+) — what changes for dumps

*Definition:* **Virtual threads** (JEP 444, JDK 21) are lightweight JVM-managed threads mounted on a few **carrier** platform threads. You can have millions, so the classic text dump stops being enough.

| | **Platform threads** | **Virtual threads** |
|---|----------------------|---------------------|
| Shown by `jstack` / `Thread.print` | ✅ Yes | ❌ Not listed individually (carriers only) |
| Dump command | `jcmd <pid> Thread.print` | `jcmd <pid> Thread.dump_to_file -format=json <file>` |
| Count | Hundreds | Up to millions |
| Spring Boot 3.2+ | default | `spring.threads.virtual.enabled=true` |

> ⚠️ **Pinning (JDK 21–23):** blocking inside `synchronized` pins the virtual thread to its carrier, so a few pinned threads can starve all carriers. Detect with JFR event `jdk.VirtualThreadPinned` (or `-Djdk.tracePinnedThreads=full` on 21–23); prefer `ReentrantLock`. JDK 24 (JEP 491) removes most `synchronized` pinning.

> ⚠️ Virtual threads remove **thread** starvation, not **resource** starvation — 10,000 virtual threads still fight over 10 Hikari connections. Keep pool limits / semaphores.

**Interview one-liner:** With virtual threads, `jstack` only shows carriers, so I use `jcmd Thread.dump_to_file -format=json`, and I watch for `synchronized` pinning via the JFR `VirtualThreadPinned` event on JDK 21.

---

## 9. How I'd actually debug a hung service in production + what interviewers probe

Scenario: `order-svc` stops responding — `/actuator/health` times out, CPU 5%, no errors except Hikari timeouts.

1. **Grafana**: `tomcat_threads_busy_threads` at max? `hikaricp_connections_pending` > 0 and `active` = max? CPU low → it's **waiting**, not computing.
2. **Keep one pod for evidence**: remove it from the Service (change a label) instead of letting K8s restart it — restarts destroy the evidence.
3. **Three dumps**, 7s apart: `jcmd 1 Thread.print -l` (or `kill -3 1` + `kubectl logs` if no JDK tools).
4. **Bottom of the dump first**: "Found one Java-level deadlock"? → fix lock order.
5. **Group threads by top frame** (fastthread.io): 190/200 in `HikariPool.getConnection` → find who holds the 10 connections — those threads are in a slow SQL or an HTTP call to ShipRocket inside `@Transactional`.
6. **Confirm downstream**: Postgres `pg_stat_activity` for long-running queries; HTTP client timeout settings.
7. **Fix + guard**: move the external call outside the transaction, add read timeouts, alert on `hikaricp_connections_pending > 0` for 1 min.

| Interviewer asks | Strong answer |
|------------------|---------------|
| "Thread dump vs heap dump?" | Thread dump = what threads are doing (hangs, CPU, deadlocks), cheap. Heap dump = what objects exist (leaks), heavy STW. |
| "BLOCKED vs WAITING?" | BLOCKED = waiting for a `synchronized` monitor; WAITING = parked/`wait()` until signalled. |
| "Why 3 dumps?" | One snapshot can't tell stuck from busy; identical stacks across dumps = stuck. |
| "Find the thread using 100% CPU?" | `top -H -p <pid>` → thread id → hex (≤17) → grep `nid` in dump. |
| "How do you detect deadlock?" | Thread dump bottom section or `ThreadMXBean.findDeadlockedThreads()`. |
| "CPU low, app hung?" | Pool starvation / downstream without timeout — look at common top frames. |
| "Virtual threads and jstack?" | jstack misses them; use `Thread.dump_to_file -format=json`; watch pinning. |

**Interview one-liner:** For a hang I keep one pod as evidence, take three thread dumps, check for deadlocks, group threads by their top frame to spot pool starvation or a stuck downstream, then fix with timeouts and pool boundaries.

---

## Quick Revision Cheat Sheet

- **Thread dump**: snapshot of all threads' state, stack, locks; `jcmd <pid> Thread.print -l`, `jstack -l`, `kill -3` (→ stdout), cheap and prod-safe.
- **States**: BLOCKED = `synchronized` monitor, WAITING/TIMED_WAITING = parked, RUNNABLE includes native socket I/O.
- **Reading**: name, `cpu=`/`elapsed=`, `nid` (hex ≤17, decimal 19+), `locked` / `waiting to lock` / `parking to wait for`.
- **Deadlock**: "Found one Java-level deadlock" at dump bottom; fix with lock ordering or `tryLock` timeout; DB deadlocks are auto-aborted.
- **3 dumps 5–10s apart**: same stack = stuck, changing = busy; analyze with fastthread.io / TMDA.
- **High CPU**: `top -H -p <pid>` → thread id → `printf '%x'` → grep `nid`; GC/C2 threads point elsewhere.
- **Pool starvation**: all threads in `HikariPool.getConnection` or socket read; fix slow SQL, timeouts, no HTTP inside transactions, own executors.
- **Virtual threads**: jstack shows carriers only → `Thread.dump_to_file -format=json`; watch `synchronized` pinning (JFR `VirtualThreadPinned`).
- **Prod flow**: Grafana → isolate pod → 3 dumps → deadlock check → group by top frame → confirm downstream → fix + alert.
