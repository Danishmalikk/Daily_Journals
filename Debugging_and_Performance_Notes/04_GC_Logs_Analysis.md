# Debugging & Performance Notes — GC Logs Analysis (-Xlog:gc*, G1 Young/Mixed/Full, Allocation Rate, Humongous, jstat, ZGC)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Why GC logs? Turning them on (unified logging)

*Definition:* A **GC log** is the JVM's own diary of every garbage collection: when it ran, why, how long the app was paused, and how much heap was freed. Since JDK 9 it is written through **unified logging** (`-Xlog`), which replaced the old `-XX:+PrintGCDetails` flags.

Real life: The fuel log of a delivery van. Average speed tells you little; the log tells you it stopped for fuel every 10 minutes because the tank leaks.

```bash
# Production-safe: rotating file, timestamps + uptime, ~1-2% overhead (usually less)
java -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m \
     -jar order-service.jar

# Already running and you forgot? Turn it on live, no restart:
jcmd <pid> VM.log output=/tmp/gc.log what=gc*=info decorators=time,uptime
```

| Old flag (JDK 8) | Unified logging (JDK 9+) |
|------------------|--------------------------|
| `-XX:+PrintGCDetails` | `-Xlog:gc*` |
| `-XX:+PrintGCDateStamps` | decorator `time` |
| `-Xloggc:gc.log` | `:file=gc.log` |
| `-XX:+UseGCLogFileRotation` | `filecount=5,filesize=20m` |

> ⚠️ In Docker/K8s, write to a **mounted volume** (or stdout with `-Xlog:gc*:stdout`). A log inside the container filesystem dies with the pod — exactly when you need it.

**Interview one-liner:** I always run services with `-Xlog:gc*` to a rotating file — it costs almost nothing and is the only reliable way to see pause times, GC causes and heap-after-GC trends.

---

## 2. Reading a G1 log line (anatomy)

*Definition:* **G1** (default GC since JDK 9) splits the heap into equal-size **regions** (Eden, Survivor, Old, Humongous). Each GC event has an id `GC(n)` and several tagged lines.

```
[2026-10-14T10:02:11.532+0530][812.345s][info][gc,start] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[...][info][gc,heap ] GC(42) Eden regions: 150->0(150)
[...][info][gc,heap ] GC(42) Survivor regions: 10->12(19)
[...][info][gc,heap ] GC(42) Old regions: 300->305
[...][info][gc,heap ] GC(42) Humongous regions: 4->4
[...][info][gc      ] GC(42) Pause Young (Normal) (G1 Evacuation Pause) 1840M->1270M(4096M) 23.456ms
[...][info][gc,cpu  ] GC(42) User=0.15s Sys=0.01s Real=0.02s
```

```
 Pause Young (Normal) (G1 Evacuation Pause)  1840M -> 1270M (4096M)  23.456ms
 └── type ──────────┘ └──── cause ────────┘  before   after  total   pause (STW)
```

- `Eden 150->0` → all of Eden evacuated (normal). `Old 300->305` → 5 regions **promoted**.
- `Real` ≫ `User+Sys` → GC threads were waiting (CPU throttled container, swapping, slow disk for logging).
- The number that matters most over time: **heap AFTER GC** (`1270M`).

Memory trick: **"Type, Cause, Before → After (Total), Time"** — read every line left to right like a receipt.

**Interview one-liner:** A G1 line gives the pause type and cause, heap before → after (total), and the stop-the-world duration; I track heap-after-GC and pause time over hours, not single lines.

---

## 3. Young vs Mixed vs Full GC

*Definition:* **Young GC** cleans only Eden/Survivor. **Mixed GC** cleans young regions **plus** some old regions picked after a concurrent mark. **Full GC** stops everything and compacts the **whole** heap — G1's emergency fallback.

