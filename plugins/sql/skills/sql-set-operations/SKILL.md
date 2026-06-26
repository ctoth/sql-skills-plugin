---
name: sql-set-operations
description: Guides SQL set operations and the default-to-UNION cost trap — UNION sorts and de-duplicates its whole combined result (expensive) while UNION ALL just appends, so use UNION ALL whenever duplicates are impossible or wanted. Covers INTERSECT/EXCEPT (native set intersection and difference, so they aren't reinvented with joins or NOT EXISTS) and their [ALL] multiset forms, union-compatibility (columns align by position not name, equal count, compatible types), where ORDER BY/FETCH are legal (only on the final compound query — parentheses isolate a branch), the precedence rule (INTERSECT binds tighter than UNION/EXCEPT in standard SQL), that set ops treat NULLs as NOT distinct (two NULLs collapse — the opposite of `=`), and VALUES as an inline table constructor. Auto-invokes when writing or editing UNION/UNION ALL/INTERSECT/EXCEPT, compound SELECTs, VALUES row-set constructors, or on "combine two queries" / "rows in A but not B" / "why are my duplicates gone" / "MINUS" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Set Operations

> "UNION effectively appends the result of query2 to the result of query1 ... Furthermore, it eliminates duplicate rows from its result, in the same way as DISTINCT, unless UNION ALL is used."
> — [PostgreSQL — Combining Queries](https://www.postgresql.org/docs/current/queries-union.html)

> "For the purposes of determining duplicate rows for the results of compound SELECT operators, NULL values are considered equal to other NULL values and distinct from all non-NULL values."
> — [SQLite — Compound Select Statements](https://www.sqlite.org/lang_select.html#compound_select_statements)

A compound query stacks two or more `SELECT`s with `UNION`, `UNION ALL`, `INTERSECT`, or `EXCEPT` to compute set union, intersection, and difference over **whole rows**. Two facts from the foundation (`sql-relational-and-null-discipline`) decide everything here: a query result is a *set*, and `NULL` is the unknown marker. Set operators lean hard on both — they treat the result as a set (so they de-duplicate, and `UNION` pays to sort), and they fold NULLs together for dedup even though `NULL = NULL` is UNKNOWN in a `WHERE`. This skill owns the operators; the NULL theory routes back to foundation.

---

## 1. `UNION` vs `UNION ALL` — the Default-to-`UNION` Cost Trap

`UNION` does two things: it appends `query2` to `query1` **and** "eliminates duplicate rows from its result, in the same way as `DISTINCT`, unless `UNION ALL` is used" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)). That dedup is not free — it sorts or hashes the entire combined result to find duplicates. `UNION ALL` skips it: it "returns all the rows from the SELECT to the left ... and all the rows from the SELECT to the right" ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)).

The reflex is to write `UNION` everywhere. When the branches are disjoint by construction (different `WHERE` ranges, different tables with no overlap) or when duplicates are *wanted*, `UNION` does a full dedup pass for nothing — a slow query hammering prod for an answer `UNION ALL` would give correctly and cheaply.

```sql
-- WRONG — disjoint branches (no id can be both), but UNION still sorts/dedups the whole result
SELECT id, name FROM active_users   WHERE region = 'EU'
UNION
SELECT id, name FROM active_users   WHERE region = 'US';

-- RIGHT — disjoint by construction; append without the dedup pass
SELECT id, name FROM active_users   WHERE region = 'EU'
UNION ALL
SELECT id, name FROM active_users   WHERE region = 'US';
```

Rule of thumb: reach for `UNION ALL` by default; use bare `UNION` **only** when you actually need duplicate elimination and have measured that you do. Note also that `UNION` gives "no guarantee" of row order — it is still a set, so add `ORDER BY` if order matters (§5).

---

## 2. Set Operators Treat NULLs as NOT Distinct (Opposite of `=`)

