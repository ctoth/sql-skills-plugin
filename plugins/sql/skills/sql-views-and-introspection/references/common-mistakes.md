# Common SQL View & Introspection Mistakes

Anti-patterns in LLM-generated SQL around views and schema introspection, each with
wrong/right code and a primary-source citation. The skill (`sql-views-and-introspection`)
states the rules; this file holds the high-frequency failure modes. All RIGHT examples are
standard/portable SQL unless flagged; non-standard spellings route to
`sql-standard-vs-dialect-map`.

---

## 1. Introspecting via `pg_catalog` instead of `INFORMATION_SCHEMA`

**The problem:** The model lists tables/columns by querying PostgreSQL system catalogs
(`pg_catalog.pg_class`, `pg_attribute`). The query works on PostgreSQL and breaks
everywhere else — pg_catalog is "specific to PostgreSQL and ... modeled after
implementation concerns," whereas the information schema "is defined in the SQL standard
and can therefore be expected to be portable" ([PostgreSQL — Information Schema](https://www.postgresql.org/docs/current/information-schema.html)).

```sql
-- WRONG — PostgreSQL-only; fails on MySQL, SQL Server, SQLite
SELECT relname FROM pg_catalog.pg_class WHERE relkind = 'r';

-- RIGHT — standard, portable across PG / MySQL / MariaDB / SQL Server
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public' AND table_type = 'BASE TABLE';
```

*Source: [PostgreSQL — Information Schema](https://www.postgresql.org/docs/current/information-schema.html). Depth: this skill, §4.*

---

## 2. `SHOW TABLES` as if it were portable / a result set

**The problem:** The model uses `SHOW TABLES` (or `SHOW COLUMNS`) for discovery. It is a
MySQL/MariaDB dialect statement, not standard SQL, and not a relational result set you can
filter or join. It does not exist on PostgreSQL or SQL Server.

```sql
-- WRONG — MySQL/MariaDB dialect; not standard; can't be filtered/joined
SHOW TABLES;

-- RIGHT — standard and queryable
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';
```

*Source: [Wikipedia — Information schema](https://en.wikipedia.org/wiki/Information_schema). Depth: this skill, §4; dialect map owns `SHOW`.*

---

## 3. Assuming `INFORMATION_SCHEMA` exists on SQLite

**The problem:** The model writes a portable `information_schema.tables` query and runs it
against SQLite, where it errors — SQLite is among the "RDBMSs that do not support
`information_schema`" ([Wikipedia — Information schema](https://en.wikipedia.org/wiki/Information_schema)). SQLite uses a schema table instead.

```sql
-- WRONG — no such table on SQLite
SELECT table_name FROM information_schema.tables;

-- RIGHT (SQLite only) — query the schema table
SELECT name FROM sqlite_schema WHERE type = 'table';
-- column detail comes from PRAGMA table_info('orders'), not a standard view
```

*Source: [SQLite — The Schema Table](https://www.sqlite.org/schematab.html); [Wikipedia — Information schema](https://en.wikipedia.org/wiki/Information_schema). Depth: this skill, §5; SQLite catalog/PRAGMA map owned by `sql-standard-vs-dialect-map`.*

---

## 4. Filtered/security view without `WITH CHECK OPTION`

**The problem:** The model creates an updatable filtered view as a "security boundary" but
omits `WITH CHECK OPTION`. A write through the view can set values that fall outside the
view's `WHERE`; the row is created but "vanishes" from the view (and escapes the filter).
The option exists precisely so "new rows are checked to ensure that they are visible
through the view. If they are not, the update will be rejected" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)).

```sql
-- WRONG — INSERT can set owner_id to someone else; row escapes the view
CREATE VIEW my_notes AS
  SELECT id, owner_id, body FROM notes WHERE owner_id = current_user_id();

-- RIGHT — reject any write whose row wouldn't be visible through the view
CREATE VIEW my_notes AS
  SELECT id, owner_id, body FROM notes WHERE owner_id = current_user_id()
  WITH CHECK OPTION;
```

*Source: [PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html). Depth: this skill, §3.*

---

## 5. Using `LOCAL` where layered views need `CASCADED`

**The problem:** The model adds `WITH LOCAL CHECK OPTION` on a view built atop another
filtered view, assuming the underlying filter is still enforced. With `LOCAL`, "any
conditions defined on underlying base views are not checked," so a write can violate the
lower view's predicate. The default (and usually correct) choice is `CASCADED`, which
checks "the view and all underlying base views" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)).

```sql
-- WRONG — LOCAL skips the base view's tenant filter
CREATE VIEW v2 AS SELECT * FROM v1 WHERE active
  WITH LOCAL CHECK OPTION;

-- RIGHT — CASCADED (also the default) checks v2's AND v1's conditions
CREATE VIEW v2 AS SELECT * FROM v1 WHERE active
  WITH CASCADED CHECK OPTION;
```

*Source: [PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html). Depth: this skill, §3.*

---

## 6. Expecting to write through a join/aggregate view

**The problem:** The model issues `INSERT`/`UPDATE`/`DELETE` against a view containing a
join, aggregate, `DISTINCT`, or `GROUP BY` and is surprised by a rejection. Only simple
views are "automatically updatable"; a view with more than one `FROM` entry, or with
aggregates/`DISTINCT`/`GROUP BY`/set operations, "is read-only by default" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)).

