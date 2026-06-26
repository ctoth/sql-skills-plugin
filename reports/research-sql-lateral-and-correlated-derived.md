# Research: sql-lateral-and-correlated-derived

Research backing the skill `sql-lateral-and-correlated-derived` (skill #9 in `reports/skill-plan-sql.md`).
Each entry: source URL, section, verbatim quote, and why it matters for the skill.

Accessed: 2026-06-26.

---

## Source note: modern-sql.com/feature/lateral is DEAD (HTTP 404)

The spec named `https://modern-sql.com/feature/lateral` as a primary source. As of 2026-06-26 that URL
returns **HTTP 404 Not Found** (verified twice via WebFetch, and a site-scoped WebSearch surfaced no
LATERAL feature/use-case page). modern-sql.com still publishes LATERAL-adjacent material (e.g. its
ORDER-BY history mentions "the need for ORDER BY in subqueries emerged ... in lateral joins"), but no
canonical feature page exists to cite. I substituted an equivalent standard-SQL primary treatment of the
top-N-per-group pattern (Wikibooks, which uses the exact `JOIN LATERAL ... FETCH FIRST n ROWS` recipe and
the `ROW_NUMBER()` alternative) plus the MySQL reference manual for the "derived table cannot normally
refer to preceding tables" framing and the version-portability claims. The PostgreSQL docs carry the
authoritative semantics. This substitution is recorded in `sources.yaml`.

---

## 1. PostgreSQL — Table Expressions: LATERAL Subqueries

URL: https://www.postgresql.org/docs/current/queries-table-expressions.html (section "7.2.1.5. LATERAL Subqueries")

**(a) What LATERAL allows — the core distinction from a plain derived table**
> "This allows them to reference columns provided by preceding `FROM` items."
> "(Without `LATERAL`, each subquery is evaluated independently and so cannot cross-reference any other `FROM` item.)"

Why it matters: This is THE load-bearing fact of the skill — a normal subquery/derived table in `FROM` is
evaluated in isolation and cannot see sibling `FROM` items; `LATERAL` is exactly the keyword that lifts that
restriction. Drives the §1 WRONG/RIGHT pair.

**(b) Per-row evaluation semantics**
> "When a `FROM` item contains `LATERAL` cross-references, evaluation proceeds as follows: for each row of the `FROM` item providing the cross-referenced column(s), or set of rows of multiple `FROM` items providing the columns, the `LATERAL` item is evaluated using that row or row set's values of the columns."

Why it matters: Explains the execution model — the lateral body runs once per outer row, like a correlated
subquery but able to return multiple rows and multiple columns. Justifies "one pass, multiple columns"
versus N scalar subqueries.

**(c) Implicit LATERAL for table (set-returning) functions**
> "Table functions appearing in `FROM` can also be preceded by the key word `LATERAL`, but for functions the key word is optional; the function's arguments can contain references to columns provided by preceding `FROM` items in any case."

Why it matters: Backs the "set-returning-function expansion" section — function FROM-items get implicit
lateral scoping, so `SELECT ... FROM t, unnest(t.arr)` already cross-references `t` without writing LATERAL.

**(d) The canonical LEFT JOIN LATERAL ... ON true pattern**
> "It is often particularly handy to `LEFT JOIN` to a `LATERAL` subquery, so that source rows will appear in the result even if the `LATERAL` subquery produces no rows for them."
> Example: `SELECT m.name FROM manufacturers m LEFT JOIN LATERAL get_product_names(m.id) pname ON true WHERE pname IS NULL;`

Why it matters: `ON true` is the idiom for the top-N recipe — the join predicate is moved inside the lateral
body, so the ON clause is a trivial truth. `LEFT JOIN LATERAL` preserves outer rows that have zero matches
(the difference between CROSS APPLY and OUTER APPLY).

---

## 2. PostgreSQL — SELECT reference: LATERAL keyword placement

URL: https://www.postgresql.org/docs/current/sql-select.html (FROM clause / LATERAL)

**(a) Placement before a sub-SELECT**
> "The `LATERAL` key word can precede a sub-`SELECT` `FROM` item. This allows the sub-`SELECT` to refer to columns of `FROM` items that appear before it in the `FROM` list. (Without `LATERAL`, each sub-`SELECT` is evaluated independently and so cannot cross-reference any other `FROM` item.)"

Why it matters: Confirms keyword goes immediately before the subquery, and the "appear before it" left-to-right
ordering rule — the lateral body can only see FROM items to its left.

**(b) LATERAL before a function call is a noise word**
> "`LATERAL` can also precede a function-call `FROM` item, but in this case it is a noise word, because the function expression can refer to earlier `FROM` items in any case."

Why it matters: Reinforces 1(c) from the SELECT-reference side — for functions LATERAL is optional/implied.

**(c) Left-of-JOIN reference rule and the RIGHT JOIN prohibition**
> "A `LATERAL` item can appear at top level in the `FROM` list, or within a `JOIN` tree. In the latter case it can also refer to any items that are on the left-hand side of a `JOIN` that it is on the right-hand side of."
> "The column source table(s) must be `INNER` or `LEFT` joined to the `LATERAL` item ... although a construct such as `X RIGHT JOIN LATERAL Y` is syntactically valid, it is not actually allowed for `Y` to reference `X`."

Why it matters: Explains why the recipe uses `INNER`/`LEFT JOIN LATERAL` (never RIGHT) — the source rows must
be to the left so there's a well-defined per-row set to evaluate against.

---

## 3. MySQL — Lateral Derived Tables

URL: https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html

**(a) Derived tables normally cannot see siblings; LATERAL lifts that**
> "A derived table cannot normally refer to (depend on) columns of preceding tables in the same `FROM` clause. As of MySQL 8.0.14, a derived table may be defined as a lateral derived table to specify that such references are permitted."

**(b) It's illegal in plain SQL-92 without LATERAL**
> "However, the query is illegal in SQL-92 because derived tables cannot depend on other tables in the same `FROM` clause."
> "The derived table is dependent on the `salesperson` table and thus fails without `LATERAL`"

Why it matters: A second-engine confirmation of the central distinction, plus the precise version
portability fact (MySQL 8.0.14+) for the portability block.

---

## 4. Wikibooks — Retrieve Top N Rows per Group (standard-SQL substitute for modern-sql)

URL: https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group

**(a) Problem definition**
> "Often there is the requirement to access the first or top n rows for every unique value of a given column: the cheapest product (= first row) within a product group ... the rows with the highest version number per entity within a historic table, the newest 10 log entries per user"

**(b) The JOIN LATERAL + FETCH FIRST recipe (verbatim example)**
> ```sql
> SELECT p3.*
> FROM   product p1
> JOIN LATERAL (SELECT *
>               FROM   product p2
>               WHERE  p1.product_group = p2.product_group
>               ORDER BY p2.prize DESC
>               FETCH FIRST 1 ROW ONLY
>              ) p3 ON p1.id = p3.id
> ```
> "Every row of p1 is joined with the first row (FETCH FIRST) of p2 within the same group (p1.product_group = p2.product_group)."

**(c) The window-function alternative**
> ```sql
> SELECT tmp.*
> FROM (SELECT product.*,
>              row_number() OVER (PARTITION BY product_group ORDER BY prize DESC) AS rownumber_per_group
>       FROM product) tmp
> WHERE rownumber_per_group < 2
> ```
> window functions "offer a very flexible and rich set of features"

Why it matters: Primary standard-SQL treatment of both the LATERAL recipe and the ROW_NUMBER alternative —
backs the top-N-per-group section and the LATERAL-vs-window tradeoff (route to `sql-window-functions`).

---

## 5. SQL Boy — The Power of SQL LATERAL Joins (and CROSS APPLY)

URL: https://www.hisqlboy.com/blog/understanding-sql-lateral-joins

> "LATERAL keyword (in PostgreSQL/MySQL) or CROSS APPLY (in SQL Server/Oracle)"
> "CROSS APPLY and OUTER APPLY, the closely related SQL Server syntax"

Why it matters: Citable support for the dialect-spelling mapping — SQL Server (and Oracle, which also has
both) spell `JOIN LATERAL` as `CROSS APPLY` and `LEFT JOIN LATERAL` as `OUTER APPLY`. Oracle 12c+ supports
both the standard `LATERAL`/`CROSS APPLY`/`OUTER APPLY` spellings. Routes to `sql-standard-vs-dialect-map`.

---

## Synthesis for the skill

- **Central claim**: a plain derived table/subquery in FROM is "evaluated independently and so cannot
  cross-reference any other FROM item"; `LATERAL` is the keyword that grants left-to-right visibility,
  evaluated once per outer row (PostgreSQL 7.2.1.5; MySQL 8.0.14).
- **Top-N-per-group**: `LEFT JOIN LATERAL (SELECT ... WHERE child.fk = parent.id ORDER BY ... FETCH FIRST n ROWS ONLY) ON true`.
  `LEFT` + `ON true` keeps parents with zero children; the correlation predicate moves inside the body.
- **Set-returning functions**: implicit lateral — function FROM-items can reference preceding columns with
  no keyword.
- **Beats N correlated scalar subqueries**: a single lateral body returns multiple columns in one pass;
  N scalar subqueries in the SELECT list re-scan the child per column. But for a top-N report across the
  WHOLE table, a window function (`ROW_NUMBER() OVER (PARTITION BY ...)`) is usually clearer/cheaper —
  LATERAL shines when you only need the top-N for a filtered/small set of outer rows.
- **APPLY**: `CROSS APPLY` = `[CROSS] JOIN LATERAL`; `OUTER APPLY` = `LEFT JOIN LATERAL` (SQL Server, Oracle 12c+).
- **Portability**: PostgreSQL ✓, MySQL 8.0.14+ ✓, SQL Server CROSS/OUTER APPLY, Oracle 12c+ (LATERAL + APPLY),
  SQLite ✗ (no LATERAL/APPLY — use a correlated subquery or window function).
