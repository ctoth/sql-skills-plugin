---
name: sql-joins
description: Guides correct join composition and the single most damaging join bug — putting a filter on the null-able side of an outer join in WHERE instead of ON, which silently demotes a LEFT JOIN to an INNER JOIN and drops the very rows you were looking for. Covers INNER/LEFT/RIGHT/FULL/CROSS semantics with wrong/right SQL, ON vs USING vs NATURAL (and why NATURAL JOIN is a schema-change landmine that joins on every same-named column), self-joins, detecting accidental cross products from a missing or typo'd predicate, one-to-many fan-out that silently inflates SUM/COUNT, and join-key nullability under three-valued logic. Auto-invokes when writing or editing JOIN clauses, ON/USING/NATURAL conditions, queries filtering an outer-joined table, multi-table FROM lists, or on "my LEFT JOIN is dropping rows" / "duplicate rows after join" / "my totals doubled" / "why did I get a cross product" symptoms. Builds on sql-relational-and-null-discipline.
allowed-tools: Read, Glob, Grep
---

# SQL Joins

> "This is because a restriction placed in the `ON` clause is processed _before_ the join, while a restriction placed in the `WHERE` clause is processed _after_ the join. That does not matter with inner joins, but it matters a lot with outer joins."
> — [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)

> "The join operation transforms data from a normalized model into a denormalized form that suits a specific processing purpose."
> — [Markus Winand — SQL JOIN](https://use-the-index-luke.com/sql/join)

A join recombines normalized tables row by row — conceptually "for each row, look up the matching rows on the other side" ([Winand — Nested Loops](https://use-the-index-luke.com/sql/join/nested-loops-join)). Two things decide whether you get the right answer: **which rows match** (the join condition) and **what happens to rows that don't** (inner drops them; outer keeps them null-extended). A join condition is a predicate, and `ON` keeps a pair only when it evaluates to TRUE — exactly like `WHERE`. This skill assumes the set semantics and three-valued logic of the foundation skill **`sql-relational-and-null-discipline`** (`ON` keeps only TRUE; `NULL = NULL` is UNKNOWN; a `WHERE` filter drops UNKNOWN). The NULL truth tables there underpin every outer-join surprise below.

---

## 1. The Five Join Types

Each type differs only in **what happens to unmatched rows**. Definitions are PostgreSQL's; the behavior is standard.

```sql
-- INNER — keep only matched pairs. "For each row R1 of T1, the joined table has a
-- row for each row in T2 that satisfies the join condition with R1."
SELECT * FROM orders o INNER JOIN customers c ON c.id = o.customer_id;

-- LEFT OUTER — keep every left row; null-extend the right when no match.
-- "the joined table always has at least one row for each row in T1."
SELECT * FROM customers c LEFT JOIN orders o ON o.customer_id = c.id;

-- RIGHT OUTER — converse of LEFT: keep every right row. (Rewrite as LEFT by swapping sides.)
SELECT * FROM orders o RIGHT JOIN customers c ON c.id = o.customer_id;

-- FULL OUTER — keep unmatched rows from BOTH sides, null-extended.
SELECT * FROM customers c FULL JOIN orders o ON o.customer_id = c.id;

-- CROSS — every combination. "If the tables have N and M rows ... N * M rows."
SELECT * FROM sizes CROSS JOIN colors;
```

The definitional quotes: INNER — "For each row R1 of T1, the joined table has a row for each row in T2 that satisfies the join condition with R1"; LEFT — "for each row in T1 that does not satisfy the join condition ... a joined row is added with null values in columns of T2 ... always at least one row for each row in T1"; FULL adds the same null-extension "for each row of T2 that does not satisfy the join condition with any row in T1"; CROSS — "a Cartesian product ... N \* M rows" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). Prefer `LEFT JOIN` over `RIGHT JOIN`: "keep everything in the table I started from" reads more clearly than its converse.

**Portability:** INNER/LEFT/CROSS are universal. `FULL JOIN` and `RIGHT JOIN` are standard and supported by PostgreSQL and SQLite (3.39.0+, 2022), but **MySQL has no `FULL JOIN`** — emulate it with `LEFT ... UNION ... RIGHT`; route that spelling to **`sql-standard-vs-dialect-map`**.

---

## 2. THE CENTERPIECE — `ON` vs `WHERE` on an Outer Join

This is the single most damaging join bug. The rule: **a predicate filtering the null-able (outer) side of a `LEFT JOIN` belongs in `ON`, not `WHERE`.** Put it in `WHERE` and the `LEFT JOIN` silently collapses into an `INNER JOIN`.

Why: "a restriction placed in the `ON` clause is processed _before_ the join, while a restriction placed in the `WHERE` clause is processed _after_ the join. That does not matter with inner joins, but it matters a lot with outer joins" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). The `LEFT JOIN` null-extends unmatched left rows (the right columns become NULL); then a `WHERE o.status = 'paid'` runs *after* that and tests `NULL = 'paid'`, which is UNKNOWN, and `WHERE` keeps only TRUE — so every null-extended row is discarded (foundation §2).

```sql
-- WRONG — looks like "all customers, with their paid orders if any."
-- Actually returns ONLY customers who have a paid order. The LEFT JOIN is demoted to INNER:
-- customers with no paid order get NULL o.status, and `NULL = 'paid'` is UNKNOWN -> dropped.
SELECT c.id, c.name, o.amount
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE  o.status = 'paid';

-- RIGHT — move the filter on the OUTER table into the ON clause.
-- Unmatched customers survive (null-extended); matched rows are restricted to paid orders.
SELECT c.id, c.name, o.amount
FROM   customers c
LEFT JOIN orders o
       ON o.customer_id = c.id
      AND o.status = 'paid';
```

Two cases where a `WHERE` filter is correct and intended:

```sql
-- LEGIT (a): filter on the PRESERVED (left) table — fine in WHERE; it does not touch
-- the null-extension of the right side.
SELECT c.id, o.amount
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE  c.region = 'EU';

-- LEGIT (b): the deliberate ANTI-JOIN — "customers with NO orders."
-- The WHERE NULL test AFTER the join is exactly the point.
SELECT c.id, c.name
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE  o.id IS NULL;
```

Rule of thumb: **join conditions and filters on the outer table go in `ON`; filters on the preserved table (or an intentional `IS NULL` unmatched-row test) go in `WHERE`.** (The `WHERE ... IS NULL` anti-join is one valid idiom; the `NOT EXISTS` form and the semi/anti-join discussion live in **`sql-subqueries-and-exists`**.)

---

## 3. `ON` vs `USING` vs `NATURAL` — Prefer Explicit `ON`

All three express the join condition; they differ in how much you spell out.

- **`ON`** is "the most general kind of join condition: it takes a Boolean value expression of the same kind as is used in a `WHERE` clause. A pair of rows ... match if the `ON` expression evaluates to true" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). Use it by default — it is explicit and survives schema changes.
- **`USING (col, ...)`** is "a shorthand ... where both sides of the join use the same name for the joining column(s) ... `USING (a, b)` produces the join condition `ON T1.a = T2.a AND T1.b = T2.b`," and it "suppresses redundant columns" so the joined column appears once ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). Acceptable because it names the columns explicitly.
- **`NATURAL`** is the landmine. It "is a shorthand form of `USING`: it forms a `USING` list consisting of **all column names that appear in both input tables** ... If there are no common column names, `NATURAL JOIN` behaves like `CROSS JOIN`" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)).

