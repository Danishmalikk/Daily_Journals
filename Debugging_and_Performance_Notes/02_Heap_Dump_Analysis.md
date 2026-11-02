# Debugging & Performance Notes — Heap Dump Analysis (jcmd, jmap, HeapDumpOnOutOfMemoryError, Eclipse MAT, Dominator Tree, Retained Size, GC Roots)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a heap dump? (and when you need one)

*Definition:* A **heap dump** is a snapshot file (`.hprof`) of **every object on the Java heap** at one moment — its class, fields, and who references whom. You open it in **Eclipse MAT**, VisualVM, or JProfiler to find **what is filling memory and why it can't be garbage-collected**.

Real life: A photo of a cluttered warehouse plus a map of which shelf is holding up which box. You find the one giant shelf (a static cache) that nobody is allowed to empty.

```
Grafana: jvm_memory_used_bytes{area="heap"} after each GC
   │      /|    /|     /|      /|
   │     / |   / |    / |  ___/ |   ← "floor" after GC keeps RISING = leak
   │  /|/  |__/  |___/  |_/     |
   │_/                            → java.lang.OutOfMemoryError: Java heap space
   └──────────────────────────────── time
```

| Symptom | Likely meaning | Heap dump helps? |
|---------|----------------|------------------|
| Sawtooth with **flat** floor | Normal | No |
| Sawtooth with **rising** floor | Memory leak | ✅ Yes |
| `OutOfMemoryError: Java heap space` | Leak or heap too small | ✅ Yes |
| `GC overhead limit exceeded` | Heap almost full, GC thrashing | ✅ Yes |
| `OutOfMemoryError: Metaspace` | Class/classloader leak | ✅ Yes (classloaders) |
| Pod `OOMKilled` (exit 137) | Container limit hit, maybe off-heap | ⚠️ Often no — native memory |

**Interview one-liner:** A heap dump is a full snapshot of heap objects and their references; I take one when heap-after-GC keeps rising or on OutOfMemoryError, and analyze it in Eclipse MAT.

---

## 2. Taking a heap dump — jcmd, jmap, on OOM

*Definition:* You can take a dump **on demand** (`jcmd`, `jmap`, VisualVM, JProfiler) or **automatically** when the JVM throws `OutOfMemoryError`.

```bash
# ✅ Preferred on-demand (JDK 9+). Live objects only by default → triggers a full GC first
jcmd 1 GC.heap_dump /dumps/order-svc-$(date +%s).hprof
jcmd 1 GC.heap_dump -all /dumps/all.hprof          # include unreachable garbage too
jcmd 1 GC.heap_dump -gz=1 /dumps/heap.hprof.gz     # compressed (JDK 15+)

# Older equivalent
jmap -dump:live,format=b,file=/dumps/heap.hprof 1

# ✅ ALWAYS set these in prod JVM options
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps                  # directory → file named java_pid<pid>.hprof
-XX:+ExitOnOutOfMemoryError              # die fast so K8s restarts a clean pod
```

| Method | When | Notes |
|--------|------|-------|
| `jcmd <pid> GC.heap_dump` | On demand | Modern, recommended |
| `jmap -dump:live,...` | On demand | Legacy, still works |
| `-XX:+HeapDumpOnOutOfMemoryError` | Automatic at OOM | Captures the exact failing state |
| VisualVM / JProfiler "Heap Dump" button | Local/staging | GUI convenience |

> ⚠️ Taking a dump is **stop-the-world** — a 4GB heap can freeze the app for seconds and write a ~4GB file. Pull the pod out of the load balancer first (or dump a single replica), and make sure the disk has room.

> ⚠️ Heap dumps contain **everything in memory**: customer PII, JWTs, DB passwords, Razorpay keys. Treat `.hprof` files as secrets — encrypted S3 bucket, short retention, never attach to a public ticket.

**Interview one-liner:** I use `jcmd <pid> GC.heap_dump` on demand and always run prod with `-XX:+HeapDumpOnOutOfMemoryError` plus a `HeapDumpPath`, remembering a dump pauses the JVM and contains sensitive data.

---

## 3. Class histogram — the cheap first look

