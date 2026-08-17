---
name: sql-expressions-case-and-functions
description: Guides portable scalar expressions in SQL — use standard `CASE`, `COALESCE`, `NULLIF`, `||` concatenation, and the keyword string functions (`SUBSTRING(... FROM ... FOR ...)`, `TRIM([LEADING|TRAILING|BOTH] c FROM s)`, `POSITION(sub IN s)`, `OVERLAY`, `CHAR_LENGTH`, `UPPER`/`LOWER`), plus `CAST(x AS type)` for conversion, instead of vendor spellings (`IFNULL`/`ISNULL`/`NVL`/`IIF`, `SUBSTR`/`INSTR`/`LEFT`/`RIGHT`/`LENGTH`, `+` for concat, `::`/`CONVERT`). Warns that `||` and arithmetic return NULL if any operand is NULL (the silent-blank concat trap; Oracle deviates by treating NULL as `''`), that a `CASE` with no `ELSE` defaults to NULL and short-circuits with no fall-through, that `NULLIF(a,b)` yields NULL when equal (the `x / NULLIF(y,0)` divide-by-zero guard), and that `GREATEST`/`LEAST` (standardized in SQL:2023) disagree across engines on NULL. Auto-invokes when writing or editing `CASE`, `COALESCE`/`NULLIF`/`GREATEST`/`LEAST`, string functions, string concatenation, or type conversions/`CAST`.
allowed-tools: Read, Glob, Grep
---

# SQL Expressions: CASE and Functions

> "A `CASE` expression does not evaluate any subexpressions that are not needed to determine the result. ... If the `ELSE` clause is omitted and no condition is true, the result is null."
> — [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html)

> "The `COALESCE` function returns the first of its arguments that is not null. Null is returned only if all arguments are null."
> — [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html)

A scalar expression is the small print of a query: the `CASE` that buckets a value, the `COALESCE` that supplies a default, the `||` that builds a label, the `CAST` that changes a type. Each has a *standard* spelling and a pile of vendor-specific ones, and each interacts with `NULL` in a way that fails silently rather than loudly. This skill is about writing those expressions so they mean the same thing on every engine and don't quietly emit a NULL. It builds directly on the NULL semantics owned by the foundation skill **`sql-relational-and-null-discipline`** — the *why* of three-valued logic lives there; this skill applies it to the everyday expression toolkit.

---

## 1. Simple vs Searched `CASE` — Short-Circuit, No Fall-Through, Implicit `ELSE NULL`

`CASE` is "a generic conditional expression, similar to if/else statements in other programming languages" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)). Two forms:

- **Simple**: `CASE expr WHEN value THEN result ... END` — compares `expr` against each `value` by equality.
- **Searched**: `CASE WHEN condition THEN result ... END` — evaluates arbitrary boolean conditions.

Three facts decide correctness. (a) It **short-circuits**: it "does not evaluate any subexpressions that are not needed to determine the result" — `WHEN`s are tested top-down and the first TRUE wins. (b) There is **no fall-through** — unlike a C `switch`, exactly one branch is taken; no `break` needed. (c) With no `ELSE`, "if the `ELSE` clause is omitted and no condition is true, the result is null" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)).

```sql
-- WRONG — no ELSE, so any grade outside the listed bands is silently NULL,
-- and that NULL then poisons any downstream arithmetic or concatenation
SELECT CASE WHEN score >= 90 THEN 'A'
            WHEN score >= 80 THEN 'B' END AS grade
FROM exams;

-- RIGHT — make the default explicit; one branch always fires
SELECT CASE WHEN score >= 90 THEN 'A'
            WHEN score >= 80 THEN 'B'
            ELSE 'F' END AS grade
FROM exams;
```

The simple form is sugar for the searched form, but **beware NULL**: simple `CASE x WHEN NULL THEN ...` never matches, because `x = NULL` is UNKNOWN, not TRUE (foundation §3). Test NULL with a searched `WHEN x IS NULL`.

---

## 2. `COALESCE` — the Standard Null-Substitution, Not `IFNULL`/`NVL`/`ISNULL`/`IIF`

