---
name: sql-lateral-and-correlated-derived
description: Guides LATERAL — a derived table in FROM that may reference columns of preceding FROM items, which a plain subquery cannot. It is the clean, standard answer to top-N-per-group ("latest 3 orders per customer"), per-row set-returning-function expansion, and pulling several correlated values in one pass instead of N correlated scalar subqueries in the SELECT list. Covers the LEFT JOIN LATERAL (...) ON true recipe with ORDER BY + FETCH FIRST n ROWS, implicit LATERAL for table functions, the left-to-right visibility rule, and the SQL Server/Oracle CROSS APPLY / OUTER APPLY spelling of (CROSS/LEFT) JOIN LATERAL. Teaches when a window function top-N is the better tool instead. Auto-invokes when writing or editing LATERAL / CROSS APPLY / OUTER APPLY, top-N-per-group or "newest N per group" queries, per-row table-function expansion (unnest/json_table/string functions in FROM), or a query carrying one correlated scalar subquery per output column. Builds on the foundation sql-relational-and-null-discipline.
allowed-tools: Read, Glob, Grep
---

# SQL LATERAL and Correlated Derived Tables

> "This allows them to reference columns provided by preceding `FROM` items. (Without `LATERAL`, each subquery is evaluated independently and so cannot cross-reference any other `FROM` item.)"
> — [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)

> "A derived table cannot normally refer to (depend on) columns of preceding tables in the same `FROM` clause. As of MySQL 8.0.14, a derived table may be defined as a lateral derived table to specify that such references are permitted."
> — [MySQL — Lateral Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html)

A subquery in the `FROM` clause is, by default, a sealed box: it is computed once, on its own, and knows
nothing about the other tables it sits beside. `LATERAL` is the one keyword that opens that box — it lets a
`FROM` subquery see the columns of the items to its left and re-run for each of their rows. That single
capability is the difference between "join to a fixed table" and "join to a query *parameterized by the
current row*," and it is the reason top-N-per-group, per-row function expansion, and multi-value correlation
have a clean one-pass answer instead of the gymnastics LLMs reach for. This skill builds on the set
semantics and three-valued logic in the foundation `sql-relational-and-null-discipline`.

---

## 1. What LATERAL Is — and Why a Plain Derived Table Can't See Its Siblings

In a normal `FROM` list, each subquery "is evaluated independently and so cannot cross-reference any other
`FROM` item" ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)).
So the obvious-looking query — a derived table that filters by the outer row — is simply illegal: in MySQL's
words it "is illegal in SQL-92 because derived tables cannot depend on other tables in the same `FROM` clause"
([MySQL — Lateral Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html)).

```sql
-- WRONG — the derived table references c.id, but a plain FROM subquery can't see c.
-- Error: "invalid reference to FROM-clause entry" / "Unknown column 'c.id'"
SELECT c.id, recent.total
FROM customers c
JOIN (SELECT SUM(amount) AS total
      FROM orders o
      WHERE o.customer_id = c.id) recent ON true;   -- c.id not in scope here

-- RIGHT — LATERAL puts the preceding FROM items in scope, evaluated per outer row
SELECT c.id, recent.total
FROM customers c
JOIN LATERAL (SELECT SUM(amount) AS total
              FROM orders o
              WHERE o.customer_id = c.id) recent ON true;
```

`LATERAL` goes immediately before the subquery: "The `LATERAL` key word can precede a sub-`SELECT` `FROM`
item. This allows the sub-`SELECT` to refer to columns of `FROM` items that appear before it in the `FROM`
list" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). Visibility is
strictly **left to right** — the lateral body sees only items written before it. Execution is **per row**:
"for each row of the `FROM` item providing the cross-referenced column(s) ... the `LATERAL` item is evaluated
using that row['s] ... values of the columns" ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)).

---

## 2. Top-N-per-Group — the Recipe LLMs Reinvent Badly

"Latest 3 orders per customer," "cheapest product per group," "newest 10 log entries per user" — this is the
top-N-per-group problem: "access the first or top n rows for every unique value of a given column"
([Wikibooks — Top N per Group](https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group)).
A correlated scalar subquery can only return *one* row and *one* column, so it can't do this. `LATERAL` can:
put the per-group `ORDER BY ... FETCH FIRST n ROWS ONLY` inside the lateral body and join with `ON true`.

```sql
-- WRONG — a scalar subquery returns at most one value; "top 3" overflows it
-- ("more than one row returned by a subquery used as an expression")
SELECT c.id,
       (SELECT o.id FROM orders o WHERE o.customer_id = c.id
        ORDER BY o.created_at DESC FETCH FIRST 3 ROWS ONLY) AS recent_orders
FROM customers c;

-- RIGHT — LEFT JOIN LATERAL: the correlation lives in the body, ON true is the join
SELECT c.id, o.id AS order_id, o.created_at, o.amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.created_at, o.amount
    FROM orders o
    WHERE o.customer_id = c.id          -- references the outer row
    ORDER BY o.created_at DESC, o.id DESC   -- total order (foundation §1)
    FETCH FIRST 3 ROWS ONLY
) o ON true;
```

Two design choices matter. Use **`LEFT JOIN LATERAL`** (not inner) so a customer with zero orders still
appears — it "so that source rows will appear in the result even if the `LATERAL` subquery produces no rows
for them" ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)).
And tie-break the `ORDER BY` on a unique column so "latest" is deterministic, exactly as the foundation
demands of any `FETCH FIRST`. (`RIGHT JOIN LATERAL` is disallowed: the source must be to the left, so an
`INNER` or `LEFT` join is required ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).)

