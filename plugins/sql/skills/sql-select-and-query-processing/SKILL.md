---
name: sql-select-and-query-processing
description: Guides the logical clause-evaluation order of a SELECT — FROM → WHERE → GROUP BY/aggregates → HAVING → SELECT-list/window → DISTINCT → UNION → ORDER BY → OFFSET/FETCH — and why that order, not the written order, decides what each clause can reference. A SELECT-list alias is computed late, so it is illegal in WHERE/HAVING (write out the expression) but legal in ORDER BY; aggregate conditions belong in HAVING, not WHERE; DISTINCT dedups the WHOLE row, not one column; and SELECT * is the "Implicit Columns" antipattern that breaks on schema change. Auto-invokes when writing or editing SELECT statements, WHERE/GROUP BY/HAVING/DISTINCT/ORDER BY clauses, column aliases referenced in another clause, ORDER BY ordinals or NULLS FIRST/LAST, or on "column alias does not exist" / "must appear in the GROUP BY" / "why is this column not allowed here" errors. Builds on the sql-relational-and-null-discipline foundation.
allowed-tools: Read, Glob, Grep
---

# SQL SELECT and Query Processing

> "The general processing of SELECT is as follows: ... 3. If the WHERE clause is specified, all rows that do not satisfy the condition are eliminated. ... 4. If the GROUP BY clause is specified ... the output is combined into groups ... If the HAVING clause is present, it eliminates groups ... 5. The actual output rows are computed using the SELECT output expressions ... 6. SELECT DISTINCT eliminates duplicate rows ... 8. If the ORDER BY clause is specified, the returned rows are sorted ... 9. If the LIMIT (or FETCH FIRST) or OFFSET clause is specified, the SELECT statement only returns a subset of the result rows."
> — [PostgreSQL — SELECT (Description)](https://www.postgresql.org/docs/current/sql-select.html)

> "An output column's name can be used to refer to the column's value in ORDER BY and GROUP BY clauses, but not in the WHERE or HAVING clauses; there you must write out the expression instead."
> — [PostgreSQL — SELECT (SELECT List)](https://www.postgresql.org/docs/current/sql-select.html)

A SELECT is **written** `SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY …`, but it is **evaluated** in a different order, and almost every "column does not exist here" / "must appear in the GROUP BY clause" error is the gap between the two. This skill builds directly on **`sql-relational-and-null-discipline`** — that foundation owns the rule that a result is an unordered *set* (no order without `ORDER BY`) and that `WHERE`/`HAVING` keep only rows evaluating to TRUE. Here we add the second half of the mental model: the **logical clause-evaluation order**, and what it makes legal where.

---

## 1. The Logical Evaluation Order (and why it is not the written order)

PostgreSQL documents "the general processing of SELECT" as a numbered pipeline where the output of each phase is the input of the next ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). Reordered to the canonical teaching sequence:

| # | Phase | What it does |
|---|---|---|
| 1 | `FROM` / `JOIN` | "All elements in the FROM list are computed" — produce the working row set ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 2 | `WHERE` | "all rows that do not satisfy the condition are eliminated" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 3 | `GROUP BY` + aggregates | "the output is combined into groups ... and the results of aggregate functions are computed" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 4 | `HAVING` | "it eliminates groups that do not satisfy the given condition" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 5 | `SELECT` list + window functions | "The actual output rows are computed using the SELECT output expressions" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 6 | `DISTINCT` | "SELECT DISTINCT eliminates duplicate rows from the result" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 7 | `UNION`/`INTERSECT`/`EXCEPT` | combine with other SELECTs ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 8 | `ORDER BY` | "the returned rows are sorted in the specified order" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |
| 9 | `OFFSET` / `LIMIT` / `FETCH FIRST` | "only returns a subset of the result rows" ([PG](https://www.postgresql.org/docs/current/sql-select.html)) |

SQLite documents the same shape — it "describe[s] the way the data returned by a SELECT statement is determined as a series of steps": FROM-clause input, then WHERE filtering, then result-row/GROUP BY generation, then DISTINCT, then ORDER BY, then LIMIT ([SQLite — SELECT](https://www.sqlite.org/lang_select.html)). The order is not a PostgreSQL quirk; it is the SQL processing model.

**The single load-bearing consequence:** the `SELECT` list is computed at step 5, *after* `WHERE` (step 2) and the grouping/`HAVING` phase (steps 3–4). So anything you name in the SELECT list — an alias, a window function, an aggregate output — does not yet exist when `WHERE` runs. That is why the rules in §2–§4 are not arbitrary syntax restrictions; they fall straight out of the pipeline.

---

## 2. Alias Visibility — Illegal in WHERE/HAVING, Legal in ORDER BY

Because the SELECT list is computed at step 5, "an output column's name can be used to refer to the column's value in ORDER BY and GROUP BY clauses, but not in the WHERE or HAVING clauses; there you must write out the expression instead" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). `ORDER BY` (step 8) runs after the SELECT list exists, so it alone can name it: "A sort_expression can also be the column label or number of an output column" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)).

```sql
-- WRONG — alias "net" does not exist yet at WHERE (step 2 < step 5)
--   PostgreSQL: ERROR: column "net" does not exist
SELECT price - discount AS net
FROM line_items
WHERE net > 0;

-- RIGHT — repeat the expression in WHERE; the alias is only for output/ORDER BY
SELECT price - discount AS net
FROM line_items
WHERE price - discount > 0
ORDER BY net;          -- legal: ORDER BY (step 8) runs after the SELECT list
```

If repeating the expression is painful, wrap it in a subquery or CTE so the alias becomes a real column of an inner result the outer `WHERE` can reference — that, not alias-in-WHERE, is the portable fix.

**Portability nuance.** The *reliably portable* rule is: aliases are visible only in `ORDER BY`. PostgreSQL additionally permits aliases in `GROUP BY` as a documented extension — "Although query output columns are nominally computed in the next step, they can also be referenced (by name or ordinal number) in the GROUP BY clause" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)) — and SQLite is more permissive still about resolving a result-column alias early ([SQLite — SELECT](https://www.sqlite.org/lang_select.html)). Do not rely on alias-in-`GROUP BY`/`HAVING` for portable code; repeat the expression. Route dialect specifics to `sql-standard-vs-dialect-map`.

One more ORDER BY precision: the alias must stand alone. "an output column name ... cannot be used in an expression — for example, this is not correct: `... ORDER BY sum + c`" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)).

