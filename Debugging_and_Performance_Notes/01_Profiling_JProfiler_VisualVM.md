# Debugging & Performance Notes — Profiling (JProfiler, VisualVM, Sampling vs Instrumentation, Flame Graphs, JFR, async-profiler)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is profiling? (metrics tell you *that*, profilers tell you *where*)

*Definition:* **Profiling** is measuring **where** a running program spends its CPU time, memory allocations, and lock/wait time — down to the method and line — so you fix the real bottleneck instead of guessing.

Real life: Grafana is the car's dashboard ("engine hot, fuel low"). A profiler is the mechanic's diagnostic scanner plugged into the engine that tells you *which* cylinder is misfiring.

```
Prometheus/Grafana:  "p95 of POST /checkout jumped 180ms → 1.4s, CPU 85%"   ← WHAT / WHEN
APM trace (Tempo):   "1.1s spent inside OrderService.applyCoupons()"          ← WHICH SPAN
Profiler:            "72% of CPU in String.format() inside CouponRule.match()" ← WHICH LINE
```

| Tool | Answers | Granularity |
|------|---------|-------------|
| **Metrics** (Micrometer → Prometheus) | Is something wrong? | Service / endpoint |
| **Tracing** (OpenTelemetry, APM) | Which hop/span is slow? | Request / span |
| **Profiler** (JProfiler, async-profiler, JFR) | Which method burns CPU/memory? | Method / line / allocation site |

**Interview one-liner:** Metrics detect the problem, traces locate the slow component, and a profiler pinpoints the exact methods burning CPU, allocating memory, or waiting on locks.

---

## 2. Sampling vs instrumentation

*Definition:* **Sampling** takes a snapshot of every thread's stack at a fixed interval (e.g. every 10ms) and counts which methods show up most. **Instrumentation** rewrites bytecode to add timing code at every method entry/exit, giving exact call counts and timings.

Real life: Sampling = a teacher glancing at the class every 5 minutes and noting who is on their phone. Instrumentation = a CCTV camera on every desk recording every second.

| Feature | **Sampling** | **Instrumentation** |
|---------|--------------|---------------------|
| How | Periodic stack snapshots | Bytecode injected into methods |
| Overhead | Low (~1–5%) | High (can slow small methods 10x+) |
| Invocation counts | ❌ No (only estimates) | ✅ Exact |
| Accuracy for tiny hot methods | Statistical, good enough | Distorted — timing code costs more than the method |
| Safe in production? | ✅ Usually (JFR, async-profiler) | ❌ Rarely — staging/local only |
| JProfiler / VisualVM name | "Sampling" / "Sampler" tab | "Instrumentation" / "Profiler" tab |

Memory trick: **"Sampling = Snapshots, Instrumentation = Injection."**

> ⚠️ **Safepoint bias:** classic JVMTI samplers (VisualVM Sampler, older JProfiler sampling) can only grab stacks at JVM **safepoints**, so tight loops without safepoint polls get under-reported. async-profiler and JFR avoid this by sampling via OS signals / `AsyncGetCallTrace`-style APIs. (JProfiler 12+ also offers "async sampling" on Linux.)

**Easy rule of thumb:** Start with **sampling** to find the hot area; switch to **instrumentation with filters** (only `com.shop.*`) when you need exact call counts in a staging run.

**Interview one-liner:** Sampling periodically snapshots stacks with low overhead and is production-safe; instrumentation injects timing into every method for exact counts but adds heavy overhead and distorts small methods.

---

## 3. CPU profiling — hot spots, call tree, self vs total time

*Definition:* A **hot spot** is a method with high **self time** (time spent in its own code, not its callees). The **call tree** is a top-down view showing which caller path led to that time.

| Term | Meaning | Example |
|------|---------|---------|
| **Self time** | Time in the method's own body | `CouponRule.match()` regex = 640ms |
| **Total time** | Self + everything it calls | `checkout()` = 1,100ms |
| **Hot Spots view** (JProfiler) | Methods sorted by self time, expand to see callers (back traces) | "Who is burning CPU?" |
| **Call Tree view** | Top-down from thread entry point | "What path led there?" |

```
Call Tree (JProfiler, sampling, POST /checkout)               total   self
└─ 92.1%  OrderController.checkout()                          1100ms   2ms
   ├─ 71.4%  CouponService.applyCoupons()                      790ms   5ms
   │   └─ 70.2%  CouponRule.match()                            775ms  10ms
   │       └─ 66.0%  java.util.regex.Pattern.compile()  ◄ HOT  730ms 730ms
   └─ 18.3%  OrderRepository.save()  (JDBC wait)               200ms   1ms
```