`COALESCE` "returns the first of its arguments that is not null. Null is returned only if all arguments are null," and like `CASE` it short-circuits — "arguments to the right of the first non-null argument are not evaluated" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)). It takes any number of arguments and is the portable way to supply a default.

```sql
-- WRONG — vendor-specific, non-portable spellings of the same idea
SELECT IFNULL(nickname, 'guest') FROM users;   -- MySQL / SQLite
SELECT NVL(nickname, 'guest')    FROM users;    -- Oracle
SELECT ISNULL(nickname, 'guest') FROM users;    -- SQL Server

-- RIGHT — standard, runs everywhere, accepts a fallback chain
SELECT COALESCE(nickname, full_name, 'guest') FROM users;
```

`COALESCE(a, b)` is the two-argument case those vendor functions cover; it also extends to a fallback *chain*, which the binary vendor functions cannot. Route the dialect spellings to **`sql-standard-vs-dialect-map`**.

---

## 3. `NULLIF` — Manufacturing a NULL, and the Divide-by-Zero Guard

`NULLIF` is the inverse of `COALESCE`: it "returns a null value if value1 equals value2; otherwise it returns value1" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)). Its highest-value use is converting a forbidden divisor into a NULL so the division yields NULL instead of raising a "division by zero" error.

```sql
-- WRONG — raises "division by zero" the moment any orders count is 0
SELECT revenue / orders AS avg_order_value FROM stores;

-- RIGHT — NULLIF turns a 0 divisor into NULL; NULL division yields NULL, no error
SELECT revenue / NULLIF(orders, 0) AS avg_order_value FROM stores;
```

The result is NULL for the zero-order rows (handle that with `COALESCE(..., 0)` on the outside if a number is required). `NULLIF` also strips sentinels — `NULLIF(rating, -1)` turns a `-1` "no rating" placeholder back into the NULL it should have been (the sentinel antipattern is foundation §10).

---

## 4. The `||` NULL-Concat Trap (and the Oracle Deviation)

`||` is the SQL-standard concatenation operator: `'Post' || 'greSQL' → PostgreSQL` ([PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html)). The trap is that, like arithmetic, **concatenation propagates NULL** — a single NULL operand makes the entire expression NULL (foundation §6). This is how a report field silently goes blank: one nullable middle name nullifies the whole assembled name.

```sql
-- WRONG — if middle_name IS NULL, full_name is NULL for that row, not a partial string
SELECT first_name || ' ' || middle_name || ' ' || last_name AS full_name FROM people;

-- RIGHT — coalesce each nullable operand to '' before concatenating
SELECT first_name || COALESCE(' ' || middle_name, '') || ' ' || last_name AS full_name
FROM people;
```

Two portability hazards. **(a) SQL Server uses `+` for string concatenation**, not `||` — `+` on a NULL is also NULL there, and `+` collides with numeric addition. **(b) Oracle deviates**: because Oracle treats `''` as NULL, its `||` "ignores" a NULL operand and treats it as the empty string, so `'a' || NULL` yields `'a'` on Oracle but `NULL` on standard engines (foundation §6, §11). PostgreSQL's separate `concat()` function also ignores NULLs — convenient, but **non-standard**; reach for explicit `COALESCE` when you want portable, predictable behavior. Route all of this to **`sql-standard-vs-dialect-map`**.

---

## 5. Standard String Functions vs Vendor Spellings