---

## 3. WHERE vs HAVING — Row Filter vs Group Filter

`WHERE` runs at step 2, before grouping; `HAVING` runs at step 4, after aggregates are computed. So **`WHERE` cannot see an aggregate, and `HAVING` is the only place a condition on an aggregate is legal**: HAVING "eliminates groups that do not satisfy the given condition" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- WRONG — aggregate in WHERE; aggregates are not computed until step 3
--   PostgreSQL: ERROR: aggregate functions are not allowed in WHERE
SELECT customer_id, COUNT(*) AS orders
FROM orders
WHERE COUNT(*) > 5
GROUP BY customer_id;

-- RIGHT — pre-aggregation row filters go in WHERE; aggregate filters go in HAVING
SELECT customer_id, COUNT(*) AS orders
FROM orders
WHERE status = 'paid'          -- filter rows BEFORE grouping (step 2)
GROUP BY customer_id
HAVING COUNT(*) > 5;           -- filter groups AFTER aggregating (step 4)
```

Rule of thumb: a condition on a *raw column* that should shrink the input belongs in `WHERE` (it also runs earlier, so it is usually cheaper); a condition on an *aggregate* (`COUNT`, `SUM`, `AVG` …) can only live in `HAVING`. Aggregate semantics, the GROUP BY functional-dependency rules, and bare-column handling are owned by `sql-aggregation-and-grouping`; this skill owns only the *placement* that the evaluation order dictates.

---

## 4. ORDER BY Does Not Affect Grouping or DISTINCT

`ORDER BY` is step 8 — the last thing before paging. It changes only the order rows are *returned*, never which rows or groups are produced. Sorting before a `GROUP BY`, or expecting `ORDER BY` to influence `DISTINCT`, is a category error: those phases (steps 3 and 6) have already happened. As the foundation skill states, the result is a set until `ORDER BY` imposes an order on the way out.

---

## 5. DISTINCT Dedups the Whole Row, Not One Column

A frequent misconception is that `SELECT DISTINCT a, b` deduplicates on `a` and "picks" a `b`. It does not. "If SELECT DISTINCT is specified, all duplicate rows are removed from the result set (one row is kept from each group of duplicates)" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)) — distinctness is evaluated across the **entire output row**, every selected column together.

```sql
-- MISCONCEPTION — this does NOT return one row per customer.
-- It returns every DISTINCT (customer_id, order_date) PAIR.
SELECT DISTINCT customer_id, order_date FROM orders;
```

`DISTINCT` is also opt-in: "SELECT ALL specifies the opposite: all rows are kept; that is the default" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html); same default in [SQLite](https://www.sqlite.org/lang_select.html)).

If you genuinely want "one row per `customer_id`," you want PostgreSQL's `DISTINCT ON (customer_id)`, which "keeps only the first row of each set of rows where the given expressions evaluate to equal" — but note "the 'first row' of each set is unpredictable unless ORDER BY is used" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). **`DISTINCT ON` is a PostgreSQL extension, not standard SQL** — the portable equivalent is a window-function `ROW_NUMBER()` filter. Route the spelling and portability to `sql-standard-vs-dialect-map` and the technique to `sql-window-functions`.

---

## 6. SELECT * vs Explicit Columns — the "Implicit Columns" Antipattern

`SELECT *` is convenient and, in production code, fragile. `*` "is substituted" with "all columns in the input data" at result-generation time ([SQLite — SELECT](https://www.sqlite.org/lang_select.html)) — meaning what `*` returns is resolved against *whatever the schema is right now*. Bill Karwin catalogs this as the **"Implicit Columns"** antipattern ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). Concrete failure modes:

- **Schema-change fragility / ordinal breakage.** Code that reads results positionally (`row[2]`) silently reads the wrong column the moment someone adds or reorders a column upstream — no error, just wrong data.
- **Over-fetch.** `*` drags every column, including large `TEXT`/`BLOB`/`JSON` payloads you never use, inflating I/O and network.
- **Broken `INSERT INTO t SELECT *`.** A `*` feeding an `INSERT` ties the insert to column count/order and breaks when either changes.
- **Lost index-only scans.** Listing only the columns you need lets the planner satisfy the query from an index; `*` usually forces a heap fetch (see `sql-indexing-and-sargability`).

```sql
-- WRONG — fragile: meaning depends on current column set/order, over-fetches
SELECT * FROM users WHERE id = 42;

