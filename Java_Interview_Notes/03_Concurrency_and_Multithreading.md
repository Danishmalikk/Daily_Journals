# Java Interview Notes — Multithreading & Concurrency

> Simple-language notes for interview revision. Read top to bottom.

---

## 1. Thread Lifecycle (the 5 states)

*Definition:* A **thread** is a single path of execution inside a program. Its **lifecycle** is the set of states it passes through from creation to finish.

Real life: A worker — hired (New), ready at desk (Runnable), running a task (Running), waiting for a tool someone else holds (Blocked/Waiting), goes home (Terminated).

```
New → Runnable → Running → Terminated
                   ↑  ↓
            Blocked / Waiting / Timed-Waiting
```

| State | Meaning |
|-------|---------|
| **New** | Thread object created, but `start()` not called yet |
| **Runnable** | `start()` called, ready to run (waiting for CPU) |
| **Blocked** | Waiting to enter a `synchronized` lock another thread holds |
| **Waiting** | Waiting indefinitely (`wait()`, `join()`) until notified |
| **Timed-Waiting** | Waiting for a fixed time (`sleep(1000)`, `wait(500)`) |
| **Terminated** | Thread finished or stopped |

> Note: Java has no separate "Running" state in the enum — it's part of `RUNNABLE`. The OS decides when a runnable thread actually runs.

**Interview one-liner:** A thread goes New → Runnable → (Blocked/Waiting/Timed-Waiting) → Terminated. `start()` makes it Runnable; locks make it Blocked; `wait()`/`join()` make it Waiting.

---

## 2. Runnable vs Callable vs Thread

*Definition:* These are the three ways to define work for a thread. **Thread** is the worker itself; **Runnable** and **Callable** are tasks you hand to a worker.

| | Thread | Runnable | Callable |
|--|--------|----------|----------|
| Type | Class (extend it) | Interface | Interface |
| Method | `run()` | `run()` | `call()` |
| Returns value? | ❌ No | ❌ No | ✅ Yes |
| Throws checked exception? | ❌ No | ❌ No | ✅ Yes |

```java
// Runnable — no result
Runnable r = () -> System.out.println("working");
new Thread(r).start();

// Callable — returns a result
Callable<Integer> c = () -> 2 + 3;
Future<Integer> f = executor.submit(c);
System.out.println(f.get());   // 5
```

> Prefer **Runnable/Callable** over extending **Thread** — Java allows only one parent class, so extending Thread wastes your single inheritance.

**Interview one-liner:** Thread is the worker; Runnable is a task with no return; Callable is a task that returns a value and can throw. Prefer implementing Runnable/Callable over extending Thread.

---

## 3. ExecutorService & Thread Pools

*Definition:* An **ExecutorService** is a manager that holds a **pool** of reusable threads and runs your tasks on them — so you don't create a new thread for every task (which is slow and memory-heavy).

Real life: A taxi company with a fixed fleet of cars, instead of buying a new car for every customer.

```java
ExecutorService pool = Executors.newFixedThreadPool(3);
pool.submit(() -> System.out.println("task"));
pool.shutdown();   // always shut down when done
```

### Types of pools

| Pool | What it does | Best for |
|------|--------------|----------|
| **Fixed** (`newFixedThreadPool(n)`) | Exactly n threads; extra tasks wait in a queue | Steady, known load |
| **Cached** (`newCachedThreadPool()`) | Creates threads as needed, reuses idle ones, kills after 60s | Many short, bursty tasks |
| **Scheduled** (`newScheduledThreadPool(n)`) | Runs tasks after a delay or on a repeat | Timers, periodic jobs |
| **Work-Stealing** (`newWorkStealingPool()`) | Idle threads "steal" tasks from busy threads' queues | Many small parallel tasks |

> Always call `shutdown()`. Otherwise the JVM may never exit (pool threads keep it alive).

**Interview one-liner:** ExecutorService reuses a pool of threads instead of creating new ones. Fixed = set size, Cached = grows/shrinks, Scheduled = delayed/repeated, Work-Stealing = idle threads steal work.

---

## 4. Future & CompletableFuture

### Future
*Definition:* A **Future** is a placeholder for a result that isn't ready yet — you submit a task now and call `get()` later to fetch the result (which blocks until it's done).

```java
Future<Integer> f = pool.submit(() -> 10 * 2);
Integer result = f.get();   // BLOCKS until ready → 20
```
Limitation: `get()` blocks, and you can't easily chain steps.

