# Common SQL Window-Function Mistakes
## Contents

- [1. LASTVALUE over the default frame returns the current row, not the partition's last](#1-lastvalue-over-the-default-frame-returns-the-current-row-not-the-partitions-last)
- [2. Filtering on a window result in WHERE/HAVING](#2-filtering-on-a-window-result-in-wherehaving)
- [3. RANK where ROWNUMBER was meant (ties produce extra rows)](#3-rank-where-rownumber-was-meant-ties-produce-extra-rows)
- [4. Running total that jumps on tied sort keys (RANGE vs ROWS)](#4-running-total-that-jumps-on-tied-sort-keys-range-vs-rows)
- [5. Treating PARTITION BY like GROUP BY](#5-treating-partition-by-like-group-by)
- [6. LAG/LEAD producing NULL at partition edges](#6-laglead-producing-null-at-partition-edges)
- [7. Relying on a window's ORDER BY to order the final result](#7-relying-on-a-windows-order-by-to-order-the-final-result)
- [8. Copy-pasting an identical OVER (...) instead of a named WINDOW](#8-copy-pasting-an-identical-over-instead-of-a-named-window)
- [9. Assuming GROUPS/EXCLUDE or QUALIFY are portable](#9-assuming-groupsexclude-or-qualify-are-portable)


Anti-patterns in LLM-generated SQL around window functions, each with wrong/right code and a
primary-source citation. The skill (`sql-window-functions`) states the rules; this file holds the
high-frequency failure modes. All RIGHT examples are standard/portable SQL; non-standard spellings
(e.g. `QUALIFY`) are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. `LAST_VALUE` over the default frame returns the current row, not the partition's last

**The problem:** The model writes `LAST_VALUE(x) OVER (... ORDER BY ...)` expecting the partition maximum/last, but the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which ends at the current row. The docs warn this "is likely to give unhelpful results for `last_value`" ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)). No error — just a wrong number.

```sql
-- WRONG — returns each row's own salary, not the dep maximum
SELECT empno, dep, salary,
       LAST_VALUE(salary) OVER (PARTITION BY dep ORDER BY salary) AS dep_max
FROM emp;

-- RIGHT — widen the frame explicitly...
SELECT empno, dep, salary,
       LAST_VALUE(salary) OVER (
         PARTITION BY dep ORDER BY salary
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS dep_max
FROM emp;

-- RIGHT — ...or just use an aggregate window (frames the whole partition with no ORDER BY)
SELECT empno, dep, salary,
       MAX(salary) OVER (PARTITION BY dep) AS dep_max
FROM emp;
```

*Source: [PostgreSQL — Window Functions](https://www.postgresql.org/docs/current/functions-window.html); [PostgreSQL — Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS). Depth: this skill, §4.*

---

## 2. Filtering on a window result in `WHERE`/`HAVING`

**The problem:** The model writes `WHERE row_number() OVER (...) = 1`. Window functions "are forbidden ... in `GROUP BY`, `HAVING` and `WHERE` clauses ... because they logically execute after the processing of those clauses" ([PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)). It is a syntax error, and the cause is execution order, not a typo.

```sql
-- WRONG — window not allowed in WHERE
SELECT * FROM emp
WHERE ROW_NUMBER() OVER (PARTITION BY dep ORDER BY salary DESC) <= 3;

-- RIGHT — wrap in a CTE/subquery, filter outside (the top-N-per-group pattern)
WITH ranked AS (
  SELECT empno, dep, salary,
         ROW_NUMBER() OVER (PARTITION BY dep ORDER BY salary DESC, empno) AS rn
  FROM emp
)
SELECT empno, dep, salary FROM ranked WHERE rn <= 3;
```

*Source: [PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html); [modern-sql.com — QUALIFY](https://modern-sql.com/caniuse/qualify). Depth: this skill, §6.*

---

## 3. `RANK` where `ROW_NUMBER` was meant (ties produce extra rows)

**The problem:** "Exactly one row per group" is implemented with `RANK() = 1`, but `rank` "Returns the rank of the current row, with gaps" — tied rows all get rank 1, so a tie yields multiple rows. `row_number` "counting from 1" is distinct per row ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)).

```sql
-- WRONG — two players tied for top score both get rank 1 -> two rows
... WHERE rnk = 1   -- where rnk = RANK() OVER (PARTITION BY team ORDER BY score DESC)

-- RIGHT — ROW_NUMBER with a total order guarantees exactly one
... WHERE rn = 1    -- where rn = ROW_NUMBER() OVER (PARTITION BY team ORDER BY score DESC, player_id)
```

*Source: [PostgreSQL — Window Functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §2.*

---

## 4. Running total that jumps on tied sort keys (`RANGE` vs `ROWS`)

**The problem:** A running total uses the default frame (`RANGE`), which includes the current row's whole peer group. When the sort key has duplicates, every tied row shows the same cumulative sum. A per-row running total needs `ROWS` ([PostgreSQL — Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)).

```sql
-- WRONG — two rows sharing sale_date both show the day's full total
SUM(amount) OVER (ORDER BY sale_date)

-- RIGHT — ROWS counts physical rows; add a tiebreaker for determinism
SUM(amount) OVER (ORDER BY sale_date, id
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

*Source: [PostgreSQL — Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS). Depth: this skill, §5.*

---

## 5. Treating `PARTITION BY` like `GROUP BY`

**The problem:** The model expects `OVER (PARTITION BY dep)` to collapse to one row per department. It does not — "the rows retain their separate identities" ([PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)). Conversely, the model reaches for a self-join to show both detail and aggregate when a window does it in one pass.

```sql
-- WRONG — expecting one row per dep from a window (you get every row)
SELECT dep, AVG(salary) OVER (PARTITION BY dep) FROM emp;  -- N rows, not one-per-dep

-- RIGHT — use GROUP BY to collapse...
SELECT dep, AVG(salary) FROM emp GROUP BY dep;
-- ...or keep the window when you WANT every row plus its dep average
SELECT empno, dep, salary, AVG(salary) OVER (PARTITION BY dep) AS dep_avg FROM emp;
```

*Source: [PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html). Depth: this skill, §1.*

---

## 6. `LAG`/`LEAD` producing NULL at partition edges

**The problem:** `LAG(x)` on the first row of a partition has "no such row," so it returns the default, which "If omitted ... defaults to ... `NULL`" ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)). A delta computed from it then becomes NULL (foundation skill §6 — NULL propagation).

```sql
-- WRONG — first row's mom_change is NULL, then arithmetic with it stays NULL
SELECT month, revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change FROM monthly;

-- RIGHT — supply a default for the edge (third argument)
SELECT month, revenue - LAG(revenue, 1, 0) OVER (ORDER BY month) AS mom_change FROM monthly;
```

*Source: [PostgreSQL — Window Functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §3.*

---

## 7. Relying on a window's `ORDER BY` to order the final result

**The problem:** The model assumes `OVER (ORDER BY ts)` also sorts the output rows. It does not — the window `ORDER BY` orders rows only for the function's computation; the result set is still unordered without an outer `ORDER BY` (foundation skill §1).

```sql
-- WRONG — output row order is undefined despite the window ORDER BY
SELECT id, ts, ROW_NUMBER() OVER (ORDER BY ts) AS rn FROM events;

-- RIGHT — order the query itself
SELECT id, ts, ROW_NUMBER() OVER (ORDER BY ts) AS rn FROM events ORDER BY ts, id;
```

*Source: [PostgreSQL — Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html). Depth: foundation `sql-relational-and-null-discipline` §1; this skill, §1.*

---

## 8. Copy-pasting an identical `OVER (...)` instead of a named `WINDOW`

**The problem:** Several SELECT-list functions repeat the same long `OVER (PARTITION BY ... ORDER BY ...)`. Beyond verbosity, divergent copies drift. A named `WINDOW` defines it once.

```sql
-- WRONG (smell) — three copies that must be kept in sync by hand
SELECT RANK() OVER (PARTITION BY dep ORDER BY salary DESC),
       DENSE_RANK() OVER (PARTITION BY dep ORDER BY salary DESC),
       AVG(salary) OVER (PARTITION BY dep ORDER BY salary DESC)
FROM emp;

-- RIGHT — define once, reference by name
SELECT RANK() OVER w, DENSE_RANK() OVER w, AVG(salary) OVER w
FROM emp
WINDOW w AS (PARTITION BY dep ORDER BY salary DESC);
```

*Source: [PostgreSQL — Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS). Depth: this skill, §7.*

---

## 9. Assuming `GROUPS`/`EXCLUDE` or `QUALIFY` are portable

**The problem:** The model uses `GROUPS`/`EXCLUDE` frames or a `QUALIFY` clause assuming they run everywhere. `GROUPS`/`EXCLUDE` need PostgreSQL 11+ or SQLite 3.28+ and are absent from MySQL 8/MariaDB; `QUALIFY` is non-standard and unsupported by PostgreSQL, MySQL, MariaDB, SQL Server, and Oracle ([modern-sql.com — QUALIFY](https://modern-sql.com/caniuse/qualify); [SQLite — window functions](https://www.sqlite.org/windowfunctions.html)).

```sql
-- WRONG (portability) — QUALIFY is rejected on Postgres/MySQL/SQLite/Oracle/SQL Server
SELECT * FROM emp
QUALIFY ROW_NUMBER() OVER (PARTITION BY dep ORDER BY salary DESC) <= 3;

-- RIGHT — the portable wrap-in-CTE form works on every engine with window functions
WITH ranked AS (
  SELECT emp.*, ROW_NUMBER() OVER (PARTITION BY dep ORDER BY salary DESC, empno) AS rn
  FROM emp
)
SELECT * FROM ranked WHERE rn <= 3;
```

*Source: [modern-sql.com — QUALIFY](https://modern-sql.com/caniuse/qualify); [modern-sql.com — ROWS framing](https://modern-sql.com/caniuse/over_rows_between). Depth: this skill, §8–§9; dialect spellings owned by `sql-standard-vs-dialect-map`.*
