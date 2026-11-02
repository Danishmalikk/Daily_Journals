# Debugging & Performance Notes — Memory Leak Detection (Heap Dumps, MAT, JFR OldObjectSample, NMT, OOMKilled)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a memory leak in a GC language?

*Definition:* In Java a **memory leak** is not "forgot to free" — it's objects that are **no longer needed but still reachable** from a GC root (static field, live thread, JNI ref), so the garbage collector is **not allowed** to remove them.

Real life: A warehouse that never throws out old parcels because each one still has a delivery slip pinned to the "active" board. The cleaner only bins parcels with no slip.

```
GC roots (static fields, thread stacks, JNI)
   │
   ▼
static Map<Long, Order> ORDER_CACHE ──► Order#1 ──► 48 OrderItems ──► byte[] invoicePdf
                                    ──► Order#2 ──► ...
                                    ──► ... 2 million orders, never removed
   Reachable = alive = never collected  →  heap fills  →  OutOfMemoryError
```

| | C / C++ leak | Java / Node leak |
|---|---|---|
| Cause | Forgot `free()` | Kept a **reference** you don't need |
| Object reachable? | No (lost pointer) | **Yes** — that's why GC can't help |
| Fix | Free memory | Remove the reference / bound the structure |

**Interview one-liner:** In a GC language a leak is memory that is still reachable from a GC root but no longer useful — the GC works correctly, our code just keeps a reference.

---

## 2. Sawtooth vs rising baseline — reading the graph

*Definition:* A healthy JVM heap graph is a **sawtooth**: allocations push it up, GC drops it down to the same floor. A leak shows a **rising baseline** — the floor after each GC keeps climbing.

```
Healthy (sawtooth, flat floor)          Leak (rising floor)
 MB                                      MB                       ┌ Full GCs → OOM
1500 ┤ /|  /|  /|  /|  /|               1500 ┤             /|/|/|/|█
1000 ┤/ | / | / | / | / |               1000 ┤      /|  /| / 
 500 ┤  |/  |/  |/  |/  |/               500 ┤ /| / |/  |/        ← floor climbs
     └──────────────────── time              └──────────────────── time
```

```promql
# Heap floor per pod: min heap over a window (a slowly rising line = leak)
min_over_time(sum by (pod) (jvm_memory_used_bytes{area="heap"})[1h:1m])
# Live data after GC (Micrometer)
jvm_gc_live_data_size_bytes
# Node.js (prom-client default metrics)
nodejs_heap_size_used_bytes
```

> ⚠️ High heap usage alone is **not** a leak — G1 happily uses most of the heap before collecting. Judge the **floor after GC** over hours/days, and compare against traffic (more users = more live sessions, not a leak).

**Easy rule of thumb:** Flat floor = fine. Floor rising at constant traffic, surviving Full GCs = leak.

**Interview one-liner:** I don't look at peak heap; I look at heap-after-GC over hours — a sawtooth with a flat floor is healthy, a floor that keeps rising at steady traffic is a leak.

---

## 3. Common causes (and the fix for each)

*Definition:* Almost every Java leak is one of a handful of patterns where a **long-lived object** holds **short-lived data**.

```java
// ❌ 1. Unbounded "cache" in a static map — grows forever
private static final Map<String, Rate> RATES = new ConcurrentHashMap<>();
// ✅ bounded + expiring cache (Caffeine)
private final Cache<String, Rate> rates = Caffeine.newBuilder()
        .maximumSize(10_000).expireAfterWrite(Duration.ofMinutes(15)).build();

// ❌ 2. ThreadLocal in a thread pool (Tomcat threads live forever)
private static final ThreadLocal<TenantContext> CTX = new ThreadLocal<>();
CTX.set(ctx);  handle(request);           // never removed → stays on the pooled thread
// ✅ always clean up
try { CTX.set(ctx); handle(request); } finally { CTX.remove(); }

// ❌ 3. Listener registered per request, never unregistered
eventBus.register(new OrderStatusListener(order));
// ✅ unregister in finally / use weak listeners / register once at startup
```

| Cause | Typical example in our stack | Fix |
|-------|------------------------------|-----|
| **Static collections** | `static List<Event>` buffer for "later" | Bound it, flush it, or don't make it static |
| **Unbounded caches** | `HashMap` caching ShipRocket rates per pincode | Caffeine/Redis with `maximumSize` + TTL |
| **ThreadLocal in pools** | Tenant/user context, `SimpleDateFormat` | `remove()` in `finally` |
| **Listeners / callbacks** | Event listeners, Node `emitter.on()` per request | Unregister; Node warns `MaxListenersExceededWarning` |
| **Unclosed resources** | `InputStream`, JDBC `ResultSet`, HTTP responses | try-with-resources |
| **Mutable map keys** | Key's `hashCode` changes after insert → entry unreachable by `get`/`remove` | Immutable keys (records) |
| **ClassLoader leaks** | Redeploying WAR on Tomcat, JDBC driver not deregistered | Restart process; avoid hot redeploy; `OutOfMemoryError: Metaspace` is the tell |