```
Eden fills ─► Young GC ─► (old gen > IHOP ~45%) ─► Concurrent Start ─► Concurrent Mark
                                                                        │
                                       Remark + Cleanup (short pauses) ◄┘
                                                │
                       Prepare Mixed ─► Mixed GCs (young + garbage-rich old regions)
                                                │
             can't keep up / to-space exhausted ▼
                                    Pause Full (G1 Compaction Pause)   💀 seconds
```

| GC type | Log text | Cleans | Typical pause | Worry? |
|---------|----------|--------|---------------|--------|
| **Young** | `Pause Young (Normal)` | Eden + Survivor | 5–50 ms | No, unless very frequent |
| **Concurrent cycle** | `Concurrent Mark Cycle`, `Pause Remark`, `Pause Cleanup` | Marks old gen while app runs | Remark few ms | No |
| **Mixed** | `Pause Young (Mixed)` | Young + some Old | 10–100 ms | No — healthy G1 |
| **Full** | `Pause Full (G1 Compaction Pause)` | Everything, single compaction | 1–10+ s | **Yes** |

Common Full GC causes in the cause field: `(G1 Compaction Pause)` after evacuation failure, `(Metadata GC Threshold)` (Metaspace), `(System.gc())`.

> ⚠️ A healthy G1 service may go **days without a Full GC**. One Full GC = investigate. Back-to-back Full GCs = outage incoming.

**Interview one-liner:** Young GCs clean Eden, mixed GCs reclaim old regions after a concurrent mark, and a Full GC is G1's stop-the-world fallback — in a tuned service it should almost never appear.

---

## 4. Pause time vs throughput

*Definition:* **Pause time (latency)** is how long a single GC freezes your app. **Throughput** is the % of total time the app spends doing real work instead of GC. You trade one for the other.

Real life: A restaurant can close for 1 big clean at night (high throughput, one long pause) or wipe tables constantly (short pauses, more total cleaning effort).

```
GC overhead % = total pause time / wall-clock time
e.g. 840 young GCs × 12 ms = 10 s paused in 1 hour  →  0.28%  ✅  (keep < 5%)
```

| Collector | Optimises | Pause | Use for |
|-----------|-----------|-------|---------|
| **Parallel** | Throughput | 100 ms – seconds | Batch jobs, reports, ETL |
| **G1** (default) | Balanced, pause goal | ~10–200 ms | Most Spring Boot APIs |
| **ZGC** (generational in 21) | Latency | < 1 ms (mostly) | Large heaps, strict p99 |
| **Shenandoah** | Latency | < 10 ms | Similar to ZGC (Red Hat builds) |

> ⚠️ A GC pause adds **directly** to API latency. A 400 ms pause on a pod means every in-flight `/checkout` request on that pod got +400 ms — it shows up in your **p99**, not your average.

**Easy rule of thumb:** API with p99 SLA → G1 (or ZGC for big heaps). Offline batch → Parallel.

**Interview one-liner:** Pause time is how long each GC freezes the app, throughput is the share of time not spent in GC; G1 balances both with a pause goal, Parallel maximises throughput, ZGC minimises pauses.

---

## 5. Allocation rate, promotion and humongous objects

*Definition:* **Allocation rate** = how fast the app creates new objects (MB/s). **Promotion** = objects surviving enough young GCs (or overflowing Survivor) get copied to Old gen. A **humongous object** is one ≥ **half a G1 region**; it is allocated directly in Old gen in contiguous regions.

```
Allocation rate ≈ Eden size / time between young GCs
  Eden 150 regions × 4 MB = 600 MB, young GC every 2 s  →  ~300 MB/s

Promotion rate  ≈ Old-regions growth per young GC
  Old 300 → 305 regions × 4 MB = 20 MB per GC  →  ~10 MB/s
```