```sql
-- WRONG — multi-table join view; INSERT/UPDATE rejected
CREATE VIEW user_orders AS
  SELECT u.id, u.name, o.total FROM users u JOIN orders o ON o.user_id = u.id;
-- UPDATE user_orders SET total = 0 ...  -> rejected

-- RIGHT — write to the base table, or use an INSTEAD OF trigger (vendor-specific)
UPDATE orders SET total = 0 WHERE user_id = 42;
```

*Source: [PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html). Depth: this skill, §2; INSTEAD OF triggers owned by `sql-standard-vs-dialect-map`.*

---

## 7. Reaching for `sqlite_master` / vendor catalogs by reflex

**The problem:** Asked to "list the tables," the model emits `SELECT name FROM
sqlite_master` (or another vendor catalog) even when the target engine is unknown or
explicitly PostgreSQL/MySQL. `sqlite_master` is the legacy alias for `sqlite_schema` and
exists only in SQLite ([SQLite — The Schema Table](https://www.sqlite.org/schematab.html)). When portability matters, default to `information_schema` and only drop to a
vendor catalog after confirming the engine.

```sql
-- WRONG — SQLite-only legacy catalog, emitted for a Postgres target
SELECT name FROM sqlite_master WHERE type = 'table';

-- RIGHT (portable default) — works on PG / MySQL / MariaDB / SQL Server
SELECT table_name FROM information_schema.tables WHERE table_type = 'BASE TABLE';
```

*Source: [SQLite — The Schema Table](https://www.sqlite.org/schematab.html); [PostgreSQL — Information Schema](https://www.postgresql.org/docs/current/information-schema.html). Depth: this skill, §§4–5.*

---

## 8. Treating a plain view as a stored/cached snapshot

**The problem:** The model expects a `CREATE VIEW` to materialize results for speed, or to
"freeze" data as of creation time. A plain view stores no data: "the query is run every
time the view is referenced" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)). A snapshot needs a *materialized* view, which is vendor-specific and must
be refreshed.

```sql
-- WRONG (assumption) — this does NOT cache; it re-runs the query every reference
CREATE VIEW daily_totals AS SELECT day, SUM(amount) FROM sales GROUP BY day;

-- RIGHT — a materialized view stores results, but is vendor-specific and needs REFRESH
CREATE MATERIALIZED VIEW daily_totals AS
  SELECT day, SUM(amount) FROM sales GROUP BY day;
-- REFRESH MATERIALIZED VIEW daily_totals;   -- PostgreSQL syntax; route to dialect map
```

*Source: [PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html). Depth: this skill, §1; materialized-view refresh owned by `sql-standard-vs-dialect-map`.*
