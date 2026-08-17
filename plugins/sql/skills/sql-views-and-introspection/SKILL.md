---
name: sql-views-and-introspection
description: Guides SQL views and portable schema introspection — a view is a stored query (`CREATE VIEW`), not a table, re-run on every reference; only simple single-table views are automatically updatable while joins/aggregates/DISTINCT/GROUP BY/set-operation views are read-only by default; `WITH [LOCAL|CASCADED] CHECK OPTION` rejects an INSERT/UPDATE through a filtered view whose new row would fall outside the view (so it can't silently vanish or escape a security filter); and schema discovery should query the SQL-standard `INFORMATION_SCHEMA` (portable across PostgreSQL/MySQL/MariaDB/SQL Server) rather than vendor catalogs (`pg_catalog`, `sqlite_master`, `SHOW TABLES`, `PRAGMA`). Notes that SQLite has no `INFORMATION_SCHEMA` at all. Auto-invokes when writing or editing `CREATE VIEW`, updatable/security views, `WITH CHECK OPTION`, querying catalog/metadata, `INFORMATION_SCHEMA`/`SHOW`/`PRAGMA`/`pg_catalog`/`sqlite_master`, or generating schema-discovery or migration tooling queries.
allowed-tools: Read, Glob, Grep
---

# SQL Views and Introspection

> "`CREATE VIEW` defines a view of a query. The view is not physically materialized. Instead, the query is run every time the view is referenced in a query."
> — [PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)

> "The information schema is defined in the SQL standard and can therefore be expected to be portable and remain stable — unlike the system catalogs, which are specific to PostgreSQL and are modeled after implementation concerns."
> — [PostgreSQL — The Information Schema](https://www.postgresql.org/docs/current/information-schema.html)

A view is a *named query*, not a second copy of the data, and the database's *metadata is itself queryable as tables*. Both facts are governed by the relational set semantics in the foundation skill `sql-relational-and-null-discipline` — a view's result is an unordered set with no order unless its query says `ORDER BY`, and `WITH CHECK OPTION` is exactly the WHERE-keeps-TRUE rule (a row "vanishes" from a view precisely when the view's predicate stops being TRUE for it). This skill covers two things the standard pins down and the dialects scatter: building views that behave, and discovering schema *portably*.

---

## 1. What a View Is, and What It's For

A view is a stored `SELECT`. "The view is not physically materialized. Instead, the query is run every time the view is referenced" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)). It costs nothing extra at rest; it re-runs its query each time. Views earn their keep in three ways:

- **Encapsulation** — hide a gnarly join/aggregate behind a stable name, so callers depend on the view's shape, not the underlying tables.
- **Security** — expose a filtered or column-projected slice (`WHERE tenant_id = current_user_id()`, or omitting a `salary` column) and grant on the view, not the base table. Grants belong to `sql-privileges-and-access-control`.
- **Compatibility** — keep an old column name/shape alive as a view after the base table is refactored.

```sql
-- A security view: callers see only their tenant's active rows
CREATE VIEW active_customers AS
  SELECT id, name, email
  FROM customers
  WHERE tenant_id = current_setting('app.tenant_id')::int
    AND deleted_at IS NULL;
```

A plain (non-materialized) view has no stored data and no refresh. *Materialized* views (a stored, refreshable snapshot) are a vendor extension with engine-specific `REFRESH` syntax — out of scope here; route to `sql-standard-vs-dialect-map`.

---

## 2. Updatable vs Read-Only Views — Only Simple Views Are Writable

You can sometimes `INSERT`/`UPDATE`/`DELETE` *through* a view, but only when it is trivial enough to map each view row back to exactly one base-table row. "Simple views are automatically updatable: the system will allow `INSERT`, `UPDATE`, `DELETE`, and `MERGE` statements to be used on the view in the same way as on a regular table." A view qualifies only if ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)):

- exactly one entry in its `FROM` list (a table or another updatable view);
- no `WITH`, `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`, or `OFFSET` at the top level;
- no set operations (`UNION`, `INTERSECT`, `EXCEPT`) at the top level;
- no aggregates, window functions, or set-returning functions in the select list.

"A more complex view that does not satisfy all these conditions is read-only by default: the system will not allow an `INSERT`, `UPDATE`, `DELETE`, or `MERGE` on the view" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)).

```sql
-- WRITABLE — single table, plain projection: INSERT/UPDATE map back cleanly
CREATE VIEW v_users AS SELECT id, name, email FROM users;

-- READ-ONLY by default — a JOIN can't unambiguously route a write to one base row
CREATE VIEW v_user_orders AS
  SELECT u.id, u.name, o.total
  FROM users u JOIN orders o ON o.user_id = u.id;
-- INSERT INTO v_user_orders ... -> rejected
```