Every common string operation has a SQL-standard keyword form. Prefer it; the positional vendor aliases differ in argument order (a real bug source — `STRPOS`/`INSTR` reverse `POSITION`'s arguments).

| Need | Standard (portable) | Common vendor spellings |
|---|---|---|
| Concatenate | `a \|\| b` | `+` (SQL Server); `CONCAT(a, b)` |
| Substring | `SUBSTRING(s FROM start FOR len)` | `SUBSTR(s, start, len)` |
| First/last *n* chars | `SUBSTRING(s FROM 1 FOR n)` / `SUBSTRING(s FROM n)` | `LEFT(s, n)` / `RIGHT(s, n)` |
| Trim chars | `TRIM([LEADING\|TRAILING\|BOTH] c FROM s)` | `TRIM(s, c)`, `LTRIM`/`RTRIM` |
| Find substring index | `POSITION(sub IN s)` | `STRPOS(s, sub)` (args reversed), `INSTR(s, sub)` |
| Replace a slice | `OVERLAY(s PLACING new FROM start FOR len)` | manual `SUBSTRING` + `\|\|` |
| Length in characters | `CHAR_LENGTH(s)` / `CHARACTER_LENGTH(s)` | `LENGTH(s)`, `LEN(s)` |
| Upper / lower case | `UPPER(s)` / `LOWER(s)` | (same — these are standard) |

```sql
-- WRONG — vendor positional spellings; SUBSTR/LEFT/STRPOS are not standard keywords,
-- and STRPOS reverses POSITION's argument order
SELECT SUBSTR(code, 1, 3), LEFT(name, 1), STRPOS(email, '@') FROM accounts;

-- RIGHT — standard keyword forms
SELECT SUBSTRING(code FROM 1 FOR 3),
       SUBSTRING(name FROM 1 FOR 1),
       POSITION('@' IN email)
FROM accounts;
```

The standard `TRIM` form is `TRIM(BOTH 'xyz' FROM 'yxTomxx') → Tom` ([PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html)); the comma form `TRIM(s, 'xyz')` is non-standard. All string functions still obey NULL propagation: `UPPER(NULL)` is NULL. Per-engine availability lives in **`sql-standard-vs-dialect-map`**.

---

## 6. `CAST(x AS type)` — the Portable Conversion, and Implicit-Coercion Hazards

`CAST(x AS type)` is the SQL-standard explicit conversion and runs on every engine. The vendor shortcuts — PostgreSQL's `x::type` and SQL Server's `CONVERT(type, x)` — are not portable.

```sql
-- WRONG — :: is PostgreSQL-only; CONVERT is SQL Server-only
SELECT '2026-06-26'::date, total::numeric(10,2) FROM invoices;   -- PostgreSQL only

-- RIGHT — standard CAST, identical meaning everywhere
SELECT CAST('2026-06-26' AS DATE), CAST(total AS NUMERIC(10,2)) FROM invoices;
```

**Implicit coercion is the quieter hazard.** Engines disagree wildly on when they auto-convert: PostgreSQL is strict and will reject `WHERE int_col = '5'` mixing in some contexts; MySQL and SQLite coerce eagerly (a string `'5abc'` may become `5`, or a string-to-number comparison may turn an indexed column scan non-sargable). Two consequences worth making explicit with `CAST`: comparing a number to a string literal, and relying on automatic numeric widening. Be explicit rather than trusting each engine's coercion rules. Numeric precision/rounding details belong to **`sql-data-types-and-numerics`**; date/time conversions and templating belong to **`sql-datetime-and-intervals`**.

---

## 7. `GREATEST` / `LEAST` — Standardized in SQL:2023, but NULL-Divergent

`GREATEST` and `LEAST` "select the largest or smallest value from a list of any number of expressions" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)). They were only *standardized* in SQL:2023 (feature **T054**) ([modern-sql.com — SQL:2023](https://modern-sql.com/standard/2023)) — long implemented, newly standard. They are **row-wise** (compare columns across one row), unlike `MAX`/`MIN` which aggregate down a column.

The portability landmine is NULL handling, which differs by engine:

> "NULL values in the argument list are ignored. The result will be NULL only if all the expressions evaluate to NULL. (This is a deviation from the SQL standard. According to the standard, the return value is NULL if any argument is NULL.)"
> — [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html)

So `GREATEST(10, NULL, 20)` is **20** on PostgreSQL/MySQL (NULL ignored) but **NULL** under the standard (and on Oracle). Same call, different answer. Don't rely on the ignore-NULL behavior; coalesce first if the inputs can be NULL.

```sql
-- WRONG — relies on engine-specific NULL handling; result differs PG vs Oracle
SELECT GREATEST(bid_a, bid_b, bid_c) AS top_bid FROM auctions;

-- RIGHT — make NULL handling explicit so every engine agrees
SELECT GREATEST(COALESCE(bid_a, 0), COALESCE(bid_b, 0), COALESCE(bid_c, 0)) AS top_bid
FROM auctions;
```

The other SQL:2023 scalar additions in the same family — `LPAD`/`RPAD` (T055) and multi-character `TRIM` (T056) — are now standard but only on recent engines; `ANY_VALUE` (T626) is an aggregate that belongs to `sql-aggregation-and-grouping`. Treat all as "portable in intent, gate on version."

---

## 8. Portability Snapshot

The standard spellings below run everywhere; the vendor columns are why the same logic breaks on a different engine. Route every dialect spelling to **`sql-standard-vs-dialect-map`**.

| Behavior | Standard | PostgreSQL | MySQL | SQL Server | Oracle |
|---|---|---|---|---|---|
| Null default | `COALESCE` | `COALESCE` | `COALESCE`/`IFNULL` | `COALESCE`/`ISNULL` | `COALESCE`/`NVL` |
| Conditional | `CASE` | `CASE` | `CASE`/`IF()` | `CASE`/`IIF` | `CASE`/`DECODE` |
| String concat | `\|\|` | `\|\|` | `CONCAT()` (no `\|\|` by default) | `+`/`CONCAT` | `\|\|` |
| `'a' \|\| NULL` | `NULL` | `NULL` | `NULL` | `NULL` | `'a'` (deviation) |
| Substring | `SUBSTRING(s FROM n FOR m)` | both | `SUBSTR` too | `SUBSTRING(s,n,m)` | `SUBSTR` |
| Cast | `CAST(x AS t)` | `CAST`/`::` | `CAST` | `CAST`/`CONVERT` | `CAST` |
| `GREATEST(x, NULL)` | `NULL` | ignores NULL | ignores NULL | (no `GREATEST`; use `CASE`) | `NULL` |
| `LPAD`/`RPAD` | SQL:2023 | yes | yes | no (use `RIGHT`+`REPLICATE`) | yes |

The two cells that bite hardest: Oracle's `'a' || NULL → 'a'` (because `'' IS NULL` there — foundation §11), and `GREATEST`'s NULL divergence (§7). When in doubt, `COALESCE` the operands and you remove the engine dependence.

---

## 9. Who Suffers When Expressions Go Wrong

These mistakes don't raise errors — they ship wrong strings and wrong numbers:

- The **analyst** reading a customer report where a whole column of "full name" rows came back blank, because `first || ' ' || middle || ' ' || last` hit a NULL middle name and `||` nullified the entire field (§4). No error fired; the export just had empty cells, and a mail-merge went out addressed to nobody.
- The **on-call engineer** paged at 2 a.m. because a query that ran fine in staging (PostgreSQL) crashed in production (Oracle) on `STRPOS`/`IFNULL` — vendor functions that simply don't exist on the target engine (§2, §5). The standard `POSITION`/`COALESCE` would have run on both.
- The **data scientist** whose "highest bid" feature was silently NULL on every row with a missing sub-bid after a database migration, because `GREATEST` ignored NULLs on the old engine and propagated them on the new one (§7).
- The **next maintainer** who can't move the codebase off one vendor because `::`, `+`-concat, `NVL`, and `CONVERT` are sprinkled through every query (§6, §8).

Writing the standard spelling and coalescing nullable operands is empathy in code: the report stays populated, the migration doesn't crash, and the next engine swap is a config change, not a rewrite.

---

## 10. Routing to Related Skills

- **`sql-relational-and-null-discipline`** (foundation) — *why* `||` and arithmetic propagate NULL, three-valued logic, the sentinel antipattern, Oracle's empty-string equivalence. Every NULL claim here routes back to it.
- **`sql-datetime-and-intervals`** — date/time `CAST`, typed literals, `EXTRACT`, and temporal formatting (§6 hands off here).
- **`sql-data-types-and-numerics`** — numeric precision, rounding, and overflow behind `CAST(x AS NUMERIC(p,s))` (§6).
- **`sql-standard-vs-dialect-map`** — the full per-engine spelling table for every vendor function named here (`IFNULL`/`NVL`/`ISNULL`/`IIF`, `SUBSTR`/`LEFT`/`RIGHT`/`STRPOS`/`INSTR`, `+`, `::`/`CONVERT`).
- **`sql-json`** — JSON value extraction and the `CAST`-to/from-JSON conversions that are out of scope here.

---

## 11. Reference Files

High-frequency expression anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
