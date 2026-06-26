---
name: sql-aggregation-and-grouping
description: Guides aggregation correctness — every column in the SELECT list of a grouped query must be either named in GROUP BY or wrapped in an aggregate (the functional-dependency rule), because a non-grouped, non-aggregated column has no single value per group. Bans the "ambiguous groups" antipattern that errors on standard-conformant engines but silently returns an arbitrary row's value on MySQL with ONLY_FULL_GROUP_BY disabled. Teaches WHERE (filters rows before grouping, no aggregates) vs HAVING (filters groups after, aggregates allowed), the standard SQL:2003 `FILTER (WHERE ...)` clause for conditional aggregation instead of fragile CASE-inside-aggregate, COUNT(*)/COUNT(col)/COUNT(DISTINCT) and aggregate NULL handling, GROUPING SETS/ROLLUP/CUBE plus GROUPING() instead of hand-UNIONed subtotal levels, and ordered-set aggregates (LISTAGG/ARRAY_AGG/percentile, WITHIN GROUP). Auto-invokes when writing or editing GROUP BY/HAVING, aggregate functions (SUM/COUNT/AVG/MIN/MAX), FILTER, LISTAGG/STRING_AGG/GROUP_CONCAT/ARRAY_AGG, ROLLUP/CUBE/GROUPING SETS, percentile/WITHIN GROUP, or any multi-level subtotal report, and on "only_full_group_by" / "must appear in the GROUP BY clause" / "subtotal" / "rollup" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Aggregation and Grouping

> "If `FILTER` is specified, then only the input rows for which the _filter_clause_ evaluates to true are fed to the aggregate function; other rows are discarded."
> — [PostgreSQL — Aggregate Expressions §4.2.7](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)

> "A clause of the form `ROLLUP ( e1, e2, e3, ... )` represents the given list of expressions and all prefixes of the list including the empty list ... This is commonly used for analysis over hierarchical data; e.g., total salary by department, division, and company-wide total."
> — [PostgreSQL — GROUPING SETS, CUBE, and ROLLUP §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)

`GROUP BY` collapses many rows into one row per group, and every value the query emits for that group must be *well-defined for the whole group* — either a grouping key or the result of an aggregate that folds the group's rows into one value. That single idea drives every rule below. This skill builds on the foundation **`sql-relational-and-null-discipline`**: aggregates *skip* NULL rather than propagate it (`COUNT(col)` undercounts, `SUM`/`AVG` ignore NULLs, `SUM` of no rows is NULL), and the deep NULL theory lives there — this skill assumes it and routes back.

---

## 1. The Functional-Dependency Rule — and the "Ambiguous Groups" Antipattern

After `GROUP BY`, the result has **one row per group**. So every expression in the `SELECT` list (and `ORDER BY`, and `HAVING`) must produce a single value per group: it must be either a column listed in `GROUP BY` (or functionally dependent on one — e.g. determined by a grouped primary key) **or** wrapped in an aggregate. A bare non-grouped, non-aggregated column has *no single value* for a group that spans several distinct values of it.

```sql
-- WRONG — name is neither grouped nor aggregated; which name, of the many in this dept?
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;

-- RIGHT — aggregate the ambiguous column (or add it to GROUP BY if that's the intent)
SELECT department, MAX(name) AS sample_name, COUNT(*)
FROM employees
GROUP BY department;

-- RIGHT — group finer if you actually want a row per (department, name)
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department, name;
```

Standard-conformant engines (PostgreSQL, and MySQL with the default `ONLY_FULL_GROUP_BY` SQL mode) **reject** the WRONG form with an error like *"column must appear in the GROUP BY clause or be used in an aggregate function."* The danger is **MySQL with `ONLY_FULL_GROUP_BY` disabled** (legacy configs, some hosts): it runs the WRONG query without error and returns an **arbitrary** value for `name` from *some* row in the group — the value can change between runs, versions, or plans. That silent, plausible-but-wrong answer is the "ambiguous groups" antipattern. Write to the standard rule and you are correct everywhere.

---

## 2. WHERE vs HAVING — Filter Rows Before, Filter Groups After