*Definition:* A **class histogram** lists, per class, the **instance count and shallow bytes** on the heap — no dump file, takes a second or two. Comparing two histograms a few minutes apart shows which class keeps growing.

```bash
jcmd 1 GC.class_histogram | head -8        # (jmap -histo:live 1 is the legacy form)
```
```
 num     #instances         #bytes  class name (module)
-------------------------------------------------------
   1:       4523190      361855200  [B (java.base@21.0.4)
   2:       4501022      108024528  java.lang.String (java.base@21.0.4)
   3:       1500340       72016320  com.shop.order.OrderEvent        ◄ business object, 1.5M?!
   4:       1500412       48013184  java.util.concurrent.ConcurrentHashMap$Node
```

- `[B` = `byte[]` (String contents live here), `[C` = `char[]`, `[Ljava.lang.Object;` = `Object[]`.
- `byte[]`/`String` at the top is normal; a **domain class** in the top 5 with millions of instances is a red flag.
- Take histogram → wait 5 min → take again → diff the counts.

**Interview one-liner:** Before a full dump I take two class histograms with `jcmd GC.class_histogram` minutes apart — a domain class whose count keeps climbing is my leak suspect.

---

## 4. Shallow size vs retained size

*Definition:* **Shallow size** is the memory of the object itself (header + fields). **Retained size** is the memory that would be **freed if this object were garbage-collected** — itself plus everything only reachable through it.

```
        static OrderCache.INSTANCE
                 │
        ConcurrentHashMap   shallow: 64 B     retained: 1.4 GB  ◄ the real culprit
        ┌────────┼────────┐
     Node[]    Node ...  Node                 (each Node → OrderEvent → byte[] payload)
```

| | **Shallow size** | **Retained size** |
|---|------------------|-------------------|
| Counts | Only the object's own fields | Object + everything it alone keeps alive |
| A `HashMap` with 1M entries | ~48 bytes | Hundreds of MB |
| Use for | Spotting many small objects | **Finding who to blame** |

Memory trick: **"Shallow = the suitcase, Retained = the suitcase + everything packed inside."**

> ⚠️ If an object is also referenced from somewhere else, it is **not** in your retained size. Retained = "what would be freed if *only* this went away".

**Interview one-liner:** Shallow size is the object alone; retained size is everything that would be freed if it were collected — I always sort by retained size to find the leak owner.

---

## 5. Dominator tree — who really owns the memory

*Definition:* In MAT's **dominator tree**, object X **dominates** Y if **every** path from the GC roots to Y goes through X. The tree groups memory under its true owner, so the biggest retained-size node at the top is usually the leak.

```
MAT → Dominator Tree (sorted by Retained Heap)
Class Name                                              Shallow    Retained   %
class com.shop.order.OrderCache @ 0x7a1c0               16         1.41 GB   78.4%
└─ java.util.concurrent.ConcurrentHashMap @ 0x7a1d8     64         1.41 GB   78.4%
   └─ java.util.concurrent.ConcurrentHashMap$Node[]     8,388,624  1.41 GB   78.3%
org.apache.tomcat.util.threads.TaskThread "http-nio-8080-exec-3"  120  34 MB  1.9%
```

- Right-click → **"Group by class / package"** to see `com.shop.*` totals.
- **Histogram view** = per class; **dominator tree** = per owner. Use the dominator tree for blame.

**Interview one-liner:** The dominator tree shows which object exclusively keeps the most memory alive — the top retained-size node is usually the leak's owner, like a static cache map.

---

## 6. Leak Suspects report & path to GC roots

*Definition:* **GC roots** are references the GC always treats as alive: thread stacks (local variables), static fields (via loaded classes), JNI references, and monitors. An object leaks when a chain from a GC root still reaches it. MAT's **Leak Suspects** report auto-finds big retainers, and **Path to GC Roots** shows the exact chain.