### CompletableFuture (the modern upgrade)
*Definition:* A **CompletableFuture** is a Future you can **chain and combine** without blocking — you describe "do this, then that" and it runs in the background.

```java
CompletableFuture.supplyAsync(() -> 5)
    .thenApply(n -> n * 2)        // transform: 5 → 10
    .thenAccept(n -> print(n));   // use result, no return
```

| Method | What it does |
|--------|--------------|
| `thenApply` | Transform the result (returns a value) |
| `thenCompose` | Chain another CompletableFuture (flatten, avoids nesting) |
| `thenCombine` | Merge results of two futures |
| `allOf` | Wait for **all** futures to finish |
| `anyOf` | Finish when **any** one future completes |

> `thenApply` vs `thenCompose`: use **Apply** when your function returns a plain value, **Compose** when it returns another CompletableFuture (otherwise you get a nested `Future<Future<...>>`).

**Interview one-liner:** Future holds a result but `get()` blocks. CompletableFuture chains tasks non-blocking — `thenApply` transforms, `thenCompose` chains another future, `allOf`/`anyOf` wait for all/any.

---

## 5. `synchronized` keyword (method vs block)

*Definition:* **`synchronized`** puts a lock around code so only **one thread at a time** can run it — preventing two threads from corrupting shared data.

Real life: A single toilet with one key — only the person holding the key gets in; others wait.

```java
// Method-level — locks the whole method
synchronized void deposit(int amt) { balance += amt; }

// Block-level — locks only the critical part (faster)
void deposit(int amt) {
    doSomeStuff();              // runs in parallel
    synchronized (this) {       // only this part is locked
        balance += amt;
    }
}
```

| | synchronized method | synchronized block |
|--|--------------------|--------------------|
| Locks | Whole method | Only chosen lines |
| Lock object | `this` (or class for static) | Any object you pick |
| Performance | Slower (bigger locked area) | Faster (smaller locked area) |

**Interview one-liner:** `synchronized` lets only one thread run the locked code. Method-level locks the whole method; block-level locks just the critical section, so it's faster.

---

## 6. `volatile` keyword (visibility)

*Definition:* **`volatile`** tells Java that a variable is shared — always read/write it from **main memory**, not a thread's private cache — so every thread sees the latest value.

Real life: A shared whiteboard everyone reads from directly, instead of each person keeping a stale photocopy.

```java
volatile boolean running = true;

void stop() { running = false; }   // other threads see it immediately
void run()  { while (running) {} } // without volatile, may loop forever
```

> `volatile` gives **visibility** but NOT **atomicity**. `count++` is still unsafe with volatile (it's read-modify-write = 3 steps). For counters use `AtomicInteger` or `synchronized`.

**Interview one-liner:** `volatile` guarantees every thread sees the latest value (visibility) by skipping the CPU cache — but it does NOT make compound actions like `count++` atomic.

---

## 7. ReentrantLock vs synchronized

*Definition:* A **ReentrantLock** is a flexible, manual version of `synchronized` — you call `lock()` and `unlock()` yourself, which gives extra powers but also extra responsibility.

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    balance += amt;     // critical section
} finally {
    lock.unlock();      // MUST unlock in finally
}
```

| Feature | synchronized | ReentrantLock |
|---------|--------------|---------------|
| Lock/unlock | Automatic | Manual (`lock`/`unlock`) |
| Try without blocking | ❌ | ✅ `tryLock()` |
| Timeout | ❌ | ✅ `tryLock(2, SECONDS)` |
| Fairness (FIFO order) | ❌ | ✅ optional |
| Interruptible wait | ❌ | ✅ |

> "Reentrant" = a thread already holding the lock can acquire it again without deadlocking itself. Both `synchronized` and `ReentrantLock` are reentrant.

**Easy rule of thumb:** Use `synchronized` for simple cases. Use `ReentrantLock` when you need `tryLock`, a timeout, fairness, or interruptible locking.

**Interview one-liner:** synchronized is automatic and simple; ReentrantLock is manual but adds tryLock, timeouts, fairness, and interruptibility — just remember to unlock in a `finally`.

---

## 8. Deadlock, Livelock, Starvation

*Definition:* These are three ways threads get **stuck** and stop making progress.

| Problem | What happens | Real life |
|---------|--------------|-----------|
| **Deadlock** | Two threads each hold a lock the other needs — both wait forever | Two people each grab one chopstick, both wait for the other |
| **Livelock** | Threads keep reacting to each other and changing state, but no work gets done | Two people in a hallway both step the same way repeatedly |
| **Starvation** | A thread never gets CPU/lock because others keep grabbing it first | A polite person never gets through a pushy crowd |

### Deadlock example
```java
synchronized (lockA) {
    synchronized (lockB) { ... }   // Thread 1: A then B
}
// Thread 2 does: B then A  →  DEADLOCK
```

### Prevention
- **Lock ordering** → always acquire locks in the same global order (A then B everywhere).
- **tryLock with timeout** → give up if you can't get the lock in time.
- **Avoid nested locks** when possible.
- **Fair locks** → fight starvation with `new ReentrantLock(true)`.

**Interview one-liner:** Deadlock = both stuck waiting on each other's lock; Livelock = both active but no progress; Starvation = one thread never gets a turn. Prevent deadlock with consistent lock ordering and tryLock timeouts.

---

## 9. Race Conditions

*Definition:* A **race condition** is a bug where the result depends on the **unpredictable timing** of threads — two threads touch shared data at once and corrupt it.

```java
int count = 0;
void inc() { count++; }   // read, +1, write = 3 steps