For deduping a compound result, "NULL values are considered equal to other NULL values and distinct from all non-NULL values" ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)). So two rows that are NULL in the same column **collapse into one** under `UNION`/`INTERSECT`/`EXCEPT`, and a NULL row in one branch *matches* a NULL row in the other for `INTERSECT`/`EXCEPT`.

This is the exact reversal of comparison logic: in a `WHERE`, `NULL = NULL` is UNKNOWN and nothing matches a NULL (foundation §3). In a set operation, NULLs are *not distinct* — they match. Same two NULLs, opposite outcome depending on whether you are comparing or set-deduping.

```sql
-- Both rows are (1, NULL). Under UNION they are "equal" and collapse to ONE row:
SELECT 1, NULL
UNION
SELECT 1, NULL;          -- => a single row (1, NULL), not two

-- INTERSECT matches the NULLs to each other:
SELECT 1, NULL
INTERSECT
SELECT 1, NULL;          -- => (1, NULL) — the NULLs are treated as not distinct
```

This mirrors how `GROUP BY`/`DISTINCT` fold NULLs (foundation §9). The *why* — three-valued logic and the grouping/uniqueness duality — is owned by `sql-relational-and-null-discipline`; route there. This skill only flags that set ops sit on the "fold NULLs" side of the line.

---

## 3. `INTERSECT` and `EXCEPT` — Don't Reinvent Them, and `[ALL]` Is Multiset

`INTERSECT` "returns all rows that are both in the result of query1 and in the result of query2" and `EXCEPT` "returns all rows that are in the result of query1 but not in the result of query2 ... (sometimes called the *difference*)" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)). These are native set intersection and difference over whole rows. LLMs routinely rebuild them with self-joins, `IN`, or `NOT EXISTS` — when the branches are full row-sets, the operator is clearer and the engine optimizes it directly.

By default both **eliminate duplicates**: "Duplicate rows are eliminated unless `INTERSECT ALL` is used" / "unless `EXCEPT ALL` is used" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)). The `[ALL]` variants switch to **multiset (bag) semantics** — they keep duplicate copies instead of collapsing them:

- `INTERSECT ALL`: if a row appears *m* times on the left and *n* times on the right, it appears `min(m, n)` times in the result.
- `EXCEPT ALL`: a row appearing *m* times on the left and *n* times on the right appears `max(m - n, 0)` times.

```sql
-- Rows present in BOTH queries, duplicates collapsed:
SELECT product_id FROM cart_a
INTERSECT
SELECT product_id FROM cart_b;

-- Rows in the first query but not the second (set difference / anti-join over whole rows):
SELECT email FROM all_subscribers
EXCEPT
SELECT email FROM unsubscribed;
```

`EXCEPT` is a *set-based* anti-join (whole-row difference). The *predicate-based* anti-join — "rows in A with no matching row in B on a key" — is `NOT EXISTS`, and the `NOT IN`+NULL trap lives in **`sql-subqueries-and-exists`**. Pick `EXCEPT` when you are differencing identical column-shapes; pick `NOT EXISTS` when matching on a join key.

**Portability:** SQLite has **no** `INTERSECT ALL` / `EXCEPT ALL` — its grammar attaches `ALL` only to `UNION`, and INTERSECT/EXCEPT there always dedup ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)). Oracle spells `EXCEPT` as `MINUS`. Route both to `sql-standard-vs-dialect-map` (§6).

---

## 4. Columns Align by Position, Not Name — Equal Count, Compatible Types

To combine queries "the two queries must be \"union compatible\", which means that they return the same number of columns and the corresponding columns have compatible data types" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)); SQLite states it as "all the constituent SELECTs must return the same number of result columns" ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)). Pairing is strictly **by ordinal position** — the first column of each branch, the second of each, and so on. Column *names* are irrelevant to the matching; the result takes its names from the first query.

```sql
-- WRONG — columns line up by POSITION, so this unions name with email and email with name
SELECT name, email FROM customers
UNION ALL
SELECT email, name FROM leads;        -- silently mismatched; types may even coerce and "work"

-- RIGHT — same order, same types, in both branches
SELECT name, email FROM customers
UNION ALL
SELECT name, email FROM leads;
```

