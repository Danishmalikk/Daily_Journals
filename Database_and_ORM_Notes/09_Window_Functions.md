# Database Notes — Window Functions (ROW_NUMBER, RANK, PARTITION BY)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Window Function?

*Definition:* A **window function** performs a calculation **across a set of related rows** (a "window") **without collapsing them into one row**. Unlike `GROUP BY` (which squashes groups into single summary rows), a window function keeps every row and just **adds** a computed column.

Real life: In a class ranked by marks, each student keeps their own row but also gets a "rank" column. You didn't merge students — you *added* info per row while looking at the whole class.

### The key difference from GROUP BY
```
GROUP BY:          collapses rows → one row per group (you lose the detail)
Window function:   keeps all rows → adds a calculation alongside each
```

```sql
-- GROUP BY: 1 row per dept (detail lost)
SELECT dept, AVG(salary) FROM emp GROUP BY dept;

-- Window: every employee row kept + dept average shown next to each
SELECT name, dept, salary, AVG(salary) OVER (PARTITION BY dept) AS dept_avg
FROM emp;
```

**Interview one-liner:** A window function computes across a group of rows but keeps every row (adds a column), unlike GROUP BY which collapses groups into single rows.

---

## 2. The OVER() clause — defines the "window"

*Definition:* The **`OVER()`** clause is what makes a function a *window* function. It defines **which rows** the function looks at — optionally split into groups (`PARTITION BY`) and ordered (`ORDER BY`).

```sql
function() OVER (
    PARTITION BY column   -- split rows into groups (optional)
    ORDER BY column       -- order within each group (optional)
)
```

- No `PARTITION BY` → the window is the **whole result set**.
- With `PARTITION BY dept` → a **separate window per department**.

---

## 3. PARTITION BY — split into groups

*Definition:* **`PARTITION BY`** divides the rows into **groups (partitions)**, and the window function restarts its calculation for each group. It's like "GROUP BY for the window" — but without collapsing rows.

```sql
SELECT name, dept, salary,
       RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS dept_rank
FROM employees;
```
```
name    dept   salary  dept_rank
Alice   Eng    9000      1     ← ranking restarts per dept
Bob     Eng    8000      2
Carol   Eng    8000      2
David   Sales  7000      1     ← new partition, rank resets to 1
Eve     Sales  6000      2
```

> `PARTITION BY` (window, keeps rows) vs `GROUP BY` (aggregates, collapses rows) — a very common interview comparison.

**Interview one-liner:** PARTITION BY splits rows into groups and restarts the window calculation per group, without collapsing the rows like GROUP BY does.

---

## 4. The ranking functions: ROW_NUMBER, RANK, DENSE_RANK

All three number rows within a window — the difference is **how they handle ties**.

```sql
SELECT name, salary,
   ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
   RANK()       OVER (ORDER BY salary DESC) AS rank,
   DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```
```
name    salary  row_num  rank  dense_rank
Alice   9000      1        1       1
Bob     8000      2        2       2     ← tie
Carol   8000      3        2       2     ← tie (same rank)
David   7000      4        4       3
                           ↑        ↑
                    RANK skips 3   DENSE_RANK doesn't skip
```

| Function | Ties get… | After a tie… | Use for |
|----------|-----------|--------------|---------|
| **ROW_NUMBER** | Different numbers (1,2,3,4) | continuous | Unique row numbering, pagination |
| **RANK** | Same number (1,2,2,4) | **skips** (gap) | Competition ranking ("two 2nd places, no 3rd") |
| **DENSE_RANK** | Same number (1,2,2,3) | **no gap** | Ranking without skipping numbers |

Memory trick: **ROW_NUMBER** = always unique; **RANK** = ties + gaps; **DENSE_RANK** = ties, no gaps (dense = packed together).

**Interview one-liner:** ROW_NUMBER gives unique sequential numbers; RANK gives ties the same number but skips the next; DENSE_RANK gives ties the same number with no gaps.

---

## 5. The "Top N per group" pattern (the #1 interview use-case)

Question: *"Get the top 2 highest-paid employees in each department."* You can't do this with GROUP BY alone. Window functions nail it:

```sql
SELECT * FROM (
    SELECT name, dept, salary,
           ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn <= 2;        -- top 2 per department
```
This is the classic "greatest-N-per-group" solution. If interviewers ask one window-function question, it's usually this.

---

## 6. Other handy window functions

| Function | What it does |
|----------|--------------|
| **LAG(col)** | Value from the **previous** row (e.g. compare to last month) |
| **LEAD(col)** | Value from the **next** row |
| **SUM() OVER(...)** | Running total / cumulative sum |
| **AVG() OVER(...)** | Moving/group average alongside each row |
| **FIRST_VALUE / LAST_VALUE** | First/last value in the window |
| **NTILE(n)** | Split rows into n buckets (e.g. quartiles) |

```sql
-- Running total of sales by date
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) AS running_total
FROM sales;

-- Compare each month to the previous (LAG)
SELECT month, revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS change_vs_prev
FROM monthly;
```

**Interview one-liner:** Beyond ranking, LAG/LEAD compare to neighboring rows, and SUM/AVG OVER give running totals and moving averages — all keeping every row.

---

## 7. When to use window functions vs GROUP BY

| Need | Use |
|------|-----|
| One summary row per group (total per dept) | **GROUP BY** |
| Keep all rows + add a calculation | **Window function** |
| Rank rows within groups | **Window** (RANK/ROW_NUMBER) |
| Top-N per group | **Window** (ROW_NUMBER + filter) |
| Running totals, prev/next comparisons | **Window** (SUM OVER, LAG/LEAD) |

**Easy rule of thumb:** If you need to keep individual rows *and* show a group-level calculation or ranking → window function. If you only need the summary → GROUP BY.

---

## Quick Revision Cheat Sheet

- **Window function** = calculate across related rows but **keep every row** (unlike GROUP BY which collapses).
- **OVER()** defines the window; **PARTITION BY** splits into groups (restarts per group); **ORDER BY** orders within.
- **ROW_NUMBER** = unique (1,2,3,4); **RANK** = ties same, skips next (1,2,2,4); **DENSE_RANK** = ties same, no gap (1,2,2,3).
- **Top-N per group**: `ROW_NUMBER() OVER (PARTITION BY g ORDER BY x DESC)` then filter `rn <= N`.
- **LAG/LEAD** = previous/next row; **SUM/AVG OVER** = running totals / moving averages.
- GROUP BY for summaries; window functions to keep rows + add per-row calculations.
