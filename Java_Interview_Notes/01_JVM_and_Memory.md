# Java Interview Notes — JVM, Memory & Core Concepts

> Simple-language notes for interview revision. Read top to bottom.

---

## 1. JVM Architecture (ClassLoader, Heap, Stack, Metaspace)

**What is JVM?**
JVM = Java Virtual Machine. It is the engine that actually *runs* your Java program.
You write `.java` → compiler makes `.class` (bytecode) → JVM runs that bytecode.

**Why JVM is special:** "Write once, run anywhere". Same `.class` file runs on Windows, Mac, Linux — because each OS has its own JVM that understands the same bytecode.

### Main parts of JVM

```
   .java  --(javac)-->  .class (bytecode)  --->  JVM  --->  runs on CPU
```

**1) ClassLoader**
   *Definition:* The ClassLoader is the part of the JVM that finds your `.class` files (the compiled bytecode) and loads them into memory so the program can use them. Think of it as the "loader/fetcher" that brings classes in only when they are first needed (lazy loading).

   Three types (parent-child order):
   - **Bootstrap** → loads core Java classes (java.lang.*, etc.)
   - **Extension/Platform** → loads extra library classes
   - **Application** → loads YOUR classes (from classpath)

   Rule: **Delegation** — a loader first asks its parent to load. Only if parent can't, it loads itself. (Stops duplicate/fake core classes.)

**2) Heap**
   *Definition:* The Heap is the large memory area where every object you create with `new` is actually stored. It is shared by all threads in the program, and it's the only area the Garbage Collector cleans up.

   - Every `new` object goes here.
   - Shared by all threads.
   - Garbage Collector cleans this area.

**3) Stack**
   *Definition:* The Stack is a per-thread memory area that keeps track of method calls — for each method it stores its local variables and references. When a method is called a block (frame) is added on top, and when the method finishes that block is removed automatically.

   - Each method call = one "stack frame".
   - When method ends, its frame is removed automatically.
   - Stores primitives (int, char) and *references* to heap objects.

**4) Metaspace**
   *Definition:* Metaspace is the memory area that stores information *about* your classes themselves — their structure, method definitions, field names, etc. (not the objects, but the "blueprint" of each class). It lives outside the heap in native OS memory.

   - Before Java 8 this was called **PermGen** (fixed size, caused `OutOfMemoryError: PermGen`).
   - From Java 8 → **Metaspace**, lives in native memory and grows automatically.

**5) PC Register**
   *Definition:* The PC (Program Counter) Register is a small per-thread pointer that remembers which instruction the thread is currently running, so after switching between threads the JVM knows exactly where to continue.

**6) Execution Engine**
   *Definition:* The Execution Engine is the part that actually *executes* the loaded bytecode and turns it into real work on the CPU. It reads instructions, runs them, and manages memory cleanup.

   - **Interpreter** → reads bytecode line by line (slow but starts fast).
   - **JIT (Just-In-Time) compiler** → converts hot/frequently-used code into native machine code (fast).
   - **GC** → garbage collector.

### Quick memory picture
```
+-------------------- JVM Memory --------------------+
|  Heap (objects, shared)                            |
|  Metaspace (class info)                            |
|  Stack (per thread: methods, locals)              |
|  PC Register (per thread)                          |
|  Native Method Stack                               |
+---------------------------------------------------+
```

**Interview one-liner:** Heap = objects (shared), Stack = method calls & locals (per thread), Metaspace = class info, ClassLoader = loads classes with parent delegation.

---

## 2. JDK vs JRE vs JVM

*Definition:* These are three layers of Java tooling. **JVM** is the engine that runs Java, **JRE** is the JVM plus the libraries needed to *run* programs, and **JDK** is the JRE plus the tools needed to *build* programs (like the compiler).

Think of it like nested boxes: **JDK contains JRE contains JVM.**

