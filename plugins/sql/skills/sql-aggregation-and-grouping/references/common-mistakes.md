# Common SQL Aggregation & Grouping Mistakes

Anti-patterns in LLM-generated SQL around `GROUP BY`, aggregates, conditional aggregation, and
multi-level subtotals, each with wrong/right code and a primary-source citation. The skill
(`sql-aggregation-and-grouping`) states the rules; this file holds the high-frequency failure modes.
All RIGHT examples are standard/portable SQL; non-standard spellings are flagged and routed to
`sql-standard-vs-dialect-map`. NULL-in-aggregate theory is owned by the foundation skill
`sql-relational-and-null-discipline`.

---

## 1. Selecting a non-grouped, non-aggregated column ("ambiguous groups")

**The problem:** The model puts a column in the `SELECT` list that is neither in `GROUP BY` nor inside
an aggregate. Standard engines (and MySQL with default `ONLY_FULL_GROUP_BY`) reject it; MySQL with that
mode disabled silently returns an **arbitrary** row's value for that column — a value that can change
between runs or versions, with no error.

```sql
-- WRONG — name has no single value per department
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;

-- RIGHT — aggregate the column...
SELECT department, MAX(name) AS sample_name, COUNT(*)
FROM employees GROUP BY department;

-- RIGHT — ...or group by it if a row per (department, name) is the intent
SELECT department, name, COUNT(*)
FROM employees GROUP BY department, name;
```

*Source: [PostgreSQL — GROUP BY / functional dependency](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP). Depth: this skill, §1.*

---

## 2. Putting an aggregate condition in `WHERE` instead of `HAVING`

**The problem:** The model writes `WHERE SUM(...) > 1000`. `WHERE` filters rows *before* groups exist,
so aggregates are illegal there — it errors. Aggregate conditions belong in `HAVING`, which filters
groups after aggregation.

```sql
-- WRONG — aggregate not allowed in WHERE
SELECT customer_id, SUM(amount)
FROM orders
WHERE SUM(amount) > 1000
GROUP BY customer_id;

-- RIGHT — per-row filter in WHERE, per-group filter in HAVING
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE status = 'paid'
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

*Source: [PostgreSQL — HAVING / GROUP BY](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP). Depth: this skill, §2; clause order owned by `sql-select-and-query-processing`.*

---

## 3. Misusing `HAVING` for a row-level filter

**The problem:** The model filters non-aggregate conditions in `HAVING`. It often still works, but it
filters *after* grouping rather than before — less clear and forcing the engine to group rows it will
then discard. Non-aggregate conditions belong in `WHERE`.

```sql
-- WRONG (smell) — status is a per-row condition, not a per-group one
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING customer_id > 0 AND SUM(amount) > 100;

-- RIGHT — push the row condition down to WHERE
SELECT customer_id, SUM(amount)
FROM orders
WHERE customer_id > 0
GROUP BY customer_id
HAVING SUM(amount) > 100;
```

*Source: [PostgreSQL — GROUP BY and HAVING](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP). Depth: this skill, §2.*

---

## 4. Hand-rolling conditional aggregation when `FILTER` exists

**The problem:** The model joins or self-unions to compute "count of paid vs refunded," or writes
verbose CASE expressions, when the standard `FILTER (WHERE ...)` clause does it in one pass. "If
`FILTER` is specified, then only the input rows for which the _filter_clause_ evaluates to true are fed
to the aggregate function" ([PostgreSQL §4.2.7](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)).

```sql
-- WRONG (verbose / fragile) — when FILTER is available
SELECT customer_id,
       COUNT(CASE WHEN status='paid' THEN 1 END) AS paid
FROM orders GROUP BY customer_id;

-- RIGHT — standard SQL:2003 FILTER (Postgres/SQLite; recent MySQL/Oracle/SQL Server)
SELECT customer_id,
       COUNT(*) FILTER (WHERE status='paid') AS paid
FROM orders GROUP BY customer_id;
```

*Source: [modern-sql.com — FILTER](https://modern-sql.com/feature/filter); [PostgreSQL §4.2.7](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES). Depth: this skill, §3; dialect availability owned by `sql-standard-vs-dialect-map`.*

---

## 5. `COUNT(CASE ... THEN col END)` counting the wrong thing

**The problem:** The model emulates a conditional count with `COUNT(CASE WHEN cond THEN col END)`, but
`COUNT(expr)` ignores NULLs, so this counts only rows where `cond` is true **and** `col` is non-null —
an undercount. The THEN value must be a non-NULL constant.

```sql
-- WRONG — undercounts when col can be NULL among matching rows
SELECT COUNT(CASE WHEN status='paid' THEN amount END) AS paid_count
FROM orders;

-- RIGHT — count with a guaranteed non-null marker
SELECT COUNT(CASE WHEN status='paid' THEN 1 END) AS paid_count FROM orders;
-- ...or sum 1/0
SELECT SUM(CASE WHEN status='paid' THEN 1 ELSE 0 END) AS paid_count FROM orders;
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html) (count(expression) counts non-null rows). Depth: this skill, §3–4.*

---

## 6. `COUNT(col)` where `COUNT(*)` was meant

**The problem:** The model counts a nullable column expecting a row count. `count(*)` "computes the
number of input rows"; `count(expression)` counts only rows "in which the input value is not null"
([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)).
They silently disagree whenever the column has NULLs.

