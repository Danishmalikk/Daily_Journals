# Database Notes — Transactions (@Transactional: propagation, isolation)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Transaction?

*Definition:* A **transaction** is a group of database operations treated as **one single unit** — either **all** of them succeed together, or **none** of them apply (everything rolls back). There's no halfway.

Real life: A bank transfer — debit ₹100 from A *and* credit ₹100 to B. If the credit fails, the debit must be undone too. You can't lose money halfway. That all-or-nothing bundle is a transaction.

```
BEGIN
  debit A  -100
  credit B +100
COMMIT   ← both saved
   -- or --
ROLLBACK ← if anything fails, both undone
```

**Interview one-liner:** A transaction is an all-or-nothing unit of DB work — it commits fully or rolls back entirely, never partially.

---

## 2. @Transactional — the easy way in Spring

*Definition:* **`@Transactional`** is a Spring annotation that wraps a method in a transaction automatically. Spring opens a transaction before the method, commits if it finishes normally, and rolls back if it throws an exception.

```java
@Service
class BankService {

    @Transactional
    public void transfer(Long from, Long to, double amount) {
        accountRepo.debit(from, amount);
        accountRepo.credit(to, amount);
        // if either line throws → BOTH are rolled back automatically
    }
}
```

> Spring implements `@Transactional` using **AOP proxies** (see the AOP file). That leads to two classic gotchas below.

### Gotcha 1 — rollback only on unchecked exceptions by default
```java
// Rolls back: RuntimeException & Error (unchecked)
// Does NOT roll back: checked exceptions, by default!
@Transactional(rollbackFor = Exception.class)   // force rollback on checked too
```

### Gotcha 2 — self-invocation doesn't work
Calling a `@Transactional` method from **another method in the same class** bypasses the proxy → no transaction. (Same proxy limitation as AOP.)

**Interview one-liner:** `@Transactional` auto-wraps a method in a transaction (commit on success, rollback on exception). By default it only rolls back on unchecked exceptions; use `rollbackFor` for checked ones.

---

## 3. Propagation — what happens when transactions meet

*Definition:* **Propagation** decides how a transactional method behaves when it's called **from within another transaction** — should it join the existing one, start a new one, or run without any?

Real life: You're already in a meeting (an active transaction) and someone starts a sub-topic. Do they continue *in the same meeting* (join), or step out for a *separate meeting* (new), or just chat *without any meeting* (none)?

| Propagation | If a transaction already exists… | If none exists… |
|-------------|----------------------------------|-----------------|
| **REQUIRED** (default) | Join it | Create a new one |
| **REQUIRES_NEW** | Suspend it, start a fresh independent one | Create a new one |
| **NESTED** | Run in a nested savepoint (can roll back just this part) | Create a new one |
| **SUPPORTS** | Join it | Run with no transaction |
| **MANDATORY** | Join it | ❌ Throw error (must already exist) |
| **NEVER** | ❌ Throw error | Run with no transaction |
| **NOT_SUPPORTED** | Suspend it, run non-transactionally | Run with no transaction |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAudit(String msg) {
    // runs in its OWN transaction — commits even if the caller rolls back
}
```

> **REQUIRED vs REQUIRES_NEW** is the key interview pair: REQUIRED reuses the caller's transaction (rollback affects everything); REQUIRES_NEW is independent (e.g. audit logs that must persist even if the main work fails).

**Interview one-liner:** Propagation controls how a method joins or starts transactions. REQUIRED (default) joins or creates one; REQUIRES_NEW always runs in its own independent transaction.

---

## 4. Isolation Levels — how transactions affect each other

*Definition:* **Isolation level** controls **how much one running transaction can see of another's uncommitted/in-progress changes**. Higher isolation = more correctness but less concurrency (slower).

### First, the 3 read problems isolation prevents:

| Problem | What happens |
|---------|--------------|
| **Dirty read** | You read data another transaction changed but **hasn't committed** (might be undone) |
| **Non-repeatable read** | You read a row twice and get **different values** (someone updated it between your reads) |
| **Phantom read** | You run the same query twice and get **different rows** (someone inserted/deleted rows) |

### The 4 isolation levels (low → high)

| Level | Dirty read | Non-repeatable | Phantom | Notes |
|-------|------------|----------------|---------|-------|
| **READ_UNCOMMITTED** | ✅ possible | ✅ possible | ✅ possible | Fastest, least safe |
| **READ_COMMITTED** | ❌ prevented | ✅ possible | ✅ possible | Common default (Postgres, Oracle) |
| **REPEATABLE_READ** | ❌ | ❌ prevented | ✅ possible | MySQL InnoDB default |
| **SERIALIZABLE** | ❌ | ❌ | ❌ prevented | Safest, slowest (acts like one-at-a-time) |

```
Safety:       UNCOMMITTED < COMMITTED < REPEATABLE_READ < SERIALIZABLE
Performance:  UNCOMMITTED > COMMITTED > REPEATABLE_READ > SERIALIZABLE
```

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void process() { ... }
```

> Memory trick — each level fixes the next problem: COMMITTED kills dirty reads, REPEATABLE_READ kills non-repeatable reads, SERIALIZABLE kills phantoms.

**Interview one-liner:** Isolation levels trade safety vs speed: READ_UNCOMMITTED (dirty reads) → READ_COMMITTED → REPEATABLE_READ → SERIALIZABLE (no anomalies but slowest). Each higher level prevents one more read problem.

---

## 5. Other useful @Transactional options

```java
@Transactional(
    readOnly = true,                 // optimization hint for pure reads
    timeout = 5,                     // seconds before auto-rollback
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    rollbackFor = Exception.class    // roll back on checked exceptions too
)
```
- `readOnly = true` → tells the DB/Hibernate "no writes", enabling optimizations.
- `timeout` → roll back if it runs too long.

---

## 6. How it ties to ACID

Transactions are how databases deliver **ACID** (next file): Atomicity (all-or-nothing), Consistency, **Isolation** (the levels above), Durability. `@Transactional` is your tool to control Atomicity (rollback) and Isolation.

---

## Quick Revision Cheat Sheet

- **Transaction** = all-or-nothing unit of DB work (commit fully or rollback).
- **@Transactional**: auto commit on success, rollback on exception (via AOP proxy).
- Default rollback only on **unchecked** exceptions → use `rollbackFor` for checked.
- **Self-invocation** bypasses the proxy → no transaction.
- **Propagation** = how methods join/start transactions: **REQUIRED** (join or create, default) vs **REQUIRES_NEW** (own independent one).
- **Isolation** = how much one txn sees of another's changes.
- Read problems: **dirty** (uncommitted), **non-repeatable** (changed value), **phantom** (changed rows).
- Levels: READ_UNCOMMITTED → READ_COMMITTED → REPEATABLE_READ → SERIALIZABLE (safer but slower).
- `readOnly=true` and `timeout` are handy extras.
