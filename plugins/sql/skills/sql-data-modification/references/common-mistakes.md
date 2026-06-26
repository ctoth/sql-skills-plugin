# Common SQL Data-Modification (DML) Mistakes

Anti-patterns in LLM-generated `INSERT`/`UPDATE`/`DELETE`/`TRUNCATE`, each with wrong/right code and a
primary-source citation. The skill (`sql-data-modification`) states the rules; this file holds the
high-frequency failure modes. All RIGHT examples are standard/portable SQL unless flagged; non-standard
spellings are routed to `sql-standard-vs-dialect-map`. Upsert/insert-or-update is owned by
`sql-merge-and-upsert`.

---

## 1. `UPDATE` / `DELETE` with no `WHERE` — the whole-table wipe

**The problem:** The model writes an `UPDATE` or `DELETE` and forgets (or mis-scopes) the `WHERE`, so it hits every row. It is valid SQL and raises no error. "If the `WHERE` clause is absent, the effect is to delete all rows in the table" ([PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)); for UPDATE, "all rows in the table are modified" ([SQLite — UPDATE](https://www.sqlite.org/lang_update.html)).

```sql
-- WRONG — rewrites every row; no error, no undo outside a transaction
UPDATE users SET status = 'inactive';

-- RIGHT — explicit transaction, SELECT-first to confirm scope, then write, then verify count
BEGIN;
SELECT count(*) FROM users WHERE last_login < '2024-01-01';     -- expect N
UPDATE users SET status = 'inactive' WHERE last_login < '2024-01-01';
COMMIT;   -- ROLLBACK if the affected count was not N
```

*Source: [PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html); [SQLite — UPDATE](https://www.sqlite.org/lang_update.html). Depth: this skill, §2.*

---

## 2. A loop of single-row `INSERT`s instead of one set-based statement

**The problem:** The model drives inserts from application code, one statement per row — N round trips, and N commits if not wrapped in a transaction. The multi-row `VALUES` form "creates one or more new rows" ([SQLite — INSERT](https://www.sqlite.org/lang_insert.html)) in a single statement.

```python
# WRONG — 100k statements, 100k round trips
for r in rows:
    db.execute("INSERT INTO t (a, b) VALUES (?, ?)", [r.a, r.b])
```

```sql
-- RIGHT — many rows, one statement
INSERT INTO t (a, b) VALUES (1, 'x'), (2, 'y'), (3, 'z');

-- RIGHT — copy in bulk straight from a query
INSERT INTO t (a, b) SELECT a, b FROM staging WHERE loaded = false;
```

*Source: [PostgreSQL — INSERT](https://www.postgresql.org/docs/current/sql-insert.html); [SQLite — INSERT](https://www.sqlite.org/lang_insert.html). Depth: this skill, §1.*

---

## 3. Relying on non-standard `UPDATE ... FROM` / `DELETE ... USING`

**The problem:** The model reaches for a join-update without knowing it is a vendor extension that "is not part of the SQL standards, each product implements UPDATE-FROM differently" ([SQLite — UPDATE](https://www.sqlite.org/lang_update.html)); `DELETE ... USING` "syntax is not standard" ([PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html)). It breaks or behaves differently when ported.

```sql
-- WRONG (non-portable) — engine-specific join-update
UPDATE inventory SET qty = qty - s.amt FROM sales s WHERE inventory.item_id = s.item_id;

-- RIGHT (portable) — correlated subquery in SET + WHERE EXISTS guard
UPDATE inventory
   SET qty = qty - (SELECT s.amt FROM sales s WHERE s.item_id = inventory.item_id)
 WHERE EXISTS (SELECT 1 FROM sales s WHERE s.item_id = inventory.item_id);
```

*Source: [PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html); [PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html); [SQLite — UPDATE](https://www.sqlite.org/lang_update.html). Depth: this skill, §3; dialect spellings owned by `sql-standard-vs-dialect-map`.*

---

## 4. Correlated-subquery UPDATE without a `WHERE EXISTS` guard

**The problem:** The model rewrites a join-update as a subquery in `SET` but omits the `WHERE EXISTS`. Rows with no matching subquery row get `SET col = (empty scalar subquery)` = `NULL`, silently nulling rows that should have been left untouched (an empty scalar subquery yields NULL — see `sql-relational-and-null-discipline`).

```sql
-- WRONG — every row with no matching sale gets qty = NULL
UPDATE inventory
   SET qty = qty - (SELECT s.amt FROM sales s WHERE s.item_id = inventory.item_id);

-- RIGHT — limit the touched rows to those the subquery actually matches
UPDATE inventory
   SET qty = qty - (SELECT s.amt FROM sales s WHERE s.item_id = inventory.item_id)
 WHERE EXISTS (SELECT 1 FROM sales s WHERE s.item_id = inventory.item_id);
```

*Source: [PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html) ("referencing other tables only within sub-selects is safer"). Depth: this skill, §3.*

---

## 5. A second `SELECT` to fetch the just-inserted id instead of `RETURNING`

**The problem:** The model inserts, then runs a separate `SELECT` to recover the generated key — an extra round trip, and a race: re-identifying "the row I just inserted" is unreliable under concurrency. `RETURNING` "avoids performing an extra database query ... and is especially valuable when it would otherwise be difficult to identify the modified rows reliably" ([PostgreSQL — RETURNING](https://www.postgresql.org/docs/current/dml-returning.html)).

```sql
-- WRONG — extra round trip; under load may return another session's row
INSERT INTO users (email) VALUES ('joe@x.com');
SELECT id FROM users WHERE email = 'joe@x.com';

-- RIGHT — one statement returns the server-generated key
INSERT INTO users (email) VALUES ('joe@x.com') RETURNING id;
```

(MySQL has no `RETURNING`; use `LAST_INSERT_ID()` there — route to `sql-standard-vs-dialect-map`.)

*Source: [PostgreSQL — RETURNING](https://www.postgresql.org/docs/current/dml-returning.html); [SQLite — RETURNING](https://www.sqlite.org/lang_returning.html). Depth: this skill, §4.*

---

## 6. `INSERT ... ON CONFLICT DO UPDATE` reset to the old value (forgetting `excluded.`)

**The problem:** When converting an insert into an upsert, the model references the bare column name in `DO UPDATE SET`, which is the *old stored* value, not the value being inserted — making the update a no-op. The conflict-and-upsert mechanics are owned by `sql-merge-and-upsert`; this is the routing pointer.

```sql
-- WRONG — `views` here is the OLD row; this never increments
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON CONFLICT (url) DO UPDATE SET views = views + 1;

-- RIGHT — excluded.col is the value you tried to insert
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON CONFLICT (url) DO UPDATE SET views = page_views.views + excluded.views;
```

*Source: routed to `sql-merge-and-upsert`. Depth: this skill, §4 (routing); full treatment in `sql-merge-and-upsert`.*

---

## 7. `TRUNCATE` assumed to be transactional / rollback-able everywhere

**The problem:** The model uses `TRUNCATE` to empty a table inside a transaction expecting to be able to roll it back. On PostgreSQL it is transactional, but on MySQL and Oracle `TRUNCATE` auto-commits and cannot be undone. `TRUNCATE` also takes no `WHERE`, so it cannot do a partial delete.

```sql
-- WRONG (on MySQL/Oracle) — this commits immediately; the ROLLBACK does nothing
BEGIN;
TRUNCATE TABLE staging;
ROLLBACK;   -- table is still empty on MySQL/Oracle

-- RIGHT — use DELETE when you need transactional safety or a predicate
BEGIN;
DELETE FROM staging WHERE batch_id = :batch;
COMMIT;   -- or ROLLBACK, cleanly, everywhere
```

*Source: [PostgreSQL — DELETE](https://www.postgresql.org/docs/current/sql-delete.html) ("TRUNCATE provides a faster mechanism to remove all rows"). Depth: this skill, §5; per-engine transactional behavior owned by `sql-standard-vs-dialect-map`.*

---

## 8. `INSERT` without an explicit column list

**The problem:** The model writes `INSERT INTO t VALUES (...)` relying on positional column order. Any schema change (a new column, a reorder) silently shifts every value into the wrong column or errors. Naming columns makes the insert robust and self-documenting.

```sql
-- WRONG — positional; breaks the moment the table gains or reorders a column
INSERT INTO users VALUES ('joe@x.com', 'Joe', 't');

-- RIGHT — explicit column list; order- and addition-proof
INSERT INTO users (email, name, active) VALUES ('joe@x.com', 'Joe', true);
```

*Source: [PostgreSQL — INSERT](https://www.postgresql.org/docs/current/sql-insert.html) (column-list syntax). Depth: this skill, §1.*