Writing through a complex view requires an `INSTEAD OF` trigger (PostgreSQL/SQL Server) or a vendor mechanism — engine-specific, route to `sql-standard-vs-dialect-map`. Default position: treat join/aggregate views as read-only.

---

## 3. `WITH CHECK OPTION` — Stop Rows From Vanishing Through the View (centerpiece)

An *updatable* filtered view has a silent trap: nothing stops you from inserting or updating a row through the view such that the new row no longer matches the view's `WHERE`. The write succeeds, but the row immediately falls *outside* the view — it "vanishes" from the view's results and, if the view was a security boundary, escapes the filter entirely.

`WITH CHECK OPTION` closes this. "When this option is specified, `INSERT`, `UPDATE`, and `MERGE` commands on the view will be checked to ensure that new rows satisfy the view-defining condition (that is, the new rows are checked to ensure that they are visible through the view). If they are not, the update will be rejected" ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)).

```sql
-- WRONG — a security view WITHOUT the check: the write escapes the filter
CREATE VIEW my_rows AS
  SELECT id, owner_id, body FROM notes WHERE owner_id = current_user_id();
-- This INSERT succeeds, but the new row has someone else's owner_id:
INSERT INTO my_rows (id, owner_id, body) VALUES (99, 'OTHER_USER', 'leaked');
-- The row is now in notes, invisible through my_rows, owned by another user.

-- RIGHT — CHECK OPTION rejects any write whose row wouldn't be visible here
CREATE VIEW my_rows AS
  SELECT id, owner_id, body FROM notes WHERE owner_id = current_user_id()
  WITH CHECK OPTION;
-- The same INSERT is now REJECTED: owner_id <> current_user_id() fails the check.
```

This is the WHERE/CHECK logic from `sql-relational-and-null-discipline` applied to writes: a row is "visible through the view" only when the view predicate is TRUE for it, so a predicate that goes FALSE on the new value is rejected. (Mind three-valued logic: a predicate that evaluates to `UNKNOWN` on a NULL is *not* TRUE, so the check still rejects it.)

### LOCAL vs CASCADED

When views are layered on views, the option's scope matters ([PostgreSQL — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)):

- **`LOCAL`** — "New rows are only checked against the conditions defined directly in the view itself. Any conditions defined on underlying base views are not checked (unless they also specify the `CHECK OPTION`)."
- **`CASCADED`** — "New rows are checked against the conditions of the view and all underlying base views." And: "If the `CHECK OPTION` is specified, and neither `LOCAL` nor `CASCADED` is specified, then `CASCADED` is assumed."

So the default is **CASCADED** (check this view *and* every view beneath it). `LOCAL` checks only the top view — which can let a write through that violates a lower view's filter. Prefer the default `CASCADED` for security views unless you have a specific reason.

---

## 4. Portable Introspection — Query `INFORMATION_SCHEMA`, Not Vendor Catalogs