```java
// ❌ Compiles the regex on EVERY call (classic hot spot)
boolean match(String code) { return code.matches("^[A-Z]{4}\\d{2}$"); }

// ✅ Compile once, reuse
private static final Pattern CODE = Pattern.compile("^[A-Z]{4}\\d{2}$");
boolean match(String code) { return CODE.matcher(code).matches(); }
```

> ⚠️ Check the **thread state filter**. JProfiler/VisualVM can show "Runnable" vs "All states". Threads blocked on a DB socket look "hot" in wall-clock mode but use zero CPU — that's an I/O problem, not a CPU one.

**Interview one-liner:** I sort by self time in the Hot Spots view to find who burns CPU, then use back traces or the call tree to see which business path calls it.

---

## 4. Flame graphs — reading a profile in 10 seconds

*Definition:* A **flame graph** stacks sampled call stacks: each box is a method, **width = share of samples** (time), the y-axis is stack depth, and the top edge is what was actually on CPU.

```
          [Pattern.compile ████████████]          ← widest top box = hot spot
     [CouponRule.match ███████████████████]  [JDBC ███]
  [CouponService.applyCoupons ██████████████████████]  [OrderRepository.save ███]
[OrderController.checkout ███████████████████████████████████████████████]
[Thread.run / Tomcat NioEndpoint ███████████████████████████████████████████████████]
```

- **Look for wide plateaus at the top** — that's self time.
- **x-axis is NOT time** — boxes are sorted alphabetically to merge identical stacks.
- Colors encode frame type (Java vs JVM/native vs kernel), **not** heat.
- A **diff / differential flame graph** (before vs after deploy) shows regressions.

Memory trick: **"Wide is the price, tall is just the path."**

**Interview one-liner:** In a flame graph, width is time and the top edge is on-CPU code, so I look for the widest boxes near the top; the x-axis is alphabetical, not a timeline.

---

## 5. Memory & allocation profiling

*Definition:* **Allocation profiling** records **which code allocates how many objects/bytes**, while **live-object (heap) profiling** shows what is still retained after GC. High allocation rate = GC pressure and latency; growing live set = leak.

| View | Tool | Question it answers |
|------|------|---------------------|
| **Allocation hot spots / call tree** | JProfiler "Allocation Call Tree", async-profiler `-e alloc` | Who creates garbage fastest? |
| **Recorded objects / live objects** | JProfiler "Live Memory", VisualVM Sampler → Memory | What classes fill the heap right now? |
| **Heap walker / heap dump** | JProfiler Heap Walker, Eclipse MAT | Why is this object still alive? |
| **GC telemetry** | VisualVM Monitor tab, JFR GC events, GCeasy | How often / how long does GC pause? |

```
VisualVM → Sampler → Memory (per-class, live)
Class                                    Bytes [%]     Bytes          Instances
byte[]                                   41.2%         612,334,120    1,903,221
java.lang.String                         12.8%         190,112,064    7,921,336
com.fasterxml.jackson.databind.node.ObjectNode  9.6%   142,557,000    2,970,000   ◄ suspicious
```

```java
// ❌ Parses the full ShipRocket JSON into a tree per request, then reads 2 fields
JsonNode root = mapper.readTree(body);
// ✅ Bind only what you need into a small record
record Rate(String courier, BigDecimal charge) {}
Rate r = mapper.readValue(body, Rate.class);
```

> ⚠️ High allocation rate ≠ leak. If heap-after-GC is flat, it's **churn** (fix for latency). If heap-after-GC keeps rising, it's a **leak** → take a heap dump (see `02_Heap_Dump_Analysis.md`).

**Interview one-liner:** Allocation profiling shows who creates garbage (GC pressure), live-object views show what is retained (leaks), and I confirm leaks with a heap dump in Eclipse MAT.

---

## 6. Attaching to remote / containerized JVMs

*Definition:* **Remote profiling** means connecting a desktop profiler (JProfiler GUI, VisualVM) to a JVM running on a server, in Docker, or in a Kubernetes pod — via **JMX**, a **profiler agent**, or by running a CLI profiler **inside** the container and copying results out.

