# Database Notes — NoSQL (MongoDB, Redis — when to choose over SQL)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. SQL vs NoSQL — the core difference

*Definition:* **SQL (relational) databases** store data in **structured tables** (rows & columns) with a fixed **schema** and relationships, using SQL. **NoSQL databases** store data in **flexible** formats (documents, key-value, etc.) with no rigid schema, built to **scale horizontally** across many servers.

Real life:
- **SQL** = a structured spreadsheet with strict columns — every row must fit the format.
- **NoSQL** = a folder of free-form sticky notes — each can hold whatever it needs.

```
SQL:    fixed tables, relationships, strong consistency (ACID)
NoSQL:  flexible structure, easy horizontal scaling, often eventual consistency (BASE)
```

**Interview one-liner:** SQL = structured tables with a fixed schema and ACID; NoSQL = flexible schema designed to scale horizontally, often trading strict consistency for speed and scale.

---

## 2. The 4 types of NoSQL databases

| Type | Stores data as | Example | Good for |
|------|----------------|---------|----------|
| **Document** | JSON-like documents | **MongoDB** | Flexible, nested records (products, profiles) |
| **Key-Value** | Simple key → value pairs | **Redis** | Caching, sessions, fast lookups |
| **Column-family** | Columns grouped in families | Cassandra | Huge write volume, time-series |
| **Graph** | Nodes & relationships | Neo4j | Connected data (social networks, recommendations) |

The two you must know for interviews are **MongoDB** (document) and **Redis** (key-value).

---

## 3. MongoDB — document database

*Definition:* **MongoDB** stores data as **documents** (BSON — binary JSON) inside **collections** (like tables). Each document is a flexible, possibly nested JSON object, and documents in the same collection don't all need the same fields.

```javascript
// A document in the "users" collection
{
  "_id": 1,
  "name": "Danish",
  "email": "d@app.com",
  "addresses": [                       // nested array — no separate table needed
    { "city": "Delhi", "pin": "110001" },
    { "city": "Mumbai", "pin": "400001" }
  ],
  "hobbies": ["coding", "cricket"]
}
```

### SQL vs MongoDB vocabulary
| SQL | MongoDB |
|-----|---------|
| Table | Collection |
| Row | Document |
| Column | Field |
| JOIN | Embedding (nest data) or `$lookup` |

👍 **Strengths:** flexible schema (add fields anytime), nested data avoids joins, scales horizontally, fast for read/write of whole documents.
👎 **Weaknesses:** complex multi-document transactions are newer/harder, joins are awkward, can duplicate data.

**Interview one-liner:** MongoDB is a document database storing flexible JSON-like documents in collections — great when your data is nested, varied, or evolving without a fixed schema.

---

## 4. Redis — in-memory key-value store

*Definition:* **Redis** is an **in-memory** key-value database — it keeps data in **RAM** for extremely fast access (microseconds). It's mainly used as a **cache**, session store, and for real-time features.

```
SET user:1:name "Danish"        # store
GET user:1:name                 # → "Danish"  (microsecond-fast)
EXPIRE user:1:name 3600         # auto-delete after 1 hour (TTL)
INCR page:views                 # atomic counter
```

Redis supports rich data types: strings, lists, sets, sorted sets, hashes — useful for leaderboards, queues, rate limiting.

### Why so fast?
- Lives in **RAM** (not disk) → no disk seek.
- Simple key-based lookups.
- Single-threaded event loop (no lock contention).

> Because it's in-memory, RAM is limited and costlier than disk — so Redis holds *hot* data, not your entire dataset. It can persist to disk (snapshots/AOF) for durability, but its core role is **speed**.

### Common Redis use cases
| Use case | Why Redis |
|----------|-----------|
| **Caching** | Avoid hitting the slow main DB for repeated reads |
| **Session store** | Fast, shared across servers |
| **Rate limiting** | Atomic counters with expiry |
| **Leaderboards** | Sorted sets rank in real time |
| **Pub/Sub & queues** | Real-time messaging |

**Interview one-liner:** Redis is an in-memory key-value store that's extremely fast (RAM-based), used mainly for caching, sessions, counters, and real-time features — not as the primary system of record.

---

## 5. When to choose NoSQL over SQL (the decision)

### Choose **SQL** when:
- Data is **structured** and relationships matter (orders ↔ customers ↔ products).
- You need **strong consistency & ACID** (money, banking, inventory).
- You run **complex queries / joins / reporting**.
- The schema is stable.

### Choose **NoSQL** when:
- Data is **flexible / unstructured / rapidly changing** (varied product attributes).
- You need to **scale horizontally** to massive read/write volume.
- You favor **speed & availability** over strict consistency (social feeds, logs, IoT).
- Specific needs: **caching** (Redis), **real-time analytics**, huge document/JSON storage.

| Need | Pick |
|------|------|
| Banking, orders, anything needing ACID | **SQL** |
| Caching, sessions, real-time counters | **Redis** |
| Flexible documents, content, catalogs | **MongoDB** |
| Complex joins & reporting | **SQL** |
| Massive scale, eventual consistency OK | **NoSQL** |

**Easy rule of thumb:** Start with **SQL** (it handles most apps well). Add **Redis** for caching/speed. Reach for **MongoDB/other NoSQL** when the schema is genuinely flexible or you need horizontal scale SQL can't easily give.

---

## 6. Polyglot persistence (the modern reality)

*Definition:* **Polyglot persistence** means using **multiple databases together**, each for what it's best at — not forcing one DB to do everything.

```
A typical real app uses several at once:
  PostgreSQL  → core transactional data (orders, users)   [ACID]
  Redis       → caching + sessions                         [speed]
  MongoDB     → product catalog / flexible content         [flexible]
  Elasticsearch → full-text search
```
> Interview gold: it's **not "SQL vs NoSQL"** — mature systems use **both**. Show you'd pick the right tool per job.

---

## 7. SQL vs NoSQL (the money table)

| | SQL | NoSQL |
|--|-----|-------|
| Schema | Fixed, predefined | Flexible / dynamic |
| Structure | Tables (rows/cols) | Documents, key-value, graph, columns |
| Scaling | Vertical (mostly) | Horizontal (built-in) |
| Consistency | Strong (ACID) | Often eventual (BASE) |
| Joins | ✅ Powerful | ❌ Limited / different |
| Best for | Structured, relational, transactions | Flexible, huge scale, speed |
| Examples | MySQL, PostgreSQL | MongoDB, Redis, Cassandra |

---

## Quick Revision Cheat Sheet

- **SQL** = fixed schema, tables, relationships, **ACID** → structured + transactional data.
- **NoSQL** = flexible schema, horizontal scaling, often **eventual consistency (BASE)**.
- **4 NoSQL types**: document (MongoDB), key-value (Redis), column-family (Cassandra), graph (Neo4j).
- **MongoDB**: JSON-like **documents** in **collections**; flexible, nested data, no rigid schema.
- **Redis**: **in-memory** key-value store; microsecond-fast; caching, sessions, counters, leaderboards (not primary store).
- **Choose SQL** for ACID/relationships/joins; **choose NoSQL** for flexibility/scale/speed.
- **Polyglot persistence**: real apps use several DBs together — pick the right tool per job.
