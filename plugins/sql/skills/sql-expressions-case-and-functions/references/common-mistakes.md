# Common SQL Expression Mistakes

Anti-patterns in LLM-generated SQL around `CASE`, `COALESCE`/`NULLIF`/`GREATEST`/`LEAST`, string
functions, concatenation, and `CAST`, each with wrong/right code and a primary-source citation. The
skill (`sql-expressions-case-and-functions`) states the rules; this file holds the high-frequency
failure modes. All RIGHT examples are standard/portable SQL; non-standard spellings are flagged and
routed to `sql-standard-vs-dialect-map`. NULL theory itself is owned by the foundation skill
`sql-relational-and-null-discipline`.

---

## 1. `CASE` with no `ELSE` silently returning NULL

**The problem:** The model writes a `CASE` covering the "expected" branches and omits `ELSE`. Any row
that matches no `WHEN` gets NULL — "if the `ELSE` clause is omitted and no condition is true, the
result is null" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)) — and that NULL then propagates through any later arithmetic or concatenation.

```sql
-- WRONG — scores below 80 yield NULL grade, not a value
SELECT CASE WHEN score >= 90 THEN 'A' WHEN score >= 80 THEN 'B' END AS grade FROM exams;

-- RIGHT — explicit default
SELECT CASE WHEN score >= 90 THEN 'A' WHEN score >= 80 THEN 'B' ELSE 'F' END AS grade FROM exams;
```

*Source: [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html). Depth: skill §1.*

---

## 2. Simple `CASE ... WHEN NULL` that never matches

**The problem:** The model writes `CASE status WHEN NULL THEN 'unknown' ...` expecting it to catch NULL
status. Simple `CASE` compares by equality, and `status = NULL` is UNKNOWN, never TRUE, so the branch
never fires (foundation §3).

```sql
-- WRONG — the WHEN NULL branch is dead code
SELECT CASE status WHEN 'active' THEN 1 WHEN NULL THEN -1 ELSE 0 END FROM jobs;

-- RIGHT — use a searched CASE with IS NULL
SELECT CASE WHEN status IS NULL THEN -1 WHEN status = 'active' THEN 1 ELSE 0 END FROM jobs;
```

*Source: [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html); foundation §3. Depth: skill §1.*

---

## 3. Vendor null-substitution functions instead of `COALESCE`

**The problem:** The model emits `IFNULL`/`NVL`/`ISNULL`/`IIF`, which exist only on one engine and crash
on the others. `COALESCE` "returns the first of its arguments that is not null" and is standard
([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)).

```sql
-- WRONG — engine-specific; each fails on the others
SELECT NVL(nickname, 'guest') FROM users;       -- Oracle only
SELECT ISNULL(nickname, 'guest') FROM users;     -- SQL Server only

-- RIGHT — standard, and supports a fallback chain
SELECT COALESCE(nickname, full_name, 'guest') FROM users;
```

*Source: [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html). Depth: skill §2; dialect map owned by `sql-standard-vs-dialect-map`.*

---

## 4. Unguarded division that raises divide-by-zero

**The problem:** The model divides by a column that can be 0, crashing the query. `NULLIF(y, 0)` turns
the 0 into NULL so the division yields NULL instead of erroring — "`NULLIF` returns a null value if
value1 equals value2" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)).

```sql
-- WRONG — "division by zero" the first time orders = 0
SELECT revenue / orders AS avg_order_value FROM stores;

-- RIGHT — NULLIF guards the divisor
SELECT revenue / NULLIF(orders, 0) AS avg_order_value FROM stores;
```

*Source: [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html). Depth: skill §3.*

---

## 5. Concatenation silently nullified by one NULL operand

**The problem:** The model builds a string with `||` over a nullable column; one NULL makes the entire
result NULL, blanking the field. Concatenation propagates NULL (foundation §6).

```sql
-- WRONG — full_name is NULL whenever middle_name IS NULL
SELECT first_name || ' ' || middle_name || ' ' || last_name AS full_name FROM people;

-- RIGHT — coalesce the nullable parts to ''
SELECT first_name || COALESCE(' ' || middle_name, '') || ' ' || last_name AS full_name FROM people;
```

*Source: [PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html); foundation §6. Depth: skill §4.*

---

## 6. Using `+` for string concatenation

**The problem:** The model (often coming from SQL Server) concatenates strings with `+`. That is
SQL Server-specific, collides with numeric addition, and is not the standard operator — standard SQL
and PostgreSQL use `||` (`'Post' || 'greSQL' → PostgreSQL`) ([PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html)).

```sql
-- WRONG — '+' concat is SQL Server only; elsewhere it's numeric add or an error
SELECT first_name + ' ' + last_name AS full_name FROM people;

-- RIGHT — standard concatenation operator
SELECT first_name || ' ' || last_name AS full_name FROM people;
```

*Source: [PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html). Depth: skill §4, §8; dialect map owned by `sql-standard-vs-dialect-map`.*

---

## 7. Vendor string functions with reversed/positional arguments