---

## 3. Per-Row Set-Returning-Function Expansion (Implicit LATERAL)

When you place a table/set-returning function in `FROM` and want it fed by each outer row — unnest an array
column, expand a JSON document, split a string — that is lateral too, but the keyword is **optional**:
"Table functions appearing in `FROM` can also be preceded by the key word `LATERAL`, but for functions the
key word is optional; the function's arguments can contain references to columns provided by preceding
`FROM` items in any case" ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)).
The SELECT reference calls it "a noise word" for function items ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- RIGHT — implicit lateral: unnest() reads o.tag_ids from the row to its left, no keyword needed
SELECT o.id, t.tag_id
FROM orders o, unnest(o.tag_ids) AS t(tag_id);

-- RIGHT — same thing written explicitly; LEFT JOIN LATERAL keeps orders with empty arrays
SELECT o.id, t.tag_id
FROM orders o
LEFT JOIN LATERAL unnest(o.tag_ids) AS t(tag_id) ON true;
```

The bare-comma form is an implicit `CROSS JOIN LATERAL`, so an order whose array is empty produces **no**
rows; switch to `LEFT JOIN LATERAL ... ON true` when you must keep those parents.

---

## 4. One LATERAL Beats N Correlated Scalar Subqueries

A dashboard that wants several facts about each customer's most recent order often grows one correlated
scalar subquery *per column* — each re-scanning `orders` independently. `LATERAL` evaluates the body once
per outer row and hands back **many columns at once** ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)).

```sql
-- WRONG — three correlated subqueries, three scans of orders, easy to let them drift out of sync
SELECT c.id,
       (SELECT o.id        FROM orders o WHERE o.customer_id = c.id ORDER BY o.created_at DESC FETCH FIRST 1 ROWS ONLY) AS last_id,
       (SELECT o.amount    FROM orders o WHERE o.customer_id = c.id ORDER BY o.created_at DESC FETCH FIRST 1 ROWS ONLY) AS last_amount,
       (SELECT o.created_at FROM orders o WHERE o.customer_id = c.id ORDER BY o.created_at DESC FETCH FIRST 1 ROWS ONLY) AS last_at
FROM customers c;