// Two threads run inc() 1000 times each.
// Expected 2000, but you often get less — updates get lost.
```
Why: `count++` isn't atomic. Thread A reads 5, Thread B reads 5, both write 6 → one increment lost.

### How to avoid
- `synchronized` or `ReentrantLock` around the shared update.
- **Atomic** classes (`AtomicInteger`) for counters.
- **Immutable** objects (can't be corrupted if they never change).
- Avoid sharing — use `ThreadLocal` or local variables.

**Interview one-liner:** A race condition is when threads access shared data at the same time and the outcome depends on timing. Fix it with synchronization, atomic variables, or immutability.

---

## 10. Semaphore, CountDownLatch, CyclicBarrier, Phaser

*Definition:* These are **coordination tools** that control how threads wait for each other or for permits.

| Tool | What it does | Real life |
|------|--------------|-----------|
| **Semaphore** | Allows only N threads at once (permits) | Parking lot with N spots |
| **CountDownLatch** | One/many threads wait until a counter hits 0 (one-time) | Wait for 3 services to boot, then start |
| **CyclicBarrier** | All threads wait at a point, then all proceed together (reusable) | Race: everyone waits at the line, then GO |
| **Phaser** | Like a flexible, multi-phase CyclicBarrier (threads can join/leave) | Multi-round game with changing players |

```java
// Semaphore — max 2 threads in the section
Semaphore sem = new Semaphore(2);
sem.acquire();  try { useResource(); } finally { sem.release(); }

// CountDownLatch — wait for 3 tasks to finish
CountDownLatch latch = new CountDownLatch(3);
// each task: latch.countDown();
latch.await();   // blocks until count = 0
```

> Key difference: **CountDownLatch** is one-shot (can't reuse). **CyclicBarrier** resets and can be reused for repeated rounds.

**Interview one-liner:** Semaphore limits concurrent access (N permits); CountDownLatch waits for a count to reach zero (one-time); CyclicBarrier makes threads wait for each other then proceed together (reusable); Phaser is a flexible multi-phase barrier.

---

## 11. ThreadLocal

*Definition:* A **ThreadLocal** gives each thread its **own private copy** of a variable — so threads never share it and can't corrupt each other's value.

Real life: Everyone gets their own personal notebook; no one writes in yours.

```java
ThreadLocal<Integer> userId = ThreadLocal.withInitial(() -> 0);

userId.set(42);          // only this thread sees 42
int id = userId.get();   // 42 here; a different thread sees its own value
userId.remove();         // clean up to avoid memory leaks
```

> Common use: storing per-request data (user ID, DB connection, `SimpleDateFormat`) in web servers, where each request runs on its own thread.
>
> **Gotcha:** in thread pools, threads are reused — always `remove()` after use or old values leak into the next task.

**Interview one-liner:** ThreadLocal gives each thread its own isolated copy of a variable — great for per-thread data like user context. Always `remove()` in thread pools to avoid leaks.

---

## 12. Atomic classes (CAS)

*Definition:* **Atomic** classes (like `AtomicInteger`) let you update a value safely from many threads **without locks**, using a CPU trick called **CAS (Compare-And-Swap)**.

### What is CAS?
*Definition:* **CAS** = "if the value is still what I expect, swap it to the new one; otherwise retry." It's one un-interruptible CPU instruction, so no other thread can sneak in.

```
CAS(memory, expected, new):
   if memory == expected → set memory = new, return true
   else → return false (someone changed it, try again)