```
Laptop (JProfiler / VisualVM)
    │  kubectl port-forward pod/order-svc-7d9 8849:8849 9010:9010
    ▼
K8s Pod ── container: java -agentpath:libjprofilerti.so=port=8849 ... -jar app.jar
```

```bash
# Option A: VisualVM over JMX (add to JAVA_TOOL_OPTIONS in the Deployment)
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.rmi.port=9010          # same port → works through port-forward
-Dcom.sun.management.jmxremote.authenticate=false     # ONLY because it's reachable via port-forward
-Dcom.sun.management.jmxremote.ssl=false
-Djava.rmi.server.hostname=127.0.0.1                  # RMI stub points to the tunnel, not pod IP

# Option B: JProfiler agent baked into the image (nowait = don't block startup)
-agentpath:/opt/jprofiler/bin/linux-x64/libjprofilerti.so=port=8849,nowait

# Option C (my default): profile INSIDE the container, copy the file out
kubectl exec -it order-svc-7d9 -- jcmd 1 JFR.start duration=60s settings=profile filename=/tmp/cpu.jfr
kubectl cp order-svc-7d9:/tmp/cpu.jfr ./cpu.jfr     # open in JDK Mission Control
```

> ⚠️ Container gotchas: the JVM is usually **PID 1**; `jcmd`/attach must run as the **same user** and see the same `/tmp` (so run it via `kubectl exec` / `docker exec`, not from the host). Slim JRE / distroless images lack `jcmd` — use a JDK debug image, **ephemeral debug container** (`kubectl debug --target=app`), or `jattach`.

> ⚠️ Never expose an unauthenticated JMX port on a Service/LoadBalancer — JMX can execute code. Tunnel it.

**Interview one-liner:** For containers I prefer running JFR or async-profiler inside the pod via `kubectl exec` and copying the file out; for live GUIs I tunnel JMX or the JProfiler agent port with `kubectl port-forward`, never exposing it publicly.

---

## 7. JFR + JMC — the built-in, always-on flight recorder

*Definition:* **Java Flight Recorder (JFR)** is a low-overhead (~1% with default settings) event recorder built into the JVM (free since JDK 11). It records CPU samples, allocations, GC, locks, I/O, and thread events into a `.jfr` file you analyze in **JDK Mission Control (JMC)** or with the `jfr` CLI.

Real life: An aircraft black box — always recording, you only pull the tape when something goes wrong.

```bash
# Always-on rolling recording from startup (keeps last 6h / 250MB)
-XX:StartFlightRecording=settings=default,maxage=6h,maxsize=250m,dumponexit=true,filename=/dumps/app.jfr

# Ad-hoc on a running JVM
jcmd 1 JFR.start name=slow settings=profile duration=120s filename=/tmp/slow.jfr
jcmd 1 JFR.dump  name=slow filename=/tmp/now.jfr     # snapshot without stopping
jcmd 1 JFR.stop  name=slow

# Analyze without a GUI (JDK 21+ has `jfr view`)
jfr summary /tmp/slow.jfr
jfr view hot-methods /tmp/slow.jfr
jfr print --events jdk.ExecutionSample /tmp/slow.jfr | head
```

| Settings | Overhead | Use |
|----------|----------|-----|
| `default` | ~1% | Always-on in prod |
| `profile` | ~2–3% | Short targeted capture during an incident |

**Interview one-liner:** JFR is the JVM's built-in ~1% overhead flight recorder — I run it continuously in prod with a rolling buffer and dump the last N minutes when an incident happens, then analyze in JMC.

---

## 8. async-profiler — flame graphs in production

*Definition:* **async-profiler** is an open-source, low-overhead sampling profiler for HotSpot JVMs that uses `perf_events`/signals plus `AsyncGetCallTrace`, so it has **no safepoint bias** and sees Java, JVM (GC/JIT), native, and kernel frames. Output: HTML flame graph, JFR, or collapsed stacks.

```bash
# v3+ CLI is `asprof` (older: profiler.sh)
asprof -d 30 -e cpu   -f /tmp/cpu.html   1     # CPU flame graph, 30s, PID 1
asprof -d 30 -e alloc -f /tmp/alloc.html 1     # allocation flame graph
asprof -d 30 -e lock  -f /tmp/lock.html  1     # lock contention
asprof -d 30 -e wall -t -f /tmp/wall.html 1    # wall-clock per thread (finds I/O waits)
```

