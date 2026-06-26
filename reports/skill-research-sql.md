# Domain Research: Standard SQL Skills (ISO/IEC 9075, SQL:2011 / 2016 / 2023)

Phase-1 domain map for an AI-agent skill set (SKILL.md files) covering **modern, portable, standard SQL** — the ISO/IEC 9075 surface and the idioms that work across compliant engines. Vendor-specific behavior (Postgres-only, MySQL-only, etc.) is deferred to separate plugins; this plugin teaches *the standard and the portable idiom*, and names divergence only to keep the agent safe.

Target: a comprehensive set, **more comprehensive than the 25-skill golang plugin** (~22–26 SQL skills). Each candidate records the six required fields: (1) skill name, (2) one-line scope, (3) why skill-worthy + what LLMs get wrong, (4) primary-source deep links, (5) portability note, (6) which of the four coverage dimensions it serves.

The four coverage dimensions referenced throughout:
- **TC** = technical correctness (the SQL is *right* — correct results, correct NULL semantics, no silent data loss)
- **HX** = human experience (readability, maintainability, reviewability, predictable behavior for the next engineer)
- **TR** = testing reality (how you *prove* the query is correct; reproducibility; edge-case coverage)
- **PI** = process integration (how SQL lives in an app/codebase: parameterization, migrations, introspection, CI, tooling)

A note on method: the ISO standard text is paywalled, so every factual claim below is anchored to a **public** primary source — the Postgres/SQLite manuals (which implement and document the features), Markus Winand's modern-sql.com and use-the-index-luke.com (vendor-neutral standards tracking), and the public SQL:2023 feature summaries by Peter Eisentraut and Markus Winand. These are the pages a per-skill researcher should `WebFetch`.

---

## 1. Landscape — what "standard SQL" means right now

SQL is **ISO/IEC 9075**, a multi-part standard. The relevant editions:

### SQL:1992 / SQL:1999 / SQL:2003 (the bedrock)
- **SQL-92** is still the de-facto "lowest common denominator" most engines fully implement: core SELECT/JOIN/subquery/set-ops, `CASE`, `CAST`, the information schema, the four isolation levels, `GRANT`/`REVOKE`.
- **SQL:1999** added recursive queries (`WITH RECURSIVE`), `GROUPING SETS`/`ROLLUP`/`CUBE`, boolean type, common table expressions, triggers, arrays.
- **SQL:2003** added **window functions** (`OVER`), the `MERGE` statement, `GENERATED`/identity columns, sequence generators, `XML` (SQL/XML). Window functions and MERGE are the two that matter most for a modern agent.
  - Reference summary: https://en.wikipedia.org/wiki/SQL:2003 and https://modern-sql.com/standard

### SQL:2011 (ISO/IEC 9075:2011) — temporal
- Main addition: **temporal databases**. Two orthogonal mechanisms:
  - **System-versioned tables**: `PERIOD FOR SYSTEM_TIME (start, end)` + `WITH SYSTEM VERSIONING`; query history with `FOR SYSTEM_TIME AS OF`, `FROM ... TO ...`, `BETWEEN ... AND ...`. The system maintains row history automatically.
  - **Application-time period tables** (a.k.a. valid-time): `PERIOD FOR <name> (start, end)`; `UPDATE/DELETE ... FOR PORTION OF` with automatic period splitting.
  - Combining both yields **bitemporal** tables.
- Also: enhanced window functions (`NTILE`, `LEAD`/`LAG`, `NTH_VALUE`, frame `RANGE` improvements), `OFFSET ... FETCH`.
- Sources: https://en.wikipedia.org/wiki/SQL:2011 ; https://modern-sql.com/standard/2011

### SQL:2016 (ISO/IEC 9075:2016) — JSON + row pattern recognition
Per Markus Winand's "What's new in SQL:2016" (https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016), the headline additions:
- **SQL/JSON**: constructors `JSON_OBJECT`, `JSON_ARRAY`, `JSON_OBJECTAGG`, `JSON_ARRAYAGG`; query functions `JSON_EXISTS`, `JSON_VALUE`, `JSON_QUERY`; the table function `JSON_TABLE`; the **SQL/JSON path language** (with `lax`/`strict` modes and filter expressions); the `IS [NOT] JSON` predicate. (Note: SQL:2016 modeled JSON over *character strings*; SQL:2023 adds a real `JSON` type.)
- **Row Pattern Recognition** — `MATCH_RECOGNIZE`: regex-style pattern matching across *rows* (time-series, sessionization, "find V-shapes"). Optional features R010/R020/R030.
- **`LISTAGG`** — ordered-set aggregate that concatenates group values into a delimited string, with `ON OVERFLOW` handling.
- **Polymorphic table functions (PTF)**; `DECFLOAT`; trig/log functions; named & defaulted routine arguments; `JOIN ... USING` with correlation names; `CAST ... FORMAT` for date/time templating.
- Part 2 grew ~18% and added 44 new optional features.

### SQL:2023 (ISO/IEC 9075:2023) — JSON type + property graphs
Per Peter Eisentraut's "SQL:2023 is finished" (https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new) and https://modern-sql.com/standard/2023, three buckets:
- **Smaller language changes**: `UNIQUE NULLS [NOT] DISTINCT` (F292 — control how NULLs interact with UNIQUE constraints); `ANY_VALUE` aggregate (T626); `GREATEST`/`LEAST` (T054); `LPAD`/`RPAD` (T055) and multi-char `LTRIM`/`RTRIM`/`BTRIM` (T056); optional `VARCHAR` length (T081); boolean cycle-mark values in recursive CTEs (T133); non-decimal integer literals `0x`/`0o`/`0b` (T661) and underscores in numeric literals (T662); `ORDER BY` of grouped tables by non-selected columns (F868).
- **JSON**: a real **`JSON` data type** (T801) with `JSON_SERIALIZE`, `JSON_SCALAR`, plus enhanced JSON type and comparison semantics; **simplified accessor** dot/subscript syntax (T860–T864); **JSON path item methods** (`.bigint()`, `.boolean()`, `.date()`, `.decimal()`, `.number()`, `.string()`, `.time()`, `.timestamp()`, T865–T878).
- **Part 16, SQL/PGQ — Property Graph Queries**: `CREATE PROPERTY GRAPH` over existing tables, queried with the `GRAPH_TABLE (... MATCH <pattern> COLUMNS (...))` operator using ASCII-art vertex/edge patterns. This is the same pattern language that seeds the standalone **GQL** (ISO/IEC 39075:2024) graph query language. Sources: https://modern-sql.com/standard/2023 ; https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq ; https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard

**Implementation reality (portability caveat that pervades this plugin):** *no* engine implements all of standard SQL, and the most advanced features (MATCH_RECOGNIZE, SQL/PGQ, system-versioned tables, JSON_TABLE) are unevenly implemented. The agent must distinguish "in the standard" from "available here." Oracle/DB2/SQL Server lead on the exotic features; Postgres/SQLite/MySQL/MariaDB/DuckDB are the readable engines and lag on some. modern-sql.com's per-feature compatibility tables are the canonical portability reference.

---

## 2. Primary sources & readable databases — master list (cite these repeatedly)

### Standards tracking (vendor-neutral, by experts)
- **modern-sql.com** (Markus Winand) — the single best vendor-neutral standard reference, with per-feature compatibility matrices:
  - Three-valued logic / NULL: https://modern-sql.com/concept/three-valued-logic
  - `NULL` handling & `IS DISTINCT FROM`: https://modern-sql.com/concept/null
  - `FILTER`: https://modern-sql.com/feature/filter
  - `OVER` / window functions: https://modern-sql.com/feature/over
  - `LATERAL`: https://modern-sql.com/feature/lateral
  - `WITH` / CTE: https://modern-sql.com/feature/with
  - `FETCH FIRST` / `OFFSET`: https://modern-sql.com/feature/fetch-first
  - `MATCH_RECOGNIZE`: https://modern-sql.com/feature/match_recognize
  - `JSON_TABLE`: https://modern-sql.com/feature/json_table
  - Standard catalog by revision: https://modern-sql.com/standard , /standard/2011 , /standard/2016 , /standard/2023