| Symptom | Likely cause in our stack | Fix |
|---------|---------------------------|-----|
| Young GC every 200 ms | Huge allocation rate: building giant JSON, `findAll()` of 100k orders | Paginate, stream, reuse buffers |
| High promotion | Objects live "medium" long (request-scoped caches, big batches) | Bigger young gen / Survivor, shorter lifetimes |
| `Humongous regions: 4->90` | 8 MB `byte[]` for PDF invoices, big `ArrayList` of rows | Stream the file, chunk the list, `-XX:G1HeapRegionSize=16m` |
| `(G1 Humongous Allocation)` as cause | Same — humongous allocs trigger concurrent cycles | Same |

```bash
# region size is printed at startup
[0.012s][info][gc,init] Heap Region Size: 4M
```

> ⚠️ Humongous objects waste the tail of their last region and fragment Old gen — a classic trigger for to-space exhausted → **Full GC**, even when heap "looks" half empty.

**Interview one-liner:** I estimate allocation rate from Eden size over young-GC interval, watch promotion into Old gen, and treat many humongous regions as a red flag — usually big arrays from unpaginated queries or file buffers.

---

## 6. Signs of trouble (what to look for)

*Definition:* GC trouble is when GC stops being background noise and starts **stealing latency or memory** — visible as patterns across many log lines, not one bad line.

```
Heap AFTER GC over 6 hours
Healthy:   ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁   flat baseline
Leak:      ▁▂▂▃▃▄▄▅▅▆▆▇▇███   rising baseline → Full GCs → OOM
```

| Sign in logs | Meaning | Next step |
|--------------|---------|-----------|
| Back-to-back `Pause Full` reclaiming little (`3980M->3950M`) | Live data ≈ heap. Leak or heap too small | Heap dump (see Memory Leak notes) |
| Old gen after mixed GC keeps rising | Memory leak / unbounded cache | Compare two heap dumps |
| `To-space exhausted` / `Evacuation Failure` | No free regions to copy into | More heap, fewer humongous objects |
| `Pause Full (Metadata GC Threshold)` | Metaspace full — classloader leak, heavy proxies | `-XX:MaxMetaspaceSize`, check redeploys |
| `Pause Full (System.gc())` | Code/library calling `System.gc()` | Find caller, or `-XX:+DisableExplicitGC` (careful: NIO direct buffers) |
| `Real` ≫ `User` | Container CPU throttled | Check CPU limits / `container_cpu_cfs_throttled_seconds_total` |
| Young pause slowly growing | Too much live data in young gen | Look at what survives |

> ⚠️ `java.lang.OutOfMemoryError: GC overhead limit exceeded` is a **Parallel GC** message (≥98% time in GC, <2% recovered). G1 just keeps doing Full GCs and finally throws `Java heap space`.

**Interview one-liner:** The red flags are back-to-back Full GCs that reclaim little, a rising heap-after-GC baseline, evacuation failures and Metaspace-triggered Full GCs — each points to a leak, an undersized heap or humongous allocations.

---

## 7. Tools — GCeasy, GCViewer, jstat, Grafana

*Definition:* Raw logs are too long to read by eye; **analysers** turn them into pause histograms, heap-after-GC graphs and allocation-rate numbers.

| Tool | What it's good for |
|------|--------------------|
| **GCeasy** (gceasy.io) | Upload log → pause stats, allocation/promotion rate, problem detection. ⚠️ don't upload logs with secrets to a SaaS |
| **GCViewer** (open source, local) | Offline graphs of heap/pauses, safe for prod logs |
| **JDK Mission Control + JFR** | GC events with allocation stack traces |
| **jstat** | Live per-second sampling on the box |
| **Micrometer → Prometheus → Grafana** | Long-term trends + alerts across all pods |

