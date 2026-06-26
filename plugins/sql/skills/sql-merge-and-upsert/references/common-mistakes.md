# Common SQL Merge & Upsert Mistakes

Anti-patterns in LLM-generated SQL around insert-or-update, each with wrong/right code and a
primary-source citation. The skill (`sql-merge-and-upsert`) states the rules; this file holds the
high-frequency failure modes. RIGHT examples favor portable/standard SQL; dialect spellings are flagged
and routed to `sql-standard-vs-dialect-map`.

---

## 1. App-side SELECT-then-INSERT-or-UPDATE (the check-then-act race)

**The problem:** The model reads a row in application code, branches, and then INSERTs or UPDATEs. Between the read and the write another session changes the world: both branches INSERT (duplicate-key crash, or duplicate rows), or both UPDATE (lost update). The check and the act are separate statements, so a retry loop cannot close the gap.

```sql
-- WRONG — conceptually: SELECT, branch in app, then write (two statements, racy)
SELECT 1 FROM users WHERE email = :email;   -- if absent ->
INSERT INTO users (email, name) VALUES (:email, :name);   -- concurrent session does the same -> dup

-- RIGHT — one atomic statement; the engine serializes on the unique index
INSERT INTO users (email, name) VALUES (:email, :name)
ON CONFLICT (email) DO UPDATE SET name = excluded.name;
```