- **use-the-index-luke.com** (Markus Winand, "SQL Performance Explained") — vendor-neutral indexing/perf:
  - ToC: https://use-the-index-luke.com/sql/table-of-contents
  - Anatomy of an index: https://use-the-index-luke.com/sql/anatomy
  - The WHERE clause (sargability, column order): https://use-the-index-luke.com/sql/where-clause
  - The join operation: https://use-the-index-luke.com/sql/join
  - Clustering / covering indexes: https://use-the-index-luke.com/sql/clustering
  - Sorting & grouping (pipelined order by): https://use-the-index-luke.com/sql/sorting-grouping
  - Partial results / pagination: https://use-the-index-luke.com/sql/partial-results
  - **No-offset / keyset pagination**: https://use-the-index-luke.com/no-offset
  - DML index cost: https://use-the-index-luke.com/sql/dml
- **Feature summaries**: Peter Eisentraut SQL:2023 — https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new and https://peter.eisentraut.org/blog/2023/04/18/postgresql-and-sql-2023 ; Wikipedia revision pages https://en.wikipedia.org/wiki/SQL:2023 , /wiki/SQL:2016 , /wiki/SQL:2011

### Readable engine manuals (implement + document the standard)
- **PostgreSQL** (closest large open-source engine to the standard; explicitly notes standard conformance):
  - SELECT (DISTINCT ON, LATERAL, FETCH, FOR UPDATE): https://www.postgresql.org/docs/current/sql-select.html
  - Queries / table expressions (joins, GROUPING SETS): https://www.postgresql.org/docs/current/queries-table-expressions.html
  - WITH / recursive CTE: https://www.postgresql.org/docs/current/queries-with.html
  - Window function tutorial: https://www.postgresql.org/docs/current/tutorial-window.html ; window syntax: https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS ; window functions list: https://www.postgresql.org/docs/current/functions-window.html
  - Aggregate functions / FILTER / ordered-set: https://www.postgresql.org/docs/current/functions-aggregate.html
  - JSON functions & path: https://www.postgresql.org/docs/current/functions-json.html
  - MERGE: https://www.postgresql.org/docs/current/sql-merge.html
  - Data types: https://www.postgresql.org/docs/current/datatype.html
  - Constraints: https://www.postgresql.org/docs/current/ddl-constraints.html ; generated columns: /ddl-generated-columns.html ; identity: /ddl-identity-columns.html
  - CREATE VIEW (WITH CHECK OPTION): https://www.postgresql.org/docs/current/sql-createview.html
  - information_schema: https://www.postgresql.org/docs/current/information-schema.html
  - Transaction isolation: https://www.postgresql.org/docs/current/transaction-iso.html
  - GRANT / privileges: https://www.postgresql.org/docs/current/sql-grant.html ; ddl-priv: /ddl-priv.html
  - EXPLAIN / using EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
  - PREPARE (parameterized): https://www.postgresql.org/docs/current/sql-prepare.html
- **SQLite** (small, precise, very readable; documents its deviations explicitly):
  - SELECT: https://www.sqlite.org/lang_select.html ; WITH: https://www.sqlite.org/lang_with.html
  - Window functions: https://www.sqlite.org/windowfunctions.html
  - UPSERT (its non-MERGE upsert): https://www.sqlite.org/lang_upsert.html
  - NULL handling quirks: https://www.sqlite.org/nulls.html ; flexible typing: https://www.sqlite.org/datatype3.html ; "quirks": https://www.sqlite.org/quirks.html
  - Query planner / optimizer overview: https://www.sqlite.org/queryplanner.html ; https://www.sqlite.org/optoverview.html ; EXPLAIN QUERY PLAN: https://www.sqlite.org/eqp.html
  - JSON: https://www.sqlite.org/json1.html
- **MySQL / MariaDB** (widely deployed; many standard deviations to flag):
  - MySQL window functions: https://dev.mysql.com/doc/refman/8.4/en/window-functions.html ; CTE: https://dev.mysql.com/doc/refman/8.4/en/with.html ; JSON: https://dev.mysql.com/doc/refman/8.4/en/json.html
  - MariaDB system-versioned tables: https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables
- **DuckDB** (analytics; strong modern-SQL ergonomics, good for "nice-to-have" standard sugar):
  - Friendly SQL / features: https://duckdb.org/docs/sql/dialect/friendly_sql ; window functions: https://duckdb.org/docs/sql/functions/window_functions

### Books & papers (conceptual grounding)
- **E. F. Codd (1970), "A Relational Model of Data for Large Shared Data Banks"** — the relational model itself: https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf
- **Bill Karwin, "SQL Antipatterns" (and "Volume 1")** — the canonical anti-patterns catalog (EAV/"Entity-Attribute-Value", "Naive Trees"/adjacency-only, "Implicit Columns"/`SELECT *`, "Fear of the Unknown"/NULL misuse, "Index Shotgun", "Ambiguous Groups", "Random Selection", "Spaghetti Query", "Jaywalking"/comma-separated lists): https://pragprog.com/titles/bksqla/sql-antipatterns/ and the newer https://pragprog.com/titles/bksqla1/sql-antipatterns-volume-1/
- **Joe Celko, "SQL for Smarties"** — advanced idioms (gaps-and-islands, nested-set trees, auxiliary number/calendar tables, set-based thinking). Topical reference.
- **C. J. Date, "SQL and Relational Theory" / "Database in Depth"** — rigorous treatment of why NULL and duplicate rows are problematic; relational discipline.
- **Row Pattern Recognition paper**: "Specification of Row Pattern Recognition in the SQL Standard and its Implementations" (Datenbank-Spektrum): https://link.springer.com/article/10.1007/s13222-022-00404-3
- **SQL/PGQ vs GQL**: "GQL and SQL/PGQ: Theoretical Models and Expressive Power": https://arxiv.org/pdf/2409.01102

---

## 3. Candidate skills (each with the six required fields)

Ordered roughly foundation → core query → modern/advanced → DDL → transactions/security → performance/style. Skill IDs are kebab-case and `sql-` prefixed.

---