```

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();   // safe ++ with no lock
count.compareAndSet(5, 6); // set to 6 only if currently 5

AtomicReference<String> ref = new AtomicReference<>("a");
ref.compareAndSet("a", "b");
```

| | synchronized | Atomic (CAS) |
|--|--------------|--------------|
| Uses locks? | ✅ Yes | ❌ No (lock-free) |
| Speed | Slower (threads wait) | Faster (retry loop) |
| Best for | Complex critical sections | Single counters/references |

> CAS is the engine behind `ConcurrentHashMap`, `AtomicInteger`, and most lock-free code. Downside: under heavy contention, lots of retries.

**Interview one-liner:** Atomic classes update values lock-free using CAS (compare-and-swap): change the value only if it still equals the expected one, else retry. Faster than locks for simple counters.

---

## 13. Fork/Join Framework

*Definition:* The **Fork/Join framework** splits a big task into smaller subtasks (**fork**), runs them in parallel, then combines the results (**join**) — built for divide-and-conquer.

Real life: To count a huge pile of votes, split it among many people, then add up their counts.

```java
class SumTask extends RecursiveTask<Long> {
    protected Long compute() {
        if (small enough) return computeDirectly();
        SumTask left = ...; SumTask right = ...;
        left.fork();                 // run left in parallel
        return right.compute() + left.join();  // combine
    }
}
new ForkJoinPool().invoke(new SumTask(...));
```

> Uses **work-stealing**: an idle thread steals subtasks from a busy thread's queue, keeping all cores busy. This is what powers parallel streams (`list.parallelStream()`).

**Interview one-liner:** Fork/Join splits a big task into subtasks (fork), runs them in parallel, and merges results (join). It uses work-stealing to keep all CPU cores busy — and powers parallel streams.

---

## 14. Virtual Threads (Java 21 — Project Loom)

*Definition:* **Virtual threads** are super-lightweight threads managed by the JVM (not the OS) — you can run **millions** of them cheaply, unlike normal "platform" threads which are heavy and limited.

Real life: Old threads = hiring full-time staff (expensive, limited). Virtual threads = tickets in a queue the JVM juggles over a few real workers.

```java
// Old way — limited (~few thousand max)
Thread.ofPlatform().start(task);

// Virtual thread — can have millions
Thread.ofVirtual().start(task);

// Best with executor
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
    ex.submit(task);
}
```

| | Platform thread | Virtual thread |
|--|-----------------|----------------|
| Managed by | OS | JVM |
| Cost | Heavy (~1MB stack) | Very light (~few KB) |
| How many | Thousands | Millions |
| Best for | CPU-heavy work | Blocking I/O (DB, HTTP calls) |

> When a virtual thread blocks (e.g. waiting on I/O), the JVM "parks" it and reuses the real carrier thread for other work — so blocking becomes cheap. No need for pools or async callbacks.

**Interview one-liner:** Virtual threads (Java 21) are JVM-managed lightweight threads — you can run millions cheaply. The JVM parks them on blocking I/O and reuses real threads, making blocking code scale like async.

---

## Quick Revision Cheat Sheet

- **Lifecycle**: New → Runnable → (Blocked/Waiting/Timed-Waiting) → Terminated.
- **Runnable** = no return; **Callable** = returns value + throws; prefer over extending **Thread**.
- **ExecutorService** reuses a pool: Fixed, Cached, Scheduled, Work-Stealing. Always `shutdown()`.
- **Future** blocks on `get()`; **CompletableFuture** chains: thenApply, thenCompose, allOf, anyOf.
- **synchronized**: one thread at a time; block-level is faster than method-level.
- **volatile**: visibility only, NOT atomicity (`count++` still unsafe).
- **ReentrantLock** > synchronized when you need tryLock, timeout, fairness; unlock in `finally`.
- **Deadlock** (mutual wait), **Livelock** (active no progress), **Starvation** (never gets turn). Fix deadlock with lock ordering.
- **Race condition** = timing-dependent corruption; fix with sync, atomics, or immutability.
- **Semaphore** (N permits), **CountDownLatch** (one-time count to 0), **CyclicBarrier** (reusable meet-point), **Phaser** (multi-phase).
- **ThreadLocal** = per-thread copy; `remove()` in pools.
- **Atomic** = lock-free via **CAS** (compare-and-swap, retry if changed).
- **Fork/Join** = split→parallel→merge with work-stealing; powers parallel streams.
- **Virtual threads** (Java 21) = millions of cheap JVM threads; best for blocking I/O.
