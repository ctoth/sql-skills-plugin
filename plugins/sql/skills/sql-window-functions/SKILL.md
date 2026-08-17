---
name: sql-window-functions
description: Guides window functions — the highest-leverage modern-SQL technique and the frame-clause traps LLMs reliably botch. A window function computes across related rows without collapsing them (PARTITION BY is not GROUP BY). The centerpiece — the default frame with ORDER BY is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW, which makes LAST_VALUE return the current row, not the partition's last, and makes running totals jump on tied sort keys; fix with an explicit ROWS frame (or MAX() OVER). Covers ROW_NUMBER/RANK/DENSE_RANK/NTILE tie behavior, LAG/LEAD offsets, FIRST_VALUE/LAST_VALUE/NTH_VALUE, ROWS vs RANGE vs GROUPS, named WINDOW reuse, EXCLUDE, and that a window result cannot be filtered in WHERE/HAVING (wrap in a CTE/subquery — the top-N-per-group pattern). Auto-invokes when writing or editing OVER (...), PARTITION BY, frame clauses (ROWS/RANGE/GROUPS), ranking/offset/value functions, running or moving totals, "top-N per group", or "filter on row_number" / "WHERE row_number() = 1" requests.
allowed-tools: Read, Glob, Grep
---

# SQL Window Functions

> "Window functions do not cause rows to become grouped into a single output row like non-window aggregate calls would. Instead, the rows retain their separate identities."
> — [PostgreSQL — Window Functions (tutorial)](https://www.postgresql.org/docs/current/tutorial-window.html)

> "The default framing option is `RANGE UNBOUNDED PRECEDING`, which is the same as `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. With `ORDER BY`, this sets the frame to be all rows from the partition start up through the current row's last `ORDER BY` peer."
> — [PostgreSQL — Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)

A window function attaches a per-row computation that looks across *other* rows — a running total, a rank, the previous row's value — **without collapsing the result the way `GROUP BY` does**. That one property makes a whole class of reports a single pass instead of a self-join. But the power comes with one quiet trap that the documentation itself flags as "likely to give unhelpful results": the *default frame*. Most of this skill orbits that trap. This skill assumes the set semantics and three-valued logic of the foundation skill `sql-relational-and-null-discipline` — windows compute over ordered partitions, but the result is still an unordered set unless the outer query has its own `ORDER BY`.

---

## 1. What a Window Function Is — `PARTITION BY` Is Not `GROUP BY`

A window function "performs a calculation across a set of table rows that are somehow related to the current row ... However, window functions do not cause rows to become grouped into a single output row like non-window aggregate calls would. Instead, the rows retain their separate identities" ([PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)). `GROUP BY dep` returns **one** row per department; `OVER (PARTITION BY dep)` returns **every** row, each carrying its department's aggregate alongside.

"The `PARTITION BY` clause within `OVER` divides the rows into groups, or partitions ... For each row, the window function is computed across the rows that fall into the same partition as the current row" ([PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)). Omit `PARTITION BY` and the whole result is one partition.

```sql
-- WRONG — GROUP BY collapses to one row per dep; you lose empno/salary detail
SELECT dep, AVG(salary) FROM emp GROUP BY dep;   -- can't show each employee too

-- RIGHT — the window keeps every row AND attaches the dep average
SELECT empno, dep, salary,
       AVG(salary) OVER (PARTITION BY dep) AS dep_avg
FROM emp;
```

The window `ORDER BY` orders rows *within the partition* for the function's purposes; it does not order the final output. A bare window query has no defined row order — that is the foundation skill's rule (`sql-relational-and-null-discipline` §1), and it still applies.

---

## 2. Ranking Functions — and the Tie Behavior That Bites

Three ranking functions differ **only in how they treat ties** (peer rows equal on the window `ORDER BY`):

- `row_number` "Returns the number of the current row within its partition, counting from 1" — every row distinct, ties broken arbitrarily.
- `rank` "Returns the rank of the current row, with gaps" — ties share a rank, then it **skips** (1, 1, 3).
- `dense_rank` "Returns the rank of the current row, without gaps; this function effectively counts peer groups" — ties share a rank, **no skip** (1, 1, 2).

(All three quoted from [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html).) `ntile(n)` "Returns an integer ranging from 1 to the argument value, dividing the partition as equally as possible" — for quartiles/deciles.

```sql
-- Scores 90, 90, 80:
ROW_NUMBER() OVER (ORDER BY score DESC)  -- 1, 2, 3   (arbitrary which 90 is #1)
RANK()       OVER (ORDER BY score DESC)  -- 1, 1, 3   (gap after the tie)
DENSE_RANK() OVER (ORDER BY score DESC)  -- 1, 1, 2   (no gap)
```

```sql
-- WRONG — "the single top scorer per team" but ties produce TWO rows
WHERE RANK() OVER (PARTITION BY team ORDER BY score DESC) = 1   -- (also illegal in WHERE, see §6)

-- RIGHT — ROW_NUMBER guarantees exactly one; break the tie deterministically
ROW_NUMBER() OVER (PARTITION BY team ORDER BY score DESC, player_id)
```

Picking `RANK` where you meant `ROW_NUMBER` is a silent off-by-one (or off-by-many) in leaderboards and top-N. If you need exactly one row per group, use `ROW_NUMBER` with a **total** order; if "all rows tied for first" is intended, use `RANK`/`DENSE_RANK`.

---

## 3. Offset Functions (`LAG`/`LEAD`) and Value Functions (`FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`)

`lag` "Returns _value_ evaluated at the row that is _offset_ rows before the current row within the partition; if there is no such row, instead returns _default_ ... If omitted, _offset_ defaults to 1 and _default_ to `NULL`" ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)); `lead` looks *after*. These are the row-to-row delta tools (this month vs last month).

