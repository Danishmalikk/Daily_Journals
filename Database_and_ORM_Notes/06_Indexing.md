# Database Notes — Indexing (B-Tree, Composite, Covering Index)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is an Index?

*Definition:* A database **index** is a separate, sorted data structure that lets the database **find rows fast** without scanning the whole table — like a book's index that points you straight to the right page instead of reading every page.

Real life: To find "Hibernate" in a 1000-page textbook, you don't read all pages. You check the **index** at the back ("Hibernate → page 420") and jump there. A DB index does exactly this for rows.

```
Without index:  scan all 1,000,000 rows   → SLOW (full table scan)
With index:     jump via sorted structure → FAST (e.g. log time)
```

> Trade-off: indexes make **reads faster** but **writes slower** (every insert/update/delete must also update the index) and use **extra storage**. So you index selectively, not everything.

**Interview one-liner:** An index is a sorted lookup structure that speeds up searches by avoiding full table scans — at the cost of slower writes and extra storage.

---

## 2. B-Tree Index (the default & most common)

*Definition:* A **B-Tree (Balanced Tree) index** stores keys in a sorted, balanced tree so the DB can find any value in a few steps (logarithmic time). It's the **default** index type in most databases.

```
                [ M ]
              /       \
        [ D  H ]      [ R  W ]
       /   |   \      /   |   \
   ...leaves (actual sorted keys → row pointers)...
```

- Balanced = all leaves at the same depth → consistent, fast lookups.
- Keeps data **sorted**, so it's great for:
  - Exact matches: `WHERE id = 5`
  - Ranges: `WHERE age BETWEEN 20 AND 30`
  - Sorting: `ORDER BY name`
  - Prefix searches: `WHERE name LIKE 'Da%'` (but NOT `'%da%'`)

```sql
CREATE INDEX idx_users_email ON users(email);   -- B-tree by default
```

> There's also a **Hash index** (super fast for exact `=` matches only, but useless for ranges/sorting). B-Tree is the versatile default; mention Hash as the exact-match alternative.

**Interview one-liner:** A B-Tree index keeps keys sorted in a balanced tree for fast exact-match, range, sorting, and prefix queries — the default index type.

---

## 3. Composite Index (multi-column)

*Definition:* A **composite index** is an index on **two or more columns together**, used to speed up queries that filter/sort on those columns as a group.

```sql
CREATE INDEX idx_name_age ON users(last_name, first_name, age);
```

### The Leftmost-Prefix Rule (the key concept!)
A composite index works **left-to-right**. It can be used only if your query uses the columns **from the left**, without skipping.

Index on `(last_name, first_name, age)` helps these:
```sql
WHERE last_name = 'Malik'                                  ✅ (uses 1st)
WHERE last_name = 'Malik' AND first_name = 'Danish'        ✅ (1st + 2nd)
WHERE last_name = 'Malik' AND first_name='Danish' AND age=25 ✅ (all 3)
```
But **NOT** these:
```sql
WHERE first_name = 'Danish'        ❌ (skips the leftmost last_name)
WHERE age = 25                     ❌ (skips the first two)
```

Memory trick: think of a **phone book** sorted by (last name, first name). You can find "all Maliks" or "Malik, Danish" easily — but you can't quickly find "everyone named Danish" because it's sorted by last name first.

> **Column order matters!** Put the most selective / most-filtered column first, and match the order your queries use.

**Interview one-liner:** A composite index covers multiple columns and follows the leftmost-prefix rule — it helps queries that use the columns left-to-right without skipping the first.

---

## 4. Covering Index

*Definition:* A **covering index** is an index that contains **all the columns a query needs** — so the database answers the query **entirely from the index**, never touching the actual table. Fastest possible read.

```sql
-- Query:
SELECT first_name, last_name FROM users WHERE last_name = 'Malik';

-- Covering index (includes everything the query reads):
CREATE INDEX idx_cover ON users(last_name, first_name);
```
Here the index already has `last_name` (for the filter) **and** `first_name` (for the SELECT). The DB reads only the index → no extra trip to the table.

```
Normal index:    find in index → then fetch full row from table (2 steps)
Covering index:  find in index → answer is already there            (1 step)
```

> In MySQL, EXPLAIN shows `Using index` when a covering index is used. In Postgres/SQL Server you can add non-key columns with `INCLUDE (...)` to build covering indexes.

**Interview one-liner:** A covering index includes all columns a query needs, so it's answered straight from the index without reading the table — the fastest read path.

---

## 5. Other index types (quick awareness)

| Type | Best for |
|------|----------|
| **B-Tree** | General purpose (default): equality, ranges, sorting |
| **Hash** | Exact `=` matches only (no ranges) |
| **Composite** | Multi-column filters (leftmost-prefix rule) |
| **Covering** | Query fully answered from the index |
| **Unique** | Enforce uniqueness + speed (e.g. email) |
| **Full-text** | Searching text/words inside documents |
| **Partial** | Index only some rows (`WHERE active = true`) |

---

## 6. When NOT to index (interview value)

Indexes aren't free. Avoid over-indexing:
- **Small tables** → full scan is already fast.
- **Columns rarely used in WHERE/JOIN/ORDER BY** → wasted index.
- **Heavy write tables** → each index slows every insert/update.
- **Low-selectivity columns** (e.g. a `gender` with 2 values) → index barely helps.

**Easy rule of thumb:** Index columns you frequently **filter, join, or sort** on. Don't index everything — it slows writes and wastes space.

---

## 7. Clustered vs Non-clustered (good to mention)

- **Clustered index** → defines the **physical order** of rows on disk (usually the primary key). One per table.
- **Non-clustered index** → a separate structure pointing to the rows. Many per table.

```
Clustered:     rows physically sorted by the index key (the table IS the index)
Non-clustered: separate index → pointer → actual row
```

---

## Quick Revision Cheat Sheet

- **Index** = sorted lookup structure → fast reads, avoids full table scans; costs slower writes + storage.
- **B-Tree** (default): balanced sorted tree → equality, ranges, sorting, prefix (`LIKE 'a%'`).
- **Hash** index: exact `=` only.
- **Composite**: multi-column; **leftmost-prefix rule** — must use columns left-to-right; column order matters.
- **Covering index**: includes all columns the query needs → answered from index alone (fastest).
- **Don't over-index**: skip small tables, rarely-filtered or low-selectivity columns, write-heavy tables.
- **Clustered** = physical row order (1 per table); **Non-clustered** = separate pointer structure (many).
