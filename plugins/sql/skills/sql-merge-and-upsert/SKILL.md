---
name: sql-merge-and-upsert
description: >-
  Guides atomic upsert — never a race-prone "SELECT then INSERT-or-UPDATE" in application code. Teaches the standard `MERGE` statement (`USING ... ON ...` with `WHEN MATCHED THEN UPDATE/DELETE` and `WHEN NOT MATCHED THEN INSERT`) and maps the real-world alternatives — `INSERT ... ON CONFLICT (col) DO UPDATE/DO NOTHING` in PostgreSQL/SQLite (with `excluded.col`) and `INSERT ... ON DUPLICATE KEY UPDATE` in MySQL/MariaDB (with the `new.col` row alias) — since `MERGE` is unevenly adopted (PostgreSQL only since v15. Auto-invokes when writing or editing `MERGE`, `INSERT ... ON CONFLICT` / `ON DUPLICATE KEY UPDATE`, any upsert / "insert-or-update" / "create-or-update" logic, or check-then-act read-modify-write code that selects a row and then decides to insert or update.
allowed-tools: Read, Glob, Grep
---

# SQL Merge and Upsert

> "An UPSERT is an ordinary INSERT statement that is followed by one or more ON CONFLICT clauses."
> — [SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html)

> "You may also wish to consider using `INSERT ... ON CONFLICT` as an alternative statement which offers the ability to run an `UPDATE` if a concurrent `INSERT` occurs. ... If a target row is modified more than once, a uniqueness violation or cardinality violation will occur."
> — [PostgreSQL — MERGE, Notes](https://www.postgresql.org/docs/current/sql-merge.html)

"Upsert" is "a portmanteau of update and insert" — a statement that "inserts a record ... if the record does not exist or, if the record already exists, updates the existing record" ([Wikipedia — Merge (SQL)](https://en.wikipedia.org/wiki/Merge_(SQL))). The hard part is not the spelling; it is doing it **atomically**, in one statement the database serializes — instead of the check-then-act pattern most application code reaches for first. This skill assumes the set semantics and three-valued logic of `sql-relational-and-null-discipline` and the constraint machinery of `sql-constraints-and-integrity`: an upsert is only meaningful against a `UNIQUE`/`PRIMARY KEY` constraint, so those two foundations decide everything below.

---

## 1. The Check-Then-Act Race — the centerpiece

The instinct is to SELECT, branch in the application, then INSERT or UPDATE. That is a **read-modify-write across a gap**: between the SELECT and the write, another session can change the world. It is the classic check-then-act concurrency bug.

```python
# WRONG — app-side select-then-write. Two sessions, same key, interleaved:
row = db.execute("SELECT 1 FROM users WHERE email = ?", [email]).fetchone()
if row is None:
    db.execute("INSERT INTO users (email, name) VALUES (?, ?)", [email, name])  # both sessions get here
else:
    db.execute("UPDATE users SET name = ? WHERE email = ?", [name, email])
```

Under load this fails two ways. If `email` is `UNIQUE`, both sessions see "absent," both INSERT, and the second INSERT raises a **duplicate-key error** (a crash). If it is *not* unique, you get **two duplicate rows**, or — when both take the UPDATE branch — a **lost update**, where one session's write silently overwrites the other's. No amount of `SELECT` retry closes the gap; the check and the act are separate statements.

The fix is a single atomic statement. The engine evaluates the conflict and the write together, serializing concurrent writers on the unique index:

```sql
-- RIGHT — one atomic upsert; the DB resolves the conflict, not the app
INSERT INTO users (email, name) VALUES (:email, :name)
ON CONFLICT (email) DO UPDATE SET name = excluded.name;
```

Why this is safe (and the precise isolation-level / MVCC behavior) is owned by `sql-transactions-and-isolation`; this skill's rule is simply: **never branch in application code between a read and the write it depends on — express insert-or-update as one statement.**

---

## 2. Standard `MERGE` — Syntax and Arms

`MERGE` was "officially introduced in the SQL:2003 standard, and expanded in the SQL:2008 standard" ([Wikipedia — Merge (SQL)](https://en.wikipedia.org/wiki/Merge_(SQL))). It joins a target table to a `data_source` on a condition, then applies arms by match status:

```sql
MERGE INTO target_table_name [ AS target_alias ]
    USING data_source ON join_condition
    WHEN MATCHED [ AND condition ] THEN { UPDATE SET ... | DELETE | DO NOTHING }
    WHEN NOT MATCHED [ AND condition ] THEN { INSERT (...) VALUES (...) | DO NOTHING }
```

That arm grammar is from the [PostgreSQL — MERGE Synopsis](https://www.postgresql.org/docs/current/sql-merge.html). A worked upsert:

```sql
-- RIGHT — standard MERGE: update on match, insert on no match
MERGE INTO inventory AS t
USING (VALUES (:sku, :qty)) AS s(sku, qty) ON t.sku = s.sku
WHEN MATCHED THEN UPDATE SET qty = t.qty + s.qty
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (s.sku, s.qty);
```

`MERGE` shines for **set-based** merges — synchronizing a whole staging table into a target with UPDATE + INSERT + DELETE arms in one pass — not just single-row upserts. PostgreSQL extends the standard with `WHEN NOT MATCHED BY SOURCE` (rows present in the target but absent from the source — useful for expiry/delete) and `WHEN NOT MATCHED [BY TARGET]` ([PostgreSQL — MERGE](https://www.postgresql.org/docs/current/sql-merge.html)):

```sql
-- RIGHT — full sync: update changed rows, insert new ones, delete rows no longer in the source
MERGE INTO inventory AS t
USING staging AS s ON t.sku = s.sku
WHEN MATCHED AND t.qty <> s.qty THEN UPDATE SET qty = s.qty
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (s.sku, s.qty)
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

The `AND condition` on an arm (here `t.qty <> s.qty`) makes the arm conditional — a use case `ON CONFLICT` cannot express as cleanly, which is why `MERGE` is the right tool for multi-arm set-based work even though `ON CONFLICT` wins for single-row upserts.

---

## 3. `INSERT ... ON CONFLICT` (PostgreSQL / SQLite)

The PostgreSQL/SQLite upsert. An ordinary INSERT followed by `ON CONFLICT (target) DO UPDATE SET ...` or `DO NOTHING` ([SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html)). The key subtlety is `excluded`:

> "Column names in the expressions of a DO UPDATE refer to the original unchanged value of the column, before the attempted INSERT. To use the value that would have been inserted had the constraint not failed, add the special 'excluded.' table qualifier to the column name." — [SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html)

```sql
-- RIGHT — bare column = stored row; excluded.col = the value you tried to insert
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON CONFLICT (url) DO UPDATE SET views = page_views.views + excluded.views;

-- RIGHT — insert if new, ignore (do not overwrite) if it already exists
INSERT INTO tags (name) VALUES (:name)
ON CONFLICT (name) DO NOTHING;
```

A trailing `WHERE` on the `DO UPDATE` lets you skip the write entirely when nothing changed — avoiding a no-op row version (and a redundant trigger/`updated_at` bump). The `ON CONFLICT (...) WHERE expr` and `DO UPDATE SET ... WHERE expr` forms are both in the [SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html) syntax:

```sql
-- RIGHT — only overwrite when the incoming name actually differs
INSERT INTO users (email, name) VALUES (:email, :name)
ON CONFLICT (email) DO UPDATE SET name = excluded.name
WHERE users.name IS DISTINCT FROM excluded.name;  -- null-safe compare; see foundation skill
```

```sql
-- WRONG — forgetting excluded: this resets views to the OLD stored value, a no-op
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON CONFLICT (url) DO UPDATE SET views = views + 1;  -- "views" here = old row, not the new 1
```

The conflict target `(url)` is not optional decoration — see §5.

---

## 4. `INSERT ... ON DUPLICATE KEY UPDATE` (MySQL / MariaDB)

MySQL's spelling. "If you specify an `ON DUPLICATE KEY UPDATE` clause and a row to be inserted would cause a duplicate value in a `UNIQUE` index or `PRIMARY KEY`, an `UPDATE` of the old row occurs" ([MySQL — INSERT ... ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)). Unlike `ON CONFLICT`, there is **no explicit conflict target** — it fires on *any* unique key (the footgun in §6). To reference the proposed new values, the old `VALUES(col)` function is "deprecated, and subject to removal in a future version of MySQL"; use the row alias instead:

```sql
-- RIGHT — modern MySQL 8.0.19+: row alias `new` is the analog of excluded.*
INSERT INTO page_views (url, views) VALUES (:url, 1) AS new
ON DUPLICATE KEY UPDATE views = page_views.views + new.views;

-- WRONG — deprecated VALUES() spelling, slated for removal
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON DUPLICATE KEY UPDATE views = views + VALUES(views);
```

The three spellings line up: `excluded.col` (PG/SQLite) ≈ `new.col` (MySQL) ≈ the source row in a `MERGE`.

---

## 5. Upsert Requires a UNIQUE / PRIMARY KEY Constraint

There is no "conflict" without a constraint that defines one. SQLite is explicit: "The UPSERT processing happens only for uniqueness constraints. A 'uniqueness constraint' is an explicit UNIQUE or PRIMARY KEY constraint within the CREATE TABLE statement, or a unique index" ([SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html)). MySQL only acts when a row "would cause a duplicate value in a `UNIQUE` index or `PRIMARY KEY`" ([MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)).

```sql
-- WRONG — no UNIQUE on email, so ON CONFLICT (email) has nothing to detect:
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, email TEXT, name TEXT);
INSERT INTO users (email, name) VALUES (:email, :name)
ON CONFLICT (email) DO UPDATE SET name = excluded.name;  -- ERROR: no matching unique constraint

-- RIGHT — declare the constraint that gives "conflict" a meaning
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, email TEXT NOT NULL UNIQUE, name TEXT);
```

So "make this an upsert" is first a **schema** question: which column(s) identify the same logical row? That unique key is the contract. Constraint syntax (composite keys, partial indexes, `NULLS NOT DISTINCT`, and the fact that a default `UNIQUE` lets multiple NULLs through) is owned by `sql-constraints-and-integrity`.

---

## 6. `MERGE` Is Not a Concurrency Cure-All

`MERGE` looks atomic, but it does **not** guarantee a no-error upsert under concurrency. PostgreSQL warns: "If a target row is modified more than once, a uniqueness violation or cardinality violation will occur," and explicitly steers you elsewhere — "You may also wish to consider using `INSERT ... ON CONFLICT` ... which offers the ability to run an `UPDATE` if a concurrent `INSERT` occurs. There are a variety of differences and restrictions between the two statement types and they are not interchangeable" ([PostgreSQL — MERGE, Notes](https://www.postgresql.org/docs/current/sql-merge.html)). A `MERGE` whose `WHEN NOT MATCHED` arm INSERTs can still hit a duplicate-key error if a concurrent transaction inserts the same key after `MERGE` checks for a match. The defenses are: (1) a real `UNIQUE`/`PK` constraint so the database catches the dup, and (2) for pure single-row insert-or-update, prefer `ON CONFLICT` / `ON DUPLICATE KEY`, which are purpose-built to absorb the concurrent insert; otherwise wrap `MERGE` in a **retry on unique-violation**. Isolation-level interaction and MVCC are owned by `sql-transactions-and-isolation`.

---

## 7. Portability

| Mechanism | PostgreSQL | SQLite | MySQL / MariaDB | SQL Server / Oracle / DB2 |
|---|---|---|---|---|
| Standard `MERGE` | **v15+** | **no** | **no** | yes |
| `INSERT ... ON CONFLICT` | v9.5+ | v3.24+ | no | no |
| `INSERT ... ON DUPLICATE KEY UPDATE` | no | no | yes | no |
| new-value alias | `excluded.col` | `excluded.col` | `new.col` (8.0.19+) | source alias in `USING` |

`MERGE` is genuinely standard (SQL:2003/2008) and broadly supported by the "big iron" engines — "PostgreSQL, Oracle Database, IBM Db2, Teradata, ... MS SQL ... support the standard syntax" — while `INSERT ON CONFLICT` is the PostgreSQL/SQLite idiom and `ON DUPLICATE KEY UPDATE` is the MySQL one ([Wikipedia — Merge (SQL)](https://en.wikipedia.org/wiki/Merge_(SQL))). No single statement is portable across all four: MySQL and SQLite have no `MERGE`; only PostgreSQL has both `MERGE` and `ON CONFLICT`. Pick by target engine and keep the upsert behind one well-named function. The full divergence table is owned by `sql-standard-vs-dialect-map`.

---

## 8. Who Suffers When the Upsert Is Wrong

An upsert bug almost never fails the unit test — it fails under concurrency or against a second unique index, in production:

- The **on-call engineer** paged at 2 a.m. for a duplicate-key crash that only appears under load: the app's `SELECT`-then-`INSERT` (§1) raced two requests for the same email, and the second INSERT blew up. The single-statement upsert they didn't write is the fix.
- The **analyst** reconciling a counter that's quietly low: two concurrent check-then-act paths both took the UPDATE branch and one **lost update** silently overwrote the other (§1). No error, just a wrong number.
- The **maintainer** of a MySQL table with two unique indexes, whose `ON DUPLICATE KEY UPDATE` fired on the *wrong* key and updated a row they never meant to touch — "if `a=1 OR b=2` matches several rows, only one row is updated ... avoid using an `ON DUPLICATE KEY UPDATE` clause on tables with multiple unique indexes" ([MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)). That is silent **data loss**, not a crash.
- The **developer** whose `DO UPDATE SET views = views + 1` is a no-op because they forgot `excluded.` and kept re-writing the old stored value (§3).

The atomic statement, the right unique key, and `excluded.`/`new.` used deliberately are all gifts to whoever debugs this under load.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — foundation: set semantics and three-valued logic (a NULL in the conflict key never "matches," because nothing equals NULL).
- `sql-constraints-and-integrity` — the `UNIQUE`/`PRIMARY KEY` constraint every upsert depends on (§5), composite keys, and multi-NULL uniqueness.
- `sql-transactions-and-isolation` — why the atomic statement is safe and `MERGE` is not (§1, §6): isolation levels, MVCC, lost updates, retry-on-unique-violation.
- `sql-standard-vs-dialect-map` — the full `MERGE` vs `ON CONFLICT` vs `ON DUPLICATE KEY UPDATE` divergence table and per-engine availability (§7).

---

## 10. Reference Files

High-frequency upsert anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