| Event | Finds |
|-------|-------|
| `cpu` | Code burning CPU (incl. GC/JIT threads) |
| `alloc` | Allocation hot spots (TLAB-based sampling) |
| `lock` | Contended `synchronized` / `ReentrantLock` |
| `wall` | Everything incl. sleeping/blocked — slow I/O, DB waits |

> ⚠️ In Docker/K8s, `perf_events` is often blocked (`perf_event_paranoid`, seccomp). async-profiler then falls back to timer-based sampling (`-e itimer`/`ctimer`) — still useful, but no kernel frames.

**Easy rule of thumb:** CPU high → `-e cpu`. Latency high but CPU low → `-e wall` (waiting). GC heavy → `-e alloc`.

**Interview one-liner:** async-profiler gives production-safe flame graphs without safepoint bias, with cpu, alloc, lock and wall modes — wall-clock is the one that catches I/O and lock waits.

---

## 9. How I'd actually debug a slow endpoint in production + what interviewers probe

Scenario: after a deploy, `POST /checkout` p95 goes 180ms → 1.4s and pod CPU hits 85%.

1. **Grafana first** — confirm scope: one endpoint or all? one pod or all? correlate with deploy time, GC pause panel, HikariCP `hikaricp_connections_pending`.
2. **Trace** a slow request in the APM — find the slow span (DB? ShipRocket call? in-JVM code?).
3. **If in-JVM CPU:** `kubectl exec` → `asprof -d 30 -e cpu -f /tmp/cpu.html 1` (or `jcmd 1 JFR.start ... settings=profile duration=60s`).
4. **If latency but low CPU:** `-e wall` profile or 3 thread dumps (see `03_Thread_Dump_Analysis.md`) — look for threads waiting on locks, pool, or sockets.
5. `kubectl cp` the file out, open the flame graph / JMC → find the widest top frames in `com.shop.*`.
6. Reproduce locally/staging with **JProfiler instrumentation** filtered to that package to get exact call counts.
7. Fix, deploy, and compare with a **differential flame graph** + Grafana p95 to prove it.

| Interviewer asks | Strong answer |
|------------------|---------------|
| "Sampling vs instrumentation?" | Sampling = periodic stacks, low overhead, prod-safe; instrumentation = exact counts, high overhead, staging only. |
| "Can you profile in prod?" | Yes — JFR always-on (~1%) and async-profiler for 30–60s captures; not instrumentation. |
| "What's safepoint bias?" | JVMTI samplers only see stacks at safepoints → skewed; async-profiler/JFR avoid it. |
| "How to read a flame graph?" | Width = time, top edge = on-CPU, x-axis alphabetical. |
| "CPU is low but requests are slow?" | It's waiting — wall-clock profile, thread dumps, check pool/DB/HTTP timeouts. |
| "How do you attach to a pod?" | `kubectl exec` + jcmd/asprof inside, `kubectl cp` out; or port-forward JMX/agent. |
| "Self vs total time?" | Self = own code; total = including callees. Hot spots sort by self. |

**Interview one-liner:** I go metrics → trace → low-overhead profile in prod (JFR/async-profiler) → targeted instrumentation in staging → fix → prove it with a diff flame graph and p95.

---

## Quick Revision Cheat Sheet

- **Profiling**: metrics say *that*, traces say *which hop*, profilers say *which method/line*.
- **Sampling vs instrumentation**: snapshots (low overhead, prod) vs injected bytecode (exact counts, heavy, staging); watch safepoint bias.
- **CPU profiling**: Hot Spots sorted by **self time**, call tree/back traces show the path; filter Runnable vs all states.
- **Flame graph**: width = time, top = on-CPU, x-axis alphabetical; look for wide plateaus; diff graphs for regressions.
- **Allocation profiling**: who creates garbage (GC pressure) vs live objects (leak) → confirm with heap dump in MAT.
- **Remote/containers**: `kubectl exec` + jcmd/asprof as PID 1 → `kubectl cp`; or port-forward JMX (`rmi.port` same, `hostname=127.0.0.1`) / JProfiler agent; never expose JMX.
- **JFR + JMC**: built-in ~1% recorder; always-on rolling buffer, `JFR.dump` on incident; `jfr view hot-methods` (JDK 21).
- **async-profiler**: `asprof -e cpu|alloc|lock|wall`, no safepoint bias, HTML flame graphs; falls back to itimer in locked-down containers.
- **Prod flow**: Grafana → trace → JFR/async-profiler → staging instrumentation → fix → diff flame graph + p95.