Memory trick: **"SUTLUC"** → **S**tatics, **U**nbounded caches, **T**hreadLocals, **L**isteners, **U**nclosed resources, **C**lassloaders.

**Interview one-liner:** Most leaks are a long-lived holder of short-lived data — static maps, unbounded caches, ThreadLocals on pooled threads, unregistered listeners, unclosed resources or classloaders that survive redeploys.

---

## 4. Heap dumps & comparing two in Eclipse MAT

*Definition:* A **heap dump** (`.hprof`) is a snapshot of every object on the heap and who references whom. **Eclipse MAT** (Memory Analyzer) finds what is holding memory. **Comparing two dumps** taken some time apart shows what is **growing**.

```bash
# Capture automatically on OOM (always set this in prod)
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps    # /dumps = mounted volume / EFS

# Capture on demand (live objects only → triggers a Full GC first)
jcmd <pid> GC.heap_dump /dumps/heap-1.hprof
# ... wait 30-60 min under normal traffic ...
jcmd <pid> GC.heap_dump /dumps/heap-2.hprof
kubectl cp order-service-7d9f:/dumps/heap-2.hprof ./heap-2.hprof

# Quick look without a full dump: top classes by instance count
jcmd <pid> GC.class_histogram | head -15
#  num   #instances    #bytes  class name
#    1:    2103344  168267520  com.shop.order.OrderLine        ← suspicious
```

In MAT:
1. **Leak Suspects report** → usually points straight at the culprit ("one instance of `RateCache` occupies 1.1 GB (72%)").
2. **Dominator tree** → objects sorted by **retained** size.
3. **Histogram → Compare to another heap dump** → classes whose count grew between dump 1 and 2.
4. **Path to GC Roots → exclude weak/soft references** → the exact field chain keeping it alive.

| Term | Meaning |
|------|---------|
| **Shallow heap** | Size of the object itself |
| **Retained heap** | Memory freed if this object were collected (it + everything only it holds) |
| **Dominator** | Object every path to X must pass through — the real "owner" |

> ⚠️ A heap dump **pauses** the JVM (seconds for a few GB) and the file ≈ heap size, and it contains **customer data** (emails, addresses, tokens). Take it from a pod pulled out of the load balancer, store it securely, delete it after.

**Interview one-liner:** I take two heap dumps under steady traffic, compare histograms in MAT to see which classes grew, then use the dominator tree and path-to-GC-roots to find the field that keeps them alive.

---

## 5. JFR OldObjectSample — leak hunting with low overhead

*Definition:* **Java Flight Recorder (JFR)** is a built-in, low-overhead profiler (free since JDK 11). Its **`jdk.OldObjectSample`** event samples allocations and tracks which ones **survive a long time**, recording their allocation stack trace and (optionally) the **path to GC roots** — leak detection without a full heap dump.

```bash
# Start recording on a running pod (profile settings enable OldObjectSample)
jcmd <pid> JFR.start name=leak settings=profile maxage=6h
# ... let it run under real traffic ...
jcmd <pid> JFR.dump name=leak filename=/dumps/leak.jfr path-to-gc-roots=true

# Inspect from CLI, or open in JDK Mission Control (JMC) → "Memory Leak" / Old Object Sample view
jfr print --events jdk.OldObjectSample /dumps/leak.jfr | head -40
```

| | **Heap dump** | **JFR OldObjectSample** |
|---|---|---|
| Overhead | Pause + file ≈ heap size | ~1–2%, small file |
| Shows | Every object, exact sizes | Sampled survivors + **allocation stack** |
| Best at | "What is holding memory?" | "Where in code was the leaking object created?" |
| Safe on busy prod? | Risky | Yes |