-- RIGHT — one lateral body, one ORDER BY, all columns from the same chosen row
SELECT c.id, last.id AS last_id, last.amount AS last_amount, last.created_at AS last_at
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.amount, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 1 ROWS ONLY
) last ON true;
```

Beyond fewer scans, the RIGHT version is *correct by construction*: all three values come from the **one**
row the single `ORDER BY` selected. The N-subquery form can silently return fields from different rows if
the ordering isn't perfectly identical and total in every copy. (Correlated subqueries in `WHERE`/`SELECT`
and `EXISTS` are owned by `sql-subqueries-and-exists`.)

---

## 5. LATERAL vs Window-Function Top-N — Pick the Right Tool

Top-N-per-group also has a window-function solution — number the rows per partition and filter
([Wikibooks — Top N per Group](https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group)):

```sql
-- Window-function top-N: one scan of orders, rank within each customer, keep the top 3
SELECT id, customer_id, created_at, amount
FROM (
    SELECT o.*,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC, o.id DESC) AS rn
    FROM orders o
) ranked
WHERE rn <= 3;
```

Rule of thumb:

- **Window function** wins for a report **across the whole table** (top-N for *every* group): it scans
  `orders` once and never touches `customers`.
- **`LATERAL`** wins when you start from a **small or filtered set of outer rows** and want top-N for *those*
  — it probes the (indexed) child per outer row instead of ranking the entire table, and it can also call
  set-returning functions and return shapes a window function can't. It also reads as "for each customer,
  the latest 3," which matches intent.

The depth on window framing, `RANK`/`DENSE_RANK`/`ROW_NUMBER`, and `QUALIFY` lives in `sql-window-functions`;
route there for the windowing side of this tradeoff.

---

## 6. CROSS APPLY / OUTER APPLY — the Same Idea, SQL Server's Spelling

SQL Server (and Oracle 12c+) write lateral joins as `APPLY`: the "LATERAL keyword (in PostgreSQL/MySQL) or
CROSS APPLY (in SQL Server/Oracle)" name the same operation ([SQL Boy — LATERAL Joins and CROSS APPLY](https://www.hisqlboy.com/blog/understanding-sql-lateral-joins)).
The mapping is mechanical:

- `CROSS APPLY (…)`  ≡  `[CROSS] JOIN LATERAL (…) ON true`  — drops outer rows with no match.
- `OUTER APPLY (…)`  ≡  `LEFT JOIN LATERAL (…) ON true`  — keeps outer rows with no match (NULL-extended).

```sql
-- SQL Server: top-3 orders per customer, the APPLY spelling of §2's LEFT JOIN LATERAL
SELECT c.id, o.id AS order_id, o.created_at
FROM customers c
OUTER APPLY (
    SELECT TOP (3) o.id, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
) o;
```

Oracle 12c+ accepts both `LATERAL` and `CROSS`/`OUTER APPLY`. For the full dialect table, route to
`sql-standard-vs-dialect-map`.

---

## 7. Portability

`LATERAL` is standard SQL (SQL:1999), but spelling and availability diverge:

| Engine | Lateral support | Spelling |
|---|---|---|
| PostgreSQL | yes | `[LEFT] JOIN LATERAL (…) ON true`; implicit for function FROM-items |
| MySQL | **8.0.14+** | `[LEFT] JOIN LATERAL (…) ON …` ([MySQL docs](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html)) |
| SQL Server | yes | `CROSS APPLY` / `OUTER APPLY` (no `LATERAL` keyword) |
| Oracle | **12c+** | both `LATERAL (…)` and `CROSS`/`OUTER APPLY` |
| SQLite | **no** | no `LATERAL`/`APPLY` — use a correlated subquery or window function |

SQLite is the one to plan around: with no lateral, top-N-per-group must be done with a window function
(`ROW_NUMBER() OVER (PARTITION BY …)`, §5) or a correlated subquery. Dialect spellings and the SQLite gap are
owned by `sql-standard-vs-dialect-map`.

---

## 8. Who Suffers When LATERAL Is Missed

- The **dashboard** issuing an N+1 storm of correlated scalar subqueries — one per metric, each re-scanning
  `orders` — that a single `LEFT JOIN LATERAL` collapses into one indexed probe per customer returning every
  column from the same chosen row (§4). The page is slow and, worse, the metrics can disagree because they
  came from differently-ordered scans.
- The **developer** who needed "the latest 3 orders per customer," found a scalar subquery returns only one
  value, and hand-rolled a mess of `UNION ALL`, self-joins, or `GROUP BY` with correlated `MAX` — when the
  job was one `LEFT JOIN LATERAL (… ORDER BY … FETCH FIRST 3 ROWS ONLY) ON true` (§2).
- The **maintainer** who finds three subqueries silently reading fields from *different* rows because one
  copy's `ORDER BY` wasn't total, producing a "most recent order" whose id, amount, and date don't belong
  together (§4).
- The **next engineer** porting to SQLite who discovers the `LATERAL` query simply won't parse, with no
  hint that the window-function rewrite was the portable path all along (§7).

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics, three-valued logic, and the rule
  that `FETCH FIRST` needs a **total** `ORDER BY` to be deterministic (every recipe here relies on it).
- `sql-window-functions` — the `ROW_NUMBER()/RANK()` top-N alternative and the full window/`QUALIFY` story;
  the other half of the §5 tradeoff.
- `sql-subqueries-and-exists` — correlated scalar subqueries in `WHERE`/`SELECT`, semi-/anti-joins, and
  `EXISTS`; the patterns `LATERAL` often replaces.
- `sql-joins` — `INNER`/`LEFT`/`RIGHT` semantics and `ON` vs `WHERE`, including why `LATERAL` must be inner-
  or left-joined to its source.
- `sql-standard-vs-dialect-map` — `CROSS`/`OUTER APPLY`, MySQL 8.0.14 / Oracle 12c version gates, and the
  SQLite absence.

---

## 10. Reference Files

High-frequency LATERAL / correlated-derived anti-patterns in LLM-generated SQL, each with wrong/right code
and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
