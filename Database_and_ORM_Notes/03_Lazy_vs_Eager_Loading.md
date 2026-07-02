# Database Notes — Lazy vs Eager Loading (N+1 problem & fix)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is "loading" about?

When you fetch an entity that has related entities (a `User` who has many `Orders`), a question arises: **should Hibernate load the related data right away, or wait until you actually use it?**

- **Eager** = load everything immediately.
- **Lazy** = load the related data only when you first touch it.

Real life:
- **Eager** = when you order a burger, they *also* bring fries, drink, dessert — whether you want them or not.
- **Lazy** = they bring the burger now, and only fetch the fries *if and when* you ask.

---

## 2. Lazy Loading

*Definition:* **Lazy loading** means a related entity/collection is **not** loaded from the DB until you actually access it in code. Hibernate puts a lightweight **proxy** placeholder there, and runs the query only on first use.

```java
User user = em.find(User.class, 1L);   // only the User row loaded
// orders NOT loaded yet — it's a proxy

List<Order> orders = user.getOrders(); // ← NOW Hibernate runs the orders query
```

👍 Saves memory & time if you don't need the related data.
👎 Can trigger extra queries later, or fail if the session is closed (see gotcha).

**Interview one-liner:** Lazy loading delays fetching related data until you access it — efficient, but risks extra queries or `LazyInitializationException`.

---

## 3. Eager Loading

*Definition:* **Eager loading** means the related entity/collection is loaded **immediately**, together with the main entity, in the same fetch.

```java
User user = em.find(User.class, 1L);   // User AND orders loaded right away
List<Order> orders = user.getOrders(); // already in memory, no new query
```

👍 Related data is ready instantly; no surprise later queries.
👎 Wastes resources if you didn't need it; can load huge object graphs.

**Interview one-liner:** Eager loading fetches related data upfront with the parent — convenient but can over-fetch and hurt performance.

---

## 4. The defaults (memorize this — common interview question!)

Hibernate/JPA has **different defaults** depending on the relationship type:

| Relationship | Default | Why |
|--------------|---------|-----|
| `@OneToMany` | **LAZY** | Could be many rows — don't load unless needed |
| `@ManyToMany` | **LAZY** | Same reason |
| `@ManyToOne` | **EAGER** | Usually just one parent row — cheap |
| `@OneToOne` | **EAGER** | Single related row |

Memory trick: **"to-Many = Lazy, to-One = Eager"** (the "many" sides are lazy by default).

```java
@OneToMany(fetch = FetchType.LAZY)    // default for collections
private List<Order> orders;

@ManyToOne(fetch = FetchType.EAGER)   // default for single refs
private User user;
```

> **Best practice:** prefer **LAZY everywhere** (even override the to-one defaults to lazy) and fetch what you need explicitly per query. Eager-by-default is a common source of performance bugs.

---

## 5. LazyInitializationException (the classic trap)

*Definition:* This error happens when you try to access a **lazy** field **after** the persistence context (session) has already closed — so Hibernate can't run the query anymore.

```java
@Transactional
User load() {
    return em.find(User.class, 1L);   // session closes when method returns
}

// later, OUTSIDE the transaction:
user.getOrders();   // ❌ LazyInitializationException — session is gone
```

### Fixes
- Access the lazy data **inside** the transaction (while the session is open).
- Use a **fetch join** (`JOIN FETCH`) to load it in the original query.
- Use an **`@EntityGraph`**.
- (Anti-pattern) `spring.jpa.open-in-view=true` keeps the session open in the web layer — convenient but hides problems; many teams disable it.

**Interview one-liner:** `LazyInitializationException` occurs when you access a lazy field after the session closed; fix by fetching within the transaction or using `JOIN FETCH`/`@EntityGraph`.

---

## 6. The N+1 Problem (the most important DB interview topic here)

*Definition:* The **N+1 problem** is when fetching a list of N entities triggers **1 query for the list + N extra queries** (one per entity) to load each one's related data. Result: N+1 total queries instead of 1 or 2 — terrible performance.

### How it sneaks in
```java
List<User> users = em.createQuery("SELECT u FROM User u", User.class).getResultList();
// → 1 query: SELECT * FROM users   (gets 100 users)

for (User u : users) {
    System.out.println(u.getOrders().size());  // lazy → 1 query PER user
}
// → 100 more queries (one per user)!
// TOTAL = 1 + 100 = 101 queries  😱
```

```
Without fix:  1 (users) + N (orders per user)  = N+1 queries
With fix:     1 query (join)                    = 1 query
```

### Fixes

**1. JOIN FETCH (most common):**
```java
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();   // ONE query with a join
```

**2. @EntityGraph (Spring Data):**
```java
@EntityGraph(attributePaths = "orders")
@Query("SELECT u FROM User u")
List<User> findAllWithOrders();
```

**3. Batch fetching** (`@BatchSize` / `hibernate.default_batch_fetch_size`): loads the N in a few `IN (...)` queries instead of N separate ones.

```java
@OneToMany
@BatchSize(size = 50)            // loads orders for 50 users per query
private List<Order> orders;
```

> **Why interviewers love N+1:** it's invisible in code (looks like one loop) but explodes into hundreds of queries in production. Knowing to spot it and fix with `JOIN FETCH` shows real experience.

**Interview one-liner:** N+1 = 1 query for the parents plus N queries for each parent's children. Fix it with `JOIN FETCH`, `@EntityGraph`, or batch fetching to collapse it into one/few queries.

---

## 7. Quick decision guide

| Situation | Choose |
|-----------|--------|
| Default for collections | **Lazy** (and keep it) |
| You almost always need the related data | Fetch eagerly **per query** (`JOIN FETCH`), not via EAGER annotation |
| Avoiding N+1 when looping over a list | `JOIN FETCH` / `@EntityGraph` |
| Loading many lazy collections efficiently | `@BatchSize` |

**Easy rule of thumb:** Map everything **LAZY**, then **fetch what you need explicitly** with `JOIN FETCH`/`@EntityGraph` per use case. Don't rely on EAGER.

---

## Quick Revision Cheat Sheet

- **Lazy** = load related data only when accessed (proxy until then).
- **Eager** = load related data immediately with the parent.
- **Defaults**: `@OneToMany`/`@ManyToMany` = LAZY; `@ManyToOne`/`@OneToOne` = EAGER ("to-Many=Lazy, to-One=Eager").
- **Best practice**: make everything LAZY, fetch explicitly per query.
- **LazyInitializationException**: accessing lazy data after session closed → fetch in-transaction or `JOIN FETCH`.
- **N+1 problem**: list query (1) + one query per item (N) = N+1. Fix with **JOIN FETCH**, **@EntityGraph**, or **@BatchSize**.
