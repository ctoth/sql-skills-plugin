# Common SQL Relational & NULL Mistakes

Anti-patterns in LLM-generated SQL around set semantics and three-valued logic, each with wrong/right
code and a primary-source citation. The policy root (`sql-relational-and-null-discipline`) states the
rules; this file holds the high-frequency failure modes. All RIGHT examples are standard/portable SQL;
non-standard spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Comparing to NULL with `=` instead of `IS NULL`

**The problem:** The model writes `WHERE col = NULL` (often translating from a language where `== null` works). Every row evaluates to UNKNOWN, which `WHERE` drops, so the query returns the empty set with no error. "Do _not_ write `expression = NULL` because `NULL` is not 'equal to' `NULL`" ([PostgreSQL — Comparison](https://www.postgresql.org/docs/current/functions-comparison.html)).

```sql
-- WRONG — UNKNOWN for every row; returns nothing
SELECT * FROM orders WHERE shipped_at = NULL;

-- RIGHT
SELECT * FROM orders WHERE shipped_at IS NULL;
```

*Source: [PostgreSQL — Comparison Operators](https://www.postgresql.org/docs/current/functions-comparison.html). Depth: this skill, §3.*

---

## 2. `<>` / `!=` silently dropping NULL rows

**The problem:** The model filters with `col <> 'x'` expecting "everything that isn't 'x'," but `NULL <> 'x'` is UNKNOWN, so every NULL row is excluded. "`7 <> NULL` yields null" ([PostgreSQL — Comparison](https://www.postgresql.org/docs/current/functions-comparison.html)).

```sql
-- WRONG — excludes rows where status IS NULL
SELECT * FROM tasks WHERE status <> 'done';

-- RIGHT — include the unknowns explicitly...
SELECT * FROM tasks WHERE status <> 'done' OR status IS NULL;
-- ...or use the null-safe operator
SELECT * FROM tasks WHERE status IS DISTINCT FROM 'done';
```

*Source: [PostgreSQL — Comparison Operators](https://www.postgresql.org/docs/current/functions-comparison.html). Depth: this skill, §4.*

---

## 3. `NOT IN (subquery)` that collapses to zero rows on one NULL

**The problem:** The classic anti-join footgun. `x NOT IN (SELECT y FROM t)` expands to a chain of `x <> y AND ...`; if any `y` is NULL the chain is never TRUE, so the whole query returns nothing — silently. A single NULL in the subquery breaks it.

```sql
-- WRONG — one NULL customer_id makes this return zero rows
SELECT * FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);

-- RIGHT — NOT EXISTS is null-safe and the portable anti-join
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

*Source: [modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic). Depth: this skill, §5; full rewrite owned by `sql-subqueries-and-exists`.*

---

## 4. Expecting `CHECK` to reject a NULL

**The problem:** The model writes `CHECK (qty > 0)` and assumes a NULL `qty` is rejected. But `CHECK` "rejects _false_, rather than accepting _true_," so it "accept[s] _true_ and _unknown_" — `NULL > 0` is UNKNOWN and passes ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)). This is the exact opposite of how `WHERE qty > 0` behaves on the same value.

```sql
-- WRONG — a NULL qty slips through; CHECK only rejects definite FALSE
qty INTEGER CHECK (qty > 0)

-- RIGHT — forbid the NULL separately
qty INTEGER NOT NULL CHECK (qty > 0)
```

*Source: [modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic). Depth: this skill, §2; syntax owned by `sql-constraints-and-integrity`.*

---

## 5. NULL silently nullifying an arithmetic or concatenation result

**The problem:** A computed column uses a nullable input directly; one NULL nullifies the whole expression. "Expressions and functions that process a `null` value generally return the `null` value" ([modern-sql.com — NULL](https://modern-sql.com/concept/null)).

```sql
-- WRONG — total is NULL whenever discount IS NULL
SELECT price - discount AS total FROM line_items;

-- RIGHT — coalesce nullable operands to a neutral value first
SELECT price - COALESCE(discount, 0) AS total FROM line_items;
```

*Source: [modern-sql.com — NULL](https://modern-sql.com/concept/null); [Oracle — Nulls](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html). Depth: this skill, §6.*

---

## 6. `COUNT(col)` used where `COUNT(*)` was meant

**The problem:** The model counts a nullable column expecting a row count. `count(*)` "computes the number of input rows"; `count(expression)` counts only "input rows in which the input value is not null" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). The two silently disagree whenever the column has NULLs.

```sql
-- WRONG — undercounts: rows with a NULL phone are excluded
SELECT COUNT(phone) AS user_count FROM users;

-- RIGHT — count rows
SELECT COUNT(*) AS user_count FROM users;
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html). Depth: this skill, §7.*

---

## 7. Assuming `AVG`/`SUM` treat NULL as zero

**The problem:** Two distinct errors. (a) The model thinks `AVG(col)` divides by all rows; it divides the sum of non-null values by the *count of non-null values*. (b) The model expects `SUM` over an empty/all-NULL set to be 0, but "`sum` of no rows returns null, not zero" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)).

```sql
-- WRONG — coalescing to 0 inflates the denominator and changes the average
SELECT AVG(COALESCE(score, 0)) FROM results;   -- NOT the same as AVG(score)

-- RIGHT — let AVG skip NULLs (the usual intent)
SELECT AVG(score) FROM results;

-- RIGHT — coalesce the RESULT when you need 0 for an empty sum
SELECT COALESCE(SUM(amount), 0) AS total FROM payments WHERE user_id = 42;
```

*Source: [PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html). Depth: this skill, §7.*

---

## 8. Relying on result order without `ORDER BY`

**The problem:** The model takes the "first" or "latest" row, or assumes rows come back in insertion/primary-key order, without an `ORDER BY`. "If sorting is not chosen, the rows will be returned in an unspecified order ... it must not be relied on" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)).

```sql
-- WRONG — "the latest" is undefined; the plan decides, and can change
SELECT * FROM events FETCH FIRST 1 ROWS ONLY;

-- RIGHT — a total order (tie-broken on a unique column) makes it deterministic
SELECT * FROM events ORDER BY created_at DESC, id DESC FETCH FIRST 1 ROWS ONLY;
```

*Source: [PostgreSQL — Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html); [Codd 1970](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf). Depth: this skill, §1.*

---

## 9. Expecting a default `UNIQUE` column to allow only one NULL

**The problem:** The model declares `email VARCHAR UNIQUE` and assumes at most one NULL. By default "null values are not considered equal," so multiple NULL rows are allowed ([PostgreSQL — CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html)) — the opposite of how `GROUP BY`/`DISTINCT` fold NULLs together.

```sql
-- WRONG (assumption) — this still permits MANY rows with email = NULL
email VARCHAR(320) UNIQUE

-- RIGHT — SQL:2023 lever to permit at most one NULL (not all engines)
email VARCHAR(320) UNIQUE NULLS NOT DISTINCT
-- ...or require presence
email VARCHAR(320) NOT NULL UNIQUE
```

*Source: [PostgreSQL — CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html); [SQLite — NULL Handling](https://www.sqlite.org/nulls.html). Depth: this skill, §9; syntax owned by `sql-constraints-and-integrity`.*

---

## 10. Engine-dependent NULL sort position left implicit

**The problem:** The model writes `ORDER BY col DESC` and assumes a fixed NULL position. PostgreSQL/Oracle sort NULLs as largest; MySQL/SQLite as smallest. "`NULLS FIRST` is the default for `DESC` order, and `NULLS LAST` otherwise" in PostgreSQL ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)) — but that default is not universal.

```sql
-- WRONG — NULL placement differs across engines; results disagree
SELECT * FROM events ORDER BY ended_at DESC;

-- RIGHT — pin the NULL position explicitly
SELECT * FROM events ORDER BY ended_at DESC NULLS LAST;
```

*Source: [PostgreSQL — Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html). Depth: this skill, §9; dialect spellings owned by `sql-standard-vs-dialect-map`.*

---

## 11. Avoiding NULL with a sentinel value

**The problem:** To "avoid NULL," the model stores `-1`, `0`, `''`, or `'N/A'` for missing data — the "Fear of the Unknown" antipattern. The sentinel poisons aggregates, defeats `IS NULL`, and collides with real values ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG — -1 corrupts SUM/AVG/MIN and every query must special-case it
end_date DATE NOT NULL DEFAULT '9999-12-31'   -- "no end" sentinel
... WHERE end_date = '9999-12-31'

-- RIGHT — NULL is the correct marker for "unknown / not yet set"
end_date DATE   -- nullable
... WHERE end_date IS NULL
```

*Source: [Karwin, *SQL Antipatterns* — "Fear of the Unknown"](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §10.*