Schema discovery (does this column exist? what type? list the tables) is exactly where engine lock-in sneaks in. The portable answer is the SQL-standard information schema: "an ANSI-standard set of read-only views that provide information about all of the tables, views, columns, and procedures in a database" ([Wikipedia — Information schema](https://en.wikipedia.org/wiki/Information_schema)), and it "can therefore be expected to be portable and remain stable — unlike the system catalogs, which are specific to PostgreSQL and are modeled after implementation concerns" ([PostgreSQL — Information Schema](https://www.postgresql.org/docs/current/information-schema.html)).

```sql
-- WRONG — pg_catalog is PostgreSQL-only; this query breaks on MySQL/SQL Server
SELECT relname FROM pg_catalog.pg_class WHERE relkind = 'r';

-- WRONG — SHOW TABLES is MySQL/MariaDB dialect, not standard SQL, not a queryable result set
SHOW TABLES;

-- RIGHT — standard, portable across PostgreSQL, MySQL, MariaDB, SQL Server
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public' AND table_type = 'BASE TABLE';

-- RIGHT — column metadata, portably
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'orders'
ORDER BY ordinal_position;
```

Core standard views: `information_schema.tables`, `.columns`, `.views`, `.table_constraints`, `.key_column_usage`, `.referential_constraints`, `.check_constraints` ([PostgreSQL — Information Schema](https://www.postgresql.org/docs/current/information-schema.html)). Constraint-introspection detail (FKs, check clauses) belongs to `sql-constraints-and-integrity`.

**The honest caveat:** the information schema is the portable *common denominator*. "The information schema views do not, however, contain information about PostgreSQL-specific features; to inquire about those you need to query the system catalogs" ([PostgreSQL — Information Schema](https://www.postgresql.org/docs/current/information-schema.html)). Reach for `pg_catalog` only for genuinely PG-specific facts (index methods, OIDs) — and know that query is then non-portable by definition.

---

## 5. The SQLite Deviation — No `INFORMATION_SCHEMA` At All

SQLite is the engine where the portable answer simply does not run: it has no information schema. It is listed among the "RDBMSs that do not support `information_schema`" ([Wikipedia — Information schema](https://en.wikipedia.org/wiki/Information_schema)). Instead, "every SQLite database contains a single 'schema table' that stores the schema for that database" ([SQLite — The Schema Table](https://www.sqlite.org/schematab.html)):

```sql
-- SQLite ONLY — list tables via the schema table (not portable elsewhere)
SELECT name FROM sqlite_schema WHERE type = 'table';
```

The schema table is `sqlite_schema(type, name, tbl_name, rootpage, sql)`; `type` is "'table', 'index', 'view', or 'trigger'" and `sql` "stores SQL text that describes the object" ([SQLite — The Schema Table](https://www.sqlite.org/schematab.html)). For historical compatibility it is also reachable as **`sqlite_master`** — the legacy name LLMs most often emit. Column-level detail (types, nullability) comes from `PRAGMA table_info(...)`, not a queryable standard view. The full SQLite catalog/`PRAGMA` map belongs to `sql-standard-vs-dialect-map`.

---

## 6. Portability Snapshot

| Capability | PostgreSQL | MySQL / MariaDB | SQL Server | SQLite | Oracle |
|---|---|---|---|---|---|
| `INFORMATION_SCHEMA` views | yes | yes | yes | **no** | **no** |
| Auto-updatable simple views | yes | yes | yes | via trigger | yes |
| `WITH [LOCAL\|CASCADED] CHECK OPTION` | yes | yes | yes (no LOCAL/CASCADED kw) | no | yes |
| List-tables fallback | — | `SHOW TABLES` | — | `sqlite_schema`/`sqlite_master` + `PRAGMA` | `ALL_TABLES` etc. |
| Vendor catalog | `pg_catalog` | `mysql`/IS | `sys.*` | `sqlite_schema` | data-dictionary views |

The standard says: a view is a stored query; simple single-table views are updatable; `WITH CHECK OPTION` rejects writes whose row would not be visible through the view; `INFORMATION_SCHEMA` is the portable metadata surface. The deviation that bites hardest is **SQLite has no `INFORMATION_SCHEMA`** — a schema-discovery query written against the standard returns nothing/errors there, and must fall back to `sqlite_schema`/`PRAGMA`. Oracle likewise lacks it (uses data-dictionary views like `ALL_TABLES`). All dialect spellings route to **`sql-standard-vs-dialect-map`**.

---

## 7. Who Suffers When This Is Mishandled

These mistakes don't error at write time — they fail later, somewhere else:

- The **migration tool that broke on a different engine.** It introspected with `SELECT relname FROM pg_catalog.pg_class` on a developer's PostgreSQL box, shipped, and threw "no such table: pg_class" the first time a customer pointed it at SQLite (or "Invalid object name 'pg_class'" on SQL Server). The portable `information_schema.tables` query would have run on every engine that has one — and would have made the SQLite gap *visible* at the boundary instead of at 2 a.m. in production.
- **The data that escaped a security view.** A `WHERE tenant_id = …` view without `WITH CHECK OPTION` looked like an isolation boundary. An `INSERT`/`UPDATE` through it set a *different* `tenant_id`; the write succeeded and the row landed in another tenant's data — invisible through the very view that was supposed to contain it. No error, no log, just a cross-tenant leak found in an audit. `WITH CHECK OPTION` (defaulting to `CASCADED`) would have rejected the write.
- **The maintainer who tried to write through a join view** and got a flat rejection with no obvious cause, not realizing only simple single-table views are auto-updatable (§2).

Portable introspection and `WITH CHECK OPTION` are both gifts to whoever runs this code on an engine, or with data, you didn't test.

---

## 8. Routing to Related Skills

- `sql-relational-and-null-discipline` — **foundation.** Set semantics (a view result is unordered), and the WHERE-keeps-TRUE / three-valued-logic rule that `WITH CHECK OPTION` enforces on writes (§3).
- `sql-constraints-and-integrity` — constraint introspection detail (FKs, CHECK clauses via `information_schema.table_constraints` / `.check_constraints`), and the UNKNOWN-passes-CHECK pitfall (§4).
- `sql-privileges-and-access-control` — granting on a view instead of the base table; views as a privilege/security surface (§1).
- `sql-standard-vs-dialect-map` — `SHOW TABLES`, `pg_catalog`, `sqlite_schema`/`sqlite_master`/`PRAGMA`, Oracle data-dictionary views, materialized-view `REFRESH`, and `INSTEAD OF` triggers (§§1, 2, 4, 5, 6).

---

## 9. Reference Files

High-frequency view/introspection anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
