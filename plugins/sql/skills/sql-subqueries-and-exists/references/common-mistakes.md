# Common SQL Subquery & EXISTS Mistakes
## Contents

- [1. NOT IN (subquery) over a nullable column — collapses to zero rows](#1-not-in-subquery-over-a-nullable-column-collapses-to-zero-rows)
- [2. <> ALL (subquery) used to "avoid NOT IN"](#2-all-subquery-used-to-avoid-not-in)
- [3. Scalar subquery that can return more than one row](#3-scalar-subquery-that-can-return-more-than-one-row)
- [4. Scalar subquery that returns zero rows — silent NULL](#4-scalar-subquery-that-returns-zero-rows-silent-null)
- [5. INNER JOIN for a membership test — accidental row multiplication](#5-inner-join-for-a-membership-test-accidental-row-multiplication)
- [6. SELECT  (or expensive columns) inside EXISTS](#6-select-or-expensive-columns-inside-exists)
- [7. Keeping NOT IN but forgetting the NULL guard](#7-keeping-not-in-but-forgetting-the-null-guard)


Anti-patterns in LLM-generated SQL around subqueries, `IN`/`NOT IN`, `EXISTS`, `ANY`/`ALL`, and scalar
subqueries — each with wrong/right code and a primary-source citation. The skill
(`sql-subqueries-and-exists`) states the rules; this file holds the high-frequency failure modes. All
RIGHT examples are standard/portable SQL; non-standard spellings are flagged and routed to
`sql-standard-vs-dialect-map`. Three-valued-logic basics are owned by the foundation
(`sql-relational-and-null-discipline`).

---

## 1. `NOT IN (subquery)` over a nullable column — collapses to zero rows

**The problem:** The classic anti-join footgun. `x NOT IN (SELECT y FROM t)` is `NOT (x = ANY(...))`, which expands to `x <> y1 AND x <> y2 AND ...`. If any `y` is NULL, that conjunct is UNKNOWN, the whole `AND` is never TRUE, and `WHERE` drops every row — silently, with no error. "the result of the `NOT IN` construct will be null, not true" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)).

```sql
-- WRONG — one NULL customer_id makes "find customers with no orders" return zero customers
SELECT * FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);

-- RIGHT — NOT EXISTS is null-safe and the portable anti-join
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

*Source: [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html); [modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic). Depth: this skill, §3.*

---

## 2. `<> ALL (subquery)` used to "avoid NOT IN"

**The problem:** The model rewrites `NOT IN` as `<> ALL` thinking it sidesteps the NULL trap. It does not: "`NOT IN` is equivalent to `<> ALL`" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)), so a NULL right-hand row makes the `<>` comparison UNKNOWN and `ALL` is never TRUE — identical collapse.

```sql
-- WRONG — <> ALL is literally NOT IN; same NULL collapse
SELECT * FROM products p
WHERE p.id <> ALL (SELECT product_id FROM discontinued);

-- RIGHT — anti-join via NOT EXISTS
SELECT * FROM products p
WHERE NOT EXISTS (SELECT 1 FROM discontinued d WHERE d.product_id = p.id);
```

*Source: [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html). Depth: this skill, §5.*

---

## 3. Scalar subquery that can return more than one row

**The problem:** The model uses `(SELECT col FROM t WHERE ...)` as a single value, but the predicate is not unique, so two rows match. PostgreSQL/Oracle/SQL Server raise `more than one row returned by a subquery used as an expression`; the subquery "cannot return more than one row" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)). SQLite is worse — it silently returns an undefined first row.

```sql
-- WRONG — errors (or silently picks one row on SQLite) when a SKU has multiple prices
SELECT i.name, (SELECT price FROM prices p WHERE p.sku = i.sku) AS price
FROM items i;

-- RIGHT — force a single value with an aggregate (or correlate on a unique key)
SELECT i.name, (SELECT MAX(price) FROM prices p WHERE p.sku = i.sku) AS price
FROM items i;
```

*Source: [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html); [SQLite — Expressions](https://www.sqlite.org/lang_expr.html#the_exists_operator). Depth: this skill, §1; dialect divergence §7.*

---

## 4. Scalar subquery that returns zero rows — silent NULL

**The problem:** The model assumes a matching row always exists. When none does, "If it returns zero rows, the result is taken to be null" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)) — and that NULL then propagates through arithmetic and comparisons (foundation §6), producing wrong totals or dropped rows with no error.

```sql
-- WRONG — rate is NULL for any currency missing from rates; amount * NULL = NULL
SELECT o.id, o.amount * (SELECT rate FROM rates r WHERE r.ccy = o.ccy) AS usd
FROM orders o;

-- RIGHT — supply a default so a missing row does not nullify the row
SELECT o.id, o.amount * COALESCE((SELECT rate FROM rates r WHERE r.ccy = o.ccy), 1) AS usd
FROM orders o;
```

*Source: [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html). Depth: this skill, §1.*

---

## 5. `INNER JOIN` for a membership test — accidental row multiplication

**The problem:** The model joins to test "has at least one match," but the matched side has duplicates, so each outer row is multiplied (fan-out). A semi-join via `EXISTS` cannot duplicate — "the result depends only on whether any rows are returned" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)).

```sql
-- WRONG — a customer with 3 orders appears 3 times
SELECT c.* FROM customers c JOIN orders o ON o.customer_id = c.id;

-- RIGHT — EXISTS is a true semi-join: one row per matching customer
SELECT c.* FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

*Source: [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html). Depth: this skill, §2, §6; join cardinality owned by `sql-joins`.*

---

## 6. `SELECT *` (or expensive columns) inside `EXISTS`

**The problem:** Not a correctness bug, but a tell that the model misunderstands `EXISTS`. Its SELECT list is ignored — only row existence matters: "the output list of the subquery is normally unimportant. A common coding convention is to write all `EXISTS` tests in the form `EXISTS(SELECT 1 WHERE ...)`" ([PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)).

```sql
-- AWKWARD — selecting columns that EXISTS discards; misleading to readers
SELECT * FROM customers c WHERE EXISTS (SELECT c.name, o.* FROM orders o WHERE o.customer_id = c.id);

-- RIGHT — idiomatic existence test
SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

*Source: [PostgreSQL — Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html). Depth: this skill, §2.*

---

## 7. Keeping `NOT IN` but forgetting the NULL guard

**The problem:** When `NOT IN` is genuinely preferred (e.g., a small static list), the model omits the guard that makes it safe against a nullable subquery column. The fix, when not switching to `NOT EXISTS`, is to filter NULLs out of the subquery: "add a where condition to the subquery that removes possible null values" ([modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic)).

```sql
-- WRONG — nullable customer_id can still collapse the result
SELECT * FROM customers c WHERE c.id NOT IN (SELECT customer_id FROM orders);

-- RIGHT (if NOT IN must stay) — strip NULLs from the subquery
SELECT * FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);

-- BETTER — NOT EXISTS needs no guard
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

*Source: [modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic). Depth: this skill, §3.*
