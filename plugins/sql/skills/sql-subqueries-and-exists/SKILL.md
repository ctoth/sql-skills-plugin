---
name: sql-subqueries-and-exists
description: >-
  Guides correct subquery use and owns the deep dive the foundation defers — the `NOT IN` + NULL trap, where a single NULL anywhere in the list or subquery collapses `NOT IN` to zero rows because `NOT IN` expands to a chain of `not equal to` comparisons AND'd together and the NULL conjunct is forever UNKNOWN, so `NOT EXISTS` is the safe portable anti-join (EXISTS never returns UNKNOWN). Auto-invokes when writing or editing subqueries, `IN (SELECT ...)`, `NOT IN`, `EXISTS`/`NOT EXISTS`, correlated subqueries, `ANY`/`SOME`/`ALL`, scalar subqueries in SELECT/WHERE, or on "NOT IN returns nothing" / "my orphan/anti-join query is empty" / "subquery returned more than one row" symptoms.
allowed-tools: Read, Glob, Grep
---

# SQL Subqueries and EXISTS

> "Note that if the left-hand expression yields null, or if there are no equal right-hand values and at least one right-hand row yields null, the result of the `NOT IN` construct will be null, not true."
> — [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)

> "When using a subquery, consider using not exists instead of not in ... EXISTS never returns unknown."
> — [modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic)

A subquery is a `SELECT` nested inside another statement. Most of them are harmless; one shape — `NOT IN` over a column that can be NULL — is a silent, result-destroying footgun, and it is the reason this skill exists. The foundation skill [`sql-relational-and-null-discipline`](../sql-relational-and-null-discipline/SKILL.md) states the three-valued-logic rules and *flags* the `NOT IN` trap (its §5), then explicitly defers the mechanism and the rewrite to here. **This skill owns that deep dive.** Everything below assumes the set semantics and TRUE/FALSE/UNKNOWN logic established in the foundation.

---

## 1. Scalar Subqueries — One Row, One Column, or It Bites

A *scalar* subquery stands in for a single value (in `SELECT`, `WHERE`, or an expression). It has two failure modes, neither obvious at write time:

