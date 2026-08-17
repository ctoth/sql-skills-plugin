---
name: sql-standard-vs-dialect-map
description: >-
  The portability index for SQL — maps each standard feature to whether the readable engines (PostgreSQL, SQLite, MySQL/MariaDB, plus notes on SQL Server / Oracle / DuckDB) support it and how they spell it. Covers `OFFSET … FETCH` vs `LIMIT`, `GENERATED AS IDENTITY` vs `SERIAL`/`AUTO_INCREMENT`/`IDENTITY(1,1)`, `MERGE` vs `ON CONFLICT` vs `ON DUPLICATE KEY UPDATE`, `||` vs `+` vs `CONCAT()`, `IS DISTINCT FROM` vs `at most greater than `, `LISTAGG`/`STRING_AGG`/`GROUP_CONCAT`, `INFORMATION_SCHEMA`. Auto-invokes when porting SQL between engines, targeting a specific database, choosing between a standard and a vendor spelling, or on "does Postgres/MySQL/SQLite support X" / "is this portable" / "how does engine Y spell Z" questions. The index every other SQL skill's Portability section routes to.
allowed-tools: Read, Glob, Grep
---

# SQL Standard vs Dialect Map

> "SQL is well known for its lack of portability ... The standard is generally followed, but each vendor has its own dialect with proprietary extensions, and conformance is only ever partial."
> — paraphrasing the [modern-sql.com — SQL Standard](https://modern-sql.com/standard) survey of versions and optional features

> "Although the SQL standard is fairly comprehensive ... PostgreSQL does not implement every feature, and conversely, some PostgreSQL features are not in the standard."
> — [PostgreSQL — SQL Conformance](https://www.postgresql.org/docs/current/features.html)

The SQL standard (ISO/IEC 9075) defines a mandatory **Core SQL** plus a long list of *optional* features. No engine implements all of it, every engine adds proprietary spellings, and "optional" means a conforming engine is free to omit a feature entirely. So the same statement can be standard, ubiquitous, *and* unportable at once (`LIMIT` is the textbook case — everywhere except Oracle/SQL Server, and not in the standard at all).

This skill is the **portability index** for the SQL plugin. It does not teach any feature — each feature's *meaning, traps, and correct usage* live in its own skill. This file answers exactly one question: **does engine X support feature Y, and how does it spell it?** Every other skill's "Portability" section routes here; this is the SQL analogue of a language version/feature map. It builds on the set-semantics and three-valued-logic floor in `sql-relational-and-null-discipline` (the policy root) — portability never overrides correctness.

Every row below **is** a portability statement: read it as "the standard says X; PostgreSQL ✓ / SQLite ~ / MySQL ✗ (spells Y)." Legend: **✓** native standard-ish support · **~** supported but spelled differently or version-gated · **✗** absent (use the fallback in the notes). The most-used ~13 features are tabled below; the exhaustive feature×engine matrix lives in [`references/dialect-matrix.md`](references/dialect-matrix.md).

---

## 1. Pagination — `OFFSET … FETCH` vs `LIMIT`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes (SQL Server / Oracle / DuckDB) |
|---|---|---|---|---|
| `OFFSET m ROWS FETCH FIRST n ROWS ONLY [WITH TIES]` | ✓ | ✗ | ✗ | SQL Server 2012+; **Oracle only 12c+**; DuckDB ✓ |
| `LIMIT n OFFSET m` (non-standard) | ✓ | ✓ | ✓ | SQL Server ✗ (use `TOP`/`OFFSET FETCH`); **Oracle ✗ pre-12c** (use `ROWNUM`); DuckDB ✓ |
| `WITH TIES` | ✓ | ✗ | ✗ | SQL Server/Oracle ✓ |

`FETCH FIRST` is the standard but "MySQL and SQLite do not support `fetch` at all"; `LIMIT` is ubiquitous-but-non-standard and absent on Oracle pre-12c / SQL Server. Who suffers: the dev who shipped `LIMIT 10` against an Oracle 11g target where it does not parse. Feature owner: **`sql-pagination-and-keyset`**.

---

## 2. Auto-Generated Keys — `GENERATED AS IDENTITY` vs `SERIAL`/`AUTO_INCREMENT`/`IDENTITY`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `GENERATED { ALWAYS \| BY DEFAULT } AS IDENTITY` (T174/T178) | ✓ (10+) | ✗ | ✗ | SQL Server ✗ (use `IDENTITY(1,1)`); Oracle ✓ (12c+); DuckDB ✓ |
| Vendor auto-increment | `SERIAL`/`BIGSERIAL` (legacy) | `INTEGER PRIMARY KEY` (rowid alias) | `AUTO_INCREMENT` | SQL Server `IDENTITY(1,1)`; Oracle pre-12c sequence + trigger |
| Computed `GENERATED ALWAYS AS (expr) [STORED\|VIRTUAL]` | ✓ (STORED only) | ✓ (VIRTUAL default) | ✓ (VIRTUAL default) | SQL Server computed columns; widely available |

Prefer the standard `GENERATED AS IDENTITY` on PostgreSQL; `SERIAL` is legacy. SQLite's rowid `INTEGER PRIMARY KEY` and MySQL's `AUTO_INCREMENT` are the portable substitutes. Feature owner: **`sql-generated-and-identity-columns`**.

---

## 3. Upsert — `MERGE` vs `ON CONFLICT` vs `ON DUPLICATE KEY UPDATE`

| Mechanism | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| Standard `MERGE` | ~ (**15+**) | ✗ | ✗ | SQL Server / Oracle / DB2 ✓ |
| `INSERT … ON CONFLICT (…) DO UPDATE` | ✓ (9.5+) | ✓ (3.24+) | ✗ | new-value alias `excluded.col` |
| `INSERT … ON DUPLICATE KEY UPDATE` | ✗ | ✗ | ✓ | new-value alias `new.col` (8.0.19+); else `VALUES(col)` |

There is **no single portable upsert**. `ON CONFLICT` (PG/SQLite) and `ON DUPLICATE KEY UPDATE` (MySQL) are different statements with different conflict-target and new-value-alias rules; `MERGE` arrived in PostgreSQL only in 15. Feature owner: **`sql-merge-and-upsert`**.

---

## 4. String Concatenation — `||` vs `+` vs `CONCAT()`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `a \|\| b` | ✓ | ✓ | ✗ by default (✓ only under `PIPES_AS_CONCAT`) | Oracle ✓; SQL Server ✗ (uses `+`) |
| `CONCAT(a, b, …)` | ✓ | ✓ (3.44+) | ✓ | universal escape hatch |
| `a + b` as concat | ✗ (numeric add) | ✗ | ✗ | **SQL Server only** |

`||` is standard but MySQL reads it as logical OR by default, and SQL Server uses `+`. `CONCAT()` is the most portable. NULL handling also diverges: `'a' || NULL` is `NULL` everywhere except Oracle (which treats `''` as NULL → yields `'a'`) and MySQL's `CONCAT` (NULL-propagating). Owners: **`sql-expressions-case-and-functions`**; NULL semantics in `sql-relational-and-null-discipline`.

---

## 5. Null-Safe Equality — `IS [NOT] DISTINCT FROM` vs `<=>`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `a IS [NOT] DISTINCT FROM b` | ✓ | ~ (use `IS`/`IS NOT` — they are null-safe) | ✗ (use `<=>`) | SQL Server ✗ (2022+ ✓); Oracle ✗ no native spelling |
| `a <=> b` (spaceship) | ✗ | ✗ | ✓ | MySQL-only; "equivalent to … `IS NOT DISTINCT FROM`" |

The standard `IS NOT DISTINCT FROM` is the null-aware equals; MySQL spells it `<=>`; SQLite's bare `IS`/`IS NOT` already compare NULLs as equal. Owner: **`sql-relational-and-null-discipline`** §8.

---

## 6. String Aggregation — `LISTAGG` / `STRING_AGG` / `GROUP_CONCAT`

| Standard (SQL:2016 `LISTAGG`) | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `LISTAGG(expr, sep) WITHIN GROUP (ORDER BY …)` | ✗ (use `string_agg`) | ✗ (use `group_concat`) | ✗ (use `GROUP_CONCAT`) | Oracle / DB2 / SQL Server 2017+ ✓ |
| Vendor spelling | `string_agg(expr, sep)` | `group_concat(expr, sep)` | `GROUP_CONCAT(expr SEPARATOR sep)` | array form: PG `array_agg`; DuckDB `string_agg`/`list` |

Four names for one operation; ordering syntax differs too (`WITHIN GROUP` for LISTAGG/PG, `ORDER BY` inside the call for MySQL). Owner: **`sql-aggregation-and-grouping`**.

---

## 7. Catalog Introspection — `INFORMATION_SCHEMA` vs `sqlite_master`/`PRAGMA` vs `SHOW`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `INFORMATION_SCHEMA.*` views | ✓ | ✗ | ✓ | SQL Server ✓; **Oracle ✗** (data-dictionary `ALL_*`/`USER_*`) |
| List tables / columns (vendor) | `pg_catalog`, `\dt` | `sqlite_schema`/`sqlite_master` + `PRAGMA table_info(t)` | `SHOW TABLES` / `SHOW COLUMNS` | DuckDB ✓ INFORMATION_SCHEMA + `PRAGMA` |

`INFORMATION_SCHEMA` is standard and portable across PG/MySQL/SQL Server, but **SQLite and Oracle have it not** — SQLite uses `sqlite_master`/`PRAGMA`, Oracle uses its data dictionary. Owner: **`sql-views-and-introspection`**.

---

## 8. JSON Access — standard functions vs `->`/`->>`

| Standard (SQL:2016) | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `JSON_VALUE` / `JSON_QUERY` / `JSON_TABLE` | ~ (17+) | ✗ (use `json_extract`/`json_each`) | ~ (`JSON_VALUE` 8.0.21+, `JSON_TABLE` 8.0.4+) | Oracle ✓; SQL Server partial |
| `->` (get JSON) / `->>` (get text) | ✓ | ✓ (3.38.0+) | ✓ (8.x) | **Oracle ✗ no native operator**; vendor, not standard |
| Native JSON type | `jsonb` | text + internal blob | `JSON` | DuckDB `JSON` |

The standard `JSON_*` functions are arriving unevenly (PG only at 17); the `->`/`->>` operators are near-universal but vendor extensions, absent on Oracle. Owner: **`sql-json`**.

---

## 9. Lateral Joins — `LATERAL` vs `CROSS/OUTER APPLY`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `[LEFT] JOIN LATERAL (…) ON …` (SQL:1999) | ✓ | ✗ | ~ (**8.0.14+**) | Oracle ✓ (12c+); DuckDB ✓ |
| `CROSS APPLY` / `OUTER APPLY` | ✗ | ✗ | ✗ | **SQL Server** (no `LATERAL` kw); Oracle accepts both |

SQL Server spells lateral as `APPLY` and has no `LATERAL` keyword; SQLite has neither — emulate with a correlated subquery or `ROW_NUMBER() OVER (PARTITION BY …)`. Owner: **`sql-lateral-and-correlated-derived`**.

---

## 10. Set Operations — `EXCEPT` vs `MINUS`, and the `ALL`/precedence gaps

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `UNION` / `UNION ALL` | ✓ | ✓ | ✓ | universal |
| `INTERSECT` / `EXCEPT` | ✓ | ✓ | ~ (**8.0.31+** / MariaDB 10.3+) | **Oracle spells `EXCEPT` as `MINUS`** |
| `INTERSECT ALL` / `EXCEPT ALL` | ✓ | ✗ | ~ (8.0.31+) | Oracle ✗ |
| `INTERSECT` binds tighter than `UNION`/`EXCEPT` | ✓ | ✗ (**all left-to-right**) | ✓ | parenthesize for portability |

Oracle's `MINUS`, MySQL's late (8.0.31) arrival of `INTERSECT`/`EXCEPT`, and SQLite's flat left-to-right precedence (no INTERSECT priority) are the three traps. Owner: **`sql-set-operations`**.

---

## 11. Identifier Quoting & Case-Folding — `"double"` vs backtick vs `ANSI_QUOTES`

| Behavior | Standard / PostgreSQL | SQLite | MySQL (default) | MySQL (`ANSI_QUOTES`) |
|---|---|---|---|---|
| `'abc'` | string literal | string literal | string literal | string literal |
| `"abc"` | **identifier (name)** | identifier (also accepts `'…'`/`` `…` ``) | **string literal (deviation)** | identifier |
| Native identifier quote | `"…"` | `"…"` / `` `…` `` / `[…]` | `` `…` `` (backtick) | `"…"` or `` `…` `` |
| Unquoted name case-folds to | **lower** (PG) / **upper** (standard) | case-preserving, case-insensitive compare | filesystem-dependent | same as default |

The damaging cell is MySQL's default `"abc"` = *string*: `WHERE name = "John"` passes review on MySQL, then breaks porting SQLite→PostgreSQL where `"John"` is a (nonexistent) column name. Write to the standard — single-quote every string, lowercase snake_case every identifier — and you never touch either deviation. Owner: **`sql-style-and-naming`**.

---

## 12. Boolean — `BOOLEAN` vs `BIT` vs `TINYINT(1)`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `BOOLEAN` (true/false/unknown) | ✓ | ✗ (integer 0/1, no BOOLEAN class) | ✗ (`BOOLEAN` is an alias for `TINYINT(1)`) | **SQL Server `BIT`** (0/1, no boolean type); Oracle ✗ pre-23ai (use `NUMBER(1)`/`CHAR`); DuckDB ✓ |

Only PostgreSQL (and DuckDB) have a real three-valued `BOOLEAN`. MySQL stores `TINYINT(1)`, SQL Server uses `BIT`, SQLite uses integer 0/1. Comparisons like `WHERE active` (bare boolean) are portable only where a true boolean exists. Owner: **`sql-data-types-and-numerics`**.

---

## 13. Datetime "now" and date math — `CURRENT_TIMESTAMP` vs `NOW()`/`GETDATE()`/`SYSDATE`

| Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|
| `CURRENT_TIMESTAMP` / `CURRENT_DATE` | ✓ | ✓ | ✓ | SQL Server ✓; Oracle ✓ — **the portable choice** |
| Vendor "now" | `now()` | `datetime('now')` | `NOW()`/`SYSDATE()` | SQL Server `GETDATE()`/`SYSDATETIME()`; Oracle `SYSDATE`/`SYSTIMESTAMP` |
| Date arithmetic | `ts + INTERVAL '1 day'` | `date(ts,'+1 day')` / `strftime` | `DATE_ADD(ts, INTERVAL 1 DAY)` | SQL Server `DATEADD`/`DATEDIFF`; Oracle `ts + 1` or `INTERVAL` |
| Truncate/bucket | `date_trunc('day', ts)` | `strftime('%Y-%m-%d', ts)` | `DATE_FORMAT` / `DATE(ts)` | SQL Server `DATETRUNC` (2022+); DuckDB `date_trunc` |

Use the standard `CURRENT_TIMESTAMP` (not `NOW()`) for portability. Date *math* has no portable spelling at all — `INTERVAL`, `DATEADD`, and `strftime` are mutually exclusive. SQLite has no dedicated temporal type. Owner: **`sql-datetime-and-intervals`**.

---

## 14. Default Transaction Isolation Level

| Engine | Default isolation level | Note |
|---|---|---|
| PostgreSQL | `READ COMMITTED` | the four standard level *names* are accepted everywhere |
| MySQL / InnoDB | **`REPEATABLE READ`** | the same app code runs at a *different* level per engine |
| SQLite | **`SERIALIZABLE`** | single-writer model |
| SQL Server | `READ COMMITTED` | Oracle: `READ COMMITTED` |

The level *names* are standard; the **default** differs, so identical un-configured transaction code has different concurrency semantics on each engine. Set the level explicitly for portable behavior. Owner: **`sql-transactions-and-isolation`**.

---

## 15. Cast, substring, null-default — small but pervasive

| Operation | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes |
|---|---|---|---|---|---|
| Cast | `CAST(x AS t)` | `CAST` / `x::t` | `CAST` | `CAST` | SQL Server `CAST`/`CONVERT`; `::` is PG-only |
| Substring | `SUBSTRING(s FROM n FOR m)` | both forms | `SUBSTR(s,n,m)` | `SUBSTRING`/`SUBSTR` | Oracle `SUBSTR` |
| Null default | `COALESCE(x, d)` | `COALESCE` | `COALESCE`/`IFNULL` | `COALESCE`/`IFNULL` | Oracle `NVL`; SQL Server `ISNULL` — `COALESCE` is the portable one |

Always prefer `CAST` over `::` and `COALESCE` over `IFNULL`/`NVL`/`ISNULL`. Owner: **`sql-expressions-case-and-functions`**.

---

## 16. Advanced / Low-Portability Features — confirm before recommending

| Feature | Availability | Portable fallback |
|---|---|---|
| Window functions (SQL:2003) | broad: PG 8.4+, SQLite 3.25+, **MySQL 8.0+ only**, MariaDB 10.2+ | none needed on modern engines |
| `MATCH_RECOGNIZE` (SQL:2016 R010, optional) | **very low**: Oracle 12cR1+/Trino/Snowflake/DB2 yes; PG 18+, MySQL 9.7+, SQLite 3.53+, SQL Server 2025+ | gaps-and-islands via window functions |
| Temporal / system-versioned tables (SQL:2011) | **low**: MariaDB 10.3+, SQL Server 2016+, DB2/HANA; Oracle ~ (Flashback); **PG/MySQL/SQLite none** | hand-modeled history table + triggers |
| SQL/PGQ property-graph queries (SQL:2023) | **almost none**: Oracle 23ai GA only | recursive CTE |
| `FULL [OUTER] JOIN` | PG ✓, SQLite 3.39.0+, **MySQL ✗** | `LEFT … UNION … RIGHT` |
| `QUALIFY` (filter a window inline) | **nowhere** in PG/SQLite/MySQL (Snowflake/DuckDB/Teradata yes) | wrap window in a subquery/CTE |

These are the rows where "is it portable?" is usually **no**. When the target engine is on the ✗ side, the correct answer is the fallback, not the syntax that will not parse. Deep coverage: `sql-match-recognize`, `sql-temporal-tables`, `sql-property-graph-queries`, `sql-window-functions`.

---

## 17. How to Use This Map

1. **Identify the feature** the user is about to write (upsert, pagination, JSON access…).
2. **Look up the target engine's column.** ✓ = write the standard spelling; ~ = use the noted vendor spelling or check the version gate; ✗ = use the fallback in the notes.
3. **Default to the standard spelling** when more than one engine is in play — it is the intersection that ports. Reach for a vendor spelling only when targeting one engine, and say so.
4. **For depth, route to the feature's own skill** (named per section) — this map says *whether and how*, not *why*.

The recurring lesson: a statement can be standard, common, and unportable simultaneously. Verify the cell before claiming "this works on X."

---

## 18. Who Suffers When Portability Is Assumed

Portability bugs do not error in the IDE — they parse on the dev's engine and fail on the target's:

- The **backend dev** who shipped `SELECT … LIMIT 10` to an Oracle 11g target and got `ORA-00933` in production — `LIMIT` never existed there pre-12c (§1).
- The **maintainer** porting a SQLite schema to PostgreSQL whose `WHERE name = "active"` silently broke: MySQL/SQLite tolerated the double-quoted *string*, PostgreSQL read `"active"` as a missing column (§11).
- The **data engineer** whose `INSERT … ON DUPLICATE KEY UPDATE` upsert was rewritten for a PostgreSQL warehouse and threw a syntax error — PG speaks `ON CONFLICT`, and `MERGE` only since 15 (§3).
- The **analyst** whose `EXCEPT` report returned a syntax error on Oracle, which spells set difference `MINUS` (§10).
- The **on-call engineer** debugging a concurrency anomaly that only reproduces on MySQL, because its default `REPEATABLE READ` differs from PostgreSQL's `READ COMMITTED` (§14).

Checking the cell before you write the spelling is a gift to whoever runs the query on the *other* engine.

---

## 19. Routing

This skill is the **index every other SQL skill's Portability section links to**, the way every Go skill routes to `go-version-feature-map`. Inbound from all 29 siblings; the highest-traffic outbound owners:

- `sql-pagination-and-keyset` (§1) · `sql-generated-and-identity-columns` (§2) · `sql-merge-and-upsert` (§3)
- `sql-relational-and-null-discipline` (§5, foundation/policy root) · `sql-aggregation-and-grouping` (§6) · `sql-views-and-introspection` (§7)
- `sql-json` (§8) · `sql-lateral-and-correlated-derived` (§9) · `sql-set-operations` (§10) · `sql-style-and-naming` (§11)
- `sql-data-types-and-numerics` (§12) · `sql-datetime-and-intervals` (§13) · `sql-transactions-and-isolation` (§14)
- `sql-match-recognize`, `sql-temporal-tables`, `sql-property-graph-queries` (§16, the low-portability anchors)

---

## 20. Reference Files

The exhaustive feature×engine matrix — every feature in this map plus the long tail, as full markdown tables:

[references/dialect-matrix.md](references/dialect-matrix.md)

High-frequency portability mistakes in LLM-generated SQL, each with wrong/right code and a source:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