| Term | Full form | What it gives you | Use it for |
|------|-----------|-------------------|------------|
| **JVM** | Java Virtual Machine | Just runs bytecode | The engine (inside JRE) |
| **JRE** | Java Runtime Environment | JVM + core libraries | Only **running** Java apps |
| **JDK** | Java Development Kit | JRE + dev tools (`javac`, debugger) | **Developing + running** Java |

**Simple way to remember:**
- Want to *run* a Java app → need **JRE**.
- Want to *write/compile* Java → need **JDK**.
- JVM is the actual runner inside both.

```
JDK = JRE + compiler(javac) + tools
JRE = JVM + libraries
JVM = the runtime engine
```

**Interview one-liner:** JDK is for developers (has compiler), JRE is for running apps, JVM is the engine that executes bytecode.

---

## 3. Garbage Collection (GC) — and which collector to use

**What is GC?**
Automatic memory cleanup. In Java you don't free memory manually (no `delete`). GC finds objects that are no longer used (no reference points to them) and removes them from the Heap.

**When is an object garbage?**
When **no reference** points to it.
```java
Student s = new Student();
s = null;   // the old Student object is now garbage
```

### How GC works (simple)
The heap is divided into areas called **generations**, based on the idea that *most objects die young* (used briefly then thrown away). Keeping new and old objects separate lets the GC clean the busy "young" area quickly and often.

- **Young Generation** → *Where brand-new objects are first created. Most objects live and die here quickly.* Split into Eden + 2 Survivor spaces.
- **Old (Tenured) Generation** → *Where long-living objects move after surviving several cleanups in the young area.*
- **Minor GC** → *A quick cleanup of only the Young generation. Happens frequently and is fast.*
- **Major/Full GC** → *A bigger cleanup that includes the Old generation. Slower and less frequent.*

**Stop-the-world** → *A short pause where the whole application freezes so the GC can safely clean memory. Smaller pauses = better user experience.*

### Types of Garbage Collectors — when to use what

Quick definitions first, then the comparison table:
- **Serial GC** → *Uses a single thread to clean memory. Simple, but freezes the app while running. Good only for small programs.*
- **Parallel GC** → *Uses many threads to clean faster. Aims for maximum total work done (throughput), but still pauses the app. Good for batch jobs.*
- **G1 GC** → *Splits the heap into many small regions and cleans the most-garbage-filled ones first, balancing speed and short pauses. The modern default.*
- **ZGC** → *A newer collector built for extremely short pauses (under 1ms) even on huge heaps. Great for apps that must stay responsive.*
- **Shenandoah** → *Another very-low-pause collector, similar goal to ZGC.*

| Collector | How it works | Best for |
|-----------|--------------|----------|
| **Serial GC** | Single thread, stops the world | Small apps, single-core, low memory |
| **Parallel GC** (Throughput) | Multiple threads for GC | When you want max throughput, pauses OK (batch jobs) |
| **G1 GC** (default since Java 9) | Splits heap into regions, balances pause + throughput | General purpose, large heaps, predictable pauses |
| **ZGC** | Ultra-low pause (<1ms), very large heaps | Huge heaps (GBs–TBs), latency-critical apps |
| **Shenandoah** | Low pause like ZGC | Same idea as ZGC, low-latency |

**Easy rule of thumb:**
- Small/simple app → **Serial**
- Batch processing, throughput matters → **Parallel**
- Default balanced choice → **G1**
- Need very low pause + huge heap → **ZGC**

**Interview one-liner:** GC auto-frees unreachable objects. G1 is the modern default; use ZGC when you need very low pause times on large heaps; Parallel for throughput; Serial for tiny apps.

---

## 4. Memory Model (Stack vs Heap, Object Lifecycle)

*Definition:* The **memory model** is simply how Java organizes and uses RAM while your program runs. The two main areas you must know are the **Stack** (fast, temporary, per-thread workspace for method calls) and the **Heap** (large, shared storage where all objects live).

### Stack vs Heap

| Stack | Heap |
|-------|------|
| Stores local variables + references + method calls | Stores objects (everything created with `new`) |
| One per thread | Shared by all threads |
| Fast access | Slower |
| Auto-cleared when method ends | Cleared by Garbage Collector |
| Small in size | Large |
| `StackOverflowError` if too deep | `OutOfMemoryError` if full |

