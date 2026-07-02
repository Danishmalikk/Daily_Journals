# Database Notes — JPA vs Hibernate vs JDBC (when to use what)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The big picture: three ways to talk to a database from Java

All three let your Java code read/write a database, but at **different levels of abstraction** (how much they do for you).

```
JDBC        → you write raw SQL by hand           (low level, full control)
Hibernate   → maps objects ↔ tables for you        (high level, less code)
JPA         → the standard/rulebook Hibernate follows (just the interface)
```

Real life:
- **JDBC** = driving a manual car (you control every gear).
- **Hibernate** = an automatic car (it handles the gears).
- **JPA** = the *rules of the road* both cars must follow (the standard).

---

## 2. JDBC — the low-level foundation

*Definition:* **JDBC (Java Database Connectivity)** is the basic Java API for connecting to a database and running **raw SQL** yourself. You write the SQL, send it, and manually read the results row by row.

```java
Connection con = DriverManager.getConnection(url, user, pass);
PreparedStatement ps = con.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setLong(1, 5);
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    User u = new User();
    u.setName(rs.getString("name"));   // manual mapping, column by column
}
con.close();   // you manage connections, exceptions, closing...
```

| 👍 Pros | 👎 Cons |
|---------|---------|
| Full control over SQL | Lots of boilerplate |
| Fast (no overhead) | Manual mapping (rows → objects) |
| Good for complex/tuned queries | Easy to leak connections/forget closing |

**Interview one-liner:** JDBC is the low-level API where you write raw SQL and manually map results — maximum control, maximum boilerplate.

---

## 3. Hibernate — the ORM that removes boilerplate

*Definition:* **Hibernate** is an **ORM (Object-Relational Mapping)** tool. It automatically maps Java **objects to database tables** and back, so you work with objects and it generates the SQL for you.

> **ORM** = a technique that bridges two worlds: objects in Java vs rows/tables in a database. It translates between them so you rarely write SQL.

```java
User u = session.get(User.class, 5L);   // Hibernate runs the SELECT + maps it
u.setName("Danish");
session.save(u);                          // Hibernate runs the UPDATE
```

Hibernate also adds: caching, lazy loading, dirty checking (auto-detects changes), and database independence (switch MySQL→PostgreSQL with little change).

| 👍 Pros | 👎 Cons |
|---------|---------|
| Very little code | Learning curve |
| Auto object↔table mapping | Hidden SQL can cause surprises (N+1) |
| Caching, lazy loading built in | Slight overhead vs raw JDBC |
| Database-independent | Hard to hand-tune tricky queries |

**Interview one-liner:** Hibernate is an ORM that maps objects to tables and auto-generates SQL, adding caching and lazy loading — far less code than JDBC.

---

## 4. JPA — the specification (the rulebook)

*Definition:* **JPA (Jakarta/Java Persistence API)** is a **specification** — a set of standard interfaces and annotations for ORM in Java. It defines *what* an ORM should do, but contains **no working code itself**. Hibernate is the most popular **implementation** of JPA.

```
JPA (the standard interfaces: EntityManager, @Entity, @Id ...)
   ↑ implemented by
Hibernate / EclipseLink / OpenJPA  (the actual working tools)
```

Why have a standard? So your code depends on **JPA**, not directly on Hibernate. You could swap Hibernate for EclipseLink without rewriting your app.

```java
@Entity
class User {
    @Id @GeneratedValue Long id;
    String name;
}
// EntityManager is a JPA interface; Hibernate implements it under the hood
entityManager.find(User.class, 5L);
```

**Interview one-liner:** JPA is the specification (standard interfaces + annotations) for ORM; Hibernate is the implementation that actually does the work. Code to JPA, run on Hibernate.

---

## 5. How they stack up (the money table)

| | JDBC | JPA | Hibernate |
|--|------|-----|-----------|
| What is it | Low-level API | Specification (rules) | ORM implementation |
| You write SQL? | ✅ Always | Rarely | Rarely |
| Object↔table mapping | ❌ Manual | ✅ (defined) | ✅ (does it) |
| Boilerplate | High | Low | Low |
| Control over SQL | Full | Less | Less (but possible) |
| Runnable code? | ✅ | ❌ (just interfaces) | ✅ |

```
Abstraction level:   JDBC  <  Hibernate/JPA
Control:             JDBC  >  Hibernate/JPA
Productivity:        JDBC  <  Hibernate/JPA
```

---

## 6. Where does Spring Data JPA fit?

One more layer on top (you saw this in the Spring Boot notes):
```
Spring Data JPA   → write repository interfaces, zero implementation
      ↓ uses
JPA               → the standard
      ↓ implemented by
Hibernate         → does the mapping & SQL
      ↓ uses
JDBC              → actually sends SQL to the DB
      ↓
Database
```
So even when you use Spring Data JPA, **JDBC is still there at the bottom** doing the real DB communication.

---

## 7. When to use what (the decision)

| Situation | Use |
|-----------|-----|
| Standard CRUD app, want speed of development | **JPA/Hibernate** (via Spring Data JPA) |
| Complex reporting, heavy custom SQL, max performance | **JDBC** (or native queries) |
| Need to switch databases easily | **JPA/Hibernate** (DB-independent) |
| Tiny script / one quick query | **JDBC** |
| Mix: mostly ORM, a few tuned queries | **Hibernate + native SQL** for the hot spots |

**Easy rule of thumb:** Default to **JPA/Hibernate** for productivity. Drop to **JDBC / native SQL** only for the few complex, performance-critical queries.

---

## Quick Revision Cheat Sheet

- **JDBC**: low-level API, you write raw SQL + manual mapping → full control, lots of boilerplate.
- **ORM**: maps Java objects ↔ DB tables automatically.
- **Hibernate**: the popular ORM — auto SQL, caching, lazy loading, DB-independent.
- **JPA**: the **specification** (interfaces + annotations); no code itself. Hibernate **implements** it.
- **Abstraction**: JDBC < JPA/Hibernate; **Control**: JDBC > JPA/Hibernate.
- **Spring Data JPA** sits on top of JPA → on Hibernate → on JDBC → DB.
- Default to **JPA/Hibernate**; use **JDBC/native SQL** for complex, tuned queries.