```
MAT → Leak Suspects
Problem Suspect 1
One instance of "java.util.concurrent.ConcurrentHashMap" loaded by "<system class loader>"
occupies 1,512,330,112 (78.41%) bytes.
Keywords: java.util.concurrent.ConcurrentHashMap, com.shop.order.OrderCache

Path to GC Roots (exclude weak/soft references):
com.shop.order.OrderEvent @ 0x8c1f2
 └─ val       java.util.concurrent.ConcurrentHashMap$Node
    └─ [12345] java.util.concurrent.ConcurrentHashMap$Node[]
       └─ table  java.util.concurrent.ConcurrentHashMap
          └─ CACHE  class com.shop.order.OrderCache   ◄ static field = GC root
```

> ⚠️ Always choose **"exclude weak/soft references"** — those don't prevent GC and just add noise. **"Merge Shortest Paths to GC Roots"** on a whole class shows the common chain for millions of instances.

Memory trick: **GC roots = "S-T-J-M"** → **S**tatics, **T**hread stacks, **J**NI refs, **M**onitors.

**Interview one-liner:** I run MAT's Leak Suspects report, then "Path to GC Roots excluding weak/soft references" on the suspect to see exactly which static field or thread is keeping it alive.

---

## 7. Common leak patterns in Spring Boot services

| Leak pattern | What you see in MAT | Fix |
|--------------|---------------------|-----|
| Static `Map` used as a cache | Huge `ConcurrentHashMap` under a `class` | Bounded cache (Caffeine `maximumSize` + `expireAfterWrite`) or Redis |
| `ThreadLocal` not removed in Tomcat pool | Data under `Thread.threadLocals` of `http-nio-*` threads | `try { ... } finally { tl.remove(); }` |
| Hibernate session in a long batch | Millions of entities in `StatefulPersistenceContext` | `flush()` + `clear()` every N rows, or `StatelessSession` |
| Listeners/callbacks never unregistered | Lists of lambdas/listeners growing | Unregister on shutdown, use weak listeners |
| High-cardinality Micrometer tags (`orderId`) | Millions of `Meter.Id` objects | Tag by bounded values (status, endpoint) |
| Unbounded queue/`ExecutorService` | `LinkedBlockingQueue` with millions of tasks | Bounded queue + rejection policy |

```java
// ❌ Grows forever: one entry per order, never evicted
private static final Map<String, OrderEvent> CACHE = new ConcurrentHashMap<>();

// ✅ Bounded and time-limited
private final Cache<String, OrderEvent> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .build();
```

**Easy rule of thumb:** Every cache, map, queue or `ThreadLocal` needs an answer to "**when does this entry go away?**" — if there's no answer, it's a leak waiting to happen.

**Interview one-liner:** Most production leaks I've seen are unbounded static caches, ThreadLocals left on pooled threads, huge Hibernate sessions in batches, and high-cardinality metric tags.

---

## 8. Heap dumps in Docker & Kubernetes

*Definition:* In containers, the hard parts are **where the dump goes** (container filesystem disappears on restart), **sizing the heap vs the container limit**, and **getting the file off the pod**.

```yaml
# Deployment snippet
env:
  - name: JAVA_TOOL_OPTIONS
    value: >-
      -XX:MaxRAMPercentage=75.0
      -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
      -XX:+ExitOnOutOfMemoryError
volumeMounts:
  - { name: dumps, mountPath: /dumps }
volumes:
  - name: dumps
    emptyDir: { sizeLimit: 6Gi }   # survives container restart, NOT pod deletion → or use a PVC
```

```bash
kubectl exec order-svc-7d9 -- jcmd 1 GC.heap_dump /dumps/manual.hprof
kubectl cp order-svc-7d9:/dumps/manual.hprof ./manual.hprof
# or ship straight to S3 from the pod
kubectl exec order-svc-7d9 -- aws s3 cp /dumps/manual.hprof s3://shop-debug-dumps/ --sse aws:kms
```

| Event | Exit / signal | Heap dump? |
|-------|---------------|-----------|
| Java `OutOfMemoryError` (heap full) | JVM exits (with `ExitOnOutOfMemoryError`) | ✅ Yes |
| Kernel **OOMKilled** (container RSS > limit) | Exit **137** (SIGKILL) | ❌ No — JVM never gets a chance |

> ⚠️ **OOMKilled ≠ Java OOM.** If the pod dies with 137 and no `.hprof`, the heap probably wasn't full — it's **off-heap** (thread stacks, Metaspace, direct buffers, Netty) or `-Xmx` too close to the limit. Use `-XX:NativeMemoryTracking=summary` + `jcmd 1 VM.native_memory summary`, and keep heap ≈ 75% of the limit.