### Example
```java
void demo() {
    int x = 10;                 // x (primitive) -> Stack
    Student s = new Student();  // 's' reference -> Stack
                                // actual Student object -> Heap
}
```
Here `x` and the reference `s` live on the Stack. The real `Student` object lives on the Heap. The reference just "points" to it.

### Object Lifecycle (birth to death)
1. **Created** → `new Student()` allocates on heap.
2. **In use** → at least one reference points to it.
3. **Unreachable** → all references gone (`s = null` or method ends).
4. **Garbage collected** → GC reclaims the memory.
5. (Optional) `finalize()` *used to* run before removal — now deprecated, don't rely on it.

**Interview one-liner:** Primitives & references live on the stack; objects live on the heap. Stack is per-thread and auto-cleaned; heap is shared and GC-cleaned.

---

## 5. String Pool & Interning

**Strings are special in Java** — they are **immutable** (cannot be changed once created) and Java keeps a special area called the **String Pool** (inside the heap) to save memory.

### String Pool
When you write a string **literal**, Java checks the pool:
- If same string already exists → reuse it (same reference).
- If not → create it in the pool.

```java
String a = "hello";   // goes into pool
String b = "hello";   // reuses same pooled object
System.out.println(a == b);  // true  (same reference)
```

But `new` always creates a **new object on the heap** (not pooled):
```java
String c = new String("hello");  // new heap object
System.out.println(a == c);      // false (different reference)
System.out.println(a.equals(c)); // true  (same content)
```

> Remember: `==` compares **references** (addresses). `.equals()` compares **content**.

### Interning
`intern()` forces a string into the pool (or returns the pooled one if it exists).
```java
String c = new String("hello");
String d = c.intern();          // gets the pooled "hello"
System.out.println(a == d);     // true
```

**Why immutable?** Safe to share in the pool, thread-safe, good as HashMap keys, secure.

**Interview one-liner:** Literal strings are pooled and reused; `new String()` always makes a fresh object. `==` checks reference, `.equals()` checks content. `intern()` pushes a string into the pool.

---

## 6. Autoboxing & Unboxing (Integer Cache Trap)

**Autoboxing** = primitive → wrapper object automatically (`int` → `Integer`).
**Unboxing** = wrapper → primitive automatically (`Integer` → `int`).

```java
Integer x = 5;     // autoboxing (int 5 -> Integer)
int y = x;         // unboxing (Integer -> int)
```

### The Integer Cache Trap (very common interview gotcha!)
Java **caches Integer objects from -128 to 127**. Same cached object is reused in that range.

```java
Integer a = 100, b = 100;
System.out.println(a == b);   // true  (cached, same object)

Integer c = 200, d = 200;
System.out.println(c == d);   // false (outside cache, new objects)

System.out.println(c.equals(d)); // true (content compare)
```

**Lesson:** Never use `==` to compare wrapper objects. Always use `.equals()`.

**Also watch out:** unboxing a `null` wrapper throws `NullPointerException`:
```java
Integer n = null;
int z = n;   // NullPointerException!
```

**Interview one-liner:** Integers -128 to 127 are cached, so `==` may accidentally work in that range but fails outside it. Always compare wrappers with `.equals()`.

---

## 7. Generics (Wildcards, Bounded Types, Type Erasure)

*Definition:* **Generics** let you write a class or method that works with *any* type, while still telling the compiler exactly which type you're using (like `List<String>` = a list that holds only Strings). This gives you type safety and removes the need for manual casting.

**Why generics?** Type safety + no casting. Catch type errors at compile time.

```java
List<String> list = new ArrayList<>();
list.add("hi");
String s = list.get(0);   // no cast needed, safe
// list.add(5);           // compile error -> good!
```

### Generic class / method
```java
class Box<T> {            // T = type parameter
    T value;
    void set(T v){ value = v; }
    T get(){ return value; }
}
Box<Integer> b = new Box<>();
```