Two filters operate at different stages, and choosing the wrong one is either an error or a correctness bug:

- **`WHERE`** filters individual rows **before** grouping. It runs before any group exists, so it **cannot reference an aggregate** — `WHERE COUNT(*) > 5` is illegal.
- **`HAVING`** filters whole groups **after** aggregation. It is the only place an aggregate condition belongs.

```sql
-- WRONG — aggregates don't exist yet at WHERE time
SELECT customer_id, SUM(amount)
FROM orders
WHERE SUM(amount) > 1000      -- error: aggregate not allowed here
GROUP BY customer_id;

-- RIGHT — restrict rows in WHERE, restrict groups in HAVING
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE status = 'paid'         -- per-row filter, runs first, shrinks the input
GROUP BY customer_id
HAVING SUM(amount) > 1000;    -- per-group filter, runs after aggregation
```

Push every condition that does **not** depend on an aggregate into `WHERE` (it filters before grouping, so it is cheaper and clearer); reserve `HAVING` for genuine aggregate conditions. The full clause-evaluation order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY) is owned by **`sql-select-and-query-processing`**; this skill owns only the row-vs-group distinction.

---

## 3. Conditional Aggregation: `FILTER (WHERE ...)`, Not Fragile CASE

To aggregate only a subset of a group's rows, the standard tool is the `FILTER` clause: "If `FILTER` is specified, then only the input rows for which the _filter_clause_ evaluates to true are fed to the aggregate function; other rows are discarded" ([PostgreSQL §4.2.7](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)). It is not a Postgres extension — "SQL:2003 introduced the `filter` clause as part of the optional feature 'Advanced OLAP operations' (T612)" ([modern-sql.com — FILTER](https://modern-sql.com/feature/filter)).

```sql
-- RIGHT — one pass, several conditional counts, each scoped by its own FILTER
SELECT
  COUNT(*)                              AS total,
  COUNT(*) FILTER (WHERE status = 'paid')    AS paid,
  COUNT(*) FILTER (WHERE status = 'refunded') AS refunded,
  SUM(amount) FILTER (WHERE status = 'paid')  AS paid_revenue
FROM orders
GROUP BY customer_id;
```

Where `FILTER` is unavailable, the portable emulation is a `CASE` inside the aggregate — the two are equivalent: "`SUM(<expression>) FILTER(WHERE <condition>)`" and "`SUM(CASE WHEN <condition> THEN <expression> END)`" ([modern-sql.com — FILTER](https://modern-sql.com/feature/filter)). It works precisely because **aggregates skip NULL** (foundation): the implicit `ELSE NULL` makes non-matching rows invisible to the aggregate.

```sql
-- RIGHT (portable) — CASE emulation; the omitted ELSE yields NULL, which the aggregate skips
SELECT
  COUNT(CASE WHEN status = 'paid' THEN 1 END) AS paid,
  SUM(CASE WHEN status = 'paid' THEN amount END) AS paid_revenue
FROM orders
GROUP BY customer_id;
```

One trap: to *count* matching rows with `CASE`, the THEN value must be non-NULL (e.g. `1`); `COUNT(CASE WHEN cond THEN col END)` only counts rows where `col` is also non-null. `SUM(CASE WHEN cond THEN 1 ELSE 0 END)` is an equivalent counting idiom.

---

## 4. COUNT Variants and Aggregate NULL Handling

The three `COUNT` forms answer three different questions, and they diverge exactly when the column has NULLs:

```sql
SELECT
  COUNT(*)               AS rows_total,      -- "Computes the number of input rows"
  COUNT(phone)           AS rows_with_phone, -- non-null phones only (undercount vs *)
  COUNT(DISTINCT phone)  AS distinct_phones  -- distinct non-null phone values
FROM users;
```

`count(*)` "computes the number of input rows"; `count(expression)` "computes the number of input rows in which the input value is not null" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). The `DISTINCT` form "invokes the aggregate once for each distinct value of the expression ... found in the input rows" ([PostgreSQL §4.2.7](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)). Likewise `sum`, `avg`, `min`, and `max` operate only on "the non-null input values," and "except for `count`, these functions return a null value when no rows are selected. In particular, `sum` of no rows returns null, not zero as one might expect" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). So wrap `COALESCE(SUM(x), 0)` when a query that may match no rows must return `0`, and do **not** "fix" `AVG` by coalescing inputs to 0 (it inflates the denominator). The full treatment of why aggregates skip NULL lives in the foundation skill — this section is the aggregation-side reminder.