### sql-relational-and-null-discipline  ⟵ FOUNDATION / POLICY ROOT
1. **Name**: `sql-relational-and-null-discipline`
2. **Scope**: The relational model + three-valued logic as the non-negotiable mental model: rows are unordered sets, NULL means "unknown" and propagates UNKNOWN through every comparison, and `WHERE`/`HAVING`/`ON`/`CHECK` keep only TRUE rows.
3. **Skill-worthy / LLM failures**: This is the `go-idiomatic-discipline` analogue — the root every other skill leans on. LLMs constantly write `x = NULL` (always UNKNOWN), `x <> 'a'` and expect NULLs back (they're filtered out), `WHERE col NOT IN (subquery)` that silently returns zero rows when the subquery yields one NULL, and `COUNT(col)` vs `COUNT(*)` confusion. They forget `NULL = NULL` is UNKNOWN, that `UNIQUE` lets multiple NULLs in by default, and that `IS [NOT] DISTINCT FROM` is the null-safe equality. They also assume result-set ordering without `ORDER BY`.
4. **Primary sources**: https://modern-sql.com/concept/three-valued-logic ; https://modern-sql.com/concept/null ; https://www.sqlite.org/nulls.html ; https://www.postgresql.org/docs/current/functions-comparison.html ; Codd 1970 https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf ; Karwin "Fear of the Unknown" antipattern https://pragprog.com/titles/bksqla/sql-antipatterns/
5. **Portability**: Three-valued logic is core SQL-92, fully portable. `IS DISTINCT FROM` is standard but absent in some engines (MySQL spells it `<=>`; SQLite has `IS`/`IS NOT`). `UNIQUE NULLS DISTINCT`/`NOT DISTINCT` is SQL:2023 and only newer engines accept the explicit syntax.
6. **Dimensions**: TC (primary), HX, TR.

---

### sql-select-and-query-processing
1. **Name**: `sql-select-and-query-processing`
2. **Scope**: SELECT fundamentals and the *logical* clause-evaluation order (`FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → OFFSET/FETCH`) and why it governs what's legal where.
3. **Skill-worthy / LLM failures**: LLMs reference a `SELECT`-list alias in `WHERE` (illegal — `SELECT` runs after `WHERE`), put aggregate conditions in `WHERE` instead of `HAVING`, expect `ORDER BY` to affect grouping, or assume physical execution matches written order. They confuse `DISTINCT` (whole-row) with per-column uniqueness and misuse vendor `DISTINCT ON`.
4. **Primary sources**: https://www.postgresql.org/docs/current/sql-select.html ; https://www.postgresql.org/docs/current/queries-overview.html ; https://www.sqlite.org/lang_select.html ; https://modern-sql.com/blog (clause processing); Itzik Ben-Gan logical-processing-order writeups.
5. **Portability**: The logical processing model is standard and universal. `DISTINCT ON` is Postgres-only; `LIMIT` is ubiquitous-but-nonstandard (the standard is `OFFSET/FETCH`).
6. **Dimensions**: TC, HX.

---

### sql-joins
1. **Name**: `sql-joins`
2. **Scope**: INNER/LEFT/RIGHT/FULL/CROSS joins, `ON` vs `WHERE` (especially with outer joins), `USING`/`NATURAL`, and multi-table join composition.
3. **Skill-worthy / LLM failures**: The classic outer-join bug: putting a filter on the right table in `WHERE` instead of `ON`, which silently turns a LEFT JOIN into an INNER JOIN. LLMs also emit `NATURAL JOIN` (a maintenance landmine — joins on every same-named column, breaks when schema changes), produce accidental cross products from missing predicates, and get RIGHT/FULL join direction wrong.
4. **Primary sources**: https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN ; https://www.sqlite.org/lang_select.html#strange_join_names ; https://use-the-index-luke.com/sql/join ; modern-sql JOIN USING notes.
5. **Portability**: All join types are SQL-92 standard. `FULL OUTER JOIN` is missing in MySQL (emulate via `UNION`) and older SQLite. `NATURAL`/`USING` are standard but discouraged.
6. **Dimensions**: TC (primary), HX, TR.

---

### sql-subqueries-and-exists
1. **Name**: `sql-subqueries-and-exists`
2. **Scope**: Scalar subqueries, `IN`/`NOT IN`, `EXISTS`/`NOT EXISTS`, correlated subqueries, and semi/anti-join idioms.
3. **Skill-worthy / LLM failures**: The **`NOT IN` + NULL trap** (a single NULL in the subquery makes `NOT IN` return zero rows) — `NOT EXISTS` is the safe portable anti-join. LLMs also write scalar subqueries that can return >1 row (runtime error), confuse correlated vs uncorrelated execution, and reach for `IN (subquery)` where a join or `EXISTS` is clearer/faster.
4. **Primary sources**: https://modern-sql.com/concept/three-valued-logic (NOT IN trap section) ; https://www.postgresql.org/docs/current/functions-subquery.html ; https://www.sqlite.org/lang_expr.html#the_exists_operator
5. **Portability**: Fully standard SQL-92 / SQL:1999. Universally portable.
6. **Dimensions**: TC (primary), HX.

---

### sql-aggregation-and-grouping
1. **Name**: `sql-aggregation-and-grouping`
2. **Scope**: `GROUP BY`, `HAVING`, aggregate functions, the `FILTER (WHERE ...)` clause, ordered-set aggregates (`LISTAGG`/`ARRAY_AGG`), and `GROUPING SETS`/`ROLLUP`/`CUBE`.
3. **Skill-worthy / LLM failures**: The "ambiguous groups" antipattern — selecting non-grouped, non-aggregated columns (works by accident in lax MySQL, errors elsewhere). LLMs reinvent `FILTER` with fragile `CASE WHEN ... END` inside aggregates, don't know `FILTER` exists, hand-`UNION` multiple grouping levels instead of `GROUPING SETS`/`ROLLUP`, and misunderstand `COUNT(DISTINCT)` and NULL exclusion in aggregates.
4. **Primary sources**: https://www.postgresql.org/docs/current/functions-aggregate.html ; https://modern-sql.com/feature/filter ; https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS ; https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016 (LISTAGG)
5. **Portability**: `GROUP BY`/`HAVING` core. `FILTER` is SQL:2003 standard but missing in MySQL/SQL Server (emulate with `CASE`). `GROUPING SETS`/`ROLLUP`/`CUBE` standard but uneven. `LISTAGG` is standard; Postgres uses `STRING_AGG`, SQLite/MySQL `GROUP_CONCAT`.
6. **Dimensions**: TC, HX.

---

### sql-set-operations
1. **Name**: `sql-set-operations`
2. **Scope**: `UNION [ALL]`, `INTERSECT [ALL]`, `EXCEPT [ALL]`, their dedup/ordering semantics, column-count/type matching, and `VALUES` as a table constructor.
3. **Skill-worthy / LLM failures**: LLMs default to `UNION` (which sorts + dedups — expensive) when `UNION ALL` is correct and cheaper; they put `ORDER BY` on the wrong branch; they don't know `EXCEPT`/`INTERSECT` exist (reinventing with `NOT EXISTS`/joins); they miss that set ops treat NULLs as *not distinct* (opposite of `=`).
4. **Primary sources**: https://www.postgresql.org/docs/current/queries-union.html ; https://www.sqlite.org/lang_select.html#compound_select_statements ; https://www.postgresql.org/docs/current/sql-values.html
5. **Portability**: `UNION`/`UNION ALL` universal. `INTERSECT`/`EXCEPT` standard; MySQL added them in 8.0.31, older versions lack them (MySQL historically used `MINUS`? no — Oracle uses `MINUS` for `EXCEPT`). `VALUES` as standalone table constructor is standard but syntax varies.
6. **Dimensions**: TC, HX.

---

### sql-cte-and-recursion
1. **Name**: `sql-cte-and-recursion`
2. **Scope**: `WITH` common table expressions for readability/decomposition, and `WITH RECURSIVE` for hierarchies, graph traversal, transitive closure, and series generation; cycle detection.
3. **Skill-worthy / LLM failures**: LLMs write infinite recursive CTEs (no termination / no cycle guard), get the anchor/recursive-member `UNION ALL` structure wrong, assume CTEs are always materialized (an optimization fence in some engines, inlined in others), and don't know the SQL:2023 `CYCLE` / `SEARCH` clauses. They reach for procedural loops where a recursive CTE is the portable set-based answer.
4. **Primary sources**: https://www.postgresql.org/docs/current/queries-with.html ; https://www.sqlite.org/lang_with.html ; https://modern-sql.com/feature/with ; https://modern-sql.com/feature/with-recursive (cycle/search) ; Eisentraut SQL:2023 boolean cycle marks.
5. **Portability**: `WITH` and `WITH RECURSIVE` are SQL:1999; broadly portable (Postgres, SQLite, MySQL 8+, MariaDB, SQL Server, Oracle). `SEARCH`/`CYCLE` clauses less universal. CTE-as-optimization-fence behavior diverges (Postgres ≤11 always materialized; ≥12 inlinable).
6. **Dimensions**: TC, HX (primary — decomposition/readability).

---

### sql-window-functions
1. **Name**: `sql-window-functions`
2. **Scope**: `OVER`, `PARTITION BY`, `ORDER BY`, frame clauses (`ROWS`/`RANGE`/`GROUPS`, `EXCLUDE`), ranking (`ROW_NUMBER`/`RANK`/`DENSE_RANK`/`NTILE`), offset (`LAG`/`LEAD`/`FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`), aggregate windows, named windows, and running totals.
3. **Skill-worthy / LLM failures**: The single highest-leverage modern-SQL skill, and LLMs botch frames constantly: the **default frame** with `ORDER BY` is `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`, which makes `LAST_VALUE` return the current row, not the partition's last — a notorious surprise. LLMs confuse `RANK` vs `DENSE_RANK` vs `ROW_NUMBER`, try to filter on a window result in `WHERE` (must wrap in a subquery/CTE), and mix `RANGE` vs `ROWS` semantics for running totals with duplicate sort keys.
4. **Primary sources**: https://www.postgresql.org/docs/current/tutorial-window.html ; https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS ; https://www.postgresql.org/docs/current/functions-window.html ; https://www.sqlite.org/windowfunctions.html ; https://modern-sql.com/feature/over
5. **Portability**: SQL:2003 core + SQL:2011 enhancements. Now in Postgres, SQLite 3.25+, MySQL 8+, MariaDB 10.2+, all majors. `GROUPS` mode and `EXCLUDE` are newer/less universal. Very portable for the common cases.
6. **Dimensions**: TC (primary), HX, TR.

---

### sql-gaps-and-islands
1. **Name**: `sql-gaps-and-islands`
2. **Scope**: The canonical "consecutive runs" / "gaps between runs" pattern family, solved portably with window functions (the `ROW_NUMBER` difference trick) — and where `MATCH_RECOGNIZE` supersedes it.
3. **Skill-worthy / LLM failures**: A famous, frequently-needed pattern (active-streak detection, sessionization, contiguous-range collapsing) that LLMs solve with brittle self-joins or procedural code. The set-based window solution is non-obvious and worth its own focused reference. Pairs with (and motivates) `MATCH_RECOGNIZE`.
4. **Primary sources**: https://use-the-index-luke.com (window perf) ; Itzik Ben-Gan gaps-and-islands canon (https://www.itprotoday.com / SQL Server Pro) ; Celko "SQL for Smarties" runs-and-sequences ; https://modern-sql.com/feature/match_recognize (the standard's purpose-built tool)
5. **Portability**: The window-function solution is fully portable wherever window functions exist. `MATCH_RECOGNIZE` (the cleaner answer) is barely implemented (Oracle, Trino, Snowflake, Flink — *not* Postgres/SQLite/MySQL).
6. **Dimensions**: TC, HX, TR. *(Candidate to fold into `sql-window-functions` if cutting; kept separate because it's a high-frequency, high-error standalone idiom.)*

---

### sql-lateral-and-correlated-derived
1. **Name**: `sql-lateral-and-correlated-derived`
2. **Scope**: `LATERAL` joins / correlated derived tables — referencing earlier FROM items inside a subquery in FROM; "top-N per group", per-row function expansion.
3. **Skill-worthy / LLM failures**: LLMs don't know `LATERAL` exists and reinvent top-N-per-group with window functions + filter, or with correlated scalar subqueries in the SELECT list (one per output column — N+1-in-SQL). They forget `LATERAL` is implied for set-returning functions in Postgres and get the keyword placement wrong.
4. **Primary sources**: https://modern-sql.com/feature/lateral ; https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL ; https://www.postgresql.org/docs/current/sql-select.html
5. **Portability**: `LATERAL` is SQL:1999/2003 standard. Postgres, MySQL 8.0.14+, SQL Server (`CROSS/OUTER APPLY` — non-standard spelling), Oracle 12c+. **Not in SQLite.** Flag the `APPLY` spelling.
6. **Dimensions**: TC, HX.

---

### sql-json
1. **Name**: `sql-json`
2. **Scope**: SQL/JSON (SQL:2016/2023): constructors (`JSON_OBJECT`, `JSON_ARRAY`, `JSON_OBJECTAGG`, `JSON_ARRAYAGG`), query functions (`JSON_VALUE`, `JSON_QUERY`, `JSON_EXISTS`), `JSON_TABLE`, the SQL/JSON path language (lax/strict, filters, item methods), `IS JSON`, and the SQL:2023 `JSON` type.
3. **Skill-worthy / LLM failures**: LLMs default to vendor operators (`->`, `->>`, `#>>`, `JSON_EXTRACT`, `::jsonb`) instead of the portable standard functions; they confuse `JSON_VALUE` (scalar) with `JSON_QUERY` (object/array); they don't know `JSON_TABLE` exists for shredding JSON into rows; they ignore lax-vs-strict path mode and `ON ERROR`/`ON EMPTY` clauses. They also store JSON where a normalized schema belongs (the modern EAV antipattern).
4. **Primary sources**: https://modern-sql.com/feature/json_table ; https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016 ; https://www.postgresql.org/docs/current/functions-json.html ; https://dev.mysql.com/doc/refman/8.4/en/json.html ; https://www.sqlite.org/json1.html ; Eisentraut SQL:2023 JSON section.
5. **Portability**: This is where "standard" and "available" diverge most. The *standard* SQL/JSON functions exist in Oracle, DB2, MySQL (partial), MariaDB; Postgres added `JSON_TABLE`/`JSON_QUERY`/`JSON_VALUE` in **v17** but its idiomatic path is `jsonb` operators; SQLite uses its own `json_*()` functions. The skill must teach the standard form *and* map the common vendor equivalents.
6. **Dimensions**: TC, HX, PI.

---

### sql-merge-and-upsert
1. **Name**: `sql-merge-and-upsert`
2. **Scope**: Portable upsert — the standard `MERGE` statement (SQL:2003/2016) and the portability landscape of `INSERT ... ON CONFLICT` / `ON DUPLICATE KEY` alternatives.
3. **Skill-worthy / LLM failures**: LLMs write race-prone "SELECT then INSERT or UPDATE" application logic instead of atomic upsert; they assume `MERGE` is universal (it's not — Postgres only got it in v15, SQLite has none); they don't handle the `WHEN NOT MATCHED` / `WHEN MATCHED` / `WHEN NOT MATCHED BY SOURCE` arms; classic `MERGE` concurrency footguns (it is not a substitute for proper unique constraints + conflict handling).
4. **Primary sources**: https://www.postgresql.org/docs/current/sql-merge.html ; https://www.sqlite.org/lang_upsert.html ; https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html ; https://en.wikipedia.org/wiki/Merge_(SQL)
5. **Portability**: `MERGE` is standard (SQL:2003) but historically poorly adopted: Oracle/DB2/SQL Server/Postgres15+/MySQL? (MySQL has *no* MERGE, uses `ON DUPLICATE KEY UPDATE`; SQLite uses `ON CONFLICT`). This is a *teach-the-divergence* skill.
6. **Dimensions**: TC (primary — correctness/atomicity), PI.

---

### sql-pagination-and-keyset
1. **Name**: `sql-pagination-and-keyset`
2. **Scope**: Standard pagination (`OFFSET ... FETCH FIRST n ROWS ONLY [WITH TIES]`) and the performant portable pattern — **keyset/seek pagination** — plus why `OFFSET` degrades and is unstable under concurrent writes.
3. **Skill-worthy / LLM failures**: LLMs reflexively use `LIMIT n OFFSET m`, which (a) is non-standard, (b) scans + discards `m` rows (O(offset)), and (c) skips/duplicates rows when data shifts between pages. They don't know `FETCH FIRST ... WITH TIES`, and they don't know keyset pagination (`WHERE (sort_key, id) > (:last_key, :last_id) ORDER BY ... FETCH FIRST n`).
4. **Primary sources**: https://modern-sql.com/feature/fetch-first ; https://use-the-index-luke.com/no-offset ; https://use-the-index-luke.com/sql/partial-results ; https://www.postgresql.org/docs/current/sql-select.html (FETCH/OFFSET)
5. **Portability**: `OFFSET ... FETCH` is SQL:2008/2011 standard, in Postgres, SQL Server, Oracle 12c+, DB2; **MySQL/SQLite only support `LIMIT/OFFSET`** (non-standard). Keyset pagination uses only standard `WHERE`+`ORDER BY`+row-value comparison and is the most portable.
6. **Dimensions**: TC, HX, PI (primary — app integration).

---

### sql-row-values-and-comparisons
1. **Name**: `sql-row-values-and-comparisons`
2. **Scope**: Row value constructors `(a, b, c)`, row comparisons `(a,b) > (x,y)`, `(a,b) IN ((..),(..))`, `VALUES` row lists, and their NULL/lexicographic semantics — the backbone of keyset pagination and bulk operations.
3. **Skill-worthy / LLM failures**: LLMs don't know multi-column comparisons exist and write verbose, often-wrong `(a > x) OR (a = x AND b > y)` expansions; they misuse row constructors with NULLs; they don't use `VALUES` row lists for set-based bulk inserts/joins.
4. **Primary sources**: https://www.postgresql.org/docs/current/functions-comparisons.html#ROW-WISE-COMPARISON ; https://www.postgresql.org/docs/current/sql-values.html ; https://modern-sql.com (row value constructor notes)
5. **Portability**: Row value constructors/comparisons are SQL-92 standard; well supported in Postgres/MySQL, **partial/absent in SQLite** for some forms and SQL Server (no row comparison). Flag divergence.
6. **Dimensions**: TC, HX. *(Candidate to merge into `sql-pagination-and-keyset` or `sql-select-and-query-processing`.)*

---

### sql-expressions-case-and-functions
1. **Name**: `sql-expressions-case-and-functions`
2. **Scope**: `CASE` (searched/simple), `COALESCE`/`NULLIF`/`GREATEST`/`LEAST`, standard string functions (`SUBSTRING`, `TRIM`, `OVERLAY`, `POSITION`, `||` concatenation), and `CAST` / standard type conversions.
3. **Skill-worthy / LLM failures**: LLMs use vendor functions (`IFNULL`, `ISNULL`, `NVL`, `IIF`, `SUBSTR`, `LEFT/RIGHT`, `+` for string concat in SQL Server) instead of standard `COALESCE`/`SUBSTRING`/`||`; they misuse `NULLIF`; they rely on implicit type coercion that differs per engine; they forget `||` yields NULL if any operand is NULL.
4. **Primary sources**: https://www.postgresql.org/docs/current/functions-string.html ; https://www.postgresql.org/docs/current/functions-conditional.html ; https://modern-sql.com/standard/2023 (GREATEST/LEAST/LPAD/RPAD/BTRIM)
5. **Portability**: `CASE`, `COALESCE`, `NULLIF`, `CAST`, `SUBSTRING(... FROM ... FOR ...)`, `TRIM`, `||` are SQL-92/99 standard. `GREATEST`/`LEAST`/`LPAD`/`RPAD` standardized only in SQL:2023 (long available as extensions). `||` is standard but SQL Server lacks it; MySQL treats `||` as OR by default.
6. **Dimensions**: TC, HX.

---

### sql-datetime-and-intervals
1. **Name**: `sql-datetime-and-intervals`
2. **Scope**: Standard temporal types (`DATE`, `TIME [WITH TIME ZONE]`, `TIMESTAMP [WITH TIME ZONE]`, `INTERVAL`), `EXTRACT`, datetime arithmetic with intervals, literals (`DATE '...'`, `TIMESTAMP '...'`), and `CAST`/`FORMAT`.
3. **Skill-worthy / LLM failures**: LLMs jam dates into strings, use vendor functions (`DATEADD`, `DATEDIFF`, `STRFTIME`, `NOW()` vs `CURRENT_TIMESTAMP`), botch time-zone semantics (`TIMESTAMP` vs `TIMESTAMP WITH TIME ZONE`), and do fragile string-based date math instead of `INTERVAL` arithmetic and `EXTRACT`.
4. **Primary sources**: https://www.postgresql.org/docs/current/datatype-datetime.html ; https://www.postgresql.org/docs/current/functions-datetime.html ; https://modern-sql.com (EXTRACT / temporal literals); https://www.sqlite.org/lang_datefunc.html (SQLite's deviation — no native date type)
5. **Portability**: The types/`EXTRACT`/`INTERVAL` are SQL-92/99 standard but among the *least* portably implemented: SQLite has no real date type, MySQL lacks `INTERVAL` as a stored type and lacks timezone-aware timestamps in the standard sense. High-divergence; teach standard + caveats.
6. **Dimensions**: TC (primary), PI.

---

### sql-data-types-and-numerics
1. **Name**: `sql-data-types-and-numerics`
2. **Scope**: The standard type system — exact numerics (`NUMERIC`/`DECIMAL` precision & scale, `INTEGER`/`SMALLINT`/`BIGINT`) vs approximate (`REAL`/`FLOAT`/`DOUBLE PRECISION`), `CHARACTER`/`VARCHAR`, `BOOLEAN`, `BLOB`/`CLOB`, and choosing the right type.
3. **Skill-worthy / LLM failures**: LLMs use `FLOAT`/`DOUBLE` for money (rounding errors) instead of `NUMERIC(p,s)`; they default to `VARCHAR(255)` cargo-culting; they assume `BOOLEAN` is universal (it isn't — SQL Server has `BIT`, older MySQL maps to `TINYINT`); they ignore precision/scale and overflow semantics.
4. **Primary sources**: https://www.postgresql.org/docs/current/datatype.html ; https://www.postgresql.org/docs/current/datatype-numeric.html ; https://www.sqlite.org/datatype3.html (flexible typing deviation) ; https://en.wikipedia.org/wiki/SQL#Data_types
5. **Portability**: `NUMERIC`/`DECIMAL`/`INTEGER`/`CHARACTER VARYING` are SQL-92 standard and portable. `BOOLEAN` standardized in SQL:1999 but SQL Server still lacks it. SQLite's dynamic typing is a major deviation. `DECFLOAT` (SQL:2016) is rare.
6. **Dimensions**: TC (primary), HX.

---

### sql-constraints-and-integrity
1. **Name**: `sql-constraints-and-integrity`
2. **Scope**: `PRIMARY KEY`, `UNIQUE` (and `NULLS [NOT] DISTINCT`), `FOREIGN KEY` with referential actions (`ON DELETE/UPDATE CASCADE|SET NULL|SET DEFAULT|RESTRICT|NO ACTION`), `CHECK`, `NOT NULL`, `DEFAULT`, deferrable constraints.
3. **Skill-worthy / LLM failures**: LLMs omit foreign keys entirely (relying on app-level integrity), pick wrong referential actions, forget that `UNIQUE` permits multiple NULLs, write `CHECK` constraints that pass on UNKNOWN (NULL), and don't know about deferrable constraints for circular references. They put integrity logic in application code that the database should enforce.
4. **Primary sources**: https://www.postgresql.org/docs/current/ddl-constraints.html ; https://www.sqlite.org/lang_createtable.html ; https://www.sqlite.org/foreignkeys.html ; Eisentraut SQL:2023 UNIQUE NULLS (F292)
5. **Portability**: All constraint types are SQL-92/99 standard and broadly portable. Caveats: SQLite enforces FKs only with `PRAGMA foreign_keys=ON`; MySQL/InnoDB has quirks; `CHECK` was ignored by old MySQL/MariaDB. `NULLS NOT DISTINCT` is SQL:2023 (Postgres 15+).
6. **Dimensions**: TC (primary), PI.

---

### sql-generated-and-identity-columns
1. **Name**: `sql-generated-and-identity-columns`
2. **Scope**: `GENERATED ALWAYS AS (...) STORED|VIRTUAL` computed columns, and surrogate keys via `GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY` — the standard replacement for `AUTO_INCREMENT`/`SERIAL`.
3. **Skill-worthy / LLM failures**: LLMs reach for vendor `SERIAL`/`AUTO_INCREMENT`/`IDENTITY(1,1)` instead of standard `GENERATED AS IDENTITY`; they don't know computed columns exist (recompute in every query or in app code); they confuse `STORED` vs `VIRTUAL`; they misuse `ALWAYS` vs `BY DEFAULT` (whether explicit inserts are allowed).
4. **Primary sources**: https://www.postgresql.org/docs/current/ddl-generated-columns.html ; https://www.postgresql.org/docs/current/ddl-identity-columns.html ; https://modern-sql.com/feature/generated-columns (and identity) ; https://www.sqlite.org/gencol.html
5. **Portability**: `GENERATED AS IDENTITY` is SQL:2003 standard (Postgres 10+, DB2, Oracle 12c+, SQL Server lacks the exact syntax). Generated columns are SQL:2003 (Postgres 12+ STORED only, MySQL both, SQLite both). Decent portability for the modern form; flag `SERIAL` as legacy/non-standard.
6. **Dimensions**: TC, HX, PI.

---

### sql-views-and-introspection
1. **Name**: `sql-views-and-introspection`
2. **Scope**: `CREATE VIEW`, updatable views, `WITH [LOCAL|CASCADED] CHECK OPTION`, and standard schema introspection via `INFORMATION_SCHEMA`.
3. **Skill-worthy / LLM failures**: LLMs query vendor catalogs (`pg_catalog`, `sqlite_master`, `SHOW TABLES`) instead of the portable `INFORMATION_SCHEMA`; they don't know `WITH CHECK OPTION` prevents inserts/updates through a view that would vanish from it; they assume all views are updatable.
4. **Primary sources**: https://www.postgresql.org/docs/current/sql-createview.html ; https://www.postgresql.org/docs/current/information-schema.html ; https://en.wikipedia.org/wiki/Information_schema ; https://www.sqlite.org/schematab.html (SQLite's deviation)
5. **Portability**: `INFORMATION_SCHEMA` is SQL-92 standard, in Postgres/MySQL/MariaDB/SQL Server; **SQLite does NOT have it** (uses `sqlite_master`/`PRAGMA`). `WITH CHECK OPTION` standard. Updatable-view rules vary.
6. **Dimensions**: PI (primary — introspection/tooling), TC, HX.

---

### sql-injection-and-parameterization
1. **Name**: `sql-injection-and-parameterization`
2. **Scope**: Parameterized queries / prepared statements as the *only* correct way to combine SQL with data; placeholders; why string interpolation/concatenation is never acceptable; identifiers can't be parameterized (allowlist them).
3. **Skill-worthy / LLM failures**: This is arguably the **highest-stakes** skill. LLMs routinely f-string/`%`-format/template user data straight into SQL ("just for this example"), suggest manual escaping/quoting as a "fix," and conflate parameter placeholders with identifier substitution. They produce different placeholder styles (`?`, `$1`, `:name`, `%s`) without noting they're driver-specific.
4. **Primary sources**: https://owasp.org/www-community/attacks/SQL_Injection ; https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html ; https://www.postgresql.org/docs/current/sql-prepare.html ; Karwin "SQL Injection" antipattern https://pragprog.com/titles/bksqla/sql-antipatterns/
5. **Portability**: The *principle* (prepared statements / bind parameters) is universal and standard (`PREPARE`/`EXECUTE` are in the standard). The *placeholder syntax* is driver/engine-specific — the skill teaches the concept and notes the spellings.
6. **Dimensions**: TC, PI (primary — app integration), HX.

---

### sql-transactions-and-isolation  ⟵ BOUNDARY with mvcc-skills-plugin
1. **Name**: `sql-transactions-and-isolation`
2. **Scope**: The *SQL-statement* side of transactions: `START TRANSACTION`/`COMMIT`/`ROLLBACK`, `SAVEPOINT`/`ROLLBACK TO`, `SET TRANSACTION ISOLATION LEVEL`, the four standard level *names*, autocommit, and read-only transactions. **Defers** the anomaly theory and MVCC mechanics to the mvcc plugin.
3. **Skill-worthy / LLM failures**: LLMs forget transactions exist for multi-statement invariants (the lost-update / check-then-act race), never use `SAVEPOINT`, leave transactions open, assume `READ COMMITTED` everywhere (defaults differ — Postgres RC, MySQL RR), and treat DDL as transactional when it isn't (MySQL auto-commits DDL).
4. **Primary sources**: https://www.postgresql.org/docs/current/tutorial-transactions.html ; https://www.postgresql.org/docs/current/sql-set-transaction.html ; https://www.postgresql.org/docs/current/transaction-iso.html ; https://www.sqlite.org/lang_transaction.html ; https://en.wikipedia.org/wiki/Isolation_(database_systems)
5. **Portability**: `COMMIT`/`ROLLBACK`/`SAVEPOINT`/isolation-level names are SQL-92 standard and portable. *Behavior* of each level differs sharply per engine — which is exactly why anomaly semantics are delegated.
6. **Dimensions**: TC, PI. **Overlap note**: see §5 — this skill owns *syntax & app usage*; `mvcc-isolation-levels-and-anomalies` owns the *anomaly catalog and theory*. Cross-link, don't duplicate.

---

### sql-privileges-and-access-control
1. **Name**: `sql-privileges-and-access-control`
2. **Scope**: The standard security model: `GRANT`/`REVOKE`, object vs system privileges, roles, `WITH GRANT OPTION`, least-privilege schema design (app role ≠ owner/superuser).
3. **Skill-worthy / LLM failures**: LLMs default to connecting as superuser/root, grant `ALL` broadly, don't use roles, and ignore least-privilege. They confuse object ownership with grants.
4. **Primary sources**: https://www.postgresql.org/docs/current/sql-grant.html ; https://www.postgresql.org/docs/current/ddl-priv.html ; https://www.postgresql.org/docs/current/user-manag.html ; https://en.wikipedia.org/wiki/SQL#Procedural_extensions (security model)
5. **Portability**: `GRANT`/`REVOKE`/roles are SQL-92/99 standard; broadly portable in Postgres/MySQL/SQL Server/Oracle. **SQLite has no user/privilege model** (file-permission based) — flag prominently. Privilege *names* vary.
6. **Dimensions**: PI (primary), TC.

---

### sql-indexing-and-sargability
1. **Name**: `sql-indexing-and-sargability`
2. **Scope**: Portable indexing principles: B-tree mechanics, composite-index column order (the "equality-first, then range" rule), covering indexes, **sargability** (why functions/wrapping on indexed columns kill index use), and the "index supports WHERE + ORDER BY + GROUP BY" idea.
3. **Skill-worthy / LLM failures**: LLMs wrap indexed columns in functions (`WHERE LOWER(email)=...`, `WHERE DATE(ts)=...`, `WHERE col + 0 = ...`) defeating the index; use leading wildcards (`LIKE '%x'`); get composite column order backwards; add redundant single-column indexes (the "index shotgun" antipattern); and don't realize an index can serve ordering. This is the highest-leverage *performance* skill and Winand's whole book is vendor-neutral.
4. **Primary sources**: https://use-the-index-luke.com/sql/where-clause ; https://use-the-index-luke.com/sql/anatomy ; https://use-the-index-luke.com/sql/clustering ; https://use-the-index-luke.com/sql/sorting-grouping ; https://www.sqlite.org/queryplanner.html ; Karwin "Index Shotgun" antipattern.
5. **Portability**: B-tree indexing principles are universal across all relational engines. `CREATE INDEX` is standard-ish (not in core SQL but ubiquitous). Sargability is engine-agnostic. Highly portable conceptually.
6. **Dimensions**: TC, PI (primary — performance), TR.

---

### sql-explain-and-set-based-thinking
1. **Name**: `sql-explain-and-set-based-thinking`
2. **Scope**: `EXPLAIN` as a concept (reading a plan: scans vs index access, join order, estimated vs actual rows, cardinality/selectivity), avoiding N+1, and set-based vs row-by-row ("RBAR") thinking.
3. **Skill-worthy / LLM failures**: LLMs generate per-row query loops in application code (N+1) where a single set-based query (or `LATERAL`/`JOIN`/`VALUES`) wins; they never look at a plan; they don't understand that a stale/under-estimated cardinality drives bad plans; they "optimize" by guessing instead of measuring.
4. **Primary sources**: https://www.postgresql.org/docs/current/using-explain.html ; https://www.sqlite.org/eqp.html ; https://use-the-index-luke.com/sql/testing-scalability ; Karwin "Spaghetti Query" antipattern.
5. **Portability**: `EXPLAIN` is non-standard and engine-specific in output, but the *concepts* (scan vs seek, cardinality, set-based thinking) are universal. Teach the concept; the per-engine plan format belongs in vendor plugins.
6. **Dimensions**: TR (primary — proving performance), TC, PI.

---

### sql-style-and-naming
1. **Name**: `sql-style-and-naming`
2. **Scope**: Formatting & style conventions: keyword casing, the `'string'` (literal) vs `"identifier"` (delimited identifier) distinction, identifier naming/quoting, leading-vs-trailing commas, and consistent layout for reviewability.
3. **Skill-worthy / LLM failures**: LLMs use `"double quotes"` for string literals (standard says that's an *identifier*; MySQL's lax default hides the bug until it doesn't), inconsistent casing, mixed naming (camelCase columns that then require quoting forever), and unreadable single-line mega-queries. The literal-vs-identifier quote rule is a genuine portability/correctness trap.
4. **Primary sources**: https://www.postgresql.org/docs/current/sql-syntax-lexical.html (identifiers, quoting, literals) ; https://modern-sql.com (string vs identifier) ; https://www.sqlitetutorial.net / SQL style guides (Simon Holywell https://www.sqlstyle.guide/) ; Karwin readability chapters.
5. **Portability**: The `'...'` vs `"..."` rule is SQL-92 standard; MySQL deviates (treats `"` as string unless `ANSI_QUOTES`). Naming/casing/formatting are conventions, fully portable.
6. **Dimensions**: HX (primary), TC, PI.

---

### sql-schema-design-and-normalization
1. **Name**: `sql-schema-design-and-normalization`
2. **Scope**: Portable schema-design discipline: normalization 1NF→BCNF, choosing keys, when to denormalize, and the major *structural* antipatterns (EAV, comma-separated lists/"Jaywalking", naive adjacency-only trees, polymorphic/promiscuous FKs).
3. **Skill-worthy / LLM failures**: LLMs produce EAV "flexible" schemas, store comma-separated foreign keys in a column, model hierarchies with only `parent_id` and then can't query subtrees, and over-normalize or under-normalize without rationale. These are Karwin's core chapters and are vendor-neutral design discipline.
4. **Primary sources**: Karwin "SQL Antipatterns" https://pragprog.com/titles/bksqla/sql-antipatterns/ (Jaywalking, Naive Trees, EAV, Polymorphic Associations) ; https://en.wikipedia.org/wiki/Database_normalization ; https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form ; Codd 1970 https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf ; Celko trees (nested sets).
5. **Portability**: Pure design theory — 100% portable, engine-independent.
6. **Dimensions**: HX (primary), TC, PI.

---

### sql-temporal-tables  *(advanced / lower-priority)*
1. **Name**: `sql-temporal-tables`
2. **Scope**: SQL:2011 temporal: system-versioned tables (`PERIOD FOR SYSTEM_TIME`, `WITH SYSTEM VERSIONING`, `FOR SYSTEM_TIME AS OF/FROM..TO/BETWEEN`), application-time periods (`FOR PORTION OF`), and bitemporal modeling.
3. **Skill-worthy / LLM failures**: When asked for audit history / "as-of" queries, LLMs hand-roll trigger-based history tables instead of using the standard mechanism (where available); they don't know `FOR SYSTEM_TIME AS OF` exists; they confuse system-time (when recorded) vs application/valid-time (when true).
4. **Primary sources**: https://en.wikipedia.org/wiki/SQL:2011 ; https://modern-sql.com/standard/2011 ; https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables ; https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables
5. **Portability**: SQL:2011 standard but **unevenly implemented**: MariaDB, SQL Server, DB2, Oracle (Flashback-ish) support it; **Postgres, SQLite, MySQL do NOT** have native standard temporal tables. Heavy divergence — overlaps `mvcc-time-travel-queries`. Lower priority / candidate to defer to a vendor plugin.
6. **Dimensions**: TC, PI.

---

### sql-match-recognize  *(advanced / optional)*
1. **Name**: `sql-match-recognize`
2. **Scope**: `MATCH_RECOGNIZE` (SQL:2016) — regex-over-rows pattern recognition: `PARTITION BY`/`ORDER BY`, `PATTERN`, `DEFINE`, `MEASURES`, `ONE ROW`/`ALL ROWS PER MATCH`, `AFTER MATCH SKIP`.
3. **Skill-worthy / LLM failures**: LLMs don't know it exists and solve time-series pattern problems (V-shapes, sessionization, threshold breaches) with convoluted self-joins or window-function gymnastics. When they do try it, they confuse greedy/reluctant quantifiers and the `RUNNING`/`FINAL` semantics.
4. **Primary sources**: https://modern-sql.com/feature/match_recognize ; https://link.springer.com/article/10.1007/s13222-022-00404-3 ; https://trino.io/blog/2021/05/19/row_pattern_matching.html
5. **Portability**: SQL:2016 standard but **rare**: Oracle, Trino/Presto, Snowflake, Flink, Vertica, DB2. **Not in Postgres/SQLite/MySQL/MariaDB.** Niche; include as advanced reference, flag low portability.
6. **Dimensions**: TC. *(Lowest-priority standalone skill; could be a section in `sql-gaps-and-islands`.)*

---

### sql-property-graph-queries  *(bleeding-edge / optional)*
1. **Name**: `sql-property-graph-queries`
2. **Scope**: SQL:2023 SQL/PGQ — `CREATE PROPERTY GRAPH` over relational tables and `GRAPH_TABLE (... MATCH (a)-[e]->(b) COLUMNS (...))` ASCII-art graph pattern matching; relationship to GQL.
3. **Skill-worthy / LLM failures**: Brand-new; LLMs either don't know it or hallucinate Cypher/Gremlin syntax. Useful as a forward-looking reference for "graph queries without a graph DB."
4. **Primary sources**: https://modern-sql.com/standard/2023 ; https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq ; https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard ; https://arxiv.org/pdf/2409.01102
5. **Portability**: SQL:2023 Part 16; **barely implemented** (Oracle 23ai; Postgres only experimental/extensions as of 2026). Very low portability — include only as a thin "what's coming" reference, clearly flagged.
6. **Dimensions**: TC. *(Lowest priority; defer or keep minimal.)*

---

## 4. Recommended grouping (the coherent set)

A **22-skill core** (exceeds the golang plugin's 25? — see note) plus 2–4 optional advanced skills. The foundation skill is the policy root all others reference, mirroring `go-idiomatic-discipline`.

**Tier 0 — Foundation (policy root, everything cross-links to it)**
1. `sql-relational-and-null-discipline` ← THE foundation skill

**Tier 1 — Core query (the daily 80%)**
2. `sql-select-and-query-processing`
3. `sql-joins`
4. `sql-subqueries-and-exists`
5. `sql-aggregation-and-grouping`
6. `sql-set-operations`
7. `sql-expressions-case-and-functions`
8. `sql-datetime-and-intervals`

**Tier 2 — Modern / advanced query (the "good modern shit")**
9. `sql-cte-and-recursion`
10. `sql-window-functions`
11. `sql-gaps-and-islands`
12. `sql-lateral-and-correlated-derived`
13. `sql-json`
14. `sql-merge-and-upsert`
15. `sql-pagination-and-keyset`
16. `sql-row-values-and-comparisons`

**Tier 3 — DDL / schema / integrity**
17. `sql-data-types-and-numerics`
18. `sql-constraints-and-integrity`
19. `sql-generated-and-identity-columns`
20. `sql-views-and-introspection`
21. `sql-schema-design-and-normalization`

**Tier 4 — Correctness / security / process**
22. `sql-injection-and-parameterization`
23. `sql-transactions-and-isolation` (boundary w/ mvcc)
24. `sql-privileges-and-access-control`

**Tier 5 — Performance & style**
25. `sql-indexing-and-sargability`
26. `sql-explain-and-set-based-thinking`
27. `sql-style-and-naming`

**Tier 6 — Advanced / optional (low portability; include or defer to vendor plugins)**
28. `sql-temporal-tables`
29. `sql-match-recognize`
30. `sql-property-graph-queries`

That's 27 high-value skills + 3 optional. To land in the requested **18–26** band while staying "more comprehensive than golang's 25," recommended **merges/cuts**:
- Merge `sql-row-values-and-comparisons` → into `sql-pagination-and-keyset` (its main consumer) or `sql-select-and-query-processing`.
- Merge `sql-gaps-and-islands` → as a major section of `sql-window-functions` (keep separate only if you want a dedicated cookbook).
- Treat `sql-match-recognize` + `sql-property-graph-queries` + `sql-temporal-tables` as **optional/deferred** (low portability; natural fit for vendor plugins) — or collapse the three exotic ones into a single `sql-advanced-standard-features` survey skill.

A defensible **24-skill shipping set**: Tiers 0–5 (skills 1–27) minus the three merges above = 24, with the three exotic features either folded into one survey skill (→ 25) or deferred. This is comfortably more comprehensive than the 25-skill golang plugin while every skill is genuinely standard and high-leverage.

**Cross-cutting candidate (optional 25th):** `sql-standard-vs-dialect-map` — the SQL analogue of `go-version-feature-map`: a single reference table of "standard feature → which readable engines have it / how they spell it" (LIMIT vs FETCH, SERIAL vs IDENTITY, `||` vs `+` vs `CONCAT`, `IS DISTINCT FROM` vs `<=>`, MERGE vs ON CONFLICT vs ON DUPLICATE KEY, INFORMATION_SCHEMA vs sqlite_master). Extremely useful as a portability index that other skills link to. **Strongly recommended** as the second cross-cutting anchor alongside the foundation skill.

---

## 5. Gaps & overlaps

### Overlap with `mvcc-skills-plugin` (~/code/mvcc-skills-plugin)
The mvcc plugin has a deep, rigorous 22-skill treatment of concurrency, including `mvcc-isolation-levels-and-anomalies` (owns the full anomaly catalog: dirty write P0, dirty read P1, non-repeatable read P2, phantom P3, lost update P4, read skew, write skew, the Adya G-definitions, and the critical "forbidding the 3 phenomena ≠ serializable" result), `mvcc-snapshot-isolation`, `mvcc-serializable-ssi`, `mvcc-choosing-isolation`, `mvcc-time-travel-queries`, and `mvcc-anomaly-testing`.

**Boundary recommendation:**
- `sql-transactions-and-isolation` should own only the **SQL surface**: `START TRANSACTION`/`COMMIT`/`ROLLBACK`/`SAVEPOINT`/`SET TRANSACTION ISOLATION LEVEL`, the four level *names*, autocommit, read-only/deferrable, DDL-in-transaction caveats, and *when to use a transaction at all* (multi-statement invariants / check-then-act). It should state the anomaly definitions in ≤1 paragraph and **explicitly route** the theory ("which anomalies each level permits, is-this-serializable, write skew, snapshot mechanics") to `mvcc-isolation-levels-and-anomalies` and siblings — exactly as the mvcc skills route among themselves.
- `sql-temporal-tables` (SQL:2011 `FOR SYSTEM_TIME`) overlaps `mvcc-time-travel-queries`. Boundary: the SQL skill owns the **standard DDL/query syntax**; the mvcc skill owns the **versioning/storage mechanics and snapshot semantics**. Given low portability, consider deferring the SQL temporal skill or making it a thin syntax reference that links to mvcc.

### Other overlaps / boundaries
- **Performance**: `sql-indexing-and-sargability` and `sql-explain-and-set-based-thinking` teach *portable principles* (Winand). Engine-specific plan internals (Postgres buffers, InnoDB B+tree, SQLite page cache) belong in vendor plugins and partly overlap mvcc's storage skills — keep the SQL skills vendor-neutral.
- **JSON**: `sql-json` teaches standard SQL/JSON; deep `jsonb` indexing/GIN belongs in a Postgres plugin.

### Genuine gaps (intentionally out of scope for *standard* SQL)
- **Stored procedures / SQL/PSM** (`CREATE PROCEDURE`, control flow, triggers' bodies): standardized (SQL/PSM, Part 4) but so divergently implemented (PL/pgSQL vs T-SQL vs PL/SQL vs MySQL SP) that it's effectively non-portable — recommend deferring to vendor plugins. A thin `sql-triggers-and-routines` *concept* skill is optional.
- **Full-text search, geospatial, vector/embedding types**: all vendor extensions, not standard core — out of scope.
- **`SQL/MED` (foreign data / external tables)**, **`SQL/XML`**: standardized but niche/legacy; skip unless requested.
- **Connection/driver concerns** (pooling, cursors over the wire): app-layer, not SQL-language — out of scope (touch `sql-pagination-and-keyset` / `sql-injection-and-parameterization` lightly).

---

## 6. Notes for the planning agent

- **Foundation discipline pattern**: `sql-relational-and-null-discipline` is the `go-idiomatic-discipline` analogue — every other skill should open by cross-linking it, because NULL/three-valued-logic and set-semantics underpin correctness everywhere (joins, aggregates, NOT IN, constraints, set ops).
- **Portability is the recurring teaching tension**: unlike Go (one implementation), standard SQL has *many* partial implementations. Every skill needs a crisp "standard says X; readable engines: Postgres ✓, SQLite ~, MySQL ✗ (spells it Y)" block. The proposed `sql-standard-vs-dialect-map` skill centralizes this so individual skills stay focused.
- **Readable-engine strategy for per-skill researchers**: Postgres docs are the best standard-aligned readable reference; SQLite docs are the best at explicitly stating deviations; modern-sql.com is the best vendor-neutral "is this standard + who has it" source; use-the-index-luke.com is the vendor-neutral performance bible. Prefer these four over search snippets.
- **Highest-leverage / most-LLM-error-prone** (prioritize quality here): `sql-relational-and-null-discipline`, `sql-window-functions`, `sql-injection-and-parameterization`, `sql-joins` (ON-vs-WHERE outer-join bug), `sql-indexing-and-sargability`, `sql-pagination-and-keyset`, `sql-merge-and-upsert`.
