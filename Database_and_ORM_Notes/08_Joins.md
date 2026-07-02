# Database Notes — Joins (Inner, Left, Right, Full, Self, Cross)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Join?

*Definition:* A **JOIN** combines rows from **two or more tables** based on a related column between them, so you can see connected data together (e.g. each order *with* its customer's name).

Real life: You have a list of orders (with `customer_id`) and a separate list of customers. A JOIN is like matching each order to its customer by id, producing one combined sheet.

### Our example tables
```
users                      orders
+----+--------+            +----+---------+--------+
| id | name   |            | id | user_id | amount |
+----+--------+            +----+---------+--------+
| 1  | Alice  |            | 10 |   1     |  500   |
| 2  | Bob    |            | 11 |   1     |  300   |
| 3  | Carol  |            | 12 |   2     |  700   |
+----+--------+            | 13 |   9     |  100   |  ← user_id 9 doesn't exist
                           +----+---------+--------+
```

---

## 2. INNER JOIN — only matching rows

*Definition:* **INNER JOIN** returns only the rows that have a **match in both tables**. No match = row excluded from both sides.

```sql
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```
```
Result (only matched):
Alice  500
Alice  300
Bob    700
-- Carol excluded (no orders); order 13 excluded (no user 9)
```

```
   users    orders
    (  ●●●●●●  )      ← only the overlap
```

**Interview one-liner:** INNER JOIN returns only rows that match in both tables; unmatched rows from either side are dropped.

---

## 3. LEFT JOIN — all left rows + matches

*Definition:* **LEFT JOIN** (LEFT OUTER JOIN) returns **all rows from the left table**, plus matching rows from the right. Where there's no match, right-side columns are `NULL`.

```sql
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```
```
Result (all users, even without orders):
Alice  500
Alice  300
Bob    700
Carol  NULL    ← kept, but no order → NULL
```

> Super useful for "find rows with NO match":
```sql
SELECT u.name FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;        -- users who never ordered → Carol
```

**Interview one-liner:** LEFT JOIN keeps all left-table rows; unmatched right columns become NULL — great for finding rows with no match (`WHERE right.id IS NULL`).

---

## 4. RIGHT JOIN — all right rows + matches

*Definition:* **RIGHT JOIN** is the mirror of LEFT JOIN: it returns **all rows from the right table**, plus matches from the left. Unmatched left columns are `NULL`.

```sql
SELECT u.name, o.amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;
```
```
Result (all orders, even orphan ones):
Alice  500
Alice  300
Bob    700
NULL   100     ← order 13 has user_id 9 (no such user) → NULL name
```

> In practice, people rarely use RIGHT JOIN — you can always rewrite it as a LEFT JOIN by swapping table order. Most teams standardize on LEFT JOIN for readability.

**Interview one-liner:** RIGHT JOIN keeps all right-table rows (mirror of LEFT JOIN); rarely used since swapping tables turns it into a LEFT JOIN.

---

## 5. FULL (OUTER) JOIN — everything from both

*Definition:* **FULL OUTER JOIN** returns **all rows from both tables** — matched where possible, with `NULL`s filled in on whichever side has no match.

```sql
SELECT u.name, o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```
```
Result (everything):
Alice  500
Alice  300
Bob    700
Carol  NULL    ← user with no order
NULL   100     ← order with no user
```

> ⚠️ **MySQL has no FULL OUTER JOIN!** You emulate it with `LEFT JOIN ... UNION ... RIGHT JOIN`. PostgreSQL/SQL Server support it natively. Common interview gotcha.

**Interview one-liner:** FULL OUTER JOIN returns all rows from both tables with NULLs where there's no match; MySQL lacks it (emulate with LEFT UNION RIGHT).

---

## 6. SELF JOIN — a table joined to itself

*Definition:* A **SELF JOIN** joins a table **to itself**, used when rows relate to other rows in the *same* table (like an employee whose `manager_id` points to another employee).

```
employees
+----+--------+------------+
| id | name   | manager_id |
+----+--------+------------+
| 1  | Alice  |   NULL     |
| 2  | Bob    |    1       |   ← Bob's manager is Alice
| 3  | Carol  |    1       |
+----+--------+------------+
```
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;   -- same table, two aliases
```
```
Bob    → Alice
Carol  → Alice
Alice  → NULL
```
> The trick: use **two different aliases** (`e` and `m`) so the DB treats them as separate "copies" of the table.

**Interview one-liner:** A SELF JOIN joins a table to itself (using two aliases) to relate rows within the same table — e.g. employees to their managers.

---

## 7. CROSS JOIN — every combination (Cartesian product)

*Definition:* A **CROSS JOIN** pairs **every row of the first table with every row of the second** — no join condition. Result size = rows(A) × rows(B).

```sql
SELECT s.size, c.color
FROM sizes s
CROSS JOIN colors c;
-- 3 sizes × 4 colors = 12 combinations
```
Use for: generating all combinations (sizes × colors), building calendars, test data.

> ⚠️ Accidentally writing a join **without an ON condition** creates a cross join — which can explode into millions of rows. A classic bug.

**Interview one-liner:** CROSS JOIN produces every combination of rows from both tables (Cartesian product, A×B) — useful for combinations, dangerous by accident.

---

## 8. All joins at a glance (the money table)

| Join | Returns | Unmatched rows |
|------|---------|----------------|
| **INNER** | Only matching rows | Dropped (both sides) |
| **LEFT** | All left + matches | Right = NULL |
| **RIGHT** | All right + matches | Left = NULL |
| **FULL OUTER** | All rows, both tables | NULL on the side missing |
| **SELF** | Table joined to itself | (depends on inner/left) |
| **CROSS** | Every combination (A×B) | No condition at all |

```
INNER  = ●overlap only
LEFT   = ◖all left + overlap
RIGHT  = overlap + all right◗
FULL   = ◖ everything ◗
```

---

## Quick Revision Cheat Sheet

- **JOIN** combines rows from tables on a related column.
- **INNER**: only rows matching in both.
- **LEFT**: all left rows + matches (right NULL if none) → find "no match" with `WHERE right.id IS NULL`.
- **RIGHT**: mirror of LEFT (rarely used; swap tables instead).
- **FULL OUTER**: all rows from both, NULLs where unmatched (MySQL lacks it → LEFT UNION RIGHT).
- **SELF**: table joined to itself via two aliases (employee → manager).
- **CROSS**: every combination (A×B, Cartesian) — no ON; dangerous by accident.