**The problem:** The model reaches for `SUBSTR`/`LEFT`/`RIGHT`/`STRPOS`/`INSTR`/`LENGTH`. These are
vendor spellings; `STRPOS(s, sub)` even reverses `POSITION(sub IN s)`'s arguments, producing a subtle
wrong answer rather than an error.

```sql
-- WRONG — non-standard, and STRPOS reverses POSITION's argument order
SELECT SUBSTR(code, 1, 3), LEFT(name, 1), STRPOS(email, '@') FROM accounts;

-- RIGHT — standard keyword forms
SELECT SUBSTRING(code FROM 1 FOR 3),
       SUBSTRING(name FROM 1 FOR 1),
       POSITION('@' IN email)
FROM accounts;
```

*Source: [PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html). Depth: skill §5; dialect map owned by `sql-standard-vs-dialect-map`.*

---

## 8. Non-standard `TRIM(s, chars)` comma form

**The problem:** The model writes `TRIM(name, 'xyz')`. The standard form uses the
`[LEADING|TRAILING|BOTH] chars FROM string` keyword syntax — `TRIM(BOTH 'xyz' FROM 'yxTomxx') → Tom`
([PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html)); the comma form is non-standard.

```sql
-- WRONG — comma form is not standard SQL
SELECT TRIM(name, ' x') FROM accounts;

-- RIGHT — standard keyword form (multi-character TRIM is SQL:2023 T056)
SELECT TRIM(BOTH ' x' FROM name) FROM accounts;
```

*Source: [PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html); [modern-sql.com — SQL:2023](https://modern-sql.com/standard/2023). Depth: skill §5.*

---

## 9. `::` or `CONVERT` instead of standard `CAST`

**The problem:** The model uses PostgreSQL's `::type` shorthand or SQL Server's `CONVERT(type, x)`.
Neither is portable; `CAST(x AS type)` is the standard explicit conversion.

```sql
-- WRONG — :: is PostgreSQL-only; CONVERT is SQL Server-only
SELECT total::numeric(10,2), CONVERT(varchar, created_at) FROM invoices;

-- RIGHT — standard CAST runs everywhere
SELECT CAST(total AS NUMERIC(10,2)), CAST(created_at AS VARCHAR) FROM invoices;
```

*Source: standard SQL `CAST`; [PostgreSQL — String Functions](https://www.postgresql.org/docs/current/functions-string.html) context. Depth: skill §6; numeric precision owned by `sql-data-types-and-numerics`, dates by `sql-datetime-and-intervals`.*

---

## 10. Relying on implicit coercion across number/string boundaries

**The problem:** The model compares a numeric column to a string literal (or vice versa) and trusts
the engine to coerce. PostgreSQL may reject it; MySQL/SQLite coerce eagerly and can turn an indexed
lookup into a full scan or silently parse `'5abc'` as `5`. Make the conversion explicit.

```sql
-- WRONG — relies on engine-specific implicit coercion; behavior and index use vary
SELECT * FROM orders WHERE order_id = '00042';

-- RIGHT — convert deliberately so every engine agrees
SELECT * FROM orders WHERE order_id = CAST('00042' AS INTEGER);
```

*Source: standard SQL `CAST`; engine coercion divergence. Depth: skill §6; details owned by `sql-data-types-and-numerics`.*

---

## 11. Relying on `GREATEST`/`LEAST` NULL handling

**The problem:** The model assumes `GREATEST(a, b, c)` skips NULLs (true on PostgreSQL/MySQL) or
assumes it returns NULL on a NULL arg (true under the standard and on Oracle). The same call gives
different answers: "NULL values in the argument list are ignored ... This is a deviation from the SQL
standard. According to the standard, the return value is NULL if any argument is NULL" ([PostgreSQL — Conditional](https://www.postgresql.org/docs/current/functions-conditional.html)).

```sql
-- WRONG — result depends on the engine when any bid is NULL
SELECT GREATEST(bid_a, bid_b, bid_c) AS top_bid FROM auctions;

-- RIGHT — coalesce first so NULL handling is explicit and identical everywhere
SELECT GREATEST(COALESCE(bid_a, 0), COALESCE(bid_b, 0), COALESCE(bid_c, 0)) AS top_bid FROM auctions;
```

*Source: [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html); [modern-sql.com — SQL:2023](https://modern-sql.com/standard/2023) (GREATEST/LEAST standardized as T054). Depth: skill §7; dialect map owned by `sql-standard-vs-dialect-map`.*

---

## 12. Confusing `GREATEST`/`LEAST` (row-wise) with `MAX`/`MIN` (aggregate)

**The problem:** The model uses `MAX(a, b)` to get the larger of two columns in a row, or `GREATEST`
to find the maximum down a column. `GREATEST`/`LEAST` compare expressions *across one row*; `MAX`/`MIN`
aggregate *down a column* across rows.

```sql
-- WRONG — MAX is an aggregate; MAX(col_a, col_b) is not a row-wise max (and errors in most engines)
SELECT MAX(price_usd, price_eur) FROM products;

-- RIGHT — GREATEST compares values within the same row
SELECT GREATEST(price_usd, price_eur) AS higher_price FROM products;
```

*Source: [PostgreSQL — Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html). Depth: skill §7; column aggregation owned by `sql-aggregation-and-grouping`.*
