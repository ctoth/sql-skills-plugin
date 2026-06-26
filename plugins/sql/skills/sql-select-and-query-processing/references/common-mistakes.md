# Common SELECT & Query-Processing Mistakes

Anti-patterns in LLM-generated SQL around the logical clause-evaluation order
(FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT), each with
wrong/right code and a primary-source citation. The skill (`sql-select-and-query-processing`)
states the model; this file holds the high-frequency failure modes. All RIGHT examples are
standard/portable SQL; non-standard spellings are flagged and routed to
`sql-standard-vs-dialect-map`.

---

## 1. Referencing a SELECT-list alias in `WHERE`

**The problem:** The model defines `expr AS alias` and then filters on `alias` in `WHERE`.
The SELECT list is computed at step 5, after `WHERE` (step 2), so the alias does not exist yet.
"An output column's name can be used to refer to the column's value in ORDER BY and GROUP BY
clauses, but not in the WHERE or HAVING clauses; there you must write out the expression
instead" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- WRONG — ERROR: column "net" does not exist
SELECT price - discount AS net FROM line_items WHERE net > 0;

-- RIGHT — repeat the expression in WHERE (alias stays for output/ORDER BY)
SELECT price - discount AS net FROM line_items WHERE price - discount > 0 ORDER BY net;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §2.*

---

## 2. Putting an aggregate condition in `WHERE` instead of `HAVING`

**The problem:** The model filters on `COUNT(*)`/`SUM(...)` in `WHERE`. Aggregates are computed
at step 3; `WHERE` runs at step 2, before they exist. `HAVING` "eliminates groups that do not
satisfy the given condition" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- WRONG — ERROR: aggregate functions are not allowed in WHERE
SELECT customer_id, COUNT(*) AS n FROM orders WHERE COUNT(*) > 5 GROUP BY customer_id;

-- RIGHT — row filters in WHERE, aggregate filters in HAVING
SELECT customer_id, COUNT(*) AS n
FROM orders
WHERE status = 'paid'
GROUP BY customer_id
HAVING COUNT(*) > 5;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §3; aggregate detail owned by `sql-aggregation-and-grouping`.*

---

## 3. Assuming `DISTINCT a, b` deduplicates only the first column

**The problem:** The model writes `SELECT DISTINCT customer_id, order_date` expecting one row
per `customer_id`. `DISTINCT` removes duplicate *whole rows* — "one row is kept from each group
of duplicates" across all selected columns ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- WRONG (assumption) — returns every DISTINCT (customer_id, order_date) PAIR, not one per customer
SELECT DISTINCT customer_id, order_date FROM orders;

-- RIGHT — one row per customer via aggregation...
SELECT customer_id, MAX(order_date) AS last_order FROM orders GROUP BY customer_id;
-- ...or PostgreSQL DISTINCT ON (non-standard; needs ORDER BY) — route to sql-standard-vs-dialect-map
SELECT DISTINCT ON (customer_id) customer_id, order_date
FROM orders ORDER BY customer_id, order_date DESC;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §5; portable window rewrite in `sql-window-functions`.*

---

## 4. `SELECT *` in stored queries / `INSERT ... SELECT *`

**The problem:** The model emits `SELECT *` in application code, views, or an `INSERT` feed.
`*` is substituted with "all columns in the input data" at runtime ([SQLite — SELECT](https://www.sqlite.org/lang_select.html)),
so the result silently changes when a column is added/reordered — Karwin's "Implicit Columns"
antipattern ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG — breaks/over-fetches on schema change; ties INSERT to column order
INSERT INTO archive SELECT * FROM orders;

-- RIGHT — name columns explicitly on both sides
INSERT INTO archive (id, customer_id, total, created_at)
SELECT id, customer_id, total, created_at FROM orders;
```

*Source: [SQLite — SELECT](https://www.sqlite.org/lang_select.html); [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §6.*

---

## 5. Relying on alias visibility in `HAVING`/`GROUP BY` for portable code

**The problem:** The model references a SELECT-list alias in `HAVING` or `GROUP BY`. MySQL and
SQLite resolve it; PostgreSQL rejects it in `HAVING` ("there you must write out the expression
instead", [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). The
query is silently non-portable.

```sql
-- WRONG (non-portable) — alias "total" in HAVING fails on PostgreSQL
SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id HAVING total > 100;

-- RIGHT — repeat the aggregate expression in HAVING
SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id HAVING SUM(amount) > 100;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html); [SQLite — SELECT](https://www.sqlite.org/lang_select.html). Depth: this skill, §2, §8; spellings owned by `sql-standard-vs-dialect-map`.*

---

## 6. Building an expression on an alias inside `ORDER BY`

**The problem:** The model writes `ORDER BY alias + something`. The alias may be a standalone
ORDER BY term but cannot be combined into a larger expression: "an output column name ... cannot
be used in an expression — for example, this is not correct: `... ORDER BY sum + c`"
([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)).

```sql
-- WRONG — cannot build an expression on the output alias
SELECT a + b AS sum, c FROM t ORDER BY sum + c;

-- RIGHT — order by the standalone alias, or write the full input expression
SELECT a + b AS sum, c FROM t ORDER BY (a + b) + c;
```

*Source: [PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html). Depth: this skill, §7.*

---

## 7. Using `ORDER BY` ordinals in maintained queries

**The problem:** The model sorts with `ORDER BY 2`. "The ordinal number refers to the ordinal
(left-to-right) position of the output column" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)) —
so reordering the SELECT list silently changes the sort key, exactly like positional reads.

```sql
-- WRONG (fragile) — re-binds the moment the SELECT list is reordered
SELECT name, created_at FROM users ORDER BY 2;

-- RIGHT — name the column
SELECT name, created_at FROM users ORDER BY created_at;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §7.*

---

## 8. Expecting `ORDER BY` to affect grouping, `DISTINCT`, or "first" selection

**The problem:** The model adds `ORDER BY` expecting it to influence which rows survive
`GROUP BY`/`DISTINCT`, or to make a bare `LIMIT 1` deterministic via an earlier sort. `ORDER BY`
is step 8 — it only orders the output; grouping/dedup already happened at steps 3 and 6. And
`DISTINCT ON`'s "first row of each set is unpredictable unless ORDER BY is used"
([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- WRONG — ORDER BY here does not pick which row each group keeps
SELECT DISTINCT ON (customer_id) customer_id, total FROM orders;   -- "first" is undefined

-- RIGHT — ORDER BY makes the per-group choice deterministic
SELECT DISTINCT ON (customer_id) customer_id, total
FROM orders ORDER BY customer_id, total DESC;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §4, §5; set semantics in `sql-relational-and-null-discipline`.*