Why `NATURAL` is a schema-change landmine: the join key set is **whatever columns happen to share a name today**. Add a column tomorrow — a `created_at`, `status`, `is_active`, or generic `id` — to either table and the join silently starts matching on it too, changing the result with no edit to the query. And if a rename ever leaves *no* common column, the `NATURAL JOIN` quietly degenerates into a `CROSS JOIN` (an N×M explosion, §5). The query that "worked" becomes wrong on a migration that never touched it.

```sql
-- WRONG — joins on EVERY shared column name. Add `customers.region` and `orders.region`
-- later and this silently also requires region to match; add nothing common and it CROSS JOINs.
SELECT * FROM customers NATURAL JOIN orders;

-- RIGHT — explicit, intent-stable across schema changes
SELECT * FROM customers c JOIN orders o ON o.customer_id = c.id;

-- ACCEPTABLE — USING names the column; still verify the names won't accidentally collide
SELECT * FROM customers JOIN orders USING (customer_id);
```

SQLite supports `NATURAL`, `USING`, and `ON` join constraints just as PostgreSQL does ([SQLite — SELECT](https://www.sqlite.org/lang_select.html)); the definitional wording above is PostgreSQL's and the behavior is shared. **Recommendation: always write an explicit `ON`** (or `USING` with named columns). Never `NATURAL JOIN` in code you intend to keep.

---

## 4. Self-Joins

A table joined to itself compares rows to other rows of the same table; the only requirement is **distinct aliases** so each reference is unambiguous.

```sql
-- Employees with their managers (manager_id -> employees.id), in the same table.
SELECT e.name AS employee, m.name AS manager
FROM   employees e
LEFT JOIN employees m ON m.id = e.manager_id;   -- LEFT keeps the CEO (no manager)
```

Use `LEFT JOIN` here deliberately: an `INNER` self-join would drop the top of the hierarchy (the row whose `manager_id IS NULL`), because `m.id = NULL` is UNKNOWN and never matches (§7, foundation §3).

---

## 5. Detecting an Accidental Cross Product

A missing or typo'd join predicate does not error — it produces the Cartesian product: "a Cartesian product ... If the tables have N and M rows respectively, the joined table will have N \* M rows" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). The symptom is a result far larger than either input and wildly inflated aggregates.

```sql
-- WRONG — comma join with the predicate forgotten: every order paired with every customer.
SELECT * FROM orders o, customers c;            -- N * M rows, silently

-- WRONG — typo'd predicate that's always true degenerates to a cross product
SELECT * FROM orders o JOIN customers c ON 1 = 1;

-- RIGHT — every join carries a real predicate
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;

-- RIGHT — when you genuinely want all combinations, say so explicitly
SELECT * FROM sizes CROSS JOIN colors;
```

Defenses: prefer the explicit `JOIN ... ON` form over comma-separated `FROM a, b` so a missing predicate is a visible omission; write `CROSS JOIN` when a cross product is intended so the next reader knows it was deliberate; and sanity-check the row count against the input sizes.

---

## 6. One-to-Many Fan-Out Inflates Aggregates

Joining a one-to-many relationship multiplies the "one" side by the number of matching "many" rows — INNER/LEFT both give "a row for each row in T2 that satisfies the join condition" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). If you then aggregate a column from the "one" side, it is **counted once per match**, silently inflating `SUM`/`COUNT`.

```sql
-- WRONG — each customer row is duplicated once per order line; SUM(c.credit_limit)
-- counts the credit limit once PER ORDER, not once per customer. Totals balloon.
SELECT c.id, SUM(c.credit_limit) AS total_limit, SUM(o.amount) AS spent
FROM   customers c
JOIN   orders o ON o.customer_id = c.id
GROUP  BY c.id;

-- RIGHT — aggregate the "many" side in a subquery, THEN join one-to-one
SELECT c.id, c.credit_limit, COALESCE(s.spent, 0) AS spent
FROM   customers c
LEFT JOIN (
    SELECT customer_id, SUM(amount) AS spent
    FROM   orders
    GROUP  BY customer_id
) s ON s.customer_id = c.id;
```

The tell is a duplicated dimension value (the same customer appearing on multiple rows) and an aggregate of a one-side column that is larger than reality. `COUNT(DISTINCT ...)` can paper over a count but cannot fix a `SUM`; pre-aggregate the many side instead. Grouping mechanics and functional-dependency rules belong to **`sql-aggregation-and-grouping`**.

---

## 7. Join-Key Nullability and Three-Valued Logic

An equi-join condition is a predicate, and `ON` keeps a pair only when it is TRUE. So `ON a.k = b.k` **never matches when either key is NULL**, because `NULL = NULL` is UNKNOWN, not TRUE (foundation §3). Rows with a NULL join key fall through every `INNER`/`LEFT` match and become unmatched (null-extended under `LEFT`, dropped under `INNER`).

```sql
-- These two NULL-keyed rows do NOT match each other under a normal equi-join:
-- ON a.k = b.k  ->  NULL = NULL is UNKNOWN  ->  no match

-- RIGHT — null-safe join that treats two NULL keys as equal (foundation §8)
SELECT * FROM a JOIN b ON a.k IS NOT DISTINCT FROM b.k;
```

`IS NOT DISTINCT FROM` "always return[s] true or false, never null," so it is safe in a join condition; the null-safe equality and its dialect spellings (MySQL `<=>`) are owned by the foundation skill and **`sql-standard-vs-dialect-map`**. Usually the right move is to forbid NULL join keys with `NOT NULL` rather than to match them — but know that a plain equi-join silently excludes them.

---

## 8. Portability Snapshot

The join *types* and `ON`/`USING`/`NATURAL` syntax are standard; the deviations to watch:

| Feature | PostgreSQL | SQLite | MySQL |
|---|---|---|---|
| `INNER` / `LEFT` / `CROSS JOIN` | yes | yes | yes |
| `RIGHT JOIN` | yes | yes (3.39.0+) | yes |
| `FULL [OUTER] JOIN` | yes | yes (3.39.0+) | **no — emulate with `LEFT … UNION … RIGHT`** |
| `JOIN ... USING (col)` | yes | yes | yes |
| `NATURAL JOIN` | yes (avoid) | yes (avoid) | yes (avoid) |
| Null-safe key match | `IS NOT DISTINCT FROM` | `IS NOT DISTINCT FROM` | `<=>` |

The most common portability trap is MySQL's missing `FULL JOIN`; the most common *correctness* trap is identical everywhere — the `ON`-vs-`WHERE` outer-join filter (§2). For `FULL JOIN` emulation, null-safe spellings, and other dialect differences, route to **`sql-standard-vs-dialect-map`**.

---

## 9. Who Suffers When Joins Go Wrong

A join mistake almost never errors — it returns a plausible-looking wrong answer, and someone downstream pays:

- The **analyst** whose revenue total doubled overnight because a one-to-many join fanned out the customer rows and `SUM(credit_limit)` counted each limit once per order (§6). The number was presented in a board deck as fact; the join had no error and the SQL "looked right."
- The **developer** debugging why a `LEFT JOIN` "is dropping rows" — the report of customers-with-their-orders is missing every customer who has no order, because a `WHERE o.status = 'paid'` quietly demoted the `LEFT JOIN` to an `INNER JOIN` (§2). The rows they were looking for were exactly the ones the filter deleted.
- The **on-call engineer** staring at a query that returned 40 million rows from two 6,000-row tables, because a refactor renamed a column and a `NATURAL JOIN` lost its last common name and degenerated into a `CROSS JOIN` (§3, §5). Nobody edited the query.
- The **next maintainer** who can't tell whether a comma-join cross product was intended or a forgotten predicate, because it was never written as an explicit `CROSS JOIN` (§5).

Disciplined joins are empathy in code: the filter in `ON` instead of `WHERE`, the explicit `ON` instead of `NATURAL`, the pre-aggregated subquery instead of a fan-out, and the spelled-out `CROSS JOIN` are all gifts to whoever debugs this under load.

---

## 10. Routing to Related Skills

- **`sql-relational-and-null-discipline`** (foundation) — set semantics and three-valued logic; `ON` keeps only TRUE, `NULL = NULL` is UNKNOWN, `WHERE` drops UNKNOWN. Every outer-join surprise here traces back to it.
- **`sql-subqueries-and-exists`** — semi-joins and anti-joins via `EXISTS`/`NOT EXISTS` (the null-safe alternative to the `WHERE ... IS NULL` anti-join in §2, and to `NOT IN`).
- **`sql-lateral-and-correlated-derived`** — `LATERAL` joins and correlated derived tables (a per-row "join to a subquery that references the outer row").
- **`sql-aggregation-and-grouping`** — `GROUP BY`/`HAVING` and functional-dependency rules behind the fan-out fix (§6).
- **`sql-indexing-and-sargability`** — join *performance*: nested-loops/hash/merge algorithms and which columns to index. This skill owns correctness, not plans.
- **`sql-standard-vs-dialect-map`** — `FULL JOIN` emulation in MySQL, null-safe key spellings (`<=>`), and other dialect differences.

---

## 11. Reference Files

High-frequency join anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