-- RIGHT — explicit, stable under schema change, leaner
SELECT id, email, created_at FROM users WHERE id = 42;
```

`SELECT *` is fine for ad-hoc exploration and for `EXISTS (SELECT 1 …)` probes (the list is irrelevant there). It is a liability in stored queries, views, and application code.

---

## 7. ORDER BY Mechanics — Ordinals, Expressions, NULLS FIRST/LAST

`ORDER BY` terms can be (a) an output-column **name**, (b) an output-column **ordinal**, or (c) an "arbitrary expression formed from input-column values" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
SELECT a + b AS sum, c FROM t ORDER BY sum;   -- by output name (alias)
SELECT a + b AS sum, c FROM t ORDER BY 1;     -- by ordinal: the 1st output column
```

Both forms above "sort by the first output column" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)). Caveats:

- **Ordinals are brittle.** `ORDER BY 1` re-binds when you reorder the SELECT list — the same hazard as positional reads. Prefer a name for anything maintained.
- **Name-vs-input ambiguity resolves to the output column.** "if an ORDER BY item is a simple name that could match either an output column name or a column from the table expression[,] [t]he output column is used" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)).
- **Defaults:** "ASC order is the default ... 'smaller' is defined in terms of the < operator"; and "By default, null values sort as if larger than any non-null value; that is, NULLS FIRST is the default for DESC order, and NULLS LAST otherwise" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)). NULL sort position differs by engine — the foundation skill (`sql-relational-and-null-discipline`, §9) owns that trap; write `NULLS FIRST|LAST` explicitly for portable, deterministic order.

---