```bash
jstat -gcutil <pid> 1000        # sample every 1 s
  S0     S1     E      O      M     CCS    YGC   YGCT    FGC  FGCT    CGC  CGCT    GCT
  0.00 100.00  45.21  62.10  97.30  94.85  842   9.812    0  0.000   12  0.210  10.022
  0.00 100.00  91.70  62.10  97.30  94.85  842   9.812    0  0.000   12  0.210  10.022
  0.00 100.00   3.10  62.40  97.30  94.85  843   9.825    0  0.000   12  0.210  10.035
# E = Eden %, O = Old %, YGC/FGC = counts, *GCT = total seconds. Watch O after each YGC and FGC climbing.
```

```promql
# % of time paused in GC per pod (alert if > 5%)
sum by (pod) (rate(jvm_gc_pause_seconds_sum[5m]))
# max single pause
max by (pod, cause) (jvm_gc_pause_seconds_max)
# allocation and promotion rate (bytes/s)
rate(jvm_gc_memory_allocated_bytes_total[5m])
rate(jvm_gc_memory_promoted_bytes_total[5m])
```

**Interview one-liner:** I use GCViewer or GCeasy for a log deep-dive, `jstat -gcutil` for a live look on the box, and Micrometer GC metrics in Grafana for trends and alerts on GC time % and max pause.

---

## 8. Tuning knobs, containers and the G1 vs ZGC choice

*Definition:* **GC tuning** means first giving the JVM the right **heap size**, then setting a **goal** (pause or throughput) — not copying 30 flags from a blog.

```bash
# Spring Boot in K8s, memory limit 2Gi → heap ≈ 1.5 GB, rest = metaspace, threads, direct buffers
# MaxGCPauseMillis=200 is the default; it is a goal, not a guarantee
java -XX:+UseG1GC \
     -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=75.0 \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
     -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=20m \
     -jar app.jar

java -XX:MaxRAMPercentage=75.0 -XX:+PrintFlagsFinal -version | grep -E "MaxHeapSize|UseG1GC"   # verify
```

| Knob | What it does | Gotcha |
|------|--------------|--------|
| `-XX:MaxRAMPercentage` | Heap as % of **container** limit (container-aware since JDK 10) | Default is only **25%** → 2Gi pod gets 512 MB heap |
| `-Xms = -Xmx` / Initial = Max % | No heap resizing at runtime | Uses memory upfront — fine in containers |
| `-XX:MaxGCPauseMillis` | G1 pause target; G1 sizes young gen to meet it | Too low (10 ms) → tiny young gen, more GCs, worse throughput |
| `-XX:G1HeapRegionSize` | Region size (power of 2) | Raise to make big arrays non-humongous |
| `-XX:InitiatingHeapOccupancyPercent` | When to start concurrent mark (45, adaptive) | Lower if Full GCs happen before marking finishes |
| `-Xmn` / `NewRatio` | Fixes young gen size | ❌ With G1 it **disables** the pause-time goal |

| | **G1** | **ZGC (JDK 21 generational)** |
|---|---|---|
| Pause | ~10–200 ms, goal-based | Sub-millisecond, independent of heap size |
| Heap size sweet spot | 1–32 GB | 8 GB to TBs |
| CPU / memory overhead | Lower | More CPU + headroom (concurrent work) |
| Enable | default | `-XX:+UseZGC -XX:+ZGenerational` (21); generational is default from 23 |

> ⚠️ With < 2 CPUs or < ~1792 MB memory the JVM thinks it's not a "server-class machine" and silently picks **SerialGC**. Tiny K8s pods (`cpu: 500m`) often run Serial — set `-XX:+UseG1GC` explicitly.

**Easy rule of thumb:** Size the heap right first (fixes 80% of GC issues), keep G1 default, only move to ZGC when p99 SLAs fail because of GC pauses on big heaps.

**Interview one-liner:** In containers I size heap with `MaxRAMPercentage` around 75%, keep G1 with its pause goal, never set `-Xmn` on G1, and switch to generational ZGC only for big heaps with strict tail-latency needs.

---

## 9. How I'd actually debug a GC problem in production + what interviewers probe