A *count* mismatch is a hard error; a *positional* mismatch with compatible types is silent and wrong — a real footgun when someone reorders one branch's SELECT list. List columns explicitly (never `SELECT *` in a compound query) and keep both branches in lockstep.

---

## 5. `ORDER BY` / `FETCH` Bind to the Whole Compound — Parentheses Isolate a Branch

The components of a compound query are simple SELECTs and "may not contain `ORDER BY` or `LIMIT` clauses. `ORDER BY` and `LIMIT` clauses may only occur at the end of the entire compound SELECT" ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)). A trailing clause attaches to the **final combined result**, not the last branch. PostgreSQL is explicit: without parentheses the clause "will be understood as applying to the output of the set operation rather than one of its inputs" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)) —

```text
SELECT a FROM b UNION SELECT x FROM y LIMIT 10
means   (SELECT a FROM b UNION SELECT x FROM y) LIMIT 10
not     SELECT a FROM b UNION (SELECT x FROM y LIMIT 10)
```

So to order or limit **one branch**, wrap that branch in parentheses; to order the whole result, put one `ORDER BY` at the very end.

```sql
-- WRONG — intent was "10 newest from each", but LIMIT applies to the combined result
SELECT id, ts FROM archive
UNION ALL
SELECT id, ts FROM live
ORDER BY ts DESC FETCH FIRST 10 ROWS ONLY;       -- 10 rows TOTAL, not 10 per branch

-- RIGHT — parenthesize each branch to bound it; one final ORDER BY for presentation
(SELECT id, ts FROM archive ORDER BY ts DESC FETCH FIRST 10 ROWS ONLY)
UNION ALL
(SELECT id, ts FROM live    ORDER BY ts DESC FETCH FIRST 10 ROWS ONLY)
ORDER BY ts DESC;
```

**Precedence.** When you mix operators without parentheses, "`UNION` and `EXCEPT` associate left-to-right, but `INTERSECT` binds more tightly than those two operators" ([PostgreSQL](https://www.postgresql.org/docs/current/queries-union.html)). So `query1 UNION query2 INTERSECT query3` means `query1 UNION (query2 INTERSECT query3)` — INTERSECT is the `*` to UNION/EXCEPT's `+`. **Caveat:** this is the standard/PostgreSQL rule; **SQLite gives INTERSECT no special precedence** and groups everything left-to-right, so a mixed compound means *different sets* on the two engines ([SQLite](https://www.sqlite.org/lang_select.html#compound_select_statements)). Always parenthesize mixed compounds explicitly — it is both clearer and portable.

---

## 6. `VALUES` as an Inline Table Constructor

`VALUES` "computes a row value or set of row values ... most commonly used to generate a \"constant table\" within a larger command, but it can be used on its own" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-values.html)). It is the clean way to introduce inline literal rows — instead of a chain of `SELECT 1 UNION ALL SELECT 2 ...`. Multi-row form is comma-separated tuples, and "all the rows must have the same number of elements" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-values.html)).

Because "`VALUES` is syntactically allowed anywhere that `SELECT` is" and "can also be used where a sub-SELECT might be written, for example in a `FROM` clause" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-values.html)), it works as a derived table you can join against or feed into a set operation:

```sql
-- WRONG — reinventing a constant table with a UNION ALL chain
SELECT 'EU' AS region, 0.20 AS vat
UNION ALL SELECT 'US', 0.00
UNION ALL SELECT 'JP', 0.10;

-- RIGHT — VALUES is the table constructor; alias names the columns (defaults are unportable)
SELECT * FROM (VALUES ('EU', 0.20), ('US', 0.00), ('JP', 0.10)) AS v(region, vat);

-- RIGHT — join real rows against inline data
SELECT o.id, o.amount * v.vat AS tax
FROM orders o
JOIN (VALUES ('EU', 0.20), ('US', 0.00)) AS v(region, vat) ON o.region = v.region;
```

