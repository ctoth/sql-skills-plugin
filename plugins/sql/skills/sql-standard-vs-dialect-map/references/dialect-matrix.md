# SQL Dialect Matrix — Exhaustive Feature × Engine Table
## Contents

- [A. Query & result shaping](#a-query-result-shaping)
- [B. Joins & lateral](#b-joins-lateral)
- [C. Subqueries & set operations](#c-subqueries-set-operations)
- [D. CTE & recursion](#d-cte-recursion)
- [E. Windowing & pattern matching](#e-windowing-pattern-matching)
- [F. Aggregation](#f-aggregation)
- [G. DML & upsert](#g-dml-upsert)
- [H. DDL, keys & constraints](#h-ddl-keys-constraints)
- [I. Data types](#i-data-types)
- [J. Datetime](#j-datetime)
- [K. JSON](#k-json)
- [L. Catalog / introspection & views](#l-catalog-introspection-views)
- [M. Identifiers, transactions, privileges](#m-identifiers-transactions-privileges)
- [N. Advanced / low-portability (confirm before recommending)](#n-advanced-low-portability-confirm-before-recommending)
- [Cross-cutting cautions](#cross-cutting-cautions)


The full portability matrix the parent `SKILL.md` links to. Rows = standard feature;
columns = **Standard spelling | PostgreSQL | SQLite | MySQL/MariaDB | Notes (SQL Server /
Oracle / DuckDB)**. Legend: **✓** native standard-ish support · **~** supported but
spelled differently / version-gated · **✗** absent (fallback in notes). Every row is
transcribed from a sibling skill's settled Portability section (the feature owner is
named); no claim here contradicts those skills. For the *meaning* and *traps* of any
feature, route to its owning skill.

---

## A. Query & result shaping

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `LIMIT n` | ✗ (non-standard) | ✓ | ✓ | ✓ | SQL Server ✗ (`TOP`/`OFFSET FETCH`); Oracle ✗ pre-12c (`ROWNUM`); DuckDB ✓ | pagination-and-keyset |
| `OFFSET m ROWS FETCH FIRST n ROWS ONLY` | ✓ | ✓ | ✗ | ✗ | SQL Server 2012+; Oracle 12c+; DuckDB ✓ | pagination-and-keyset |
| `WITH TIES` | ✓ | ✓ | ✗ | ✗ | SQL Server / Oracle ✓ | pagination-and-keyset |
| Row-value compare `(a,b) < (x,y)` (keyset) | ✓ | ✓ | ✗ | ~ (8.0+) | SQL Server ✗; Oracle ✓ | pagination-and-keyset |
| `DISTINCT` | ✓ | ✓ | ✓ | ✓ | universal | select-and-query-processing |
| `DISTINCT ON (cols)` | ✗ | ✓ (PG-only) | ✗ | ✗ | use `GROUP BY`/window elsewhere; DuckDB ✓ | select-and-query-processing |
| Alias visible in `WHERE` | ✗ | ✗ | ~ (loose) | ✗ | repeat the expression — portable rule: alias only in `ORDER BY` | select-and-query-processing |
| Alias visible in `HAVING`/`GROUP BY` | ✗ | ✗ (HAVING) / ✓ ext (GROUP BY) | ~ | ✓ ext | not portable; write the expression | select-and-query-processing |
| `NULLS FIRST` / `NULLS LAST` | ✓ | ✓ | ✓ | ✗ (use `col IS NULL, col`) | Oracle ✓ | relational-and-null-discipline |
| `ORDER BY ordinal` (by position) | ✓ | ✓ | ✓ | ✓ | discouraged but portable | select-and-query-processing |

## B. Joins & lateral

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `INNER` / `LEFT` / `CROSS JOIN` | ✓ | ✓ | ✓ | ✓ | universal | joins |
| `RIGHT JOIN` | ✓ | ✓ | ✓ (3.39.0+) | ✓ | — | joins |
| `FULL [OUTER] JOIN` | ✓ | ✓ | ✓ (3.39.0+) | ✗ | MySQL: `LEFT … UNION … RIGHT`; SQL Server/Oracle ✓ | joins |
| `JOIN … USING (col)` | ✓ | ✓ | ✓ | ✓ | SQL Server ✗ (use `ON`) | joins |
| `NATURAL JOIN` | ✓ | ✓ (avoid) | ✓ (avoid) | ✓ (avoid) | error-prone; avoid everywhere | joins |
| `JOIN LATERAL (…)` (SQL:1999) | ✓ | ✓ | ✗ | ~ (8.0.14+) | Oracle 12c+; DuckDB ✓ | lateral-and-correlated-derived |
| `CROSS APPLY` / `OUTER APPLY` | ✗ | ✗ | ✗ | ✗ | SQL Server (no `LATERAL` kw); Oracle accepts both | lateral-and-correlated-derived |
| Null-safe join key | `IS NOT DISTINCT FROM` | ✓ | `IS`/`IS NOT` | `<=>` | Oracle no native; SQL Server 2022+ | relational-and-null-discipline |

## C. Subqueries & set operations

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `EXISTS` / `IN` / `NOT IN` / `ANY` / `ALL` | ✓ | ✓ | ✓ | ✓ | universal | subqueries-and-exists |
| `NOT IN (… NULL …)` → empty result | ✓ | ✓ | ✓ | ✓ | a correctness trap, identical everywhere | subqueries-and-exists |
| Scalar subquery returning >1 row | error | error | **silent first row** | error | SQLite hides the bug; others raise | subqueries-and-exists |
| `UNION` / `UNION ALL` | ✓ | ✓ | ✓ | ✓ | universal | set-operations |
| `INTERSECT` / `EXCEPT` | ✓ | ✓ | ✓ | ~ (MySQL 8.0.31+ / MariaDB 10.3+) | **Oracle: `EXCEPT` is `MINUS`** | set-operations |
| `INTERSECT ALL` / `EXCEPT ALL` | ✓ | ✓ | ✗ | ~ (8.0.31+) | Oracle ✗ | set-operations |
| `INTERSECT` precedence over `UNION`/`EXCEPT` | ✓ | ✓ | ✗ (left-to-right) | ✓ | parenthesize for portability | set-operations |
| `VALUES` as a table source | ✓ | ✓ | ✓ | ~ (`ROW()`/`VALUES` vary) | Oracle via `SELECT … FROM dual` | set-operations |

## D. CTE & recursion

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `WITH` / `WITH RECURSIVE` | ✓ | ✓ | ✓ | ✓ (MySQL 8+/MariaDB) | universal on modern engines | cte-and-recursion |
| `SEARCH` / `CYCLE` clauses | ✓ | ✓ (14+) | ✗ | ✗ | fall back to manual path-array guard | cte-and-recursion |
| `[NOT] MATERIALIZED` fence | ✗ | ✓ (12+) | ✗ | ✗ | PG optimizer hint | cte-and-recursion |
| `generate_series(…)` | ✗ | ✓ | ✗ (use counter CTE) | ✗ (use counter CTE) | DuckDB ✓; SQL Server `GENERATE_SERIES` 2022+ | cte-and-recursion |

## E. Windowing & pattern matching

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| Window functions (`OVER`, `LAG`/`LEAD`, ranking) | ✓ (SQL:2003) | ✓ (8.4+) | ✓ (3.25+) | ~ (MySQL **8.0+ only**; MariaDB 10.2+) | broadly available now | window-functions |
| Default frame `RANGE … CURRENT ROW` | ✓ | ✓ | ✓ | ✓ | a trap, identical everywhere | window-functions |
| `GROUPS` frame mode + `EXCLUDE` | ✓ | ✓ (11+) | ✓ (3.28+) | ✗ | MariaDB ✗ | window-functions |
| `QUALIFY` (filter window inline) | ✗ | ✗ | ✗ | ✗ | Snowflake/DuckDB/Teradata ✓; else wrap in subquery | window-functions / gaps-and-islands |
| `MATCH_RECOGNIZE` (SQL:2016 R010, optional) | ✓ | ~ (18+) | ~ (3.53+) | ~ (9.7+) | Oracle 12cR1+/Trino/Snowflake/DB2 established; SQL Server 2025+ | match-recognize |

## F. Aggregation

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| String aggregation | `LISTAGG … WITHIN GROUP` | `string_agg` | `group_concat` | `GROUP_CONCAT(… SEPARATOR …)` | Oracle/DB2/SQL Server 2017+ `LISTAGG`; PG/DuckDB also `array_agg`/`list` | aggregation-and-grouping |
| `FILTER (WHERE …)` on aggregate | ✓ | ✓ | ✓ | ~ (recent / else `CASE`) | Oracle/SQL Server recent / `CASE` | aggregation-and-grouping |
| `GROUP BY ROLLUP` / `CUBE` / `GROUPING SETS` | ✓ | ✓ | ✗ (use `UNION ALL`) | ~ (`WITH ROLLUP` only) | Oracle/SQL Server ✓ | aggregation-and-grouping |
| `GROUPING()` function | ✓ | ✓ | ✗ | ✓ | — | aggregation-and-grouping |
| Ordered-set `percentile_cont` / `mode` | ✓ | ✓ | ✗ | ✗ (8.0+ window) | Oracle/SQL Server ✓ | aggregation-and-grouping |
| `COUNT(*)` vs `COUNT(col)` (skip NULL) | ✓ | ✓ | ✓ | ✓ | semantics identical everywhere | relational-and-null-discipline |

## G. DML & upsert

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `MERGE` | ✓ | ~ (15+) | ✗ | ✗ | SQL Server / Oracle / DB2 ✓ | merge-and-upsert |
| `INSERT … ON CONFLICT DO UPDATE` | ✗ | ✓ (9.5+) | ✓ (3.24+) | ✗ | new-value alias `excluded.col` | merge-and-upsert |
| `INSERT … ON DUPLICATE KEY UPDATE` | ✗ | ✗ | ✗ | ✓ | new-value alias `new.col` (8.0.19+) / `VALUES(col)` | merge-and-upsert |
| `INSERT … RETURNING` | ✗ (vendor; SQL:2023 adds it) | ✓ | ✓ (3.35+) | ✗ | SQL Server `OUTPUT`; Oracle `RETURNING INTO` | merge-and-upsert |
| Parameter placeholder | n/a (driver `paramstyle`) | `$1` | `?` / `:name` | `?` / `%s` | **a driver property, not a dialect feature** | injection-and-parameterization |

## H. DDL, keys & constraints

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `GENERATED … AS IDENTITY` | ✓ | ✓ (10+) | ✗ | ✗ | SQL Server `IDENTITY(1,1)`; Oracle 12c+ | generated-and-identity-columns |
| Vendor auto-increment | — | `SERIAL` (legacy) | `INTEGER PRIMARY KEY` | `AUTO_INCREMENT` | SQL Server `IDENTITY`; Oracle seq+trigger pre-12c | generated-and-identity-columns |
| Computed `GENERATED ALWAYS AS (expr)` | ✓ | ✓ (STORED only) | ✓ (VIRTUAL default) | ✓ (VIRTUAL default) | SQL Server computed columns | generated-and-identity-columns |
| FK enforced by default | ✓ | ✓ | ✗ (`PRAGMA foreign_keys=ON` per connection) | ✓ (InnoDB) | Oracle ✓ | constraints-and-integrity |
| `PRIMARY KEY` implies `NOT NULL` | ✓ | ✓ | ✗ (unless `INTEGER PK`/declared) | ✓ | Oracle ✓ | constraints-and-integrity |
| Default `UNIQUE` allows multiple NULLs | ✓ | ✓ | ✓ | ✓ | universal | constraints-and-integrity |
| `UNIQUE NULLS NOT DISTINCT` (SQL:2023 F292) | ✓ | ✓ (15+) | ✗ | ✗ | Oracle ✗ | constraints-and-integrity |
| `CHECK` enforced | ✓ | ✓ | ✓ | ✓ (8.0.16+) | Oracle ✓ | constraints-and-integrity |
| Referential actions | all 5 | all 5 | all 5 (when FKs on) | no `SET DEFAULT` | Oracle: NO ACTION/CASCADE/SET NULL only | constraints-and-integrity |
| `DEFERRABLE` constraints | ✓ | ✓ | ~ (FK only, via PRAGMA) | ✗ | Oracle ✓ | constraints-and-integrity |
| `CREATE INDEX … ON t (cols)` | ✗ (not in core standard) | ✓ | ✓ | ✓ | near-universal de-facto; spelled the same | indexing-and-sargability |
| Covering "payload" columns | — | `INCLUDE (…)` (11+) | append to key | append to key | SQL Server `INCLUDE`; vendor index types not portable | indexing-and-sargability |

## I. Data types

| Concept | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| Exact decimal | `NUMERIC`/`DECIMAL` | `numeric`/`decimal` | NUMERIC affinity (may store REAL) | `decimal` | SQL Server `decimal`/`numeric` | data-types-and-numerics |
| Boolean | `BOOLEAN` | `boolean` | ✗ (integer 0/1) | `tinyint(1)` (`BOOLEAN` alias) | SQL Server `bit`; Oracle ✗ pre-23ai; DuckDB ✓ | data-types-and-numerics |
| Binary LOB | `BLOB`/`BINARY LARGE OBJECT` | `bytea` | `BLOB` | `blob` | SQL Server `varbinary(max)` | data-types-and-numerics |
| Declared type enforced | ✓ | ✓ | ✗ (advisory affinity) | ✓ | SQL Server ✓ | data-types-and-numerics |
| String concat | `\|\|` | `\|\|` | `\|\|` / `CONCAT` (3.44+) | `CONCAT()` (no `\|\|` default) | SQL Server `+`; Oracle `\|\|` | expressions-case-and-functions |
| Null default | `COALESCE` | `COALESCE` | `COALESCE`/`IFNULL` | `COALESCE`/`IFNULL` | Oracle `NVL`; SQL Server `ISNULL` | expressions-case-and-functions |
| Conditional | `CASE` | `CASE` | `CASE` | `CASE`/`IF()` | SQL Server `IIF`; Oracle `DECODE` | expressions-case-and-functions |
| Cast | `CAST(x AS t)` | `CAST` / `::` | `CAST` | `CAST` | SQL Server `CAST`/`CONVERT`; `::` is PG-only | expressions-case-and-functions |
| Substring | `SUBSTRING(s FROM n FOR m)` | both forms | `SUBSTR(s,n,m)` | `SUBSTRING`/`SUBSTR` | Oracle `SUBSTR` | expressions-case-and-functions |
| `'' ` (empty string) = NULL | no | no | no | no | **Oracle: yes (deviation)** | relational-and-null-discipline |

## J. Datetime

| Concept | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| Native temporal types | `DATE`/`TIME`/`TIMESTAMP [WITH TIME ZONE]`/`INTERVAL` | rich | ✗ (TEXT/REAL/INTEGER) | `DATE`/`TIME`/`DATETIME`/`TIMESTAMP` | DuckDB rich | datetime-and-intervals |
| Current timestamp | `CURRENT_TIMESTAMP` | ✓ / `now()` | ✓ / `datetime('now')` | ✓ / `NOW()`/`SYSDATE()` | SQL Server `GETDATE()`; Oracle `SYSDATE`/`SYSTIMESTAMP` | datetime-and-intervals |
| Stored `INTERVAL` type | ✓ | ✓ | ✗ | ✗ (expr only) | Oracle ✓ | datetime-and-intervals |
| Date arithmetic | `ts + INTERVAL '1 day'` | `+ INTERVAL` | `date(ts,'+1 day')`/`strftime` | `DATE_ADD(…, INTERVAL …)` | SQL Server `DATEADD`/`DATEDIFF`; Oracle `ts + 1` | datetime-and-intervals |
| Truncate / bucket | `EXTRACT` + `date_trunc` (vendor) | `date_trunc` | `strftime` | `DATE_FORMAT`/`DATE()` | SQL Server `DATETRUNC` 2022+ | datetime-and-intervals |
| UTC-instant type | `TIMESTAMP WITH TIME ZONE` | `timestamptz` | n/a (you encode) | `TIMESTAMP` (UTC-converted, ~2038 cap) | — | datetime-and-intervals |

## K. JSON

| Feature | Standard (SQL:2016) | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `JSON_VALUE` / `JSON_QUERY` / `JSON_EXISTS` | ✓ | ~ (17+) | ✗ (`json_extract`) | ~ (`JSON_VALUE` 8.0.21+; no `JSON_QUERY`) | Oracle ✓ | json |
| `JSON_TABLE` | ✓ | ~ (17+) | ✗ (`json_each`/`json_tree`) | ~ (8.0.4+) | Oracle ✓ | json |
| `JSON_OBJECT` / `JSON_ARRAY` / `*AGG` | ✓ | ✓ | ~ (`json_object`/`json_array`; AGG newer) | ✓ | Oracle ✓ | json |
| `IS JSON` predicate | ✓ | ~ (16+) | `json_valid()` | partial | Oracle ✓ | json |
| `->` / `->>` operators | ✗ (vendor) | ✓ | ✓ (3.38.0+) | ✓ (8.x) | **Oracle no native operator** | json |
| Native JSON type | — | `jsonb` | text + internal blob | `JSON` | DuckDB `JSON` | json |

## L. Catalog / introspection & views

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `INFORMATION_SCHEMA.*` views | ✓ | ✓ | ✗ | ✓ | SQL Server ✓; **Oracle ✗** (`ALL_*`/`USER_*`); DuckDB ✓ | views-and-introspection |
| List-tables fallback | — | `pg_catalog`/`\dt` | `sqlite_schema`/`sqlite_master` + `PRAGMA table_info` | `SHOW TABLES`/`SHOW COLUMNS` | Oracle `ALL_TABLES` | views-and-introspection |
| Auto-updatable simple views | ✓ | ✓ | ~ (via trigger) | ✓ | Oracle ✓ | views-and-introspection |
| `WITH [LOCAL\|CASCADED] CHECK OPTION` | ✓ | ✓ | ✗ | ✓ | SQL Server ✓ (no LOCAL/CASCADED kw) | views-and-introspection |
| Materialized views | ✗ (vendor) | ✓ (`REFRESH`) | ✗ | ✗ (MariaDB ✗) | Oracle ✓; SQL Server indexed views | views-and-introspection |

## M. Identifiers, transactions, privileges

| Feature | Standard | PostgreSQL | SQLite | MySQL/MariaDB | Notes | Owner |
|---|---|---|---|---|---|---|
| `"name"` = identifier | ✓ | ✓ | ✓ | ✗ (string literal unless `ANSI_QUOTES`) | SQL Server `[name]`/`"name"` | style-and-naming |
| Native identifier quote | `"…"` | `"…"` | `"…"`/`` `…` ``/`[…]` | `` `…` `` | SQL Server `[…]` | style-and-naming |
| Unquoted name case-fold | upper | lower | preserve, case-insensitive | filesystem-dependent | Oracle upper | style-and-naming |
| Isolation level *names* | ✓ | ✓ | ✓ | ✓ | standard everywhere | transactions-and-isolation |
| **Default** isolation level | (impl-defined) | `READ COMMITTED` | `SERIALIZABLE` | `REPEATABLE READ` | SQL Server/Oracle `READ COMMITTED` | transactions-and-isolation |
| DDL rolls back in a transaction | (impl-defined) | ✓ | ✓ | ✗ (implicit commit) | Oracle implicit commit | transactions-and-isolation |
| `GRANT`/`REVOKE` + roles | ✓ (SQL-92 / SQL:1999) | ✓ | ✗ (no GRANT — file perms) | ✓ (roles 8.0+; `user@host`) | Oracle/SQL Server own privilege names | privileges-and-access-control |
| `ALL PRIVILEGES` (with `PRIVILEGES` kw) | ✓ | ✓ | n/a | ✓ | required by strict SQL | privileges-and-access-control |

## N. Advanced / low-portability (confirm before recommending)

| Feature | Standard | Where it exists | Fallback | Owner |
|---|---|---|---|---|
| System-versioned temporal tables (SQL:2011) | ✓ optional | MariaDB 10.3+, SQL Server 2016+, DB2, SAP HANA; Oracle ~ (Flashback) | hand-modeled history table + triggers (**PG/MySQL/SQLite have none**) | temporal-tables |
| Application-time periods / `FOR PORTION OF` | ✓ optional | thinner than system-versioning even where present | manual valid-time modeling | temporal-tables |
| `MATCH_RECOGNIZE` (SQL:2016 R010) | ✓ optional | Oracle 12cR1+/Trino/Snowflake/DB2/Vertica; PG 18+, MySQL 9.7+, MariaDB 12.3+, SQLite 3.53+, SQL Server 2025+ | window functions (gaps-and-islands) | match-recognize |
| SQL/PGQ property-graph queries (SQL:2023) | ✓ optional | **Oracle 23ai GA only**; PG experimental out-of-tree | recursive CTE | property-graph-queries |

---

## Cross-cutting cautions

- **"Standard" ≠ "portable."** `LIMIT` is non-standard yet near-universal; `FETCH FIRST` is standard yet missing on MySQL/SQLite. Always check the *cell*, not the standard.
- **SQLite is the most divergent reader engine** on schema/typing: advisory type affinity, FK off by default, PK allows NULL, no `INFORMATION_SCHEMA`, no `LATERAL`/`FULL JOIN`, no temporal types.
- **Oracle is the most divergent on small spellings**: `MINUS` for `EXCEPT`, `NVL`/`SYSDATE`, `''`=NULL, no `INFORMATION_SCHEMA`, no `->`/`->>`.
- **MySQL's defaults bite quietly**: `"x"` as a string, `REPEATABLE READ` default, DDL implicit commit, `||` as OR, no `FULL JOIN`.
- **Version gates matter as much as engine identity**: window functions need MySQL 8.0; `MERGE` needs PostgreSQL 15; `INTERSECT` needs MySQL 8.0.31; `MATCH_RECOGNIZE` is absent from nearly every deployed version.

When more than one engine is in play, write the **standard intersection**; reach for a vendor spelling only when you have named the single target engine.