---

## 5. Multi-Level Subtotals: `GROUPING SETS` / `ROLLUP` / `CUBE` + `GROUPING()`

To produce subtotals and grand totals in one query, do **not** hand-`UNION ALL` several `GROUP BY` queries at different granularities — those copies drift out of sync as the report evolves, and each rescans the table. Standard SQL computes multiple grouping levels in one pass: "The data selected by the `FROM` and `WHERE` clauses is grouped separately by each specified grouping set, aggregates computed for each group just as for simple `GROUP BY` clauses, and then the results returned" ([PostgreSQL §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)).

```sql
-- WRONG (drift-prone) — three queries hand-stitched; change one level, forget the others
SELECT department, division, SUM(salary) FROM emp GROUP BY department, division
UNION ALL SELECT department, NULL, SUM(salary) FROM emp GROUP BY department
UNION ALL SELECT NULL, NULL, SUM(salary) FROM emp;

-- RIGHT — ROLLUP computes (dept,div), (dept), and the grand total () in one statement
SELECT department, division, SUM(salary)
FROM emp
GROUP BY ROLLUP (department, division);
```

`ROLLUP (e1, e2, ...)` yields "all prefixes of the list including the empty list" — subtotals up a hierarchy plus the grand total ([PostgreSQL §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)). `CUBE (e1, e2, ...)` yields "the given list and all of its possible subsets (i.e., the power set)" — every cross-tab subtotal. `GROUPING SETS ((a,b), (a), ())` lets you list the exact levels you want.