**Scenario:** `/api/orders` p99 jumps from 300 ms to 2.5 s on one or two `order-service` pods, every few minutes.

1. **Correlate.** In Grafana, overlay p99 per pod with `jvm_gc_pause_seconds_max`. Spikes line up → GC suspect. Not aligned → look at DB/pool (see Slow API notes).
2. **Grab the log.** `kubectl cp order-service-7d9f:/logs/gc.log ./gc.log` (or enable live with `jcmd <pid> VM.log ...`).
3. **Classify the pauses.** `grep -E "Pause Full|To-space|Humongous|Evacuation Failure" gc.log | tail`.
4. **Check heap-after-GC trend** in GCViewer: flat → sizing/humongous issue; rising → leak → heap dump.
5. **Check allocation rate** (GCeasy / `jvm_gc_memory_allocated_bytes_total`): 1 GB/s after a release → find the endpoint (JFR allocation profiling).
6. **Fix the cause, not the GC:** paginate the `findAll()`, stream the invoice PDF, bound the cache. Then right-size heap (`MaxRAMPercentage`) and confirm pod memory limit.
7. **Verify** for 24 h: GC time < 5%, no Full GC, p99 back to baseline. Add an alert on GC time % and Full GC count.

| Interviewer asks | Strong answer |
|------------------|---------------|
| "How do you enable GC logging on JDK 17?" | `-Xlog:gc*:file=...:time,uptime:filecount=5,filesize=20m`; `jcmd VM.log` at runtime |
| "Young vs mixed vs full in G1?" | Young = Eden/Survivor; mixed = young + selected old after concurrent mark; full = STW whole-heap compaction fallback |
| "How do you spot a leak in GC logs?" | Heap-after-GC baseline rising, Full GCs reclaiming little |
| "What is a humongous object?" | ≥ half a region, allocated straight into old gen contiguous regions; causes fragmentation |
| "Pause time vs throughput?" | Single-freeze length vs % time not in GC; G1 balances, Parallel throughput, ZGC latency |
| "Heap in Docker?" | Container-aware JVM; set `MaxRAMPercentage` ~75%, leave room for non-heap |
| "When ZGC?" | Big heaps + strict p99; costs more CPU; generational in JDK 21 |
| "Would you just raise -Xmx?" | Only after checking it's not a leak — more heap delays a leak, and longer heaps can mean longer Full GCs |

**Interview one-liner:** I correlate latency spikes with GC pause metrics, pull the GC log, classify the pauses, check heap-after-GC and allocation rate, fix the allocation source, then right-size the heap and verify with alerts.

---

## Quick Revision Cheat Sheet

- **Enable logs**: `-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m`; live via `jcmd <pid> VM.log`; mount the log dir in K8s.
- **Read a line**: type + cause, heap before → after (total), pause ms; `Real ≫ User` = CPU throttling.
- **Young / Mixed / Full**: Eden only / young + some old after concurrent mark / STW whole heap — Full should be rare.
- **Pause vs throughput**: freeze length vs % time not in GC; Parallel = throughput, G1 = balanced, ZGC = latency; keep GC time < 5%.
- **Allocation & promotion**: Eden size ÷ young-GC interval; humongous = ≥ half region, straight to old, causes fragmentation.
- **Red flags**: back-to-back Full GCs, rising heap-after-GC, to-space exhausted, Metadata GC Threshold, `System.gc()`.
- **Tools**: GCViewer/GCeasy for logs, `jstat -gcutil <pid> 1000` live, Micrometer `jvm_gc_pause_seconds_*` in Grafana.
- **Tuning**: `MaxRAMPercentage=75`, `MaxGCPauseMillis` goal, never `-Xmn` with G1, force G1 on tiny pods; ZGC for big heaps + strict p99.
- **Prod debug**: correlate p99 with GC pauses → pull log → classify → heap-after-GC trend → fix allocation source → right-size → alert.