### Bounded types (restrict T)
```java
// T must be Number or its subclass
class Calc<T extends Number> {
    double square(T n){ return n.doubleValue() * n.doubleValue(); }
}
```

### Wildcards (`?`)
- `<?>` → unknown type (read-only-ish).
- `<? extends T>` → T or any **subclass** (use when you **read/produce** values).
- `<? super T>` → T or any **superclass** (use when you **write/consume** values).

```java
void printAll(List<? extends Number> list) {  // accepts List<Integer>, List<Double>...
    for (Number n : list) System.out.println(n);
}
```
Helpful memory: **PECS** → "Producer Extends, Consumer Super".

### Type Erasure (important!)
Generics exist **only at compile time**. At runtime, the type info is **erased** — `List<String>` and `List<Integer>` both become just `List`.
```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass()); // true! (both ArrayList)
```
Consequences: can't do `new T()`, can't use `instanceof List<String>`, no generic arrays.

**Interview one-liner:** Generics give compile-time type safety; bounded types restrict with `extends`; wildcards use PECS; and type erasure removes generic info at runtime (both lists are just `List`).

---

## 8. Collections Framework (List, Set, Map, Queue — internals)

*Definition:* The **Collections Framework** is Java's ready-made set of classes for storing and managing groups of objects (lists, sets, maps, queues) — so you don't have to build data structures from scratch.

Big picture:
```
Collection (interface)
 ├── List   → ordered, allows duplicates
 ├── Set    → no duplicates
 └── Queue  → FIFO ordering
Map (separate) → key-value pairs
```

Quick definitions of the four main types:
- **List** → *An ordered collection that keeps items in the sequence you add them and allows duplicates. You access items by index (position).*
- **Set** → *A collection that stores only unique items — no duplicates allowed.*
- **Queue** → *A collection that processes items in order, usually FIFO (First-In-First-Out) — like a line at a ticket counter.*
- **Map** → *Stores data as key-value pairs (like a dictionary). Each key is unique and maps to one value.*

### List (ordered, duplicates allowed, index-based)
| Class | Internals | Notes |
|-------|-----------|-------|
| **ArrayList** | Dynamic array | Fast get by index O(1); slow insert/delete in middle O(n) |
| **LinkedList** | Doubly linked list | Fast insert/delete at ends O(1); slow random access O(n) |
| **Vector** | Like ArrayList but synchronized | Legacy, thread-safe but slow |

### Set (no duplicates)
| Class | Internals | Order |
|-------|-----------|-------|
| **HashSet** | Backed by HashMap | No order |
| **LinkedHashSet** | HashMap + linked list | Insertion order |
| **TreeSet** | Red-Black tree | Sorted order |

### Queue (FIFO)
| Class | Notes |
|-------|-------|
| **LinkedList** | Can act as queue |
| **PriorityQueue** | Elements come out by priority (heap) |
| **ArrayDeque** | Double-ended queue, fast |

### Map (key → value, keys unique)
| Class | Internals | Order |
|-------|-----------|-------|
| **HashMap** | Array of buckets + linked list/tree | No order |
| **LinkedHashMap** | HashMap + linked list | Insertion order |
| **TreeMap** | Red-Black tree | Sorted by key |
| **Hashtable** | Synchronized HashMap | Legacy |

**Interview one-liner:** List = ordered + duplicates, Set = unique, Queue = FIFO, Map = key-value. ArrayList for random access, LinkedList for frequent insert/delete, TreeMap/TreeSet for sorted data.

---

## 9. HashMap Internals (hashing, collision, resize, load factor)

*Definition:* A **HashMap** stores key-value pairs and uses a trick called **hashing** to find data almost instantly (O(1) on average) instead of searching one by one.

This is one of the **most asked** interview topics. First learn the key words:
- **Hashing** → *Converting a key into a number (a hash) so the map can jump straight to where the value is stored.*
- **Bucket** → *A slot in the internal array where entries are kept. The hash decides which bucket a key goes into.*
- **Collision** → *When two different keys end up in the same bucket. The map then stores them together in that bucket.*
- **Load factor** → *How full the map is allowed to get (default 0.75 = 75%) before it grows itself.*
- **Resize (rehash)** → *When the map gets too full, it makes a bigger array and re-places all entries into new buckets.*

