---
name: sql-data-modification
description: Guides the core write statements — INSERT, UPDATE, DELETE — as set-based operations the database serializes, not row-by-row loops. Teaches the INSERT forms (single-row VALUES, multi-row VALUES in ONE statement, INSERT ... SELECT for bulk copy, DEFAULT VALUES) and bans the nightly job that fires 100k single-row INSERTs instead of one. Centers the #1 catastrophe — an UPDATE or DELETE with no WHERE rewrites or wipes the WHOLE table silently, with no error — and the discipline that prevents it (run inside an explicit transaction, SELECT the predicate first, then write). Flags that `UPDATE ... FROM` and `DELETE ... USING` are NON-STANDARD join-update extensions whose portable form is a correlated subquery in SET / `WHERE EXISTS`. Teaches RETURNING (non-standard but in PostgreSQL, SQLite 3.35+, MariaDB, Oracle; SQL:2023 standardized an OLD/NEW form) to fetch generated keys without a second round-trip, and TRUNCATE vs DELETE. Auto-invokes when writing or editing any INSERT/UPDATE/DELETE/TRUNCATE, bulk-insert or batch-load code, an application loop that inserts or updates rows one at a time, RETURNING / generated-key fetches, or on "insert many rows" / "update from another table" / "delete all rows" / "get the new id" requests. Routes upsert/insert-or-update to sql-merge-and-upsert.
allowed-tools: Read, Glob, Grep
---

# SQL Data Modification

> "If the `WHERE` clause is absent, the effect is to delete all rows in the table. The result is a valid, but empty table."
> — [PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)

