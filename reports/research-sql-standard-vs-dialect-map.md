# Research note — sql-standard-vs-dialect-map (skill #27)

Date: 2026-06-26. This anchor skill is an *index*, not a tutorial, so its primary
research source is the 29 already-drafted sibling skills: every one of them ends in a
"Portability" section that states settled per-engine claims with primary-source
citations. The map must AGREE with those claims (no contradictions). Below: what I
harvested and the confirmations.

## Harvest method

Single Grep over `plugins/sql/skills/*/SKILL.md` for `Portability` with `-A 12`,
then spot-read of every high-divergence block. The grep returned the full portability
table for nearly every skill verbatim, so the map rows are copied from settled text,
not re-derived.

## Skills harvested (portability claims pulled verbatim)

- **sql-pagination-and-keyset** — `FETCH FIRST n ROWS ONLY` standard; PG yes, Oracle 12c+, SQL Server 2012+, MySQL **no**, SQLite **no**. `LIMIT` non-standard but in PG/MySQL/SQLite. `WITH TIES` PG/Oracle/SQL Server, not MySQL/SQLite. Row-value `(a,b)<(x,y)`: PG/Oracle yes, MySQL 8.0+, SQL Server/SQLite no.
- **sql-generated-and-identity-columns** — `GENERATED AS IDENTITY` (T174/T178) PG yes, SQLite no (rowid `INTEGER PRIMARY KEY`), MySQL no (`AUTO_INCREMENT`). Computed `GENERATED ALWAYS AS (expr)`: all three; VIRTUAL default in SQLite/MySQL, PG STORED-only.
- **sql-merge-and-upsert** — `MERGE` PG **15+**, SQLite no, MySQL no, SQL Server/Oracle/DB2 yes. `ON CONFLICT` PG 9.5+/SQLite 3.24+. `ON DUPLICATE KEY UPDATE` MySQL only. New-value alias: `excluded.col` (PG/SQLite), `new.col` (MySQL 8.0.19+).
- **sql-aggregation-and-grouping** — string agg spelling: PG `string_agg`, SQLite `group_concat`, MySQL `GROUP_CONCAT`, Oracle/SQL Server `LISTAGG`. `FILTER (WHERE)` PG/SQLite yes, MySQL/Oracle recent-only. ROLLUP/CUBE/GROUPING SETS PG/Oracle yes, SQLite no, MySQL `WITH ROLLUP` only.
- **sql-relational-and-null-discipline** — `IS NOT DISTINCT FROM` PG yes, SQLite `IS` (no spelling), MySQL `<=>`, Oracle no native. Oracle `''`=NULL deviation. `NULLS FIRST/LAST` PG/SQLite/Oracle yes, MySQL needs `col IS NULL` trick.
- **sql-views-and-introspection** — `INFORMATION_SCHEMA` PG/MySQL/SQL Server yes, SQLite **no** (`sqlite_schema`/`sqlite_master`+`PRAGMA`), Oracle **no** (data-dictionary `ALL_*`). MySQL `SHOW TABLES`.
- **sql-json** — `JSON_VALUE`/`JSON_QUERY`/`JSON_TABLE` PG 17+, MySQL partial (JSON_VALUE 8.0.21+, JSON_TABLE 8.0.4+), SQLite no (`json_extract`/`json_each`), Oracle yes. `->`/`->>` PG/MySQL/SQLite(3.38+) yes, Oracle no native.
- **sql-style-and-naming** — `"abc"` is identifier in standard/PG, **string literal** in MySQL default, identifier under `ANSI_QUOTES`. Native quote: PG `"..."`, MySQL backtick. Case-fold: PG lower, standard upper.
- **sql-lateral-and-correlated-derived** — `LATERAL` (SQL:1999) PG yes, MySQL 8.0.14+, Oracle 12c+ (also APPLY), SQL Server `CROSS/OUTER APPLY` (no LATERAL kw), SQLite **no**.
- **sql-set-operations** — `INTERSECT`/`EXCEPT` PG/SQLite yes, MySQL **8.0.31+**, Oracle spells `EXCEPT` as **`MINUS`**. `INTERSECT ALL`/`EXCEPT ALL` PG yes, SQLite **no**, MySQL 8.0.31+, Oracle no. INTERSECT precedence: PG/MySQL/Oracle bind tighter; SQLite **all left-to-right**.
- **sql-transactions-and-isolation** — default level: PG `READ COMMITTED`, MySQL/InnoDB `REPEATABLE READ`, SQLite `SERIALIZABLE`. MySQL DDL implicit-commit (cannot roll back).
- **sql-data-types-and-numerics** — Boolean: PG `boolean`, SQLite integer 0/1, MySQL `tinyint(1)`, SQL Server `bit`. SQLite advisory type affinity (not enforced). Exact decimal `numeric`/`decimal` shared.
- **sql-datetime-and-intervals** — standard `CURRENT_*`/`EXTRACT`/`INTERVAL`; SQLite has no temporal type (TEXT/REAL/INTEGER + `strftime`); MySQL `TIMESTAMP` UTC-converted vs `DATETIME` naive; PG `date_trunc`, MySQL `DATE_FORMAT`.
- **sql-expressions-case-and-functions** — `COALESCE` everywhere; `IFNULL` (MySQL/SQLite), `NVL` (Oracle), `ISNULL` (SQL Server). Concat `||` standard/PG/Oracle, MySQL `CONCAT()`, SQL Server `+`. `SUBSTRING` vs `SUBSTR`. `CAST` vs `::`.
- **sql-joins** — `FULL JOIN`/`RIGHT JOIN` PG yes, SQLite 3.39.0+, MySQL **no FULL JOIN** (emulate `LEFT … UNION … RIGHT`).
- **sql-cte-and-recursion** — `WITH RECURSIVE` portable; `SEARCH`/`CYCLE` PG 14+ only; `generate_series` PG-only.
- **sql-window-functions / sql-gaps-and-islands** — window fns SQL:2003: PG 8.4+, SQLite 3.25+, MySQL **8.0+ only**, MariaDB 10.2+. `QUALIFY` nowhere (wrap in subquery). `GROUPS`/`EXCLUDE` PG 11+/SQLite 3.28+, not MySQL/MariaDB.
- **sql-match-recognize** — optional SQL:2016 R010: established in Oracle 12cR1+/Trino/Snowflake/DB2; PG **18+**, MySQL **9.7+**, MariaDB 12.3+, SQLite 3.53+, SQL Server 2025+ (i.e. absent from almost all deployed versions).
- **sql-temporal-tables** — SQL:2011 system-versioning: MariaDB 10.3+, SQL Server 2016+, DB2/SAP HANA yes; Oracle ~ (Flashback); PG/MySQL/SQLite **none**.
- **sql-property-graph-queries** — SQL/PGQ (SQL:2023): Oracle 23ai GA only; everyone else recursive-CTE fallback.
- **sql-indexing-and-sargability** — `CREATE INDEX` not in core standard but near-universal; `INCLUDE` covering PG 11+/SQL Server; GIN/GiST/BRIN/bitmap vendor-specific.
- **sql-injection-and-parameterization** — placeholder punctuation (`?`/`$1`/`:name`/`%s`) is a driver `paramstyle`, NOT a dialect feature.
- **sql-subqueries-and-exists** — `NOT IN`+NULL→empty everywhere; scalar subquery >1 row: PG/MySQL/Oracle error, **SQLite silent first row**.
- **sql-select-and-query-processing** — alias usable only in ORDER BY portably; `DISTINCT ON` PG-only.
- **sql-constraints-and-integrity** — SQLite FK off by default (`PRAGMA foreign_keys=ON`); SQLite PK allows NULL; `UNIQUE NULLS NOT DISTINCT` PG 15+.

## modern-sql confirmation

WebFetch of https://modern-sql.com/standard (2026-06-26) confirmed the page indexes
standard versions (SQL:2023/2019/2016/2011/…) and per-feature technical reports
(JSON, temporal) plus browsable BNF, but it does not itself carry quotable
prose about implementation divergence — the citable material lives on the
per-feature pages (e.g. `/feature/match_recognize`, `/caniuse/generated-always-as`,
`/concept/null`) that the sibling skills already cite. The map therefore reuses those
already-verified per-feature citations rather than inventing new ones.

## Decision

All matrix rows are transcribed from the settled sibling-skill claims above. SKILL.md
keeps the ~13 most-used features as compact tables; the exhaustive feature×engine
matrix lives in `references/dialect-matrix.md`. No claim contradicts a sibling skill.