```sql
-- Month-over-month change; the 3rd arg avoids a NULL on the first row
SELECT month, revenue,
       revenue - LAG(revenue, 1, 0) OVER (ORDER BY month) AS mom_change
FROM monthly;
```

The value functions read a specific row **of the frame**: `first_value`, `last_value`, and `nth_value(value, n)` return the value at the first / last / n'th row of the *window frame*. That phrase "of the window frame" is the entire problem — see §4.

**Frame-sensitivity matters:** `first_value`, `last_value`, `nth_value` (and aggregate windows like `SUM`/`MAX`) depend on the frame. `row_number`, `rank`, `dense_rank`, `ntile`, `lag`, `lead` **ignore** the frame — they always see the full partition. So the §4 trap hits the value/aggregate functions, never the ranking/offset ones.

---

## 4. The Centerpiece — The Default Frame and the `LAST_VALUE` Surprise

When the `OVER` clause has an `ORDER BY` but no explicit frame, the frame is **not** the whole partition. "The default framing option is `RANGE UNBOUNDED PRECEDING`, which is the same as `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. With `ORDER BY`, this sets the frame to be all rows from the partition start up through the current row's last `ORDER BY` peer" ([PostgreSQL — window calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)). SQLite is identical: "The default frame-spec is: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`" ([SQLite — window functions](https://www.sqlite.org/windowfunctions.html)).

So the frame **ends at the current row**. `LAST_VALUE` therefore returns the *current row's* value, not the partition's last — which is why the docs warn this "is likely to give unhelpful results for `last_value`" ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)).

```sql
-- WRONG — returns the CURRENT row's salary every time, not the dep's max.
-- In a financial report this silently prints the wrong "highest" number.
SELECT empno, dep, salary,
       LAST_VALUE(salary) OVER (PARTITION BY dep ORDER BY salary) AS dep_max
FROM emp;

-- RIGHT — widen the frame to the whole partition explicitly
SELECT empno, dep, salary,
       LAST_VALUE(salary) OVER (
         PARTITION BY dep ORDER BY salary
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS dep_max
FROM emp;

-- RIGHT (simpler) — for "max of the partition" just use an aggregate window,
-- which needs no ORDER BY and so frames the entire partition by default
SELECT empno, dep, salary,
       MAX(salary) OVER (PARTITION BY dep) AS dep_max
FROM emp;
```

Rule of thumb: if you wrote `LAST_VALUE`, stop and ask whether you actually want `MAX() OVER` or an explicit `UNBOUNDED FOLLOWING` frame. `FIRST_VALUE` is usually safe (the frame starts at the partition start), but it too breaks if you add a `... PRECEDING` lower bound.