> ⚠️ Analyzing a 4GB dump needs a machine with **more** than 4GB free for MAT (`-Xmx` in `MemoryAnalyzer.ini`). For huge dumps, run MAT headless on a big EC2 box: `ParseHeapDump.sh heap.hprof org.eclipse.mat.api:suspects`.

**Interview one-liner:** In K8s I write OOM dumps to a mounted volume, cap heap with `MaxRAMPercentage=75`, copy dumps out via `kubectl cp` or S3, and remember that an OOMKilled (137) pod means native memory — no heap dump will exist.

---

## 9. How I'd actually debug a memory leak in production + what interviewers probe

Scenario: `order-svc` pods restart every ~6 hours with `OutOfMemoryError: Java heap space`.

1. **Grafana**: heap-after-GC floor rising? GC pause time and frequency climbing? Restarts line up with traffic or with time? (Time-based → leak; traffic spike → sizing.)
2. **Confirm it's heap, not native**: Java OOM in logs + `.hprof` in `/dumps` → heap. Exit 137 with no dump → native/limits (NMT).
3. **Cheap first look**: two `jcmd 1 GC.class_histogram` runs 10 min apart on a live pod → which class grows.
4. **Get the dump**: use the OOM `.hprof`, or take `jcmd 1 GC.heap_dump` from a pod drained from the LB; upload to an encrypted S3 bucket.
5. **MAT**: Leak Suspects → Dominator Tree (sort by retained) → Path to GC Roots (exclude weak/soft).
6. **Map to code**: the root is e.g. `static OrderCache.CACHE` → add bounds/TTL, or move to Redis.
7. **Verify**: load test in staging, watch heap-after-GC stay flat for hours; add an alert on heap-after-GC > 80%.

| Interviewer asks | Strong answer |
|------------------|---------------|
| "Shallow vs retained size?" | Shallow = object itself; retained = what's freed if it's collected. Blame by retained. |
| "What is a dominator?" | X dominates Y if every path from GC roots to Y goes through X. |
| "What are GC roots?" | Thread stack locals, static fields, JNI refs, monitors. |
| "How do you take a dump?" | `jcmd <pid> GC.heap_dump`, `jmap -dump`, or `-XX:+HeapDumpOnOutOfMemoryError`. |
| "Does taking a dump affect prod?" | Yes — STW pause + big file; drain the pod first; dumps contain secrets. |
| "Pod OOMKilled but no heap dump?" | Kernel killed it (137) — off-heap/native memory or heap too close to limit. |
| "Can Java leak memory with a GC?" | Yes — anything still reachable from a GC root is never collected. |

**Interview one-liner:** Rising heap-after-GC → histogram diff → heap dump → MAT Leak Suspects, dominator tree and path to GC roots → bound or fix the owning reference → prove a flat heap in a soak test.

---

## Quick Revision Cheat Sheet

- **Heap dump**: `.hprof` snapshot of all objects + references; take it when heap-after-GC floor rises or on OOM.
- **Taking dumps**: `jcmd <pid> GC.heap_dump` (live by default, `-all`, `-gz`), `jmap -dump:live`, `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath`; STW + contains secrets.
- **Class histogram**: `jcmd GC.class_histogram` twice, diff counts — growing domain class = suspect.
- **Shallow vs retained**: object alone vs everything freed with it; always sort by retained.
- **Dominator tree**: groups memory under its exclusive owner; top node = likely leak.
- **Leak Suspects + GC roots**: Path to GC Roots excluding weak/soft refs; roots = Statics, Thread stacks, JNI, Monitors.
- **Spring leaks**: static maps, ThreadLocals on pooled threads, huge Hibernate sessions, high-cardinality tags, unbounded queues.
- **K8s**: dump to mounted volume, `MaxRAMPercentage=75`, `kubectl cp`/S3; OOMKilled (137) = native memory, no heap dump.
- **Prod flow**: Grafana → heap vs native → histogram diff → dump → MAT → fix reference → soak test.
