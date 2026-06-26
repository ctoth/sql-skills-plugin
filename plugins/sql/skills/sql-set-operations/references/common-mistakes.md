# Common SQL Set-Operation Mistakes

Anti-patterns in LLM-generated SQL around `UNION`/`UNION ALL`/`INTERSECT`/`EXCEPT` and `VALUES`, each
with wrong/right code and a primary-source citation. The skill (`sql-set-operations`) states the rules;
this file holds the high-frequency failure modes. All RIGHT examples are standard/portable SQL;
non-standard spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Reflexive `UNION` where `UNION ALL` was correct (the cost trap)

**The problem:** The model writes `UNION` by habit to combine two branches that cannot overlap (or
where duplicates are wanted). `UNION` "eliminates duplicate rows from its result, in the same way as
`DISTINCT`, unless `UNION ALL` is used" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)) — a full sort/hash dedup pass over the whole combined result, for nothing.

```sql
-- WRONG — branches are disjoint by region; UNION still sorts/dedups the entire result
SELECT id FROM users WHERE region = 'EU'
UNION
SELECT id FROM users WHERE region = 'US';

-- RIGHT — append without the dedup pass
SELECT id FROM users WHERE region = 'EU'
UNION ALL
SELECT id FROM users WHERE region = 'US';
```

*Source: [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html). Depth: this skill, §1.*

---

## 2. `UNION` silently dropping legitimately-duplicate rows

**The problem:** The model uses `UNION` for its append behavior, unaware it also de-duplicates — so two
genuinely distinct real-world records that happen to have identical column values collapse into one,
and a count comes out short with no error.

```sql
-- WRONG — two real refunds of the same amount on the same day collapse to one row
SELECT amount, refunded_on FROM refunds_q1
UNION
SELECT amount, refunded_on FROM refunds_q2;

-- RIGHT — keep every row; dedup only when you truly mean "distinct"
SELECT amount, refunded_on FROM refunds_q1
UNION ALL
SELECT amount, refunded_on FROM refunds_q2;
```

*Source: [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html). Depth: this skill, §1.*

---

## 3. Assuming set ops treat NULLs like `=` does

**The problem:** The model expects two NULL rows to stay separate (because `NULL = NULL` is UNKNOWN in
a `WHERE`). But for compound-query dedup, "NULL values are considered equal to other NULL values"
([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)) — so they collapse, and
`INTERSECT`/`EXCEPT` match NULLs to each other. Opposite of comparison logic.

```sql
-- Two identical (1, NULL) rows do NOT survive as two under UNION — they fold into one:
SELECT 1, NULL
UNION
SELECT 1, NULL;          -- => single row (1, NULL)

-- Use UNION ALL if both NULL rows must be preserved:
SELECT 1, NULL
UNION ALL
SELECT 1, NULL;          -- => two rows
```