---

## 5. `ROWS` vs `RANGE` on Tied Sort Keys — Running Totals That Jump

The same default frame creates a second trap for **running totals**. With `RANGE`, the frame is value-based and includes *the entire peer group* of the current row: "all rows from the partition start up through the current row's last `ORDER BY` peer" ([PostgreSQL — window calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)). So when the sort key has duplicates, every tied row gets the **same** cumulative sum — the total through the whole peer group — and the running total appears to jump in steps.

`ROWS` counts physical rows instead: "the frame starts or ends the specified number of rows before or after the current row" ([PostgreSQL — window calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)). For a true per-row running total over a non-unique sort key, you must say `ROWS`.

```sql
-- Two sales both dated '2026-01-02', amounts 10 and 20:

-- WRONG (often) — default RANGE lumps the tied dates; BOTH rows show the
-- running total through the whole day, so the per-row progression is lost.
SUM(amount) OVER (ORDER BY sale_date)                                  -- ..., 30, 30

-- RIGHT — ROWS counts rows, giving a genuine per-row running total
SUM(amount) OVER (ORDER BY sale_date, id
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)    -- ..., 10, 30
```

A third mode, `GROUPS`, counts *peer groups* rather than rows ("the specified number of _peer groups_ ... a set of rows that are equivalent in the `ORDER BY` ordering," [PostgreSQL — window calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)) — useful for "this group plus the previous two groups." `RANGE` with an offset (e.g. `RANGE BETWEEN '7 days' PRECEDING AND CURRENT ROW`) gives value-windowed moving averages over time; `RANGE` "require[s] that the `ORDER BY` clause specify exactly one column."

---

## 6. You Cannot Filter a Window Result in `WHERE`/`HAVING` — Wrap It