> "Use of `RETURNING` avoids performing an extra database query to collect the data, and is especially valuable when it would otherwise be difficult to identify the modified rows reliably."
> — [PostgreSQL — Returning Data From Modified Rows](https://www.postgresql.org/docs/current/dml-returning.html)

DML is where a query stops being a question and becomes a *change* to durable state. Two ideas govern everything below. First, every write is **set-based**: `INSERT`, `UPDATE`, and `DELETE` each operate on a whole set of rows in one statement the engine optimizes and a transaction wraps — not a loop the application drives one row at a time. Second, the *scope* of a write is decided entirely by its predicate, and a missing predicate means "everything." This skill assumes the set semantics and three-valued logic of `sql-relational-and-null-discipline` (a `WHERE` keeps only rows that evaluate to TRUE — which is exactly why a NULL-laden predicate can quietly modify the wrong rows), and it hands upsert / insert-or-update to `sql-merge-and-upsert`.

---

## 1. INSERT Forms — One Statement, Not a Loop

`INSERT` has three set-based forms. The multi-row `VALUES` form inserts many rows in a *single* statement — the grammar is `VALUES ( ... ) [, ...]`, the trailing `[, ...]` meaning "repeat the row" ([PostgreSQL — INSERT](https://www.postgresql.org/docs/current/sql-insert.html)). SQLite agrees: the `VALUES` form "creates one or more new rows in an existing table" ([SQLite — INSERT](https://www.sqlite.org/lang_insert.html)).

```sql
-- RIGHT — single row
INSERT INTO films (code, title, kind) VALUES ('UA502', 'Bananas', 'Comedy');

-- RIGHT — MANY rows in ONE statement (one round trip, one transaction)
INSERT INTO films (code, title, did, date_prod, kind) VALUES
    ('B6717', 'Tampopo',         110, '1985-02-10', 'Comedy'),
    ('HG120', 'The Dinner Game', 140, DEFAULT,      'Comedy');

-- RIGHT — bulk copy from a query: the engine does the loop, set-based
INSERT INTO films SELECT * FROM tmp_films WHERE date_prod < '2004-05-07';

-- RIGHT — a row of all defaults
INSERT INTO films DEFAULT VALUES;
```

`DEFAULT` is usable per value, and `DEFAULT VALUES` fills "all columns ... with their default values" ([PostgreSQL — INSERT](https://www.postgresql.org/docs/current/sql-insert.html)). The anti-pattern is driving inserts from application code one row at a time:

```python
# WRONG — N statements, N round trips, N transactions (unless wrapped); slow
for f in films:
    db.execute("INSERT INTO films (code, title) VALUES (?, ?)", [f.code, f.title])
```

A thousand single-row INSERTs is a thousand network round trips and (without an explicit transaction) a thousand commits. One multi-row `VALUES` or `INSERT ... SELECT` collapses that to one. Batch the rows; let the database do the loop.

---

## 2. The Missing `WHERE` — UPDATE/DELETE Without a Predicate Hits Every Row (the centerpiece)

This is the single most destructive mistake in DML, and it never errors. `UPDATE` "changes the values of the specified columns in all rows that satisfy the condition" ([PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html)) — and with **no** condition, *every* row satisfies. SQLite says it outright: "If the UPDATE statement does not have a WHERE clause, all rows in the table are modified" ([SQLite — UPDATE](https://www.sqlite.org/lang_update.html)). `DELETE` is worse: "If the `WHERE` clause is absent, the effect is to delete all rows in the table" ([PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)).

```sql
-- WRONG — no WHERE: rewrites EVERY row in the table. Valid SQL. No error. No undo outside a tx.
UPDATE users SET status = 'inactive';

-- WRONG — no WHERE: empties the entire table.
DELETE FROM users;
```

The instinct that saves you is to treat every `UPDATE`/`DELETE` as a three-step ritual: **wrap it in an explicit transaction, SELECT the rows the predicate matches first, then run the write and verify the affected-row count before COMMIT.**

```sql
-- RIGHT — verify scope before you commit to it
BEGIN;

-- 1. See exactly which rows the predicate selects (and how many).
SELECT id, status FROM users WHERE last_login < '2024-01-01';

-- 2. Run the write with the SAME predicate; check the reported row count is what you expected.
UPDATE users SET status = 'inactive' WHERE last_login < '2024-01-01';

-- 3. Wrong count or wrong rows? ROLLBACK. Right? COMMIT.
COMMIT;   -- or ROLLBACK;
```

The `SELECT`-first step turns the predicate into something you can *read* before it's something you can't take back. The transaction makes a mistake reversible: an un-`COMMIT`-ted `UPDATE`/`DELETE` rolls back cleanly. Outside a transaction, on autocommit, the wipe is durable the instant the statement returns. Why the transaction is safe (isolation, locking, MVCC) is owned by `sql-transactions-and-isolation`.

---

## 3. Join-Updates Are Non-Standard — `UPDATE ... FROM` / `DELETE ... USING`

Updating or deleting based on *another* table tempts you toward a join. Both PostgreSQL and SQLite offer one — `UPDATE ... FROM` — but it is a vendor extension, not standard SQL. PostgreSQL is explicit: "the `FROM` and `RETURNING` clauses are PostgreSQL extensions" ([PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html)). SQLite warns the construct "is not part of the SQL standards, each product implements UPDATE-FROM differently" ([SQLite — UPDATE](https://www.sqlite.org/lang_update.html)). `DELETE ... USING` is the same story — PostgreSQL flatly states "This syntax is not standard" ([PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)).

```sql
-- NON-STANDARD (PG/SQLite) — convenient, but syntax varies per engine
UPDATE inventory SET qty = qty - s.amt
  FROM sales s WHERE inventory.item_id = s.item_id;

DELETE FROM films USING producers
  WHERE producer_id = producers.id AND producers.name = 'foo';
```

The **portable** form is a correlated subquery in `SET` plus a `WHERE EXISTS` guard. PostgreSQL itself recommends it: "referencing other tables only within sub-selects is safer" ([PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html)), and calls the subquery DELETE "a more standard way" ([PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)).

```sql
-- RIGHT (portable) — correlated subquery sets the value; EXISTS limits which rows are touched
UPDATE inventory
   SET qty = qty - (SELECT s.amt FROM sales s WHERE s.item_id = inventory.item_id)
 WHERE EXISTS (SELECT 1 FROM sales s WHERE s.item_id = inventory.item_id);

-- RIGHT (portable) — anti-join delete via a subquery, no USING
DELETE FROM films
 WHERE producer_id IN (SELECT id FROM producers WHERE name = 'foo');
```

The `WHERE EXISTS` is not optional: without it, rows with no matching subquery row get `SET col = (SELECT ... )` = `NULL` (an empty scalar subquery is NULL — see `sql-relational-and-null-discipline`), silently nulling rows you meant to leave alone.

---

## 4. RETURNING — Fetch Generated Keys Without a Second Round-Trip

After an `INSERT`, you usually need the generated id. The naive path is a second statement (`SELECT ... WHERE ...`) — an extra round trip *and* a race, because identifying "the row I just inserted" reliably is itself hard under concurrency. `RETURNING` solves both: it "avoids performing an extra database query to collect the data, and is especially valuable when it would otherwise be difficult to identify the modified rows reliably" ([PostgreSQL — RETURNING](https://www.postgresql.org/docs/current/dml-returning.html)).

```sql
-- WRONG — extra round trip, and "the new id" is racy to re-identify under concurrency
INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool');
SELECT id FROM users WHERE firstname = 'Joe' AND lastname = 'Cool';  -- which Joe Cool?

-- RIGHT — one statement returns the server-generated key
INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool') RETURNING id;
```

`RETURNING` works on `UPDATE` and `DELETE` too, returning a row per affected row — handy for logging what a bulk write actually changed. SQL:2023 standardized a richer form returning both the OLD and NEW content of each modified row; PostgreSQL exposes it as `old.`/`new.` ([PostgreSQL — RETURNING](https://www.postgresql.org/docs/current/dml-returning.html)):

```sql
-- RIGHT — capture before/after in one statement (SQL:2023 old/new)
UPDATE products SET price = price * 1.10 WHERE price <= 99.99
  RETURNING name, old.price AS old_price, new.price AS new_price;
```

`RETURNING` is "not standard SQL ... modelled after PostgreSQL" ([SQLite — RETURNING](https://www.sqlite.org/lang_returning.html)) but is widely available (§6). For `INSERT ... ON CONFLICT` and other upsert spellings, route to `sql-merge-and-upsert` — that skill owns insert-or-update; this one owns the plain write.

---

## 5. TRUNCATE vs DELETE — Empty a Table Fast, but Know the Trade-offs

To remove *all* rows, `DELETE FROM t` works but scans and logs every row. `TRUNCATE` "provides a faster mechanism to remove all rows from a table" ([PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)) — it deallocates storage instead of deleting row-by-row, and typically resets identity/auto-increment counters. The catch is transactional behavior:

```sql
-- DELETE — row-by-row, fully transactional everywhere, fires row triggers, keeps identity counter
DELETE FROM staging_events;

-- TRUNCATE — fast bulk empty; resets identity; transactional on PostgreSQL,
-- but AUTO-COMMITS (cannot roll back) on MySQL and Oracle
TRUNCATE TABLE staging_events;
```

Reach for `TRUNCATE` to reset a scratch/staging table; reach for `DELETE ... WHERE` for any *partial* removal (TRUNCATE takes no predicate). The transactional divergence is the trap — on PostgreSQL a `TRUNCATE` inside `BEGIN ... ROLLBACK` is undone; on MySQL and Oracle it commits immediately and is irreversible. Per-engine specifics route to `sql-standard-vs-dialect-map`.

---

## 6. Portability

| Feature | PostgreSQL | SQLite | MySQL / MariaDB | Oracle |
|---|---|---|---|---|
| Multi-row `VALUES`, `INSERT ... SELECT` | yes | yes | yes | `INSERT ALL` / `... SELECT` |
| `RETURNING` | yes | **3.35+** | MariaDB yes / MySQL **no** | yes |
| `UPDATE ... FROM` (join-update) | yes | **3.33+** | different syntax | no — use subquery |
| `DELETE ... USING` (join-delete) | yes | use subquery | different syntax | no — use subquery |
| `TRUNCATE` rolls back in a transaction | **yes** | no `TRUNCATE` (use `DELETE`) | **no — auto-commits** | **no — auto-commits** |

The set-based core — multi-row `VALUES`, `INSERT ... SELECT`, `UPDATE`/`DELETE` with a `WHERE`, and the correlated-subquery join-update — is portable everywhere. The conveniences are not: `RETURNING` is absent in MySQL (use `LAST_INSERT_ID()` there), `UPDATE...FROM`/`DELETE...USING` spell differently per engine, and `TRUNCATE` is non-transactional on MySQL/Oracle. Keep the non-portable spellings behind one well-named function and route dialect detail to `sql-standard-vs-dialect-map`.

---

## 7. Who Suffers When DML Goes Wrong

A DML mistake is durable — it changes state, and someone inherits the damage:

- The **engineer** who ran `UPDATE accounts SET balance = 0` in a prod console, highlighted everything *except* the `WHERE` line, and hit execute — rewriting every row in the table. There was no error and no transaction; the rollback they didn't open is the restore-from-backup they now need (§2).
- The **on-call** watching a nightly load crawl for hours because it fires 100k single-row `INSERT`s — 100k round trips — instead of one multi-row `VALUES` or `INSERT ... SELECT` that finishes in seconds (§1).
- The **developer** whose "create user" endpoint does `INSERT` then `SELECT ... WHERE email = ?` to get the id, and under concurrency occasionally returns *another* request's row — a bug `RETURNING id` would have made impossible (§4).
- The **maintainer** who ported an `UPDATE ... FROM` to a new engine and got a syntax error, or worse, silently different rows — because the join-update they relied on was never standard (§3).

The `WHERE` you checked, the transaction you opened, the multi-row `VALUES` you batched, and the `RETURNING` you reached for are all gifts to whoever runs this against production.

---

## 8. Routing to Related Skills

- `sql-relational-and-null-discipline` — foundation: set semantics, and why a `WHERE` keeps only TRUE rows (so a NULL in an `UPDATE`/`DELETE` predicate silently changes the matched set).
- `sql-merge-and-upsert` — insert-or-update / `INSERT ... ON CONFLICT` / `MERGE`: the atomic upsert this skill deliberately does not duplicate.
- `sql-transactions-and-isolation` — why wrapping a write in `BEGIN ... COMMIT` makes it reversible and safe under concurrency (§2), and the autocommit trap.
- `sql-constraints-and-integrity` — what a write must satisfy: `NOT NULL`, `CHECK`, `UNIQUE`, and foreign-key cascade on `DELETE`/`UPDATE`.
- `sql-standard-vs-dialect-map` — dialect spellings of `RETURNING`, `UPDATE...FROM`/`DELETE...USING`, and `TRUNCATE`'s per-engine transactional behavior (§3, §5, §6).

---

## 9. Reference Files

High-frequency DML anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