```sql
-- WRONG — undercounts: rows with NULL phone excluded
SELECT COUNT(phone) AS users FROM users;

-- RIGHT — count rows
SELECT COUNT(*) AS users FROM users;
-- distinct non-null phones, if that was the intent
SELECT COUNT(DISTINCT phone) AS distinct_phones FROM users;
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html). Depth: this skill, §4; full NULL treatment in foundation `sql-relational-and-null-discipline` §7.*

---

## 7. Expecting `SUM` of no rows to be `0`

**The problem:** The model assumes a `SUM` over a group that matches no rows returns `0`. It returns
NULL: "`sum` of no rows returns null, not zero as one might expect" ([PostgreSQL — Aggregate
Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). Downstream code or
charts then mishandle the NULL.

```sql
-- WRONG — returns NULL, not 0, when the customer has no paid orders
SELECT SUM(amount) AS total FROM orders WHERE customer_id = 42 AND status='paid';

-- RIGHT — coalesce the RESULT of the aggregate
SELECT COALESCE(SUM(amount), 0) AS total
FROM orders WHERE customer_id = 42 AND status='paid';
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html). Depth: this skill, §4; foundation §7.*

---

## 8. Hand-`UNION`-ing subtotal levels instead of `ROLLUP`

**The problem:** The model produces detail + subtotal + grand-total rows by `UNION ALL`-ing several
`GROUP BY` queries at different granularities. The copies drift apart as the report changes, and each
rescans the table. `ROLLUP` computes them in one pass: it represents "all prefixes of the list
including the empty list" ([PostgreSQL §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)).

```sql
-- WRONG — three queries to keep in sync; one will drift
SELECT department, division, SUM(salary) FROM emp GROUP BY department, division
UNION ALL SELECT department, NULL, SUM(salary) FROM emp GROUP BY department
UNION ALL SELECT NULL, NULL, SUM(salary) FROM emp;

-- RIGHT — one statement; add GROUPING SETS/CUBE for other levels
SELECT department, division, SUM(salary)
FROM emp GROUP BY ROLLUP (department, division);
```

*Source: [PostgreSQL §7.2.4 — GROUPING SETS/ROLLUP/CUBE](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS). Depth: this skill, §5; MySQL `WITH ROLLUP` spelling owned by `sql-standard-vs-dialect-map`.*

---

## 9. Confusing a subtotal NULL with a data NULL in ROLLUP/CUBE output

**The problem:** In `ROLLUP`/`CUBE` output, subtotal rows carry NULL in the columns they don't group
by — "References to the grouping columns ... are replaced by null values in result rows for grouping
sets in which those columns do not appear" ([PostgreSQL §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)). The model treats that NULL like a real
"unknown" data value, double-counting totals or mislabeling rows. Use `GROUPING()` to tell them apart.

```sql
-- WRONG — can't distinguish "division-unknown" data rows from the dept subtotal row
SELECT department, division, SUM(salary)
FROM emp GROUP BY ROLLUP (department, division);

-- RIGHT — GROUPING(division)=1 marks a subtotal row, not real data
SELECT department, division, SUM(salary),
       GROUPING(division) AS is_subtotal
FROM emp GROUP BY ROLLUP (department, division);
```

*Source: [PostgreSQL §7.2.4](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS); [PostgreSQL — Grouping Operations](https://www.postgresql.org/docs/current/functions-aggregate.html). Depth: this skill, §5.*

---

## 10. String aggregation with no order, or the wrong dialect spelling

**The problem:** Two errors. (a) The model concatenates group values without an `ORDER BY`, so the
delimited string's order is undefined and unstable. (b) It hardcodes one engine's spelling, breaking on
others — the SQL:2016 standard `LISTAGG` is actually the *least* widely supported.

```sql
-- WRONG — undefined order inside the aggregated string
SELECT post_id, string_agg(tag, ', ') FROM post_tags GROUP BY post_id;

-- RIGHT — ordered-set aggregate with an explicit ORDER BY (Postgres/SQLite spelling)
SELECT post_id, string_agg(tag, ', ' ORDER BY tag) AS tags
FROM post_tags GROUP BY post_id;

-- RIGHT — standard SQL:2016 spelling (Oracle, DB2, SQL Server 2017+)
SELECT post_id, LISTAGG(tag, ', ') WITHIN GROUP (ORDER BY tag) AS tags
FROM post_tags GROUP BY post_id;
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html) (string_agg); [modern-sql.com — SQL:2016 LISTAGG](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016). Depth: this skill, §6; full dialect map (`STRING_AGG`/`GROUP_CONCAT`/`LISTAGG`) owned by `sql-standard-vs-dialect-map`.*

---

## 11. Using `array_agg` and expecting NULLs to be dropped

**The problem:** The model assumes all aggregates skip NULL uniformly. They don't: `string_agg`
"Concatenates the non-null input values," but `array_agg` "Collects all the input values, **including
nulls**, into an array" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). A NULL element then appears in the resulting array.

```sql
-- WRONG (surprise) — NULL tags become NULL elements in the array
SELECT post_id, array_agg(tag ORDER BY tag) FROM post_tags GROUP BY post_id;

-- RIGHT — filter the nulls out explicitly when you don't want them
SELECT post_id, array_agg(tag ORDER BY tag) FILTER (WHERE tag IS NOT NULL)
FROM post_tags GROUP BY post_id;
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html). Depth: this skill, §6.*