"Window functions are permitted only in the `SELECT` list and the `ORDER BY` clause of the query. They are forbidden elsewhere, such as in `GROUP BY`, `HAVING` and `WHERE` clauses. This is because they logically execute after the processing of those clauses" ([PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)). modern-sql puts it bluntly: "Neither the `where` nor the `having` clause allow window functions" ([modern-sql.com — QUALIFY](https://modern-sql.com/caniuse/qualify)).

This is the `WHERE row_number() = 1` error that baffles newcomers — it is a *logical-order* problem, not a syntax typo. The window is computed after `WHERE` has already run, so `WHERE` can't see it. The fix: compute the window in a subquery/CTE, then filter the outer query. "If there is a need to filter or group rows after the window calculations are performed, you can use a sub-select" ([PostgreSQL — tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)).

```sql
-- WRONG — window function not allowed in WHERE; this is a syntax error
SELECT * FROM emp
WHERE ROW_NUMBER() OVER (PARTITION BY dep ORDER BY salary DESC) <= 3;

-- RIGHT — top-N-per-group: rank in a CTE, filter outside
WITH ranked AS (
  SELECT empno, dep, salary,
         ROW_NUMBER() OVER (PARTITION BY dep ORDER BY salary DESC, empno) AS rn
  FROM emp
)
SELECT empno, dep, salary FROM ranked WHERE rn <= 3;
```

This is *the* portable top-N-per-group pattern. Some engines (Snowflake, BigQuery, DuckDB, Teradata) offer a `QUALIFY` shortcut, but it "is supported by" only a minority and **not** by PostgreSQL, MySQL, MariaDB, SQL Server, or Oracle ([modern-sql.com — QUALIFY](https://modern-sql.com/caniuse/qualify)) — so the wrap-in-CTE form is the one to write for portability. A `LATERAL`/correlated-subquery alternative for top-N lives in `sql-lateral-and-correlated-derived`.

---

## 7. Named `WINDOW` Clause — Define Once, Reuse

When several SELECT-list functions share a window, name it once with a `WINDOW` clause instead of copy-pasting the `OVER (...)` spec (and risking subtly divergent copies):

```sql
SELECT empno, dep, salary,
       RANK()       OVER w  AS rnk,
       DENSE_RANK() OVER w  AS dense_rnk,
       AVG(salary)  OVER w  AS dep_avg
FROM emp
WINDOW w AS (PARTITION BY dep ORDER BY salary DESC);
```

Note `OVER w` references the named window exactly; `OVER (w ...)` copies and modifies it and "will be rejected if the referenced window specification includes a frame clause" ([PostgreSQL — window calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)).

---

## 8. `EXCLUDE` and `GROUPS` — Brief

The frame can `EXCLUDE` rows: `EXCLUDE CURRENT ROW` (drop the current row), `EXCLUDE GROUP` (drop the current row and its ordering peers), `EXCLUDE TIES` (drop peers but keep the current row), `EXCLUDE NO OTHERS` (the default — exclude nothing) ([PostgreSQL — window calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)). Useful for "average of everyone *else* in my group": `AVG(x) OVER (PARTITION BY g EXCLUDE CURRENT ROW)`. `EXCLUDE` and `GROUPS` mode are full SQL-standard features in PostgreSQL (11+) and SQLite (3.28+), but **absent from MySQL 8 / MariaDB** — see §9.

---

## 9. Portability Snapshot

Window functions are SQL:2003 and now broadly available; the deviations to watch are the advanced frame features and pre-8 MySQL.

| Feature | PostgreSQL | SQLite | MySQL | MariaDB |
|---|---|---|---|---|
| Window functions (`OVER`, ranking, `LAG`/`LEAD`, frames) | yes (8.4+) | yes (3.25+) | yes (**8.0+ only**) | yes (10.2+) |
| Default frame = `RANGE ... CURRENT ROW` (the §4/§5 traps) | yes | yes | yes | yes |
| `GROUPS` frame mode + `EXCLUDE` clause | yes (11+) | yes (3.28+) | **no** | **no** |
| `QUALIFY` shortcut for filtering a window | no | no | no | no |

Pre-8.0 MySQL has **no** window functions at all — code must fall back to self-joins or user-variable tricks. The default-frame behavior (§4, §5) is identical across every engine that has windows, so those two traps are universal, not PostgreSQL-specific ([SQLite — window functions](https://www.sqlite.org/windowfunctions.html); [modern-sql.com — ROWS framing](https://modern-sql.com/caniuse/over_rows_between)). For dialect spellings, version gaps, and emulation on old MySQL, route to **`sql-standard-vs-dialect-map`**.

---

## 10. Who Suffers When Windows Are Misused

A window mistake rarely errors — it returns a plausible wrong number, and someone downstream trusts it:

- The **analyst** whose financial report used `LAST_VALUE(price) OVER (... ORDER BY ts)` to show "closing price" and silently got each row's *own* price instead of the period's last (§4). No error, no warning — just a wrong number presented in a board deck. The fix was four words: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.
- The **engineer** whose "running total" jumped in steps because two transactions shared a timestamp and the default `RANGE` lumped the peer group (§5). The ledger looked corrupted; the frame just needed to be `ROWS`.
- The **newcomer** staring at `WHERE row_number() = 1` and a syntax error they can't parse, because nothing told them windows run *after* `WHERE` (§6). The wrap-in-CTE pattern they didn't know is the answer.
- The **leaderboard owner** who used `RANK()` and got two "first place" rows on a tie when the business rule was exactly one (§2).

The explicit frame, the `ROWS` you wrote instead of accepting the `RANGE` default, and the CTE wrapper are gifts to whoever reads the report under deadline.

---

## 11. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics, undefined order without `ORDER BY`, and three-valued logic that windows still obey.
- `sql-gaps-and-islands` — the consecutive-runs / islands recipe built on `ROW_NUMBER` and `LAG`.
- `sql-match-recognize` — `MATCH_RECOGNIZE` regex-over-rows when window logic isn't enough.
- `sql-aggregation-and-grouping` — `GROUP BY`/`HAVING`, `FILTER`, and ordered-set aggregates (the row-collapsing counterpart to §1).
- `sql-lateral-and-correlated-derived` — the `LATERAL` top-N-per-group alternative to the §6 wrap pattern.
- `sql-pagination-and-keyset` — keyset paging instead of `ROW_NUMBER`-based offset paging.
- `sql-standard-vs-dialect-map` — version gaps (MySQL 8+), `GROUPS`/`EXCLUDE` availability, and `QUALIFY`.

---

## 12. Reference Files

High-frequency window-function anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
