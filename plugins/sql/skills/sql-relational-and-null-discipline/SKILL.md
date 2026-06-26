---
name: sql-relational-and-null-discipline
description: Guides the core correctness floor of SQL — query results are unordered sets with no defined order unless you write ORDER BY, NULL means "unknown" and propagates UNKNOWN through every comparison and arithmetic expression, and WHERE/HAVING/ON keep only rows that evaluate to TRUE while CHECK rejects only rows that evaluate to FALSE (so UNKNOWN passes). Bans the recurring NULL traps — `x = NULL` (always UNKNOWN, use IS NULL), `col <> 'a'` silently dropping NULL rows, `NOT IN (subquery)` collapsing to zero rows on a single NULL, COUNT(col) silently undercounting versus COUNT(*), and SUM/AVG silently skipping NULL. Teaches null-safe equality IS [NOT] DISTINCT FROM and UNIQUE NULLS [NOT] DISTINCT (SQL:2023). Auto-invokes when writing or editing any WHERE/HAVING/ON/CHECK predicate, comparisons against possibly-null columns, NOT IN/`<>`/`!=`, COUNT/SUM/AVG over nullable columns, GROUP BY/ORDER BY/UNIQUE on nullable columns, or on "why does my query return no rows" / "why is my average wrong" / "handle NULL" requests. The policy root every other SQL skill routes back to.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Relational and NULL Discipline

> "SQL uses a _three-valued logic_: besides _true_ and _false_, the result of logical expressions can also be _unknown_. ... The logical value _unknown_ indicates that a result _actually_ depends on a `null` value."
> — [modern-sql.com — Three-Valued Logic](https://modern-sql.com/concept/three-valued-logic)

> "If sorting is not chosen, the rows will be returned in an unspecified order. The actual order in that case will depend on the scan and join plan types and the order on disk, but it must not be relied on."
> — [PostgreSQL — Sorting Rows (ORDER BY)](https://www.postgresql.org/docs/current/queries-order.html)

A relational table is a *set* of rows, and `NULL` is a marker for absent or unknown data — not zero, not the empty string, not "nothing." Those two facts decide everything below. They are the reason a query silently returns no rows, an average comes out wrong, or a "unique" column holds three NULLs. This is the policy root for the SQL plugin: every other SQL skill assumes the set semantics and three-valued logic stated here and routes back to it.

---

## 1. A Result Is a Set — Order Is Undefined Without `ORDER BY`

Codd's relational model defines a relation as a set of tuples in which "the ordering of rows is immaterial" and "all rows are distinct" ([Codd 1970, §1.3](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf)). A SQL result therefore has **no defined order** unless you ask for one. Relying on insertion order, primary-key order, or "the order it came out last time" is a latent bug: the order "will depend on the scan and join plan types and the order on disk, but it must not be relied on" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)).

```sql
-- WRONG — "first" row is undefined; the plan can change it at any time
SELECT id, name FROM users FETCH FIRST 1 ROWS ONLY;

-- RIGHT — an explicit, total order makes the result deterministic
SELECT id, name FROM users ORDER BY created_at, id FETCH FIRST 1 ROWS ONLY;
```

The same applies to `DISTINCT`, `GROUP BY`, set operations, and `FETCH/OFFSET` paging — none of them imply an order. If you need a deterministic result, the `ORDER BY` must be **total** (break ties on a unique column).

---

## 2. Three-Valued Logic — and the WHERE/CHECK Asymmetry (the centerpiece)

SQL logic has three values: TRUE, FALSE, and UNKNOWN, where UNKNOWN "indicates that a result _actually_ depends on a `null` value" ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)). The combination rules: `AND` is FALSE as soon as any operand is FALSE (otherwise UNKNOWN if any operand is UNKNOWN); `OR` is TRUE as soon as any operand is TRUE; and crucially `NOT (UNKNOWN)` is **still UNKNOWN** — you cannot negate your way out of a NULL ([Oracle — Nulls](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html)).

The single most important rule in this skill is **how each clause treats UNKNOWN**, because they are opposites:

- `WHERE`, `HAVING`, and join `ON` clauses "require _true_ conditions. It is not enough that a condition is not _false_" — a row is kept only if its predicate is TRUE, so UNKNOWN is dropped exactly like FALSE ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)).
- `CHECK` constraints "follow the reverse logic: they reject _false_, rather than accepting _true_," so "check constraints accept _true_ and _unknown_" — a row passes unless the predicate is definitively FALSE ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)).

Same predicate, same NULL, opposite outcome depending on the clause:

```sql
-- A row with qty = NULL:
WHERE qty > 0        -- NULL > 0 is UNKNOWN -> row is DROPPED (WHERE keeps only TRUE)
CHECK (qty > 0)      -- NULL > 0 is UNKNOWN -> row is ALLOWED (CHECK rejects only FALSE)
```

That asymmetry is why a `CHECK (qty > 0)` does **not** stop a NULL `qty` from being inserted — you need `qty INTEGER NOT NULL CHECK (qty > 0)`. Constraint syntax lives in `sql-constraints-and-integrity`; the logic is owned here.

---

## 3. `= NULL` Is Always UNKNOWN — Use `IS NULL`

"Ordinary comparison operators yield null (signifying 'unknown'), not true or false, when either input is null. For example, `7 = NULL` yields null, as does `7 <> NULL`" ([PostgreSQL — Comparison Operators](https://www.postgresql.org/docs/current/functions-comparison.html)). Nothing equals NULL — "not even `null` equals `null` because each `null` could be different" ([modern-sql.com — 3VL](https://modern-sql.com/concept/three-valued-logic)). So `WHERE col = NULL` matches nothing; the documentation is explicit: "Do _not_ write `expression = NULL`" ([PostgreSQL — Comparison](https://www.postgresql.org/docs/current/functions-comparison.html)).

```sql
-- WRONG — col = NULL is UNKNOWN for every row; returns the empty set
SELECT * FROM orders WHERE shipped_at = NULL;

-- RIGHT — the dedicated null test
SELECT * FROM orders WHERE shipped_at IS NULL;
```

---

## 4. `<>` and `!=` Silently Drop NULL Rows

Because both `=` and `<>` against NULL yield UNKNOWN, a predicate like `col <> 'a'` returns **only** rows where `col` is a non-NULL value different from `'a'`. Every NULL row is silently excluded, even though a NULL is certainly "not equal to `'a'`" in plain English.

```sql
-- WRONG — drops every row where status IS NULL, usually not intended
SELECT * FROM tasks WHERE status <> 'done';

-- RIGHT — say what you mean: not 'done', including the unknowns
SELECT * FROM tasks WHERE status <> 'done' OR status IS NULL;

-- RIGHT — null-safe, terser (see §7)
SELECT * FROM tasks WHERE status IS DISTINCT FROM 'done';
```

---

## 5. The `NOT IN (… NULL …)` Trap

This is the canonical footgun. `x NOT IN (a, b, NULL)` expands to `x <> a AND x <> b AND x <> NULL`. The last conjunct `x <> NULL` is UNKNOWN, so the whole `AND` can never be TRUE — it is at best UNKNOWN, which `WHERE` drops. **A single NULL anywhere in the list or subquery collapses `NOT IN` to zero matching rows.** `IN` is forgiving (it can still return TRUE on a real match); `NOT IN` is not.

```sql
-- WRONG — if ANY customer_id in the subquery is NULL, this returns NOTHING
SELECT * FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);
```

The fix is `NOT EXISTS`, which is null-safe and the portable anti-join. The full rewrite and the semi-join/anti-join discussion belong to **`sql-subqueries-and-exists`** — route there; this skill only flags the trap.

---

## 6. NULL Propagates Through Arithmetic and Concatenation

"Expressions and functions that process a `null` value generally return the `null` value" ([modern-sql.com — NULL](https://modern-sql.com/concept/null)); Oracle states it as "any arithmetic expression containing a null always evaluates to null" ([Oracle — Nulls](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html)). So `1 + NULL` is NULL, and `'foo ' || NULL || 'bar'` is NULL — a single missing column quietly nullifies an entire computed value.

```sql
-- WRONG — if discount IS NULL, total becomes NULL for that row
SELECT price - discount AS total FROM line_items;

-- RIGHT — substitute a neutral value with COALESCE before computing
SELECT price - COALESCE(discount, 0) AS total FROM line_items;
```

The logical operators are the documented exception (`NULL OR TRUE` is TRUE; `NULL AND FALSE` is FALSE), as are aggregates (§7). **Portability note:** standard SQL and PostgreSQL make `'a' || NULL` yield NULL, but Oracle treats NULL as the empty string in `||` (because Oracle treats `''` as NULL — see §9), so the same expression yields `'a'` there. Concatenation is engine-divergent; route dialect spellings to `sql-standard-vs-dialect-map`.

---

## 7. Aggregates: `COUNT(*)` vs `COUNT(col)`, and SUM/AVG Skip NULL

Aggregates do not propagate NULL — they **skip** it, which is its own trap. `count(*)` "computes the number of input rows," but `count(expression)` "computes the number of input rows in which the input value is not null" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). `sum`, `avg`, `min`, and `max` likewise operate only on "the non-null input values."

```sql
-- These ask DIFFERENT questions when phone has NULLs:
SELECT COUNT(*)             FROM users;  -- total rows
SELECT COUNT(phone)         FROM users;  -- rows with a non-null phone (undercount vs *)
SELECT COUNT(DISTINCT phone) FROM users; -- distinct non-null phones
```

Two consequences worth internalizing:

- **`AVG(col)` is not "treat NULL as 0."** It divides the sum of non-null values by the *count of non-null values*. Coalescing to 0 instead (`AVG(COALESCE(col,0))`) inflates the denominator and lowers the average — a different, usually wrong, number.
- **`SUM` over no rows (or all-NULL) returns NULL, not 0:** "`sum` of no rows returns null, not zero as one might expect" ([PostgreSQL — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)). Wrap in `COALESCE(SUM(x), 0)` when you need a zero. Aggregate-specific grouping rules live in `sql-aggregation-and-grouping`.

---

## 8. `IS [NOT] DISTINCT FROM` — Null-Safe Equality

When you need "equal, treating two NULLs as equal," use the standard null-safe comparison. "For non-null inputs, `IS DISTINCT FROM` is the same as the `<>` operator. However, if both inputs are null it returns false, and if only one input is null it returns true" ([PostgreSQL — Comparison](https://www.postgresql.org/docs/current/functions-comparison.html)). It "provides a null-aware equals comparison" where "two `null` values are not distinct from each other" ([modern-sql.com — NULL](https://modern-sql.com/concept/null)). Crucially it always returns TRUE or FALSE — **never UNKNOWN** — so it is safe in `WHERE` and in joins.

```sql
-- NULL IS NOT DISTINCT FROM NULL  -> TRUE   (the two unknowns "match")
-- 1    IS DISTINCT FROM NULL      -> TRUE
-- NULL IS NOT DISTINCT FROM 1     -> FALSE

-- RIGHT — a null-safe join that matches NULL keys to each other
SELECT * FROM a JOIN b ON a.k IS NOT DISTINCT FROM b.k;
```

**Portability:** this is an optional SQL feature, "still not widely supported." MySQL spells it `<=>` (the "spaceship" operator), which "is equivalent to the standard SQL `IS NOT DISTINCT FROM` operator" ([MySQL — Comparison Operators](https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html)). Route dialect spellings to `sql-standard-vs-dialect-map`.

---

## 9. NULL in GROUP BY, ORDER BY, and UNIQUE — the Grouping/Uniqueness Duality

NULL is treated **inconsistently** across these three, and that inconsistency is itself a known wart:

- **`GROUP BY` / `DISTINCT` / `UNION` fold all NULLs into one group** — they treat NULLs as *not distinct*. Across every major engine, "nulls are distinct in SELECT DISTINCT" is **No** ([SQLite — NULL Handling](https://www.sqlite.org/nulls.html)).
- **A default `UNIQUE` constraint treats NULLs as distinct** — the opposite: "for the purpose of a unique constraint, null values are not considered equal, unless `NULLS NOT DISTINCT` is specified" ([PostgreSQL — CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html)). So a default `UNIQUE` column permits **multiple** NULL rows. SQLite calls the split "puzzling ... NULLs should be either distinct everywhere or nowhere" ([SQLite — NULL Handling](https://www.sqlite.org/nulls.html)). The modern lever is the SQL:2023 `UNIQUE NULLS NOT DISTINCT` (PostgreSQL 15+), which permits at most one NULL.
- **`ORDER BY` placement of NULLs differs by engine.** PostgreSQL: "null values sort as if larger than any non-null value; that is, `NULLS FIRST` is the default for `DESC` order, and `NULLS LAST` otherwise" ([PostgreSQL — ORDER BY](https://www.postgresql.org/docs/current/queries-order.html)). MySQL/SQLite sort NULLs as smallest. For portable, deterministic ordering, always say it explicitly.

```sql
-- WRONG — NULL position is engine-dependent; result order differs across DBs
SELECT * FROM events ORDER BY ended_at DESC;

-- RIGHT — explicit and portable in intent
SELECT * FROM events ORDER BY ended_at DESC NULLS LAST;

-- RIGHT — enforce single-NULL uniqueness (SQL:2023; not all engines)
ALTER TABLE accounts ADD CONSTRAINT uq_email UNIQUE NULLS NOT DISTINCT (email);
```

(MySQL historically lacks `NULLS FIRST/LAST` and needs `ORDER BY col IS NULL, col`; route to `sql-standard-vs-dialect-map`.)

---

## 10. Don't Fear NULL Into a Sentinel

The reflex to "avoid NULL" by substituting a magic value — `0`, `-1`, `''`, `'N/A'`, `'unknown'` — is the "Fear of the Unknown" antipattern ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). A sentinel poisons aggregates (a `-1` placeholder corrupts `SUM`/`AVG`/`MIN`), defeats `IS NULL` testing, collides with legitimate data, and forces every query to special-case the magic value. NULL is the *correct* marker for absent or unknown data; the discipline is to handle it with three-valued logic, `IS NULL`, `IS DISTINCT FROM`, and `COALESCE` — and to declare `NOT NULL` only where absence is genuinely impossible.

---

## 11. Portability Snapshot

The standard says: comparisons to NULL are UNKNOWN; arithmetic propagates NULL; `DISTINCT`/`UNION` fold NULLs; `WHERE` keeps TRUE, `CHECK` rejects FALSE. Consensus across engines is strong on those; the deviations to watch:

| Behavior | PostgreSQL | SQLite | MySQL | Oracle |
|---|---|---|---|---|
| Arithmetic with NULL → NULL | yes | yes | yes | yes |
| `DISTINCT`/`UNION` fold NULLs together | yes | yes | yes | yes |
| NULLs distinct in a `UNIQUE` column (multi-NULL allowed) | yes | yes | yes | yes |
| `IS NOT DISTINCT FROM` available | yes | ~ (no spelling; use `IS`) | `<=>` | no native spelling |
| `NULLS FIRST/LAST` in `ORDER BY` | yes | yes | needs `col IS NULL` trick | yes |
| `'' ` (empty string) treated as NULL | no | no | no | **yes (deviation)** |

Oracle's empty-string-is-NULL equivalence is the most damaging cross-engine difference: "Oracle Database currently treats a character value with a length of zero as null ... Oracle recommends that you do not treat empty strings the same as nulls" ([Oracle — Nulls](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html)). Code that distinguishes `''` from NULL is not portable to/from Oracle. For all dialect spellings, route to **`sql-standard-vs-dialect-map`**.

One escape hatch worth knowing: the `IS [NOT] TRUE/FALSE/UNKNOWN` predicates "always return true or false, never null" ([PostgreSQL — Comparison](https://www.postgresql.org/docs/current/functions-comparison.html)), so `WHERE (a <> b) IS NOT TRUE` deliberately *includes* the NULL/UNKNOWN rows that a bare `a <> b` would drop.

---

## 12. Who Suffers When NULL Is Mishandled

A NULL mistake never errors at write time — it returns the wrong answer silently, and someone downstream pays:

- The **debugger** chasing a report that "returns no rows" — and after an hour finds a single NULL in a `NOT IN` subquery that collapsed the entire result to empty (§5). There was no error message; the query was "valid." The `NOT EXISTS` rewrite they didn't write is the bug they're now hunting.
- The **analyst** whose dashboard average is quietly wrong because `AVG` skipped the NULLs and they assumed it treated them as zero (§7) — a financial number off by a denominator, presented in a meeting as fact.
- The **on-call engineer** staring at three rows with a NULL email in a column they were told was `UNIQUE`, because a default `UNIQUE` constraint lets multiple NULLs through (§9). The "uniqueness guarantee" the schema seemed to promise was never there.
- The **next maintainer** who can't reproduce yesterday's "first row" because the query had no `ORDER BY` and the plan changed (§1).

Disciplined NULL handling is empathy in code: the `IS NULL` you wrote instead of `= NULL`, the `NOT EXISTS` instead of `NOT IN`, and the explicit `ORDER BY` are all gifts to whoever debugs this under load.

---

## 13. Routing to Related Skills

This skill is the **policy root**: every SQL skill in this plugin links back here for set semantics and three-valued logic, the way every Go skill routes to `go-idiomatic-discipline`. Outbound:

- `sql-subqueries-and-exists` — the `NOT IN` + NULL anti-join rewrite to `NOT EXISTS` (§5).
- `sql-joins` — `ON` vs `WHERE` placement and join-key nullability interacting with three-valued logic.
- `sql-aggregation-and-grouping` — `GROUP BY`/`HAVING`, functional-dependency rules, and aggregate NULL handling depth (§7).
- `sql-constraints-and-integrity` — `CHECK`/`UNIQUE`/`NOT NULL` syntax and the UNKNOWN-passes-CHECK pitfall (§2, §9).
- `sql-standard-vs-dialect-map` — dialect spellings of null-safe equality (`<=>`), `NULLS FIRST/LAST`, and the Oracle empty-string deviation.

---

## 14. Reference Files

High-frequency NULL/relational anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