The catch: subtotal and grand-total rows carry **NULL** in the columns they did not group by — "References to the grouping columns or expressions are replaced by null values in result rows for grouping sets in which those columns do not appear" ([PostgreSQL §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)). That NULL is indistinguishable from a real NULL in the data unless you use `GROUPING()`, which "Returns a bit mask indicating which `GROUP BY` expressions are not included in the current grouping set ... each bit is 0 if the corresponding expression is included ... and 1 if it is not included" ([PostgreSQL — Grouping Operations](https://www.postgresql.org/docs/current/functions-aggregate.html)).

```sql
-- RIGHT — label subtotal rows; GROUPING(division)=1 means "this is a department-level subtotal"
SELECT
  department, division, SUM(salary) AS total,
  GROUPING(division) AS is_dept_subtotal
FROM emp
GROUP BY ROLLUP (department, division);
```

---

## 6. Ordered-Set Aggregates: `WITHIN GROUP (ORDER BY ...)`

Some aggregates only make sense relative to an ordering of the group's rows — string concatenation in a chosen order, medians/percentiles, the mode. These are *ordered-set aggregates*: "an _order_by_clause_ is _required_ ... written inside `WITHIN GROUP (...)`" ([PostgreSQL §4.2.7](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)).

```sql
-- Percentile / median (interpolating) — standard ordered-set aggregate
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY income) AS median_income
FROM households;

-- Most frequent value
SELECT mode() WITHIN GROUP (ORDER BY category) AS top_category FROM events;
```

String aggregation is the everyday case. SQL:2016 standardized **`LISTAGG`**: "a new _ordered set function_ that resembles the `group_concat` and `string_agg` functions ... It transforms values from a group of rows into a delimited string," with syntax "`LISTAGG(<expr>, <separator>) WITHIN GROUP(ORDER BY …)`" ([modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)). PostgreSQL's `string_agg` "Concatenates the non-null input values into a string," and `array_agg` "Collects all the input values, including nulls, into an array" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)) — note `array_agg` *keeps* nulls while `string_agg` drops them.

```sql
-- Postgres / SQLite spelling — comma-joined tag list per post, in name order
SELECT post_id, string_agg(tag, ', ' ORDER BY tag) AS tags
FROM post_tags
GROUP BY post_id;

-- Standard SQL:2016 spelling (Oracle, DB2, SQL Server 2017+)
SELECT post_id, LISTAGG(tag, ', ') WITHIN GROUP (ORDER BY tag) AS tags
FROM post_tags
GROUP BY post_id;
```

---

## 7. Portability Snapshot

| Feature | PostgreSQL | SQLite | MySQL | Oracle / SQL Server |
|---|---|---|---|---|
| `FILTER (WHERE ...)` on aggregates | yes | yes | recent only (else CASE) | recent only (else CASE) |
| CASE-inside-aggregate emulation | yes | yes | yes | yes |
| `GROUP BY ROLLUP / CUBE / GROUPING SETS` | yes | no (use UNION ALL) | `WITH ROLLUP` only | yes |
| `GROUPING()` function | yes | no | yes | yes |
| Ordered-set `percentile_cont` / `mode` | yes | no | no (8.0+ window fns) | yes |
| String aggregation spelling | `string_agg` | `group_concat` | `GROUP_CONCAT` | `LISTAGG` |

`FILTER` is reliable on PostgreSQL and SQLite (and recent MySQL/MariaDB/Oracle/SQL Server); for the widest reach use the CASE form, which modern-sql recommends "rather than the non-standard alternatives" ([modern-sql.com — FILTER](https://modern-sql.com/feature/filter)). String aggregation is the most divergent corner: the SQL:2016 standard spelling `LISTAGG` is actually the *least* widely implemented (Oracle/SQL Server), while PostgreSQL uses `string_agg`, and MySQL/SQLite use `GROUP_CONCAT`. Route every concrete dialect spelling — `LISTAGG` vs `STRING_AGG` vs `GROUP_CONCAT`, MySQL's `WITH ROLLUP`, and FILTER availability by version — to **`sql-standard-vs-dialect-map`**.

---

## 8. Who Suffers When Aggregation Is Wrong

Aggregation bugs, like NULL bugs, return a plausible wrong number rather than an error — and someone downstream trusts it:

- The **analyst** reading a per-department report on a MySQL box with `ONLY_FULL_GROUP_BY` off (§1): the `manager_name` column shows *an* arbitrary employee's name per department, not the manager's. No error, no warning — the number next to it (the headcount) is right, so the name looks trustworthy. It changes after the next deploy.
- The **finance team** whose quarterly rollup is wrong because someone hand-`UNION ALL`-ed the department, division, and company totals (§5), then later added a new region to the detail query but forgot one of the three subtotal copies. The grand total no longer equals the sum of its parts, and reconciliation takes a day.
- The **dashboard owner** whose "total revenue" reads NULL instead of `$0` for a new customer with no paid orders yet, because `SUM` of no rows is NULL (§4) and the chart library renders NULL as a gap.
- The **report consumer** who can't tell a `ROLLUP` grand-total row (NULL region) from a genuine "region unknown" data row (§5), because nobody added `GROUPING()` — so the totals row gets double-counted into a filter.

The aggregate written to the functional-dependency rule, the `FILTER`/`CASE` that scopes cleanly, the single `ROLLUP` instead of three drifting `UNION`s, and the `COALESCE(SUM(...), 0)` are all gifts to whoever reads the numbers.

---

## 9. Routing to Related Skills

- **`sql-relational-and-null-discipline`** (foundation) — *why* aggregates skip NULL, `COUNT(*)` vs `COUNT(col)`, `SUM`/`AVG` NULL handling, and `SUM` of no rows being NULL. This skill assumes that and routes the depth there.
- **`sql-window-functions`** — when you want the aggregate *and* the per-row detail in the same result, use `SUM(...) OVER (...)` rather than `GROUP BY`. Aggregate-as-window is owned there.
- **`sql-select-and-query-processing`** — the full clause-evaluation order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY) and why `SELECT` aliases aren't visible in `WHERE`/`GROUP BY` on some engines.
- **`sql-standard-vs-dialect-map`** — `FILTER` availability by engine/version, `LISTAGG` vs `STRING_AGG` vs `GROUP_CONCAT`, and MySQL's `WITH ROLLUP`.

---

## 10. Reference Files

High-frequency aggregation/grouping anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