### How HashMap stores data
- Internally it's an **array of "buckets"** (default size **16**).
- Each entry = key + value + hash + next pointer (a **Node**).

### Step by step `put(key, value)`
1. Compute `hashCode()` of key.
2. Spread the hash (extra mixing) and find bucket index: `index = hash & (n-1)`.
3. If bucket empty → place node there.
4. If bucket already has nodes → **collision**:
   - Compare keys with `equals()`. If same key → update value.
   - Else add to the chain (linked list) in that bucket.

### Collision handling
When two keys land in the same bucket = **collision**.
- Java stores them as a **linked list** in that bucket.
- **Java 8+ optimization:** if a bucket's list gets too long (**≥ 8 nodes** and table size ≥ 64), it converts the list into a **balanced tree (Red-Black tree)** → lookup becomes O(log n) instead of O(n). Shrinks back to list if it drops to ≤ 6.

### Load Factor & Resize
- **Load factor** = how full the map can get before growing. Default **0.75**.
- **Threshold** = capacity × load factor = 16 × 0.75 = **12**.
- When size crosses threshold → **resize**: capacity doubles (16 → 32) and all entries are **rehashed** into new buckets.
- 0.75 is a balance: lower = more memory but fewer collisions; higher = less memory but more collisions.

### Why override both `hashCode()` and `equals()`?
HashMap uses `hashCode()` to find the bucket and `equals()` to find the exact key.
If you use a custom object as a key and don't override them properly, lookups will fail.

```java
Map<String,Integer> m = new HashMap<>();
m.put("a", 1);
m.put("a", 2);   // same key -> updates, not duplicate
System.out.println(m.get("a")); // 2
```

**Interview one-liner:** HashMap = array of buckets, default 16, load factor 0.75. Uses hashCode to pick bucket, equals to match key. Collisions chain as linked list, turning into a Red-Black tree after 8 nodes. Resizes (doubles + rehashes) at 75% full.

---

## 10. ConcurrentHashMap vs HashMap vs Hashtable

*Definition:* All three store key-value pairs. The real difference is **thread safety** — whether they work correctly when *many threads (multiple tasks at once)* use them at the same time.

> **Thread-safe** → *Safe to use from multiple threads at once without data getting corrupted.* The downside is usually some speed cost from "locking" (making threads wait their turn).

All store key-value pairs. Difference = **thread safety & performance**.

| Feature | HashMap | Hashtable | ConcurrentHashMap |
|---------|---------|-----------|-------------------|
| Thread-safe? | ❌ No | ✅ Yes | ✅ Yes |
| How it locks | — | Locks **whole map** (one big lock) | Locks **small parts** (buckets/segments) |
| Performance (multi-thread) | Unsafe | Slow | Fast |
| Null key/values | 1 null key, many null values | No nulls | No nulls |
| Introduced | Java 1.2 | Java 1.0 (legacy) | Java 1.5 |

**Key idea:**
- **HashMap** → single-threaded, fastest, NOT safe for concurrent use.
- **Hashtable** → old, thread-safe but locks the entire map so it's slow.
- **ConcurrentHashMap** → modern thread-safe map; only locks the small portion being written, so multiple threads can work at the same time (much faster).

**When to use:** Single thread → HashMap. Multi-thread → ConcurrentHashMap. Never use Hashtable in new code.

**Interview one-liner:** HashMap is not thread-safe; Hashtable locks the whole map (slow); ConcurrentHashMap locks only small parts so many threads run concurrently — use it for multithreaded code.

---

## 11. Comparable vs Comparator