*Source: [PostgreSQL — MERGE, Notes](https://www.postgresql.org/docs/current/sql-merge.html) (ON CONFLICT "offers the ability to run an UPDATE if a concurrent INSERT occurs"). Depth: this skill, §1; isolation theory owned by `sql-transactions-and-isolation`.*

---

## 2. Upsert against a column with no UNIQUE / PRIMARY KEY constraint

**The problem:** The model writes `ON CONFLICT (col)` (or relies on `ON DUPLICATE KEY`) for a column that has no unique constraint. There is no conflict to detect: PostgreSQL errors ("there is no unique or exclusion constraint matching the ON CONFLICT specification"), and `ON DUPLICATE KEY UPDATE` simply never fires, inserting duplicates. "The UPSERT processing happens only for uniqueness constraints" ([SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html)).

```sql
-- WRONG — email has no UNIQUE constraint, so there is no conflict to catch
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, email TEXT, name TEXT);
INSERT INTO users (email, name) VALUES (:email, :name)
ON CONFLICT (email) DO UPDATE SET name = excluded.name;   -- ERROR / never matches

-- RIGHT — declare the unique key that defines "the same row"
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, email TEXT NOT NULL UNIQUE, name TEXT);
```

*Source: [SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html); [MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html). Depth: this skill, §5; constraint syntax owned by `sql-constraints-and-integrity`.*

---

## 3. Forgetting `excluded.` in `DO UPDATE` — overwriting with the old value

**The problem:** In `DO UPDATE SET`, a bare column name is the *existing stored* value, not the value being inserted. "Column names in the expressions of a DO UPDATE refer to the original unchanged value ... To use the value that would have been inserted ... add the special 'excluded.' table qualifier" ([SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html)). Forgetting it makes the update a no-op or writes the wrong value.

```sql
-- WRONG — "views" on the right is the OLD row; adds nothing useful
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON CONFLICT (url) DO UPDATE SET views = views + 1;

-- RIGHT — excluded.views is the value from the attempted INSERT
INSERT INTO page_views (url, views) VALUES (:url, 1)
ON CONFLICT (url) DO UPDATE SET views = page_views.views + excluded.views;
```

*Source: [SQLite — UPSERT](https://www.sqlite.org/lang_upsert.html). Depth: this skill, §3.*

---

## 4. `ON DUPLICATE KEY UPDATE` on a table with multiple unique indexes

**The problem:** MySQL's `ON DUPLICATE KEY` has no explicit conflict target — it fires on *any* unique key. With two unique indexes it can update a row matched by the *wrong* key, non-deterministically. "If `a=1 OR b=2` matches several rows, only one row is updated. In general, you should try to avoid using an `ON DUPLICATE KEY UPDATE` clause on tables with multiple unique indexes" ([MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)). This is silent data loss.

```sql
-- WRONG — t1 has UNIQUE(a) AND UNIQUE(b); which row gets updated is undefined
INSERT INTO t1 (a, b, c) VALUES (1, 2, 3) AS new
ON DUPLICATE KEY UPDATE c = new.c;

-- RIGHT — name the exact conflict target (PostgreSQL/SQLite), so intent is explicit
INSERT INTO t1 (a, b, c) VALUES (1, 2, 3)
ON CONFLICT (a) DO UPDATE SET c = excluded.c;
```

*Source: [MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html). Depth: this skill, §6, §8; dialect divergence owned by `sql-standard-vs-dialect-map`.*

---

## 5. Using the deprecated `VALUES()` function in MySQL upserts

**The problem:** The model references the new row via `VALUES(col)`. "The use of `VALUES()` to refer to the new row and columns is deprecated, and subject to removal in a future version of MySQL" ([MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)). Use a row alias (MySQL 8.0.19+).

```sql
-- WRONG — deprecated, slated for removal
INSERT INTO t1 (a, b, c) VALUES (1, 2, 3)
ON DUPLICATE KEY UPDATE c = VALUES(c);

-- RIGHT — row alias `new`
INSERT INTO t1 (a, b, c) VALUES (1, 2, 3) AS new
ON DUPLICATE KEY UPDATE c = new.c;
```

*Source: [MySQL — ON DUPLICATE KEY](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html). Depth: this skill, §4.*

---

## 6. Treating `MERGE` as a guaranteed race-free upsert

**The problem:** The model uses `MERGE` and assumes it cannot raise a duplicate-key error. Under concurrency the `WHEN NOT MATCHED ... INSERT` arm can still collide with a concurrent insert. PostgreSQL: "If a target row is modified more than once, a uniqueness violation or cardinality violation will occur," and it recommends `INSERT ... ON CONFLICT` for the concurrent-insert case ([PostgreSQL — MERGE, Notes](https://www.postgresql.org/docs/current/sql-merge.html)).

```sql
-- WRONG (assumption) — relying on MERGE alone to absorb concurrent inserts, no constraint, no retry
MERGE INTO users t USING (VALUES (:email, :name)) s(email, name) ON t.email = s.email
WHEN NOT MATCHED THEN INSERT (email, name) VALUES (s.email, s.name);

-- RIGHT — a UNIQUE constraint exists AND, for single-row insert-or-update, prefer ON CONFLICT
--          (or wrap MERGE in a retry on unique-violation)
INSERT INTO users (email, name) VALUES (:email, :name)
ON CONFLICT (email) DO UPDATE SET name = excluded.name;
```

*Source: [PostgreSQL — MERGE, Notes](https://www.postgresql.org/docs/current/sql-merge.html). Depth: this skill, §6; isolation/retry owned by `sql-transactions-and-isolation`.*

---

## 7. Reaching for `MERGE` on MySQL or SQLite

**The problem:** The model writes a standard `MERGE` statement targeting MySQL or SQLite, which do not implement it — "MySQL supports the use of `INSERT ... ON DUPLICATE KEY UPDATE`" and PostgreSQL/SQLite use `INSERT ON CONFLICT` ([Wikipedia — Merge (SQL)](https://en.wikipedia.org/wiki/Merge_(SQL))). The statement is a syntax error on those engines.

```sql
-- WRONG — MERGE does not exist in MySQL or SQLite
MERGE INTO t USING s ON t.k = s.k WHEN MATCHED THEN UPDATE SET v = s.v ...;

-- RIGHT (SQLite / PostgreSQL)
INSERT INTO t (k, v) VALUES (:k, :v) ON CONFLICT (k) DO UPDATE SET v = excluded.v;
-- RIGHT (MySQL)
INSERT INTO t (k, v) VALUES (:k, :v) AS new ON DUPLICATE KEY UPDATE v = new.v;
```

*Source: [Wikipedia — Merge (SQL)](https://en.wikipedia.org/wiki/Merge_(SQL)). Depth: this skill, §7; full table owned by `sql-standard-vs-dialect-map`.*