**Easy rule of thumb:** JFR first on prod (cheap, tells you *where it's allocated*); heap dump when you need the full reference graph.

**Interview one-liner:** JFR's OldObjectSample event samples long-lived objects with their allocation stack and path to GC roots at ~1% overhead, so I can find leaks in production without the pause of a heap dump.

---

## 6. Off-heap / native leaks — NMT and direct buffers

*Definition:* **Native (off-heap) memory** is JVM memory outside the Java heap: Metaspace, thread stacks, code cache, GC structures, **direct `ByteBuffer`s** (Netty, NIO, Kafka clients), and JNI/malloc. **Native Memory Tracking (NMT)** breaks it down.

```bash
java -XX:NativeMemoryTracking=summary -jar app.jar     # ~5-10% overhead; enable while investigating
jcmd <pid> VM.native_memory baseline
# ... wait ...
jcmd <pid> VM.native_memory summary.diff
```

```
Total: reserved=3245MB, committed=1812MB  (+410MB)
-       Java Heap (reserved=1536MB, committed=1536MB)
-           Class (reserved=1052MB, committed=62MB)
-          Thread (reserved=212MB, committed=212MB)  (thread #210 +120)   ← thread leak?
-            Code (reserved=244MB, committed=48MB)
-              GC (reserved=95MB, committed=95MB)
-           Other (reserved=420MB, committed=420MB +380MB)               ← direct ByteBuffers
```

| Error message | Area | Look at |
|---------------|------|---------|
| `OutOfMemoryError: Java heap space` | Heap | Heap dump / JFR |
| `OutOfMemoryError: Metaspace` | Class metadata | Classloader leak, generated proxies |
| `OutOfMemoryError: Cannot reserve N bytes of direct buffer memory` | Direct buffers | `-XX:MaxDirectMemorySize`, unreleased Netty `ByteBuf` (`-Dio.netty.leakDetection.level=paranoid` in test) |
| `OutOfMemoryError: unable to create native thread` | Threads | Thread leak (new `ExecutorService` per request), pod pid limit |
| RSS grows, NMT flat | malloc outside JVM tracking | glibc arenas → `MALLOC_ARENA_MAX=2`, or jemalloc |

> ⚠️ NMT only sees what the JVM allocates. Native libraries (some compression/crypto libs) are invisible to it — if RSS ≫ NMT committed, suspect native code or glibc fragmentation.

**Interview one-liner:** When the pod's RSS grows but heap is flat, I turn on Native Memory Tracking and diff a baseline to find whether threads, Metaspace or direct buffers are growing.

---

## 7. Container OOMKilled vs java.lang.OutOfMemoryError

*Definition:* **`java.lang.OutOfMemoryError`** is thrown **by the JVM** when a memory area (heap, Metaspace, direct) is full. **OOMKilled** is the **Linux kernel / cgroup** killing the whole process because the container exceeded its memory **limit** — the JVM gets no chance to react.

```bash
kubectl describe pod order-service-7d9f
#   Last State:  Terminated
#     Reason:    OOMKilled
#     Exit Code: 137          ← 128 + SIGKILL(9)
```

| | **java.lang.OutOfMemoryError** | **Container OOMKilled** |
|---|---|---|
| Who decides | JVM | Kernel cgroup (K8s / Docker limit) |
| Trigger | Heap/Metaspace/direct limit hit | Total process RSS > container limit |
| Visible as | Stack trace in logs, heap dump if flag set | Exit code 137, `OOMKilled`, no Java log |
| Typical cause | Heap leak, heap too small | Heap + non-heap > limit (`-Xmx` too close to limit), native leak |
| Fix | Find leak, size heap | `MaxRAMPercentage≈75`, cap direct/metaspace, NMT for native growth |

```
Container limit 2Gi = Heap (1.5G) + Metaspace (~150M) + threads (200 × 1M) + code cache + direct buffers + GC
                      └ MaxRAMPercentage=75 ┘ └──────────── must fit in the remaining ~500M ────────────┘
```

> ⚠️ Add `-XX:+ExitOnOutOfMemoryError` (after `HeapDumpOnOutOfMemoryError`) — a JVM that survived an OOM is in an unknown state; let K8s restart it cleanly.

**Interview one-liner:** A Java OOM is the JVM refusing an allocation in one memory area; OOMKilled with exit 137 is the kernel killing the container for exceeding its limit — usually heap set too close to the limit or native memory growth.

---

## 8. Fixes, prevention & the Node.js note

**Prevention checklist:**
- Every cache is **bounded** (size + TTL) — Caffeine in-process, Redis for shared.
- `ThreadLocal.remove()` in `finally`; prefer passing context explicitly (or `ScopedValue`, preview in 21).
- try-with-resources for streams, JDBC, HTTP responses.
- Paginate / stream large queries — `findAll()` on `orders` is a "temporary leak" that kills the pod.
- Alerts: heap floor rising, GC time %, pod restarts with `OOMKilled`, RSS vs limit > 90%.
- Load/soak test with **k6** for hours and watch the heap floor before a big release.

**Node.js / NestJS:**
```bash
node --inspect=127.0.0.1:9229 dist/main.js           # then: kubectl port-forward pod/api-7c 9229
# Chrome → chrome://inspect → Memory → take Heap snapshot, run traffic, take another → "Comparison" view
node --heapsnapshot-near-heap-limit=2 --max-old-space-size=1536 dist/main.js   # auto-snapshot before crash
# crash looks like: FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```
Common Node leaks: module-level `Map` caches, closures capturing large request objects, `emitter.on()` per request, un-cleared `setInterval`.

> ⚠️ Never expose the inspector port publicly — it allows remote code execution. Bind to `127.0.0.1` and port-forward.

**Interview one-liner:** Prevent leaks with bounded caches, cleaned ThreadLocals, try-with-resources and pagination, catch them with heap-floor alerts and soak tests; in Node I compare two heap snapshots via `--inspect` in Chrome DevTools.

---

## 9. How I'd actually debug a leak in production + what interviewers probe

**Scenario:** `order-service` pods restart every ~2 days; Grafana shows heap floor creeping up.

1. **Confirm it's a leak:** heap-after-GC rising at steady traffic? Full GCs reclaiming little in the GC log? Restart reason `OOMKilled` (native/sizing) or `OutOfMemoryError` (heap)?
2. **Correlate with a deploy:** did the trend start at a release? Diff that release first.
3. **Heap leak →** start **JFR** with `path-to-gc-roots=true` for a few hours, *or* pull one pod out of the Service, take **two heap dumps** 30–60 min apart.
4. **Analyse in MAT:** Leak Suspects → dominator tree → histogram compare → path to GC roots → the exact field (e.g. `RateCache.map`).
5. **Native leak (RSS ≫ heap) →** NMT baseline + `summary.diff`; check threads, direct buffers, Metaspace.
6. **Mitigate while fixing:** raise memory limit or schedule rolling restarts — buys time, not a fix.
7. **Fix + verify:** bound the cache / `remove()` the ThreadLocal, soak test, watch heap floor flat for days; add alert.

| Interviewer asks | Strong answer |
|------------------|---------------|
| "Can Java have memory leaks with a GC?" | Yes — reachable-but-unused objects; GC can only free unreachable ones |
| "How do you tell a leak from high usage?" | Heap floor after GC rising over time at constant load |
| "Shallow vs retained heap?" | Object's own size vs everything freed if it were gone |
| "How do you find what's leaking?" | Two heap dumps → MAT compare + dominator tree + path to GC roots; or JFR OldObjectSample |
| "Why are ThreadLocals dangerous?" | Pool threads never die, so the value lives forever unless `remove()` |
| "OOMKilled but no OutOfMemoryError?" | Kernel killed the container: heap + native > limit; check `MaxRAMPercentage`, NMT |
| "Heap dump in production — risks?" | STW pause, huge file, PII; do it on a drained pod |
| "Fix by adding memory?" | Only buys time; a real leak eventually fills any heap |

**Interview one-liner:** I confirm the leak from a rising heap floor, distinguish heap OOM from OOMKilled, capture JFR or two heap dumps from a drained pod, find the holder in MAT via dominator tree and GC-root path, fix it and verify with a soak test.

---

## Quick Revision Cheat Sheet

- **Leak in GC languages**: reachable-but-useless objects; GC frees only the unreachable.
- **Graph**: sawtooth with flat floor = healthy; rising floor after GC at steady traffic = leak.
- **Causes (SUTLUC)**: statics, unbounded caches, ThreadLocals in pools, listeners, unclosed resources, classloaders.
- **Heap dumps + MAT**: `jcmd GC.heap_dump` ×2, histogram compare, dominator tree, path to GC roots; shallow vs retained; contains PII.
- **JFR OldObjectSample**: `JFR.start settings=profile` + `JFR.dump path-to-gc-roots=true`; low overhead, gives allocation stack.
- **Native leaks**: `-XX:NativeMemoryTracking=summary` + `VM.native_memory summary.diff`; threads, Metaspace, direct buffers, glibc arenas.
- **OOMKilled vs OOME**: kernel kills container (exit 137, no Java log) vs JVM error per area; keep heap ≈ 75% of limit, `ExitOnOutOfMemoryError`.
- **Prevention & Node**: bounded caches, `remove()`, try-with-resources, paginate, soak tests, alerts; Node → `--inspect` + heap snapshot comparison.
- **Prod flow**: confirm trend → correlate deploy → JFR / two dumps from drained pod → MAT → fix → soak test → alert.