*Definition:* Both are tools that tell Java **how to sort your custom objects** (Java can't guess whether to sort Students by name or marks, so you define the rule). The difference is *where* you write that rule.

Both are used for **sorting** objects. Difference = where the sorting logic lives.

### Comparable — "natural ordering" (one way)
- Implemented **inside** the class itself.
- Method: `compareTo()`.
- Use when there's **one obvious default** sort order.

```java
class Student implements Comparable<Student> {
    int marks;
    public int compareTo(Student o) {
        return this.marks - o.marks;   // sort by marks ascending
    }
}
Collections.sort(studentList);   // uses compareTo
```

### Comparator — "custom ordering" (many ways)
- Written **outside** the class (separate logic).
- Method: `compare()`.
- Use when you want **multiple/different** sort orders.

```java
Comparator<Student> byName = (a, b) -> a.name.compareTo(b.name);
Collections.sort(studentList, byName);

// modern style
studentList.sort(Comparator.comparingInt(s -> s.marks));
studentList.sort(Comparator.comparing((Student s) -> s.name).reversed());
```

| | Comparable | Comparator |
|--|------------|------------|
| Where | Inside the class | Outside (separate) |
| Method | `compareTo(o)` | `compare(a, b)` |
| Sort orders | One (natural) | Many (custom) |
| Modifies class? | Yes | No |

**Memory trick:** Compara**ble** = **B**uilt into the class. Compara**tor** = separate sor**TOR** logic.

**Interview one-liner:** Comparable defines one natural order inside the class (`compareTo`); Comparator defines custom orders outside the class (`compare`), and you can have many.

---

## 12. Immutability & `final` keyword

### Immutable object
An object whose **state can't change** after it's created. Example: `String`.
Benefits: thread-safe, safe to share, great as map keys, no surprise changes.

### How to make a class immutable
1. Mark class `final` (can't be subclassed).
2. Make all fields `private final`.
3. No setters.
4. Set values only via constructor.
5. For mutable fields (like List), return a **copy**, not the original.

```java
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int getX() { return x; }
    public int getY() { return y; }
    // no setters -> can't change after creation
}
```

### The `final` keyword (3 uses)
*Definition:* `final` means **"this cannot be changed."** What exactly can't change depends on where you put it — on a variable, a method, or a class.

| Used on | Meaning |
|---------|---------|
| **variable** | Value can't be reassigned (constant) |
| **method** | Can't be overridden by subclass |
| **class** | Can't be extended (no subclass) |

```java
final int MAX = 100;     // can't reassign
final class Util {}      // can't extend
class A { final void f(){} }  // f can't be overridden
```

> Important: `final` on an **object reference** means the reference can't point elsewhere, but the object's **internals can still change**.
```java
final List<Integer> list = new ArrayList<>();
list.add(5);             // OK! contents can change
// list = new ArrayList<>(); // ERROR! reference can't change
```

**Interview one-liner:** Immutable = can't change after creation (like String); make fields private final + no setters + final class. `final` prevents reassignment (variable), overriding (method), or extension (class) — but a final reference's object can still mutate internally.

---

## 13. `var` — Local Variable Type Inference (Java 10+)

`var` lets the compiler **figure out the type** for you. Less typing, but it's still strongly typed (NOT like JavaScript `var`).

```java
var name = "John";          // inferred as String
var count = 10;             // inferred as int
var list = new ArrayList<String>();  // ArrayList<String>
```

### Rules / limits
- Only for **local variables** (inside methods).
- **Must initialize** at declaration: `var x;` ❌ (compiler can't infer).
- Can't use for fields, method parameters, or return types.
- Can't be `null` alone: `var x = null;` ❌.
- Type is fixed once inferred — `var x = 5; x = "hi";` ❌.

**When to use:** When the type is obvious from the right side and reduces clutter. Avoid when it hurts readability.

**Interview one-liner:** `var` infers the type at compile time for local variables only; it's still statically typed, just less verbose. Must be initialized; can't be used for fields/params/returns.

---

## 14. Records (Java 14+)

A **Record** is a short way to create an **immutable data-carrying class**. Java auto-generates the boilerplate.

### Before (old way — lots of code)
```java
class Point {
    private final int x, y;
    // constructor, getters, equals(), hashCode(), toString() ... long!
}
```

### With Record (one line!)
```java
public record Point(int x, int y) {}
```
This auto-generates:
- private final fields `x`, `y`
- constructor `Point(int x, int y)`
- accessor methods `x()` and `y()` (note: not `getX()`)
- `equals()`, `hashCode()`, `toString()`

```java
Point p = new Point(3, 4);
System.out.println(p.x());     // 3
System.out.println(p);         // Point[x=3, y=4]
```

### Key points
- Records are **immutable** (all fields final).
- Can't extend other classes (records are implicitly final).
- Can add methods and custom/compact constructors with validation:
```java
public record Point(int x, int y) {
    public Point {                       // compact constructor
        if (x < 0) throw new IllegalArgumentException("x<0");
    }
}
```

**Use for:** DTOs, simple data holders, return-multiple-values.

**Interview one-liner:** Records are concise immutable data classes (Java 14+) that auto-generate constructor, accessors, equals, hashCode, and toString. Great for DTOs.

---

## 15. Sealed Classes (Java 17+)

A **sealed class** controls **exactly which classes can extend it**. You limit inheritance.

Normally any class can extend a non-final class. Sealed says: "Only THESE specific classes may extend me."

```java
public sealed class Shape permits Circle, Square, Triangle {}

public final class Circle extends Shape {}
public final class Square extends Shape {}
public final class Triangle extends Shape {}
```

A subclass of a sealed class must be one of:
- `final` → no further extension, OR
- `sealed` → continues controlled extension, OR
- `non-sealed` → opens it back up for anyone.

### Why use it?
- You control the full set of subtypes (good for modeling fixed options).
- Works great with **pattern matching switch** — compiler knows all cases:
```java
String describe(Shape s) {
    return switch (s) {
        case Circle c   -> "circle";
        case Square sq  -> "square";
        case Triangle t -> "triangle";
        // no default needed — compiler knows all permitted types
    };
}
```

**Interview one-liner:** Sealed classes (Java 17+) restrict which classes can extend them via `permits`. Subclasses must be final, sealed, or non-sealed. Pairs perfectly with switch pattern matching.

---

## 16. `instanceof` Pattern Matching (Java 16+)

Old way — check type, then cast manually:
```java
if (obj instanceof String) {
    String s = (String) obj;   // explicit cast needed
    System.out.println(s.length());
}
```

New way — check + cast in **one step** (binds a variable):
```java
if (obj instanceof String s) {     // s is auto-cast String
    System.out.println(s.length());
}
```
`s` is only available where the check is true. Less code, no manual cast, fewer bugs.

You can even use it in conditions:
```java
if (obj instanceof String s && s.length() > 5) {
    System.out.println("long string: " + s);
}
```

### Combined with switch (modern, Java 21):
```java
String result = switch (obj) {
    case Integer i -> "int: " + i;
    case String s  -> "string: " + s;
    default        -> "unknown";
};
```

**Interview one-liner:** `instanceof` pattern matching checks the type and casts into a new variable in one step (`obj instanceof String s`), removing the explicit cast and reducing boilerplate.

---

## Quick Revision Cheat Sheet

- **JVM** runs bytecode; Heap=objects, Stack=method/locals, Metaspace=class info.
- **JDK** > **JRE** > **JVM** (development > running > engine).
- **GC**: G1 default, ZGC low-pause, Parallel throughput, Serial tiny apps.
- **String pool**: literals reused; `new String` fresh; `==` ref, `.equals()` content.
- **Integer cache** -128..127 → use `.equals()` for wrappers.
- **Generics**: compile-time safety, PECS for wildcards, type erasure at runtime.
- **HashMap**: 16 buckets, load 0.75, collisions chain→tree(8), resize doubles+rehash.
- **ConcurrentHashMap** for multithreading (locks parts, not whole map).
- **Comparable** = natural order inside class; **Comparator** = custom orders outside.
- **final**: no reassign/override/extend; immutable = no change after creation.
- **var** = local type inference; **record** = auto data class; **sealed** = controlled inheritance; **instanceof pattern** = check + cast in one.
