# Research: sql-views-and-introspection

Research backing for skill #18 (`sql-views-and-introspection`). Every claim below is
tagged with URL, the section it came from, a verbatim quote, and why it matters for the
skill. Accessed 2026-06-26.

---

## Source 1 — PostgreSQL: CREATE VIEW

URL: https://www.postgresql.org/docs/current/sql-createview.html

### 1a. What a view is / purpose

**Section:** Description.
**Verbatim:** "`CREATE VIEW` defines a view of a query. The view is not physically
materialized. Instead, the query is run every time the view is referenced in a query."

**Why it matters:** Establishes the baseline — a (non-materialized) view is a stored
query, re-run on every reference. It is the encapsulation / abstraction / security
boundary primitive. Distinguishes it from materialized views (out of scope, route to
dialect map).

### 1b. Automatically updatable (simple) views

**Section:** "Updatable Views."
**Verbatim:** "Simple views are automatically updatable: the system will allow `INSERT`,
`UPDATE`, `DELETE`, and `MERGE` statements to be used on the view in the same way as on a
regular table. A view is automatically updatable if it satisfies all of the following
conditions:
- The view must have exactly one entry in its `FROM` list, which must be a table or
  another updatable view.
- The view definition must not contain `WITH`, `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`,
  or `OFFSET` clauses at the top level.
- The view definition must not contain set operations (`UNION`, `INTERSECT` or `EXCEPT`)
  at the top level.
- The view's select list must not contain any aggregates, window functions or
  set-returning functions."

**Why it matters:** This is the exact, citeable list of what makes a view updatable vs
read-only. Single-table, no DISTINCT/GROUP BY/aggregates/set-ops → writable. Anything
fancier → read-only by default. Directly answers spec point (c).

### 1c. Complex views are read-only by default

**Section:** "Updatable Views."
**Verbatim:** "A more complex view that does not satisfy all these conditions is read-only
by default: the system will not allow an `INSERT`, `UPDATE`, `DELETE`, or `MERGE` on the
view."

**Why it matters:** Confirms the read-only default; writes to such views need INSTEAD OF
triggers (PG) / a vendor mechanism — beyond scope, but the failure mode (write rejected)
is worth naming.

### 1d. WITH CHECK OPTION — core behavior (CENTERPIECE)

**Section:** "Parameters → CHECK OPTION."
**Verbatim:** "When this option is specified, `INSERT`, `UPDATE`, and `MERGE` commands on
the view will be checked to ensure that new rows satisfy the view-defining condition (that
is, the new rows are checked to ensure that they are visible through the view). If they
are not, the update will be rejected."

**Why it matters:** This is the centerpiece. Without CHECK OPTION you can INSERT/UPDATE a
row *through* a filtered view such that the row immediately falls outside the view's WHERE
predicate — it "vanishes" from the view (and escapes any security filter the view was
meant to enforce). CHECK OPTION rejects exactly those writes. Directly answers spec
point (b).

### 1e. LOCAL vs CASCADED

**Section:** "Parameters → CHECK OPTION → LOCAL / CASCADED."
**Verbatim (LOCAL):** "New rows are only checked against the conditions defined directly
in the view itself. Any conditions defined on underlying base views are not checked
(unless they also specify the `CHECK OPTION`)."
**Verbatim (CASCADED):** "New rows are checked against the conditions of the view and all
underlying base views. If the `CHECK OPTION` is specified, and neither `LOCAL` nor
`CASCADED` is specified, then `CASCADED` is assumed."

**Why it matters:** LOCAL checks only this view's predicate; CASCADED checks this view
plus every underlying view's predicate. Default is CASCADED. Matters when views are
layered — LOCAL can let a row through that violates a lower view's filter.

---

## Source 2 — PostgreSQL: The Information Schema

URL: https://www.postgresql.org/docs/current/information-schema.html

### 2a. Portability of the information schema

**Section:** Chapter 35 intro / "The Information Schema."
**Verbatim:** "The information schema is defined in the SQL standard and can therefore be
expected to be portable and remain stable — unlike the system catalogs, which are specific
to PostgreSQL and are modeled after implementation concerns."

**Why it matters:** The headline claim of the skill: query INFORMATION_SCHEMA for
portable, stable metadata; pg_catalog is PG-specific and modeled on internals. Directly
answers spec point (a).

### 2b. Information schema lacks vendor-specific features

**Section:** Chapter 35 intro.
**Verbatim:** "The information schema views do not, however, contain information about
PostgreSQL-specific features; to inquire about those you need to query the system catalogs
or other PostgreSQL-specific views."

**Why it matters:** Honest caveat — INFORMATION_SCHEMA is the portable common denominator;
vendor-only features (e.g. PG index types) still require pg_catalog. Nuance for the
portability block.

