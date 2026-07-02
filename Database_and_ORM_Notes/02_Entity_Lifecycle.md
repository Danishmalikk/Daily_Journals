# Database Notes — Entity Lifecycle (Transient, Persistent, Detached, Removed)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is the Entity Lifecycle?

*Definition:* In JPA/Hibernate, an **entity** is a Java object mapped to a DB table row. The **entity lifecycle** is the set of **states** that object can be in — from a plain new object, to being tracked & saved by Hibernate, to being disconnected or deleted.

The key player that tracks entities is the **Persistence Context** (managed by the `EntityManager`).

*Definition:* The **Persistence Context** is like a "smart notepad" Hibernate keeps during a transaction. Any entity inside it is **managed** — Hibernate watches it for changes and syncs them to the DB automatically.

Real life: A shopping cart system —
- item you haven't added yet = **Transient**
- item in your live cart being tracked = **Persistent**
- cart from an old saved session, no longer live = **Detached**
- item you marked "remove" = **Removed**

---

## 2. The 4 states at a glance

```
   new User()           persist()/save()        commit
   ┌─────────┐  ───────►  ┌──────────┐  ───────►  saved to DB
   │TRANSIENT│            │PERSISTENT│
   └─────────┘            └──────────┘
                          │   ▲    │ remove()
            close()/detach│   │find │
                          ▼   │load ▼
                      ┌────────┐  ┌────────┐
                      │DETACHED│  │REMOVED │ ──► deleted at commit
                      └────────┘  └────────┘
```

| State | Tracked by Hibernate? | In DB? | Meaning |
|-------|----------------------|--------|---------|
| **Transient** | ❌ No | ❌ No | Brand-new object, Hibernate doesn't know it |
| **Persistent** | ✅ Yes | ✅ Yes (synced) | Managed; changes auto-saved |
| **Detached** | ❌ No | ✅ Yes | Was persistent, now disconnected from context |
| **Removed** | ✅ Yes | ⏳ Will be deleted | Marked for deletion at commit |

---

## 3. Transient — the new object

*Definition:* A **Transient** entity is a freshly created object (`new`) that Hibernate is **not** aware of. It has no link to the database and no id yet. If your app ends now, it's lost.

```java
User u = new User("Danish");   // TRANSIENT — Hibernate knows nothing about it
// not in persistence context, not in DB
```

**Interview one-liner:** Transient = a `new` object not yet associated with Hibernate or the database.

---

## 4. Persistent — the managed object

*Definition:* A **Persistent** entity is one that is **attached to the persistence context** and tracked by Hibernate. Any change you make to it is **automatically detected** (dirty checking) and saved to the DB when the transaction commits — no explicit `save()` needed.

```java
User u = new User("Danish");
em.persist(u);            // TRANSIENT → PERSISTENT (now managed)

u.setName("Danish Malik");// just change the field...
// no save() call needed! Hibernate auto-updates at commit (dirty checking)
```

Or load an existing one (already persistent):
```java
User u = em.find(User.class, 5L);  // loaded → PERSISTENT
u.setEmail("new@app.com");          // auto-synced at commit
```

> **Dirty checking** is the magic here: Hibernate remembers the original values, compares at commit, and issues UPDATEs only for what changed. A favorite interview point.

**Interview one-liner:** Persistent = managed by the persistence context; changes are auto-saved at commit via dirty checking, without calling `save()`.

---

## 5. Detached — disconnected from Hibernate

*Definition:* A **Detached** entity *was* persistent, but its persistence context closed (transaction ended) or it was explicitly detached. It still exists in the DB and holds its id, but Hibernate **no longer tracks** it — changes won't auto-save.

```java
User u = em.find(User.class, 5L);  // PERSISTENT
em.close();                         // context closed → u is now DETACHED

u.setName("Changed");              // ❌ NOT saved — nobody is tracking it
```

To reconnect a detached entity, use **`merge()`**:
```java
User managed = em.merge(u);   // DETACHED → returns a PERSISTENT copy
```

> Gotcha: `merge()` doesn't make `u` itself managed — it **returns a new managed instance**. Keep using the returned object, not the old one.

**Interview one-liner:** Detached = was persistent but the context closed; changes aren't tracked. Use `merge()` to reattach (it returns a managed copy).

---

## 6. Removed — marked for deletion

*Definition:* A **Removed** entity is one you've marked for deletion with `remove()`. It's still in the persistence context (and still in the DB) until the transaction commits — then the actual `DELETE` runs.

```java
User u = em.find(User.class, 5L);  // PERSISTENT
em.remove(u);                      // PERSISTENT → REMOVED
// row still in DB until commit, then DELETE executes
```

**Interview one-liner:** Removed = scheduled for deletion via `remove()`; the DELETE actually runs at commit.

---

## 7. The methods that move between states

| Method | Moves entity… | 
|--------|---------------|
| `persist()` / `save()` | Transient → Persistent |
| `find()` / `get()` / query | (loads as) Persistent |
| `remove()` / `delete()` | Persistent → Removed |
| `detach()` / `close()` / `clear()` | Persistent → Detached |
| `merge()` | Detached → Persistent (returns managed copy) |
| `flush()` | Pushes pending changes to DB (state unchanged) |

> **`flush()` vs `commit()`:** `flush()` sends the SQL to the DB *now* but inside the transaction (can still roll back). `commit()` makes it permanent. Hibernate auto-flushes before queries and at commit.

---

## 8. Why this matters (real interview value)

- Explains **why `u.setName()` saves without a `save()` call** (it's persistent + dirty checking).
- Explains the classic bug: *"I changed a detached object and it didn't save"* → it wasn't managed; use `merge()`.
- Connects to **LazyInitializationException**: accessing a lazy field on a **detached** entity (after the context closed) throws this. (See Lazy vs Eager file.)

**Interview one-liner:** The four states (Transient, Persistent, Detached, Removed) explain when Hibernate tracks and auto-saves your objects — and why detached changes silently don't persist.

---

## Quick Revision Cheat Sheet

- **Persistence Context** = Hibernate's tracking "notepad"; entities inside it are **managed**.
- **Transient**: new object, not tracked, not in DB.
- **Persistent**: managed → changes auto-saved at commit via **dirty checking** (no `save()` needed).
- **Detached**: was persistent, context closed → not tracked; reattach with **`merge()`** (returns managed copy).
- **Removed**: marked by `remove()`; DELETE runs at commit.
- Methods: `persist`→persistent, `remove`→removed, `detach/close`→detached, `merge`→persistent.
- **flush()** = push SQL now (still in txn); **commit()** = make permanent.
- Touching a lazy field on a detached entity → **LazyInitializationException**.