- **More than one row → runtime error.** Standard SQL and PostgreSQL raise `more than one row returned by a subquery used as an expression`. The subquery "cannot return more than one row" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)).
- **Zero rows → NULL, silently.** "If it returns zero rows, the result is taken to be null" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)); SQLite agrees: "The value of a subquery expression is NULL if the enclosed SELECT statement returns no rows" ([SQLite — Expressions](https://www.sqlite.org/lang_expr.html#the_exists_operator)). That NULL then propagates through whatever surrounds it (see the foundation on NULL propagation).

```sql
-- RISKY — errors the day two rows match (or worse on SQLite, see §7)
SELECT name, (SELECT price FROM prices p WHERE p.sku = i.sku) AS price
FROM items i;

-- SAFER — guarantee one row with an aggregate, or correlate on a key that is unique
SELECT name, (SELECT MAX(price) FROM prices p WHERE p.sku = i.sku) AS price
FROM items i;
```

**Portability wart:** SQLite does *not* error on a multi-row scalar subquery — it silently returns "the first row of the result" ([SQLite — Expressions](https://www.sqlite.org/lang_expr.html#the_exists_operator)), and without `ORDER BY` that row is undefined (foundation §1). The stricter engines fail loudly; SQLite hides the bug. Route dialect specifics to [`sql-standard-vs-dialect-map`](../sql-standard-vs-dialect-map/SKILL.md).

**Row-constructor subqueries** lift the same idea to multiple columns: `(a, b) = (SELECT x, y FROM ...)` and `(a, b) IN (SELECT x, y FROM ...)` compare a *row* of values at once. They obey the identical rules — the `=` form still requires the subquery to return at most one row, and the `IN`/`NOT IN` forms still carry the NULL behavior of §3, now triggered by a NULL in *any* compared column. They are standard SQL but unevenly supported (route spellings to `sql-standard-vs-dialect-map`); prefer an explicit `EXISTS` with an `AND`-ed join condition when portability matters.

---

## 2. `IN` and `EXISTS` — Two Spellings of a Semi-Join

A *semi-join* keeps a row from the outer table when **at least one** matching row exists in the subquery — the match multiplies nothing and adds no columns. Both `IN (SELECT ...)` and `EXISTS (...)` express it.

```sql
-- Customers who have placed at least one order — semi-join, two spellings:
SELECT * FROM customers c WHERE c.id IN (SELECT customer_id FROM orders);
SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

`EXISTS` tests only **whether rows come back**, never their contents: "If it returns at least one row, the result of `EXISTS` is 'true'; if the subquery returns no rows, the result of `EXISTS` is 'false'." Because "the result depends only on whether any rows are returned, and not on the contents of those rows, ... a common coding convention is to write all `EXISTS` tests in the form `EXISTS(SELECT 1 WHERE ...)`" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)). The `SELECT 1` is idiomatic — `SELECT *` is equivalent but noisier. For the *positive* membership test, `IN` and `EXISTS` are interchangeable in correctness. The divergence appears only under negation — §3.

---

## 3. The `NOT IN` + NULL Trap (the centerpiece)

This is the canonical SQL footgun, and the reason the foundation routes here. **A single NULL anywhere in a `NOT IN` list or subquery collapses the entire result to zero rows — silently, with no error.**

```sql
-- WRONG — if ANY customer_id in orders is NULL, this returns NOTHING.
-- "Find customers with no orders" silently becomes "find no customers."
SELECT * FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);

-- RIGHT — NOT EXISTS is null-safe and the portable anti-join.
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

### Why it happens — the expansion

`NOT IN` is the negation of `IN`, and `IN` is `= ANY`. So `x NOT IN (a, b, NULL)` is `NOT (x = a OR x = b OR x = NULL)`, which by De Morgan is the conjunction:

```text
x <> a  AND  x <> b  AND  x <> NULL
```

modern-sql.com states the mechanism exactly: "An = ANY predicate is only false (so that the negation becomes true) if all comparisons are false. This is however, not possible if there is a NULL comparison, which inevitably yields unknown" ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)). The conjunct `x <> NULL` is UNKNOWN for *every* `x` (foundation §3), and `something AND UNKNOWN` can never be TRUE — at best UNKNOWN. `WHERE` keeps only TRUE rows, so **every** row is dropped. The spec puts it flatly: "the result of the `NOT IN` construct will be null, not true" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)), and "the result of not in predicates that contain a null value is never true" ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)).

### Why `NOT EXISTS` is immune