Always supply column names with `AS v(...)`: "the default column names for `VALUES` are `column1`, `column2`, etc. in PostgreSQL, but these names might be different in other database systems" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-values.html)). Bulk-row `VALUES` for paging/seed data ties into `sql-pagination-and-keyset`.

---

## 7. Portability Snapshot

The standard says: `UNION`/`INTERSECT`/`EXCEPT` dedup, `[ALL]` keeps multiset duplicates, columns match by position, NULLs are not distinct for dedup, `ORDER BY`/`FETCH` only on the final compound, and INTERSECT binds tighter than UNION/EXCEPT. The deviations to watch:

| Behavior | PostgreSQL | SQLite | MySQL | Oracle |
|---|---|---|---|---|
| `UNION` / `UNION ALL` | yes | yes | yes | yes |
| `INTERSECT` / `EXCEPT` | yes | yes | **8.0.31+ only** | `EXCEPT` **spelled `MINUS`** |
| `INTERSECT ALL` / `EXCEPT ALL` (multiset) | yes | **no** | 8.0.31+ | **no** |
| NULLs not distinct for dedup | yes | yes | yes | yes |
| `INTERSECT` binds tighter than `UNION`/`EXCEPT` | yes | **no (all left-to-right)** | yes | yes |
| `VALUES` as standalone / FROM-clause table | yes | yes | yes (`ROW()`/`VALUES` vary) | via `SELECT ... FROM dual` |

The sharp edges: MySQL gained `INTERSECT`/`EXCEPT` only in **8.0.31** (MariaDB 10.3+) — older versions lack them entirely; Oracle spells set difference **`MINUS`**, not `EXCEPT`, and lacks the `[ALL]` multiset variants; SQLite has no `INTERSECT ALL`/`EXCEPT ALL` and gives INTERSECT no precedence. For all dialect spellings and version gaps, route to **`sql-standard-vs-dialect-map`**.

---

## 8. Who Suffers When Set Operations Are Mishandled

A set-operation mistake compiles and runs — it just returns the wrong rows or burns the wrong amount of CPU, and someone downstream pays:

- The **analyst** whose reconciliation report is short a handful of rows because a bare `UNION` silently folded *legitimately duplicate* records — two real customers with the same name and a NULL middle initial collapsed into one (§1, §2). No error; the count was just quietly wrong in a board deck.
- The **on-call engineer** watching a dashboard query peg a CPU on prod because someone wrote `UNION` across two large disjoint partitions, forcing a full sort/dedup pass that `UNION ALL` would have skipped entirely (§1).
- The **next maintainer** who reorders one branch's `SELECT` list for readability and silently swaps `name` and `email` in the merged result, because compound queries pair columns by position, not name, and the types happened to be compatible (§4).
- The **developer** porting a working `a UNION b INTERSECT c` query from PostgreSQL to SQLite and getting a *different set* back, because INTERSECT binds tighter on one engine and not the other (§5).

The `UNION ALL` you wrote when duplicates were impossible, the explicit column lists in both branches, and the parentheses around a mixed compound are all gifts to whoever runs this query at scale.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — **foundation**: set semantics, three-valued logic, and *why* set ops fold NULLs while `=` does not (§2).
- `sql-subqueries-and-exists` — the predicate-based anti-join (`NOT EXISTS`) and the `NOT IN`+NULL trap, versus `EXCEPT`'s whole-row set difference (§3).
- `sql-pagination-and-keyset` — bulk-row `VALUES` and limiting/ordering compound results (§5, §6).
- `sql-standard-vs-dialect-map` — Oracle `MINUS`, MySQL 8.0.31+ `INTERSECT`/`EXCEPT`, SQLite's missing `[ALL]` multiset variants and left-to-right precedence (§7).

---

## 10. Reference Files

High-frequency set-operation anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