## 8. Portability Snapshot

The numbered evaluation order is shared across engines; alias-visibility and DISTINCT spelling are where they diverge.

| Behavior | PostgreSQL | SQLite | MySQL |
|---|---|---|---|
| FROM→WHERE→GROUP BY→HAVING→SELECT→DISTINCT→ORDER BY→LIMIT order | ✓ strict | ✓ documented | ✓ |
| Alias usable in `ORDER BY` | ✓ | ✓ | ✓ |
| Alias usable in `WHERE`/`HAVING` | ✗ (write the expression) | ~ (resolves aliases more loosely) | `HAVING` accepts aliases (extension); `WHERE` does not |
| Alias usable in `GROUP BY` | ✓ (extension) | ~ | ✓ (extension) |
| `DISTINCT ON (...)` | ✓ (non-standard) | ✗ (use `GROUP BY`/window) | ✗ (use window) |
| `NULLS FIRST/LAST` in `ORDER BY` | ✓ | ✓ | needs `col IS NULL` trick |

SQLite ✓-but-loose: it "consider[s] [an identifier] an alias for [an output] column" when resolving names ([SQLite — SELECT](https://www.sqlite.org/lang_select.html)), so a query that references an alias in `HAVING`/`GROUP BY` may run on SQLite/MySQL and then fail on PostgreSQL with "column does not exist." Treat **alias-only-in-ORDER-BY** as the portable contract. For every divergent spelling (`DISTINCT ON`, `NULLS FIRST/LAST`, alias scope), route to **`sql-standard-vs-dialect-map`**.

---

## 9. Who Suffers When the Processing Model Is Ignored

These mistakes usually error loudly — but the error message names the symptom, not the cause, and someone burns time bridging the gap:

- The **developer** staring at `ERROR: column "net" does not exist` on a `WHERE net > 0`, certain the alias is right there two lines up — until they internalize that `WHERE` (step 2) runs before the `SELECT` list (step 5) computes `net` (§2). The fix (repeat the expression, or wrap in a CTE) is invisible without the evaluation order.
- The **analyst** who "deduped" a report with `SELECT DISTINCT customer_id, order_date` and shipped one row per *order*, not per *customer* — because `DISTINCT` is whole-row (§5). The number was wrong in a meeting, with no error to warn them.
- The **on-call engineer** paged at 3 a.m. because a nightly `INSERT INTO archive SELECT * FROM orders` started failing — or worse, *silently mismatched columns* — after a teammate added a column to `orders` (§6). The `SELECT *` that worked for two years was a latent landmine.
- The **teammate porting MySQL to PostgreSQL** whose `HAVING total > 100` (alias `total`) ran fine for years and now throws on PostgreSQL, because MySQL/SQLite resolved the alias and PostgreSQL does not (§8).

Writing the expression out, listing columns explicitly, and putting aggregate filters in `HAVING` are small acts of empathy for whoever runs this query under load on a different engine.

---

## 10. Routing to Related Skills

- **`sql-relational-and-null-discipline`** — the foundation: set semantics (no order without `ORDER BY`), three-valued logic, and NULL sort position (§4, §7). Every claim here assumes it.
- `sql-joins` — `FROM`/`JOIN` mechanics (step 1) and `ON` vs `WHERE` placement.
- `sql-aggregation-and-grouping` — GROUP BY functional-dependency rules, bare-column handling, and aggregate NULL semantics behind §3.
- `sql-window-functions` — window functions evaluate at the SELECT-list phase (step 5), so they too are illegal in `WHERE`; the portable `ROW_NUMBER()` replacement for `DISTINCT ON` (§5).
- `sql-pagination-and-keyset` — `OFFSET`/`LIMIT`/`FETCH FIRST` (step 9) and `VALUES`/row-value constructors.
- `sql-set-operations` — `UNION`/`INTERSECT`/`EXCEPT` (step 7) and their single trailing `ORDER BY`.
- `sql-standard-vs-dialect-map` — `DISTINCT ON`, `NULLS FIRST/LAST`, MySQL `LIMIT`, and per-engine alias scope (§8).

---

## 11. Reference Files

High-frequency clause-order and SELECT mistakes in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