`EXISTS` asks "did any row come back?", not "is this value equal to that one?" — there is no comparison to go UNKNOWN. "EXISTS never returns unknown" ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)); SQLite confirms cross-engine that "rows containing NULL values are not handled any differently from rows without NULL values" ([SQLite — Expressions](https://www.sqlite.org/lang_expr.html#the_exists_operator)). `NOT EXISTS` is therefore the correct, portable **anti-join** (keep outer rows with *no* match) and the recommended fix: "consider using not exists instead of not in" ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)).

### The asymmetry to remember

`IN` is forgiving: a NULL in the list only matters when there is no real match, and even then `IN` can still return TRUE on a genuine match, so positive membership "usually works." `NOT IN` is unforgiving: with any NULL present it can never be TRUE. (If you must keep `NOT IN`, add `WHERE customer_id IS NOT NULL` to the subquery — but `NOT EXISTS` is cleaner and needs no such guard.) One safe `NOT IN` case: over an *empty* set, `NOT IN` is TRUE for every row ([SQLite — Expressions](https://www.sqlite.org/lang_expr.html#the_exists_operator)).

---

## 4. Correlated vs Uncorrelated Subqueries

- **Uncorrelated:** the subquery references no column from the outer query. It can be evaluated once, independently. `WHERE id IN (SELECT customer_id FROM orders)` is uncorrelated.
- **Correlated:** the subquery references an outer-row column, so conceptually it is re-evaluated for each outer row. `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)` is correlated — `c.id` comes from the outer `customers c`.

`EXISTS`/`NOT EXISTS` anti/semi-joins are correlated by nature (the join predicate lives inside the subquery). "Conceptually re-evaluated per row" is the *semantics*, not the execution plan — every serious optimizer rewrites a correlated `EXISTS` into a single semi-join/anti-join. Whether and how it does so is a plan question; route plan reasoning to [`sql-explain-and-set-based-thinking`](../sql-explain-and-set-based-thinking/SKILL.md). Correlated subqueries in the `FROM` clause (`LATERAL`, correlated derived tables) are a distinct construct owned by [`sql-lateral-and-correlated-derived`](../sql-lateral-and-correlated-derived/SKILL.md).

---

## 5. `ANY` / `ALL` — and Why `<> ALL` Inherits the Same Trap

`ANY` (synonym `SOME`) and `ALL` compare a value against every row of a subquery with a chosen operator. Two equivalences matter:

> "`SOME` is a synonym for `ANY`. `IN` is equivalent to `= ANY`." — [PostgreSQL](https://www.postgresql.org/docs/current/functions-subquery.html)
> "`NOT IN` is equivalent to `<> ALL`." — [PostgreSQL](https://www.postgresql.org/docs/current/functions-subquery.html)

`= ANY` is just `IN`; "The result of `ANY` is 'true' if any true result is obtained" ([PostgreSQL](https://www.postgresql.org/docs/current/functions-subquery.html)). And because `<> ALL` is literally `NOT IN`, **it carries the identical NULL trap.** "The result of `ALL` is 'true' if all rows yield true" ([PostgreSQL](https://www.postgresql.org/docs/current/functions-subquery.html)) — but a NULL right-hand row makes its `<>` comparison UNKNOWN, so `ALL` is never TRUE and the row is dropped.

```sql
-- WRONG — same footgun wearing a different hat: <> ALL ≡ NOT IN
SELECT * FROM customers c
WHERE c.id <> ALL (SELECT customer_id FROM orders);

-- RIGHT — the anti-join, NULL-safe
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

Other operators (`> ALL`, `< ANY`, etc.) are legitimate and have no special NULL hazard beyond ordinary three-valued logic — but anyone reaching for `<> ALL` to "avoid `NOT IN`" has escaped nothing. Use `NOT EXISTS`.

---

## 6. Subquery vs Join — Choosing for Clarity and Plans

When a subquery and a join express the same thing, choose by intent:

- **Semi-join (membership, no extra columns):** prefer `EXISTS`/`IN`. An `INNER JOIN` used for membership can **multiply** outer rows when the subquery side has duplicates — a classic accidental fan-out (join cardinality is owned by [`sql-joins`](../sql-joins/SKILL.md)). `EXISTS` cannot duplicate: it is a pure existence test.
- **Anti-join (absence):** prefer `NOT EXISTS` (§3) over `NOT IN` (NULL-unsafe) and over `LEFT JOIN ... WHERE right.key IS NULL` (correct but easy to misplace the filter — see `sql-joins`).
- **Need columns from the other table:** use a join; that is what joins are for.
- **Single derived value per outer row:** a scalar subquery (§1) reads more clearly than a join + `GROUP BY` when you want exactly one value.

On *plans*: modern optimizers usually produce equivalent plans for `IN`, `EXISTS`, and the matching join, but not always, and `NOT IN` over a nullable column can force a notably worse plan than `NOT EXISTS`. Plan inspection and set-based reasoning belong to [`sql-explain-and-set-based-thinking`](../sql-explain-and-set-based-thinking/SKILL.md); window-function alternatives to top-N-per-group subqueries belong to [`sql-window-functions`](../sql-window-functions/SKILL.md). Pick for correctness and readability first; profile before contorting for plans.

---

## 7. Portability Snapshot

`EXISTS`, `IN`, `NOT IN`, `ANY`/`SOME`, and `ALL` are all standard SQL and present in PostgreSQL, MySQL, SQLite, Oracle, and SQL Server. The semantics in this skill — NOT IN collapsing on a NULL, NOT EXISTS being NULL-immune, scalar 0-rows→NULL — hold across all of them. The deviations to watch:

| Behavior | PostgreSQL | SQLite | MySQL | Oracle |
|---|---|---|---|---|
| `NOT IN (…NULL…)` → empty result | yes | yes | yes | yes |
| `NOT EXISTS` unaffected by NULL | yes | yes | yes | yes |
| Scalar subquery, **>1 row** | **error** | **silent first row** | error | error |
| Scalar subquery, 0 rows → NULL | yes | yes | yes | yes |
| `NOT IN` over an empty set → TRUE | yes | yes | yes | yes |

The scalar `>1 row` row is the one real wart: PostgreSQL/MySQL/Oracle/SQL Server raise a runtime error, SQLite silently takes an undefined first row ([SQLite — Expressions](https://www.sqlite.org/lang_expr.html#the_exists_operator)) — arguably worse, because it returns a plausible wrong answer instead of failing. Dialect spellings and edge cases route to [`sql-standard-vs-dialect-map`](../sql-standard-vs-dialect-map/SKILL.md).

---

## 8. Who Suffers When `NOT IN` Meets a NULL

No error fires; someone downstream pays:

- The **developer** whose nightly "find orphaned records" job is `... WHERE id NOT IN (SELECT parent_id FROM child)`. It worked in dev and in every test. In production one `parent_id` is NULL, the query returns **zero orphans**, the cleanup does nothing, and the bug ships — discovered weeks later when the orphans pile up. The `NOT EXISTS` rewrite they didn't write is the bug.
- The **on-call engineer** paged because a filter "stopped working." Nothing in the query changed; an upstream feed simply started emitting a NULL into the subquery column, and overnight `NOT IN` flipped from "most rows" to "no rows." They stare at a valid query that returns nothing and no error to grep for.
- The **analyst** whose report column is mysteriously NULL for some rows, because a scalar subquery matched zero rows and quietly became NULL (§1) instead of erroring.

Disciplined subquery code is empathy: the `NOT EXISTS` you wrote instead of `NOT IN`, and the aggregate or unique-key guard on the scalar subquery, are gifts to whoever debugs this at 3 a.m.

---

## 9. Routing to Related Skills

- [`sql-relational-and-null-discipline`](../sql-relational-and-null-discipline/SKILL.md) — the policy root: three-valued logic, `= NULL`/`<>` traps, why UNKNOWN is dropped by `WHERE`. It flags the `NOT IN` trap (§5) and defers the deep dive to here.
- [`sql-joins`](../sql-joins/SKILL.md) — semi/anti-join via join, accidental fan-out, `LEFT JOIN ... IS NULL` anti-join, `ON` vs `WHERE` placement.
- [`sql-lateral-and-correlated-derived`](../sql-lateral-and-correlated-derived/SKILL.md) — correlated subqueries and `LATERAL` in the `FROM` clause.
- [`sql-window-functions`](../sql-window-functions/SKILL.md) — window alternatives to top-N-per-group and "compare to the group" subqueries.
- [`sql-explain-and-set-based-thinking`](../sql-explain-and-set-based-thinking/SKILL.md) — plan implications of subquery vs join, correlated execution, and `NOT IN` vs `NOT EXISTS` plans.
- [`sql-standard-vs-dialect-map`](../sql-standard-vs-dialect-map/SKILL.md) — dialect spellings and the scalar-subquery `>1 row` divergence.

---

## 10. Reference Files

High-frequency subquery anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