*Source: [SQLite — Compound Select Statements](https://www.sqlite.org/lang_select.html#compound_select_statements). Depth: this skill, §2; NULL theory owned by `sql-relational-and-null-discipline`.*

---

## 4. Reinventing `INTERSECT` / `EXCEPT` with joins or `IN`

**The problem:** The model rebuilds whole-row intersection or difference with a self-join, `IN`, or
`NOT EXISTS` when the branches are full row-sets. `INTERSECT` "returns all rows that are both in the
result of query1 and in the result of query2"; `EXCEPT` returns "rows that are in the result of query1
but not in the result of query2" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)).

```sql
-- WRONG — verbose, and the NOT IN form is NULL-unsafe
SELECT email FROM all_subscribers
WHERE email NOT IN (SELECT email FROM unsubscribed);

-- RIGHT — native set difference over whole rows
SELECT email FROM all_subscribers
EXCEPT
SELECT email FROM unsubscribed;
```

*Source: [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html). Depth: this skill, §3; key-based anti-join (`NOT EXISTS`) owned by `sql-subqueries-and-exists`.*

---

## 5. Expecting `INTERSECT ALL` / `EXCEPT ALL` everywhere

**The problem:** The model writes `INTERSECT ALL` / `EXCEPT ALL` for multiset semantics, but the target
engine lacks them. Standard SQL + PostgreSQL keep duplicates with `[ALL]`; SQLite has no
`INTERSECT ALL`/`EXCEPT ALL` (its grammar attaches `ALL` only to `UNION`), and Oracle lacks the
multiset variants too.

```sql
-- WRONG (on SQLite/Oracle) — INTERSECT ALL / EXCEPT ALL is unsupported there
SELECT product_id FROM cart_a
INTERSECT ALL
SELECT product_id FROM cart_b;

-- RIGHT (portable) — plain INTERSECT (dedup); emulate multiset with a counting GROUP BY if needed
SELECT product_id FROM cart_a
INTERSECT
SELECT product_id FROM cart_b;
```

*Source: [SQLite — Compound Select Statements](https://www.sqlite.org/lang_select.html#compound_select_statements). Depth: this skill, §3; dialect gaps owned by `sql-standard-vs-dialect-map`.*

---

## 6. Mismatched column count or position across branches

**The problem:** A count mismatch errors loudly, but a *positional* mismatch with compatible types is
silent: queries combine by ordinal position, and "the two queries must be \"union compatible\" ...
they return the same number of columns and the corresponding columns have compatible data types"
([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)) — names are ignored.

```sql
-- WRONG — columns pair by POSITION; this unions name↔email and email↔name
SELECT name, email FROM customers
UNION ALL
SELECT email, name FROM leads;

-- RIGHT — identical order and types in both branches; no SELECT * in a compound query
SELECT name, email FROM customers
UNION ALL
SELECT name, email FROM leads;
```

*Source: [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html). Depth: this skill, §4.*

---

## 7. `ORDER BY` / `LIMIT` placed on a branch instead of the compound

**The problem:** The model attaches `ORDER BY`/`LIMIT`/`FETCH` to the last branch expecting per-branch
behavior. The clause "may only occur at the end of the entire compound SELECT" and applies "to the
output of the set operation rather than one of its inputs" ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements); [PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)).

```sql
-- WRONG — intent "10 newest from each"; LIMIT applies to the COMBINED result (10 total)
SELECT id, ts FROM archive
UNION ALL
SELECT id, ts FROM live
ORDER BY ts DESC FETCH FIRST 10 ROWS ONLY;

-- RIGHT — parenthesize each branch to bound it; one final ORDER BY for the whole result
(SELECT id, ts FROM archive ORDER BY ts DESC FETCH FIRST 10 ROWS ONLY)
UNION ALL
(SELECT id, ts FROM live    ORDER BY ts DESC FETCH FIRST 10 ROWS ONLY)
ORDER BY ts DESC;
```

*Source: [SQLite — Compound Select Statements](https://www.sqlite.org/lang_select.html#compound_select_statements); [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html). Depth: this skill, §5.*

---

## 8. Mixing `UNION` and `INTERSECT` without parentheses

**The problem:** The model assumes left-to-right evaluation, but in standard SQL/PostgreSQL "INTERSECT
binds more tightly than" UNION/EXCEPT ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)) — and SQLite groups everything left-to-right, so the *same* query yields different sets per engine.

```sql
-- WRONG (ambiguous) — means a UNION (b INTERSECT c) on PostgreSQL, ((a UNION b) INTERSECT c) on SQLite
SELECT id FROM a
UNION
SELECT id FROM b
INTERSECT
SELECT id FROM c;

-- RIGHT — parenthesize the intent explicitly; portable and unambiguous
SELECT id FROM a
UNION
(SELECT id FROM b INTERSECT SELECT id FROM c);
```

*Source: [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html); [SQLite — Compound Select Statements](https://www.sqlite.org/lang_select.html#compound_select_statements). Depth: this skill, §5; dialect gap owned by `sql-standard-vs-dialect-map`.*

---

## 9. Building a constant table with a `UNION ALL` chain instead of `VALUES`

**The problem:** The model stitches inline literal rows together with `SELECT ... UNION ALL SELECT ...`.
`VALUES` "computes a row value or set of row values ... most commonly used to generate a \"constant
table\"" and "is syntactically allowed anywhere that `SELECT` is" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-values.html)).

```sql
-- WRONG — a UNION ALL chain to express three constant rows
SELECT 'EU' AS region, 0.20 AS vat
UNION ALL SELECT 'US', 0.00
UNION ALL SELECT 'JP', 0.10;

-- RIGHT — VALUES as a table constructor with an explicit column alias
SELECT * FROM (VALUES ('EU', 0.20), ('US', 0.00), ('JP', 0.10)) AS v(region, vat);
```

*Source: [PostgreSQL — VALUES](https://www.postgresql.org/docs/current/sql-values.html). Depth: this skill, §6.*

---

## 10. Relying on default `VALUES` column names

**The problem:** The model references `column1`/`column2` (or whatever the engine emits) instead of
naming the columns. "The default column names for `VALUES` are `column1`, `column2`, etc. in
PostgreSQL, but these names might be different in other database systems" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-values.html)) — unportable and unreadable.

```sql
-- WRONG — depends on engine-specific default names
SELECT column1 AS region FROM (VALUES ('EU'), ('US')) ;

-- RIGHT — name the columns in the table alias
SELECT region FROM (VALUES ('EU'), ('US')) AS v(region);
```

*Source: [PostgreSQL — VALUES](https://www.postgresql.org/docs/current/sql-values.html). Depth: this skill, §6.*