### 2c. Core views available

**Section:** Chapter 35 table of contents (66 views).
**Verbatim (names):** `tables`, `columns`, `table_constraints`, `views`,
`view_table_usage`, `schemata`, `check_constraints`, `referential_constraints`,
`key_column_usage`, `constraint_column_usage`, `routines`, `sequences`, `domains`.

**Why it matters:** The concrete tables to query for schema discovery —
`information_schema.tables`, `information_schema.columns`,
`information_schema.table_constraints`, etc. Constraint-introspection detail routes to
`sql-constraints-and-integrity`.

---

## Source 3 — Wikipedia: Information schema

URL: https://en.wikipedia.org/wiki/Information_schema

### 3a. ANSI standard, read-only views

**Section:** Lead.
**Verbatim:** "the **information schema** (`information_schema`) is an ANSI-standard set of
read-only views that provide information about all of the tables, views, columns, and
procedures in a database."

**Why it matters:** Confirms it is an ANSI/ISO SQL standard (since SQL-92), not a
vendor invention — the portability argument's foundation.

### 3b. Which engines implement it / which do not

**Section:** body / "RDBMSs that do not support information_schema."
**Verbatim:** "As a notable exception among major database systems, Oracle does not as of
2015 implement the information schema." Engines listed as implementing it include MySQL,
PostgreSQL, Microsoft SQL Server, and MariaDB. SQLite and Oracle are listed under "RDBMSs
that do not support information_schema."

**Why it matters:** Drives the portability matrix: PostgreSQL / MySQL / MariaDB / SQL
Server have it; SQLite and Oracle do not. SQLite is the deviation the skill centers on;
Oracle is a secondary note.

---

## Source 4 — SQLite: The Schema Table

URL: https://www.sqlite.org/schematab.html

### 4a. sqlite_schema exists; no information_schema

**Section:** Lead.
**Verbatim:** "Every SQLite database contains a single 'schema table' that stores the
schema for that database."

**Schema:** "CREATE TABLE sqlite_schema(type text, name text, tbl_name text, rootpage
integer, sql text);"

**Column meanings (verbatim):**
- type: "will be one of the following text strings: 'table', 'index', 'view', or 'trigger'
  according to the type of object defined."
- name: "will hold the name of the object."
- tbl_name: "holds the name of a table or view that the object is associated with."
- rootpage: "stores the page number of the root b-tree page for tables and indexes."
- sql: "stores SQL text that describes the object."

**Why it matters:** SQLite has NO information_schema at all (confirmed by Source 3 too).
Schema introspection in SQLite goes through `sqlite_schema` plus `PRAGMA`. Directly
answers spec point (d).

### 4b. Alternate names

**Section:** "Alternative Names."
**Verbatim:** "The schema table can always be referenced using the name 'sqlite_schema',
especially if qualifed by the schema name like 'main.sqlite_schema' or
'temp.sqlite_schema'. But for historical compatibility, some alternative names are also
recognized, including: (1) sqlite_master (2) sqlite_temp_schema (3) sqlite_temp_master. ...
alternative (1) works anywhere."

**Why it matters:** `sqlite_master` is the legacy/widely-seen name for `sqlite_schema`.
LLMs frequently emit `sqlite_master`; this is the SQLite-specific catalog, not portable.
Column-level introspection in SQLite needs `PRAGMA table_info(...)` (the `sql` column only
gives raw DDL text). Detailed SQLite catalog/PRAGMA map routes to
`sql-standard-vs-dialect-map`.

---

## Synthesis — the four load-bearing points

1. **Query INFORMATION_SCHEMA, not vendor catalogs, when portability matters.** It is
   SQL-standard (ANSI, SQL-92+) and "can therefore be expected to be portable and remain
   stable," whereas pg_catalog / sqlite_master / `SHOW TABLES` are vendor-specific (2a,
   3a). Tooling that queries pg_catalog breaks against MySQL/SQL Server.
2. **WITH CHECK OPTION stops writes that would make the row disappear from the view.**
   Without it, an INSERT/UPDATE through a filtered view can land outside the view's
   predicate — the row vanishes from the view and escapes any security filter (1d). LOCAL
   = this view only; CASCADED (default) = this view + all underlying views (1e).
3. **Only simple single-table views are auto-updatable.** Views with joins, aggregates,
   DISTINCT, GROUP BY/HAVING, LIMIT/OFFSET, window functions, or set operations are
   read-only by default (1b, 1c).
4. **SQLite has no INFORMATION_SCHEMA.** It uses `sqlite_schema` (legacy `sqlite_master`)
   plus `PRAGMA`. So the portable query simply does not run there — the one engine where
   the standard answer fails (3b, 4a, 4b).
