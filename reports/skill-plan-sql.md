# Skill Decomposition Plan: Standard SQL Skills Plugin

This plan finalizes the skill set the drafters will build, derived from `reports/skill-research-sql.md`.
Conventions match the golang plugin: kebab-case `sql-`-prefixed skill dirs under `plugins/sql/skills/<name>/SKILL.md`,
each with a `references/` folder containing `common-mistakes.md` + `sources.yaml`; frontmatter carries
`allowed-tools: Read, Glob, Grep` and `compatibility: "Claude Code, Codex CLI, Gemini CLI"`.

---

## 0. Locked decisions (read first)

1. **Final count: 30 skills.** Comfortably more comprehensive than golang's 25 — this is the explicit goal ("the good modern shit"). Breakdown: 1 foundation policy-root + 25 core/standard skills + 1 cross-cutting dialect map + 3 advanced low-portability skills.
2. **The 3 advanced standard features stay IN as real skills** (`sql-temporal-tables`, `sql-match-recognize`, `sql-property-graph-queries`), each flagged low-portability in its description and body. Rationale: comprehensiveness is the stated objective, and these are the headline SQL:2011/2016/2023 additions — exactly "the modern shit." They are NOT collapsed into a survey; collapsing would bury the one place an agent learns the real syntax instead of hallucinating Cypher/self-joins.
3. **`sql-standard-vs-dialect-map` is IN** as the second cross-cutting anchor (the `go-version-feature-map` analogue). It is the portability index every other skill links to, so individual skills can stay focused and just route here for "who spells it how."
4. **`sql-gaps-and-islands`: KEPT SEPARATE** as a high-value pattern cookbook. Justification: it is a high-frequency, high-error standalone idiom (streak detection, sessionization, range-collapsing) whose set-based window solution is non-obvious; folding it into `sql-window-functions` would overload that skill and hide a recipe agents reach for by name. It motivates and routes to `sql-match-recognize`.
5. **`sql-row-values-and-comparisons`: FOLDED into `sql-pagination-and-keyset`.** Justification: its dominant, highest-leverage use is the keyset-pagination predicate `(sort_key, id) > (:k, :id)`, and its other uses (VALUES row-lists, multi-row `IN`) are small enough to host in `sql-pagination-and-keyset` plus a pointer from `sql-select-and-query-processing`. Keeping a separate thin skill would duplicate the row-comparison NULL discussion that already lives in the foundation skill. (If Q prefers maximal granularity, un-folding it yields 31 — flagged as an open question.)
6. **MVCC boundary (hard rule for drafters).** `sql-transactions-and-isolation` owns ONLY the SQL surface: `START TRANSACTION`/`COMMIT`/`ROLLBACK`/`SAVEPOINT`, isolation-level *names*, autocommit, read-only/deferrable, DDL-in-transaction caveats, and *when to open a transaction at all*. It states the anomaly catalog in <=1 paragraph and **routes** all anomaly theory / snapshot mechanics / "is this serializable" / write-skew to `mvcc-isolation-levels-and-anomalies` and siblings in `~/code/mvcc-skills-plugin`. `sql-temporal-tables` owns standard `FOR SYSTEM_TIME` DDL/query syntax; it routes versioning/storage/snapshot semantics to `mvcc-time-travel-queries`. **Do not duplicate MVCC theory.**

Every skill cross-links the foundation (`sql-relational-and-null-discipline`), mirroring how every Go skill routes back to `go-idiomatic-discipline`.

---

## 1. Final skill list (30)

| # | Skill | Type | Wave |
|---|-------|------|------|
| 1 | sql-relational-and-null-discipline | policy-root | Foundation |
| 2 | sql-select-and-query-processing | technique | 1 |
| 3 | sql-joins | technique | 1 |
| 4 | sql-subqueries-and-exists | technique | 1 |
| 5 | sql-aggregation-and-grouping | technique | 1 |
| 6 | sql-set-operations | technique | 1 |
| 7 | sql-window-functions | technique | 1 |
| 8 | sql-cte-and-recursion | technique | 1 |
| 9 | sql-lateral-and-correlated-derived | technique | 1 |
| 10 | sql-expressions-case-and-functions | reference/technique | 1 |
| 11 | sql-datetime-and-intervals | technique | 1 |
| 12 | sql-data-types-and-numerics | reference | 1 |
| 13 | sql-constraints-and-integrity | technique/policy | 1 |
| 14 | sql-injection-and-parameterization | policy | 1 |
| 15 | sql-merge-and-upsert | pattern | 1 |
| 16 | sql-pagination-and-keyset (absorbs row-values) | pattern | 1 |
| 17 | sql-generated-and-identity-columns | technique | 2 |
| 18 | sql-views-and-introspection | technique | 2 |
| 19 | sql-json | technique | 2 |
| 20 | sql-gaps-and-islands | pattern (cookbook) | 2 |
| 21 | sql-schema-design-and-normalization | pattern/policy | 2 |
| 22 | sql-style-and-naming | policy | 2 |
| 23 | sql-transactions-and-isolation | technique | 2 |
| 24 | sql-privileges-and-access-control | policy | 2 |
| 25 | sql-indexing-and-sargability | technique | 2 |
| 26 | sql-explain-and-set-based-thinking | technique | 2 |
| 27 | sql-standard-vs-dialect-map | reference | 2 |
| 28 | sql-temporal-tables | reference (low-portability) | 2 |
| 29 | sql-match-recognize | reference (low-portability) | 2 |
| 30 | sql-property-graph-queries | reference (low-portability) | 2 |

Coverage-dimension legend: **TC** technical correctness · **HX** human experience · **TR** testing reality · **PI** process integration.

---

## 2. Per-skill spec blocks

> Drafters: every `description` below is a *draft* to refine — it must stay rich, lead with what the skill guides, and carry explicit `Auto-invokes when ...` triggers, modeled on the golang descriptions. Keep descriptions standalone (no "this skill"). Always open the body by cross-linking the foundation skill.

---

### 1. sql-relational-and-null-discipline  ← FOUNDATION / POLICY ROOT
- **type**: policy-root
- **one-line**: The relational model + three-valued logic as the non-negotiable mental model every other SQL skill leans on.
- **draft description**: Guides the core correctness floor of SQL — rows are unordered sets (no order without `ORDER BY`), `NULL` means "unknown" and propagates `UNKNOWN` through every comparison and arithmetic, and `WHERE`/`HAVING`/`ON`/`CHECK` keep only rows that evaluate to `TRUE`. Bans the recurring NULL traps: `x = NULL` (always UNKNOWN — use `IS NULL`), `col <> 'a'` silently dropping NULL rows, `NOT IN (subquery)` collapsing to zero rows on a single NULL, and `COUNT(col)` vs `COUNT(*)`. Teaches null-safe equality `IS [NOT] DISTINCT FROM` and `UNIQUE NULLS [NOT] DISTINCT` (SQL:2023). Auto-invokes when writing or editing any `WHERE`/`HAVING`/`ON`/`CHECK` predicate, comparisons against possibly-null columns, `NOT IN`/`<>`/`!=`, `COUNT`/aggregates over nullable columns, or on "why does my query return no/extra rows" and "handle NULL" requests. The dual-axis policy root every other SQL skill routes back to.
- **scope (IN)**: set semantics & unordered rows; three-valued logic truth tables; NULL propagation in comparisons/arithmetic/concatenation; `IS NULL`/`IS NOT NULL`; `IS [NOT] DISTINCT FROM`; the `NOT IN`+NULL trap (stated here, deep-dive in subqueries skill); `COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT)`; NULL in `UNIQUE`/`GROUP BY`/`ORDER BY` (`NULLS FIRST/LAST`); `UNIQUE NULLS [NOT] DISTINCT`; "result order is undefined without ORDER BY."
- **scope boundaries (OUT)**: full join mechanics → `sql-joins`; the `NOT IN` anti-join rewrite → `sql-subqueries-and-exists`; aggregate-specific grouping rules → `sql-aggregation-and-grouping`; constraint syntax → `sql-constraints-and-integrity`; dialect spellings of null-safe equality (`<=>`, `IS`) → `sql-standard-vs-dialect-map`.
- **primary sources**: https://modern-sql.com/concept/three-valued-logic ; https://modern-sql.com/concept/null ; https://www.sqlite.org/nulls.html ; https://www.postgresql.org/docs/current/functions-comparison.html ; Codd 1970 https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf ; Karwin "Fear of the Unknown" https://pragprog.com/titles/bksqla/sql-antipatterns/
- **cross-links**: routes to → subqueries-and-exists, joins, aggregation-and-grouping, constraints-and-integrity, standard-vs-dialect-map. (Linked FROM: all.)
- **dimensions**: TC (primary), HX, TR.

---

### 2. sql-select-and-query-processing
- **type**: technique
- **one-line**: SELECT fundamentals and the logical clause-evaluation order that governs what's legal where.
- **draft description**: Guides the logical processing model of a query — `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `OFFSET/FETCH` — and why it, not the written order, decides what each clause can reference. Prevents referencing a `SELECT`-list alias in `WHERE` (illegal — `SELECT` runs after `WHERE`), putting aggregate conditions in `WHERE` instead of `HAVING`, expecting `ORDER BY` to affect grouping, and confusing whole-row `DISTINCT` with per-column uniqueness or vendor `DISTINCT ON`. Auto-invokes when writing or editing `SELECT` statements, `WHERE`/`HAVING`/`DISTINCT`/`ORDER BY` clauses, column aliases referenced elsewhere, or on "why is this column not allowed here" errors.
- **scope (IN)**: clause logical order and its consequences; alias visibility; `WHERE` vs `HAVING`; `DISTINCT` (whole-row) vs grouping; `SELECT *` vs explicit column lists (Karwin "Implicit Columns"); `ORDER BY` ordinals/expressions/`NULLS FIRST|LAST`; brief pointer to row-value/`VALUES` constructors (now hosted in pagination skill).
- **scope boundaries (OUT)**: join syntax → `sql-joins`; aggregation detail → `sql-aggregation-and-grouping`; window-result filtering → `sql-window-functions`; `OFFSET/FETCH`/`LIMIT` mechanics → `sql-pagination-and-keyset`; `DISTINCT ON` portability → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/sql-select.html ; https://www.postgresql.org/docs/current/queries-overview.html ; https://www.sqlite.org/lang_select.html ; Itzik Ben-Gan logical-query-processing writeups.
- **cross-links**: foundation; → joins, aggregation-and-grouping, window-functions, pagination-and-keyset, set-operations, standard-vs-dialect-map.
- **dimensions**: TC, HX.

---

### 3. sql-joins
- **type**: technique
- **one-line**: INNER/LEFT/RIGHT/FULL/CROSS joins, `ON` vs `WHERE`, and the outer-join filter trap.
- **draft description**: Guides correct join composition and the single most damaging join bug — putting a filter on the null-able side in `WHERE` instead of `ON`, which silently demotes a `LEFT JOIN` to an `INNER JOIN`. Covers `INNER`/`LEFT`/`RIGHT`/`FULL`/`CROSS`, `ON` vs `USING` vs `NATURAL` (and why `NATURAL JOIN` is a schema-change landmine), accidental cross products from missing predicates, and multi-table join order/direction. Auto-invokes when writing or editing `JOIN` clauses, `ON`/`USING`/`NATURAL` conditions, queries filtering an outer-joined table, or on "my LEFT JOIN is dropping rows" / "duplicate rows after join" symptoms.
- **scope (IN)**: all five join types; `ON` vs `WHERE` placement with outer joins; `USING`/`NATURAL` and why to avoid them; self-joins; cross-product detection; join column nullability interacting with three-valued logic.
- **scope boundaries (OUT)**: NULL truth tables → foundation; semi/anti-joins via EXISTS → `sql-subqueries-and-exists`; `LATERAL`/correlated joins → `sql-lateral-and-correlated-derived`; join *performance*/index support → `sql-indexing-and-sargability`; `FULL JOIN` emulation in MySQL → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN ; https://www.sqlite.org/lang_select.html ; https://use-the-index-luke.com/sql/join
- **cross-links**: foundation; → subqueries-and-exists, lateral-and-correlated-derived, indexing-and-sargability, standard-vs-dialect-map.
- **dimensions**: TC (primary), HX, TR.

---

### 4. sql-subqueries-and-exists
- **type**: technique
- **one-line**: Scalar/correlated subqueries, `IN`/`EXISTS`, and the safe portable anti-join.
- **draft description**: Guides subquery use and the `NOT IN` + NULL trap — a single NULL in the subquery makes `NOT IN` return zero rows, so `NOT EXISTS` is the safe, portable anti-join. Covers scalar subqueries (and the >1-row runtime error), correlated vs uncorrelated execution, `IN`/`EXISTS`/`ANY`/`ALL`, and choosing a join vs `EXISTS` for clarity and plan quality. Auto-invokes when writing or editing subqueries, `IN (SELECT ...)`, `NOT IN`, `EXISTS`/`NOT EXISTS`, correlated subqueries, or on "NOT IN returns nothing" / "subquery returned more than one row" errors.
- **scope (IN)**: scalar/row/table subqueries; semi-join (`EXISTS`/`IN`) and anti-join (`NOT EXISTS`); correlated subqueries; `ANY`/`ALL`; the NOT-IN-NULL trap deep-dive; subquery-vs-join tradeoffs.
- **scope boundaries (OUT)**: three-valued-logic basics → foundation; `LATERAL`/correlated derived tables in FROM → `sql-lateral-and-correlated-derived`; window-function alternatives to top-N → `sql-window-functions`; plan implications → `sql-explain-and-set-based-thinking`.
- **primary sources**: https://modern-sql.com/concept/three-valued-logic ; https://www.postgresql.org/docs/current/functions-subquery.html ; https://www.sqlite.org/lang_expr.html#the_exists_operator
- **cross-links**: foundation; → joins, lateral-and-correlated-derived, window-functions, explain-and-set-based-thinking.
- **dimensions**: TC (primary), HX.

---

### 5. sql-aggregation-and-grouping
- **type**: technique
- **one-line**: `GROUP BY`/`HAVING`, `FILTER`, ordered-set aggregates, and `GROUPING SETS`/`ROLLUP`/`CUBE`.
- **draft description**: Guides aggregation correctness — never select non-grouped, non-aggregated columns (the "ambiguous groups" antipattern that errors everywhere except lax MySQL), use the standard `FILTER (WHERE ...)` clause instead of fragile `CASE WHEN ... END` inside aggregates, and reach for `GROUPING SETS`/`ROLLUP`/`CUBE` instead of hand-`UNION`-ing grouping levels. Covers `COUNT(DISTINCT)`, NULL exclusion in aggregates, and ordered-set aggregates (`LISTAGG`/`ARRAY_AGG`). Auto-invokes when writing or editing `GROUP BY`/`HAVING`, aggregate functions (`SUM`/`COUNT`/`AVG`/`MIN`/`MAX`), `FILTER`, `LISTAGG`/`STRING_AGG`/`GROUP_CONCAT`, or multi-level rollup reports.
- **scope (IN)**: `GROUP BY`/`HAVING`; functional-dependency rule for selected columns; `FILTER` (and CASE emulation where missing); `COUNT` variants & NULL handling; `GROUPING SETS`/`ROLLUP`/`CUBE` + `GROUPING()`; ordered-set aggregates `LISTAGG`/`ARRAY_AGG` and `WITHIN GROUP`.
- **scope boundaries (OUT)**: aggregate-as-window (`OVER`) → `sql-window-functions`; NULL semantics → foundation; `LISTAGG`/`STRING_AGG`/`GROUP_CONCAT` dialect map → `sql-standard-vs-dialect-map`; clause order → `sql-select-and-query-processing`.
- **primary sources**: https://www.postgresql.org/docs/current/functions-aggregate.html ; https://modern-sql.com/feature/filter ; https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS ; https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016
- **cross-links**: foundation; → window-functions, select-and-query-processing, standard-vs-dialect-map.
- **dimensions**: TC, HX.

---

### 6. sql-set-operations
- **type**: technique
- **one-line**: `UNION [ALL]`/`INTERSECT [ALL]`/`EXCEPT [ALL]` semantics and `VALUES` as a table constructor.
- **draft description**: Guides set operations and the default-to-`UNION` cost trap — `UNION` sorts and de-duplicates (expensive); use `UNION ALL` when duplicates are impossible or wanted. Covers `INTERSECT`/`EXCEPT` (so they aren't reinvented with joins/`NOT EXISTS`), column-count/type matching, where `ORDER BY` is legal (only the final query), and that set ops treat NULLs as *not distinct* (opposite of `=`). Auto-invokes when writing or editing `UNION`/`UNION ALL`/`INTERSECT`/`EXCEPT`, compound `SELECT`s, or `VALUES` row-set constructors.
- **scope (IN)**: `UNION`/`UNION ALL` and when each is correct; `INTERSECT`/`EXCEPT` (`[ALL]` multiset semantics); NULL-as-not-distinct in set ops; column alignment; `ORDER BY`/`FETCH` placement on compound queries; `VALUES` as a standalone derived table.
- **scope boundaries (OUT)**: anti-join via `NOT EXISTS` → `sql-subqueries-and-exists`; `MINUS` (Oracle) / MySQL version gaps → `sql-standard-vs-dialect-map`; NULL equality theory → foundation.
- **primary sources**: https://www.postgresql.org/docs/current/queries-union.html ; https://www.sqlite.org/lang_select.html#compound_select_statements ; https://www.postgresql.org/docs/current/sql-values.html
- **cross-links**: foundation; → subqueries-and-exists, pagination-and-keyset (VALUES bulk rows), standard-vs-dialect-map.
- **dimensions**: TC, HX.

---

### 7. sql-window-functions
- **type**: technique
- **one-line**: `OVER`/`PARTITION BY`/frames, ranking, offset, and the default-frame `LAST_VALUE` surprise.
- **draft description**: Guides window functions — the highest-leverage modern-SQL skill — and the frame-clause traps LLMs botch. The default frame with `ORDER BY` is `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`, which makes `LAST_VALUE` return the current row, not the partition's last; running totals need explicit `ROWS` vs `RANGE` when sort keys tie. Covers `ROW_NUMBER`/`RANK`/`DENSE_RANK`/`NTILE`, `LAG`/`LEAD`/`FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`, aggregate windows, named windows, `EXCLUDE`/`GROUPS`, and that you cannot filter a window result in `WHERE` (wrap in a CTE/subquery). Auto-invokes when writing or editing `OVER (...)`, `PARTITION BY`, frame clauses (`ROWS`/`RANGE`/`GROUPS`), ranking/offset functions, running totals, or "top-N per group" / "filter on row_number" requests.
- **scope (IN)**: `OVER`, `PARTITION BY`, window `ORDER BY`, frame units & defaults, `EXCLUDE`; all ranking/offset/value functions; aggregate-as-window; named `WINDOW` clause; window-in-WHERE → wrap pattern; running/moving aggregates.
- **scope boundaries (OUT)**: the consecutive-runs recipe → `sql-gaps-and-islands`; regex-over-rows → `sql-match-recognize`; ordered-set aggregates → `sql-aggregation-and-grouping`; `LATERAL` top-N alternative → `sql-lateral-and-correlated-derived`; index support for window `ORDER BY` → `sql-indexing-and-sargability`.
- **primary sources**: https://www.postgresql.org/docs/current/tutorial-window.html ; https://www.postgresql.org/docs/current/functions-window.html ; https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS ; https://www.sqlite.org/windowfunctions.html ; https://modern-sql.com/feature/over
- **cross-links**: foundation; → gaps-and-islands, match-recognize, aggregation-and-grouping, lateral-and-correlated-derived, pagination-and-keyset.
- **dimensions**: TC (primary), HX, TR.

---

### 8. sql-cte-and-recursion
- **type**: technique
- **one-line**: `WITH` for decomposition and `WITH RECURSIVE` for hierarchies/graphs/series with cycle guards.
- **draft description**: Guides common table expressions for readable query decomposition and `WITH RECURSIVE` for hierarchies, transitive closure, and series generation — always with a termination/cycle guard so recursion can't run away. Covers the anchor + recursive-member `UNION ALL` structure, the SQL:2023 `SEARCH`/`CYCLE` clauses, and the materialization-fence divergence (Postgres <=11 always materializes; >=12 can inline). Steers away from procedural loops where a set-based recursive CTE is the portable answer. Auto-invokes when writing or editing `WITH`/`WITH RECURSIVE`, hierarchy/tree/graph traversal queries, transitive-closure or generate-series logic, or on "infinite recursion" / "tree query" requests.
- **scope (IN)**: non-recursive `WITH` for readability; recursive structure & termination; cycle detection (`CYCLE`/`SEARCH` and manual path-array guards); series generation; materialization-fence behavior.
- **scope boundaries (OUT)**: adjacency-vs-nested-set tree *modeling* → `sql-schema-design-and-normalization`; window functions used inside CTEs → `sql-window-functions`; CTE plan/perf → `sql-explain-and-set-based-thinking`; `SEARCH`/`CYCLE` availability → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/queries-with.html ; https://www.sqlite.org/lang_with.html ; https://modern-sql.com/feature/with ; Eisentraut SQL:2023 cycle marks https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new
- **cross-links**: foundation; → window-functions, schema-design-and-normalization, explain-and-set-based-thinking, standard-vs-dialect-map.
- **dimensions**: TC, HX (primary — decomposition/readability).

---

### 9. sql-lateral-and-correlated-derived
- **type**: technique
- **one-line**: `LATERAL` joins / correlated derived tables for top-N-per-group and per-row expansion.
- **draft description**: Guides `LATERAL` — a derived table in `FROM` that may reference earlier `FROM` items — the clean answer to top-N-per-group and per-row function expansion that LLMs otherwise reinvent with window-filter gymnastics or N correlated scalar subqueries in the `SELECT` list. Covers keyword placement, implicit `LATERAL` for set-returning functions (Postgres), and the non-standard `CROSS/OUTER APPLY` spelling (SQL Server). Auto-invokes when writing or editing `LATERAL` / `CROSS APPLY` / `OUTER APPLY`, top-N-per-group queries, per-row table-function expansion, or queries with one correlated scalar subquery per output column.
- **scope (IN)**: `LATERAL` semantics & placement; top-N-per-group via `LATERAL` + `FETCH`; set-returning-function expansion; `APPLY` spelling; comparison to window-function and correlated-subquery approaches.
- **scope boundaries (OUT)**: window top-N tradeoff → `sql-window-functions`; correlated subqueries in `WHERE`/`SELECT` → `sql-subqueries-and-exists`; SQLite absence / `APPLY` mapping → `sql-standard-vs-dialect-map`.
- **primary sources**: https://modern-sql.com/feature/lateral ; https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL ; https://www.postgresql.org/docs/current/sql-select.html
- **cross-links**: foundation; → window-functions, subqueries-and-exists, joins, standard-vs-dialect-map.
- **dimensions**: TC, HX.

---

### 10. sql-expressions-case-and-functions
- **type**: reference/technique
- **one-line**: `CASE`, `COALESCE`/`NULLIF`/`GREATEST`/`LEAST`, standard string functions, and `CAST`.
- **draft description**: Guides portable scalar expressions — use standard `CASE`, `COALESCE`, `NULLIF`, `||` concatenation, `SUBSTRING(... FROM ... FOR ...)`, `TRIM`, `POSITION`, `OVERLAY`, and `CAST` instead of vendor spellings (`IFNULL`/`ISNULL`/`NVL`/`IIF`, `SUBSTR`, `LEFT`/`RIGHT`, `+` for string concat). Warns that `||` (and arithmetic) yields NULL if any operand is NULL, that implicit coercion differs per engine, and covers the SQL:2023 additions `GREATEST`/`LEAST`/`LPAD`/`RPAD`/multi-char `TRIM`. Auto-invokes when writing or editing `CASE`, `COALESCE`/`NULLIF`, string functions, type conversions/`CAST`, or string concatenation.
- **scope (IN)**: searched & simple `CASE`; `COALESCE`/`NULLIF`/`GREATEST`/`LEAST`; standard string functions & `||`; `CAST` and explicit-vs-implicit conversion; NULL propagation through expressions.
- **scope boundaries (OUT)**: date/time functions & `EXTRACT` → `sql-datetime-and-intervals`; numeric type/precision → `sql-data-types-and-numerics`; vendor function map → `sql-standard-vs-dialect-map`; JSON functions → `sql-json`.
- **primary sources**: https://www.postgresql.org/docs/current/functions-string.html ; https://www.postgresql.org/docs/current/functions-conditional.html ; https://modern-sql.com/standard/2023
- **cross-links**: foundation; → datetime-and-intervals, data-types-and-numerics, standard-vs-dialect-map.
- **dimensions**: TC, HX.

---

### 11. sql-datetime-and-intervals
- **type**: technique
- **one-line**: Standard temporal types, `EXTRACT`, and `INTERVAL` arithmetic vs fragile string date math.
- **draft description**: Guides standard temporal handling — `DATE`, `TIME`/`TIMESTAMP [WITH TIME ZONE]`, `INTERVAL`, typed literals (`DATE '...'`, `TIMESTAMP '...'`), `EXTRACT`, and interval arithmetic — instead of jamming dates into strings or using vendor functions (`DATEADD`/`DATEDIFF`/`STRFTIME`/`NOW()`). Stresses the `TIMESTAMP` vs `TIMESTAMP WITH TIME ZONE` distinction and the high implementation divergence (SQLite has no native date type; MySQL lacks a stored `INTERVAL`). Auto-invokes when writing or editing date/time columns or literals, `EXTRACT`, `INTERVAL` arithmetic, time-zone-sensitive comparisons, or date-bucketing/age calculations.
- **scope (IN)**: temporal types & time-zone semantics; typed literals; `EXTRACT`/`CURRENT_*`; `INTERVAL` arithmetic; truncation/bucketing portably; `CAST`/`FORMAT` templating.
- **scope boundaries (OUT)**: general `CAST` → `sql-expressions-case-and-functions`; numeric precision → `sql-data-types-and-numerics`; system/valid-time history → `sql-temporal-tables`; per-engine function spellings → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/datatype-datetime.html ; https://www.postgresql.org/docs/current/functions-datetime.html ; https://www.sqlite.org/lang_datefunc.html
- **cross-links**: foundation; → expressions-case-and-functions, data-types-and-numerics, temporal-tables, standard-vs-dialect-map.
- **dimensions**: TC (primary), PI.

---

### 12. sql-data-types-and-numerics
- **type**: reference
- **one-line**: The standard type system and choosing exact vs approximate numerics.
- **draft description**: Guides type selection — exact `NUMERIC`/`DECIMAL(p,s)` for money and counts (never `FLOAT`/`DOUBLE`, which carry rounding error), `INTEGER`/`SMALLINT`/`BIGINT` by range, `CHARACTER VARYING` without cargo-culted `VARCHAR(255)`, `BOOLEAN` (and that SQL Server has only `BIT`, older MySQL maps to `TINYINT`), and `BLOB`/`CLOB`. Covers precision/scale, overflow semantics, and SQLite's dynamic-typing deviation. Auto-invokes when writing or editing column type declarations, `CAST` targets, money/currency or high-precision fields, or `CREATE TABLE` type choices.
- **scope (IN)**: exact vs approximate numerics; precision/scale & overflow; integer sizing; character/`VARCHAR` sizing; `BOOLEAN` portability; `BLOB`/`CLOB`; SQLite flexible typing.
- **scope boundaries (OUT)**: temporal types → `sql-datetime-and-intervals`; identity/generated columns → `sql-generated-and-identity-columns`; JSON type → `sql-json`; `CAST` expression usage → `sql-expressions-case-and-functions`; `BOOLEAN`/`BIT` map → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/datatype.html ; https://www.postgresql.org/docs/current/datatype-numeric.html ; https://www.sqlite.org/datatype3.html
- **cross-links**: foundation; → datetime-and-intervals, generated-and-identity-columns, constraints-and-integrity, standard-vs-dialect-map.
- **dimensions**: TC (primary), HX.

---

### 13. sql-constraints-and-integrity
- **type**: technique/policy
- **one-line**: `PRIMARY KEY`/`UNIQUE`/`FOREIGN KEY`/`CHECK`/`NOT NULL`/`DEFAULT` and referential actions.
- **draft description**: Guides database-enforced integrity — declare `FOREIGN KEY`s with deliberate referential actions (`ON DELETE/UPDATE CASCADE|SET NULL|SET DEFAULT|RESTRICT|NO ACTION`), choose `PRIMARY KEY` vs `UNIQUE`, and remember that `UNIQUE` permits multiple NULLs (use `NULLS NOT DISTINCT`, SQL:2023) and that a `CHECK` passes when it evaluates to UNKNOWN (NULL). Pushes integrity into the schema rather than app code, and covers deferrable constraints for circular references. Auto-invokes when writing or editing `CREATE TABLE`/`ALTER TABLE` constraints, foreign keys, `CHECK`/`UNIQUE`/`NOT NULL`/`DEFAULT`, or on "should this be enforced in the DB or the app" decisions.
- **scope (IN)**: all constraint types; referential actions; `UNIQUE` + NULL semantics & `NULLS [NOT] DISTINCT`; `CHECK` and the UNKNOWN-passes pitfall; `NOT NULL`/`DEFAULT`; deferrable constraints; SQLite `PRAGMA foreign_keys` caveat.
- **scope boundaries (OUT)**: NULL/`UNIQUE` theory → foundation; identity/generated columns → `sql-generated-and-identity-columns`; normalization/key-choice rationale → `sql-schema-design-and-normalization`; type choice → `sql-data-types-and-numerics`; FK-enforcement dialect quirks → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/ddl-constraints.html ; https://www.sqlite.org/foreignkeys.html ; https://www.sqlite.org/lang_createtable.html ; Eisentraut SQL:2023 F292 https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new
- **cross-links**: foundation; → schema-design-and-normalization, generated-and-identity-columns, data-types-and-numerics, standard-vs-dialect-map.
- **dimensions**: TC (primary), PI.

---

### 14. sql-injection-and-parameterization
- **type**: policy
- **one-line**: Parameterized queries as the only correct way to combine SQL with data — highest-stakes skill.
- **draft description**: Guides the one non-negotiable rule for combining SQL with untrusted data — use bind parameters / prepared statements, never string interpolation, concatenation, or manual escaping. Covers why placeholders are the fix (and that identifiers cannot be parameterized — allowlist them), and that placeholder syntax (`?`, `$1`, `:name`, `%s`) is driver-specific, not a dialect feature. Treats "just for this example" interpolation of user data as a defect. Auto-invokes when writing or editing any query that embeds a variable/user input, string-building of SQL, dynamic `WHERE`/`ORDER BY`/table names, ORM raw-query escape hatches, or on "build this query from input" / "is this safe from injection" requests.
- **scope (IN)**: parameterization principle; placeholder styles (driver-specific, noted not taught as dialect); identifier allowlisting; dynamic SQL done safely; why escaping/quoting is not a fix; `PREPARE`/`EXECUTE` standard surface.
- **scope boundaries (OUT)**: privileges/least-privilege → `sql-privileges-and-access-control`; transaction wrapping → `sql-transactions-and-isolation`; placeholder-spelling table → `sql-standard-vs-dialect-map`; ORM/driver pooling specifics → out of scope (app layer).
- **primary sources**: https://owasp.org/www-community/attacks/SQL_Injection ; https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html ; https://www.postgresql.org/docs/current/sql-prepare.html ; Karwin "SQL Injection" https://pragprog.com/titles/bksqla/sql-antipatterns/
- **cross-links**: foundation; → privileges-and-access-control, transactions-and-isolation, standard-vs-dialect-map.
- **dimensions**: TC, PI (primary — app integration), HX.

---

### 15. sql-merge-and-upsert
- **type**: pattern
- **one-line**: Atomic upsert via `MERGE` and the `ON CONFLICT`/`ON DUPLICATE KEY` portability landscape.
- **draft description**: Guides atomic upsert — never a race-prone "SELECT then INSERT or UPDATE" in application code. Teaches the standard `MERGE` statement (`WHEN MATCHED` / `WHEN NOT MATCHED [BY SOURCE]`) and maps the real-world alternatives (`INSERT ... ON CONFLICT` in Postgres/SQLite, `ON DUPLICATE KEY UPDATE` in MySQL), since `MERGE` is unevenly adopted (Postgres only since v15, no `MERGE` in MySQL/SQLite). Flags `MERGE` concurrency footguns — it is not a substitute for a proper `UNIQUE` constraint plus conflict handling. Auto-invokes when writing or editing `MERGE`, `INSERT ... ON CONFLICT`/`ON DUPLICATE KEY`, upsert/"insert-or-update" logic, or check-then-act read-modify-write code.
- **scope (IN)**: `MERGE` syntax & match arms; `ON CONFLICT`/`ON DUPLICATE KEY` mapping; atomicity vs app-side select-then-write; unique-constraint dependence; concurrency caveats.
- **scope boundaries (OUT)**: isolation-level interaction with concurrent upserts → `sql-transactions-and-isolation` (and MVCC plugin for theory); unique constraints → `sql-constraints-and-integrity`; full divergence table → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/sql-merge.html ; https://www.sqlite.org/lang_upsert.html ; https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html ; https://en.wikipedia.org/wiki/Merge_(SQL)
- **cross-links**: foundation; → constraints-and-integrity, transactions-and-isolation, standard-vs-dialect-map.
- **dimensions**: TC (primary — correctness/atomicity), PI.

---

### 16. sql-pagination-and-keyset  (absorbs row-values-and-comparisons)
- **type**: pattern
- **one-line**: Standard `OFFSET/FETCH`, performant keyset/seek pagination, and row-value comparisons.
- **draft description**: Guides correct, scalable pagination — prefer keyset/seek pagination (`WHERE (sort_key, id) > (:last_key, :last_id) ORDER BY sort_key, id FETCH FIRST n ROWS ONLY`) over `LIMIT n OFFSET m`, which is non-standard, scans-and-discards `m` rows (O(offset)), and skips or duplicates rows when data shifts between pages. Teaches the standard `OFFSET ... FETCH FIRST n ROWS ONLY [WITH TIES]` surface and the **row value constructor / row comparison** machinery behind keyset (`(a,b) > (x,y)` and its lexicographic/NULL semantics) instead of verbose, often-wrong `(a > x) OR (a = x AND b > y)` expansions; also covers `VALUES` row-lists for set-based bulk operations. Auto-invokes when writing or editing pagination/`LIMIT`/`OFFSET`/`FETCH` queries, infinite-scroll or cursor endpoints, multi-column `ORDER BY` with paging, row-value comparisons, or `VALUES` bulk row sets.
- **scope (IN)**: `OFFSET ... FETCH FIRST ... [WITH TIES]`; why `OFFSET` degrades & is unstable; keyset/seek pattern; **row value constructors & row comparisons** (lexicographic + NULL behavior); multi-row `IN ((..),(..))`; `VALUES` row-lists for bulk insert/join; stable tiebreak ordering.
- **scope boundaries (OUT)**: index support that makes keyset O(1)-per-page → `sql-indexing-and-sargability`; window functions for ranked paging → `sql-window-functions`; NULL comparison theory → foundation; `LIMIT` vs `FETCH` dialect table & SQLite row-comparison gaps → `sql-standard-vs-dialect-map`.
- **primary sources**: https://modern-sql.com/feature/fetch-first ; https://use-the-index-luke.com/no-offset ; https://use-the-index-luke.com/sql/partial-results ; https://www.postgresql.org/docs/current/functions-comparisons.html#ROW-WISE-COMPARISON ; https://www.postgresql.org/docs/current/sql-values.html
- **cross-links**: foundation; → indexing-and-sargability, window-functions, select-and-query-processing, set-operations, standard-vs-dialect-map.
- **dimensions**: TC, HX, PI (primary — app integration).

---

### 17. sql-generated-and-identity-columns
- **type**: technique
- **one-line**: `GENERATED ALWAYS AS (...)` computed columns and `GENERATED AS IDENTITY` surrogate keys.
- **draft description**: Guides the standard forms — `GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY` for surrogate keys instead of vendor `SERIAL`/`AUTO_INCREMENT`/`IDENTITY(1,1)`, and `GENERATED ALWAYS AS (expr) STORED|VIRTUAL` computed columns instead of recomputing in every query or in app code. Clarifies `STORED` vs `VIRTUAL` and `ALWAYS` (rejects explicit inserts) vs `BY DEFAULT` (allows them). Auto-invokes when writing or editing auto-increment/surrogate-key columns, `SERIAL`/`AUTO_INCREMENT`, computed/derived columns, or `GENERATED` clauses in `CREATE TABLE`.
- **scope (IN)**: `GENERATED AS IDENTITY` (`ALWAYS` vs `BY DEFAULT`); computed columns `STORED` vs `VIRTUAL`; `SERIAL`/`AUTO_INCREMENT` as legacy/non-standard; sequence interaction at a high level.
- **scope boundaries (OUT)**: PK/UNIQUE constraint rules → `sql-constraints-and-integrity`; type choice → `sql-data-types-and-numerics`; `SERIAL`/`AUTO_INCREMENT` mapping → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/ddl-generated-columns.html ; https://www.postgresql.org/docs/current/ddl-identity-columns.html ; https://www.sqlite.org/gencol.html ; https://modern-sql.com/feature/generated-columns
- **cross-links**: foundation; → constraints-and-integrity, data-types-and-numerics, standard-vs-dialect-map.
- **dimensions**: TC, HX, PI.

---

### 18. sql-views-and-introspection
- **type**: technique
- **one-line**: `CREATE VIEW`, `WITH CHECK OPTION`, and portable `INFORMATION_SCHEMA` introspection.
- **draft description**: Guides views and portable schema introspection — query `INFORMATION_SCHEMA` rather than vendor catalogs (`pg_catalog`, `sqlite_master`, `SHOW TABLES`) where portability matters, and use `WITH [LOCAL|CASCADED] CHECK OPTION` so rows inserted/updated through a view can't vanish from it. Covers updatable-view rules and that SQLite has no `INFORMATION_SCHEMA` (uses `sqlite_master`/`PRAGMA`). Auto-invokes when writing or editing `CREATE VIEW`, querying catalog/metadata, `INFORMATION_SCHEMA`/`SHOW`/`PRAGMA`, or generating schema-discovery/migration tooling queries.
- **scope (IN)**: `CREATE VIEW`; updatable views; `WITH CHECK OPTION`; `INFORMATION_SCHEMA` core tables; portability of introspection.
- **scope boundaries (OUT)**: materialized-view refresh (vendor) → out of scope / dialect map note; constraint introspection detail → `sql-constraints-and-integrity`; SQLite catalog & `SHOW`/`PRAGMA` map → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/sql-createview.html ; https://www.postgresql.org/docs/current/information-schema.html ; https://en.wikipedia.org/wiki/Information_schema ; https://www.sqlite.org/schematab.html
- **cross-links**: foundation; → constraints-and-integrity, privileges-and-access-control, standard-vs-dialect-map.
- **dimensions**: PI (primary), TC, HX.

---

### 19. sql-json
- **type**: technique
- **one-line**: SQL/JSON (2016/2023): constructors, `JSON_VALUE`/`JSON_QUERY`/`JSON_TABLE`, path language, `JSON` type.
- **draft description**: Guides standard SQL/JSON instead of reflexive vendor operators (`->`, `->>`, `#>>`, `JSON_EXTRACT`, `::jsonb`) — constructors (`JSON_OBJECT`/`JSON_ARRAY`/`JSON_OBJECTAGG`/`JSON_ARRAYAGG`), query functions (`JSON_VALUE` for scalars vs `JSON_QUERY` for objects/arrays vs `JSON_EXISTS`), `JSON_TABLE` to shred JSON into rows, the SQL/JSON path language (lax vs strict, filters, item methods), `IS JSON`, and the SQL:2023 `JSON` type. Warns against storing JSON where a normalized schema belongs (the modern EAV antipattern). Auto-invokes when writing or editing JSON columns/queries, `->`/`->>`/`JSON_EXTRACT`, `JSON_VALUE`/`JSON_QUERY`/`JSON_TABLE`, JSON path expressions, or JSON aggregation.
- **scope (IN)**: constructors; `JSON_VALUE`/`JSON_QUERY`/`JSON_EXISTS`; `JSON_TABLE`; path language (lax/strict, `ON ERROR`/`ON EMPTY`, item methods); `IS JSON`; SQL:2023 `JSON` type; JSON-vs-normalized-schema judgment.
- **scope boundaries (OUT)**: `jsonb` GIN indexing → vendor plugin (note + route to dialect map); EAV/normalization design depth → `sql-schema-design-and-normalization`; standard-vs-vendor operator table → `sql-standard-vs-dialect-map`.
- **primary sources**: https://modern-sql.com/feature/json_table ; https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016 ; https://www.postgresql.org/docs/current/functions-json.html ; https://dev.mysql.com/doc/refman/8.4/en/json.html ; https://www.sqlite.org/json1.html
- **cross-links**: foundation; → schema-design-and-normalization, standard-vs-dialect-map.
- **dimensions**: TC, HX, PI.

---

### 20. sql-gaps-and-islands
- **type**: pattern (cookbook)
- **one-line**: The consecutive-runs / gaps-between-runs pattern family, solved portably with window functions.
- **draft description**: Guides the canonical "gaps and islands" pattern family — detecting consecutive runs (active streaks, sessionization, contiguous-range collapsing) and the gaps between them — with the portable set-based solution (the `ROW_NUMBER`-difference trick and friends) instead of brittle self-joins or procedural loops. Points to `MATCH_RECOGNIZE` as the cleaner standard tool where engines support it. Auto-invokes when writing or editing queries for consecutive sequences, streaks/runs, sessionization, collapsing adjacent date/number ranges, or "group consecutive rows" / "find gaps" requests.
- **scope (IN)**: islands via `ROW_NUMBER` difference; gaps via `LEAD`/`LAG`; range collapsing; sessionization with a time threshold; when to escalate to `MATCH_RECOGNIZE`.
- **scope boundaries (OUT)**: window-function mechanics/frames → `sql-window-functions`; regex-over-rows engine support → `sql-match-recognize`; index support for the ordering → `sql-indexing-and-sargability`.
- **primary sources**: Itzik Ben-Gan gaps-and-islands canon (itprotoday.com) ; Celko "SQL for Smarties" runs-and-sequences ; https://modern-sql.com/feature/match_recognize ; https://use-the-index-luke.com/sql/sorting-grouping
- **cross-links**: foundation; → window-functions, match-recognize, indexing-and-sargability.
- **dimensions**: TC, HX, TR.

---

### 21. sql-schema-design-and-normalization
- **type**: pattern/policy
- **one-line**: Normalization 1NF→BCNF, key choice, deliberate denormalization, and structural antipatterns.
- **draft description**: Guides portable schema-design discipline — normalize to remove update anomalies (1NF→BCNF), choose natural vs surrogate keys deliberately, denormalize only with a stated reason, and avoid the structural antipatterns: EAV "flexible" schemas, comma-separated value lists ("Jaywalking"), adjacency-only trees that can't query subtrees, and polymorphic/promiscuous foreign keys. Auto-invokes when designing tables/schemas, writing `CREATE TABLE` for a new model, modeling hierarchies or many-to-many relationships, or on "is this schema/data model good" reviews.
- **scope (IN)**: normal forms & the anomalies they prevent; key selection; when to denormalize; EAV / Jaywalking / Naive Trees (adjacency vs nested-set vs closure-table) / Polymorphic Associations; junction tables for M:N.
- **scope boundaries (OUT)**: constraint syntax that enforces the design → `sql-constraints-and-integrity`; recursive queries over trees → `sql-cte-and-recursion`; JSON-vs-relational tradeoff detail → `sql-json`; index design → `sql-indexing-and-sargability`.
- **primary sources**: Karwin "SQL Antipatterns" https://pragprog.com/titles/bksqla/sql-antipatterns/ ; https://en.wikipedia.org/wiki/Database_normalization ; https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form ; Codd 1970 https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf
- **cross-links**: foundation; → constraints-and-integrity, cte-and-recursion, json, indexing-and-sargability.
- **dimensions**: HX (primary), TC, PI.

---

### 22. sql-style-and-naming
- **type**: policy
- **one-line**: Formatting, casing, and the `'literal'` vs `"identifier"` quoting rule for reviewable SQL.
- **draft description**: Guides readable, reviewable SQL and the genuine correctness trap in quoting — `'single quotes'` are string literals and `"double quotes"` are delimited identifiers per the standard (MySQL's lax default hides the bug until `ANSI_QUOTES` flips it). Covers keyword casing, identifier naming/quoting (snake_case so columns never need permanent quoting), leading-vs-trailing commas, and consistent layout instead of single-line mega-queries. Auto-invokes when writing or editing SQL with string vs identifier quoting, identifier naming, query formatting/layout, or on "clean up / format this SQL" requests.
- **scope (IN)**: literal vs identifier quoting (correctness + portability); keyword casing; naming conventions & when quoting is forced; comma style; multi-line layout for joins/CTEs/predicates.
- **scope boundaries (OUT)**: clause logical order → `sql-select-and-query-processing`; `ANSI_QUOTES`/identifier-quote dialect detail → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/sql-syntax-lexical.html ; https://www.sqlstyle.guide/ ; https://modern-sql.com (string vs identifier)
- **cross-links**: foundation; → select-and-query-processing, standard-vs-dialect-map.
- **dimensions**: HX (primary), TC, PI.

---

### 23. sql-transactions-and-isolation  ← MVCC BOUNDARY
- **type**: technique
- **one-line**: The SQL surface of transactions — statements, savepoints, isolation-level names; routes theory to MVCC.
- **draft description**: Guides the SQL-statement side of transactions — wrap multi-statement invariants and check-then-act sequences in `START TRANSACTION`/`COMMIT`/`ROLLBACK`, use `SAVEPOINT`/`ROLLBACK TO` for partial rollback, set isolation with `SET TRANSACTION ISOLATION LEVEL`, and know the four standard level names, autocommit, read-only/deferrable, and that DDL is not transactional everywhere (MySQL auto-commits DDL). States the anomaly catalog in one paragraph and routes the theory (which anomalies each level permits, serializability, write skew, snapshot mechanics) to the MVCC plugin. Auto-invokes when writing or editing `BEGIN`/`COMMIT`/`ROLLBACK`/`SAVEPOINT`, `SET TRANSACTION`, multi-statement write sequences, or on "should this be in a transaction" / "which isolation level" requests.
- **scope (IN)**: transaction control statements; `SAVEPOINT`; isolation-level *names* & how to set them; autocommit; read-only/deferrable; DDL-in-transaction caveats; *when to open a transaction*; <=1 paragraph anomaly summary.
- **scope boundaries (OUT)** — **HARD**: anomaly catalog/theory, snapshot isolation, SSI/serializable mechanics, "forbidding 3 phenomena ≠ serializable", write skew, choosing a level by anomaly → `mvcc-isolation-levels-and-anomalies` / `mvcc-snapshot-isolation` / `mvcc-serializable-ssi` / `mvcc-choosing-isolation` in `~/code/mvcc-skills-plugin`. Upsert concurrency → `sql-merge-and-upsert`. Default-level differences table → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/tutorial-transactions.html ; https://www.postgresql.org/docs/current/sql-set-transaction.html ; https://www.postgresql.org/docs/current/transaction-iso.html ; https://www.sqlite.org/lang_transaction.html
- **cross-links**: foundation; → merge-and-upsert, privileges-and-access-control, standard-vs-dialect-map; **MVCC**: mvcc-isolation-levels-and-anomalies (+ siblings).
- **dimensions**: TC, PI.

---

### 24. sql-privileges-and-access-control
- **type**: policy
- **one-line**: The standard security model — `GRANT`/`REVOKE`, roles, least privilege.
- **draft description**: Guides the standard access-control model — `GRANT`/`REVOKE`, object vs system privileges, roles, and `WITH GRANT OPTION` — toward least privilege: the application connects as a role that is not the owner/superuser and holds only the privileges it needs, never `ALL` granted broadly. Notes that privilege names vary and that SQLite has no user/privilege model (file-permission based). Auto-invokes when writing or editing `GRANT`/`REVOKE`/`CREATE ROLE`, designing DB user/role setup, connection-credential decisions, or on "least privilege" / "who can access" requests.
- **scope (IN)**: `GRANT`/`REVOKE`; object vs system privileges; roles & role hierarchies; `WITH GRANT OPTION`; ownership vs grants; least-privilege app-role design.
- **scope boundaries (OUT)**: injection prevention → `sql-injection-and-parameterization`; row-level security (vendor) → out of scope / dialect note; view-based access → `sql-views-and-introspection`; SQLite/dialect privilege-name map → `sql-standard-vs-dialect-map`.
- **primary sources**: https://www.postgresql.org/docs/current/sql-grant.html ; https://www.postgresql.org/docs/current/ddl-priv.html ; https://www.postgresql.org/docs/current/user-manag.html
- **cross-links**: foundation; → injection-and-parameterization, views-and-introspection, standard-vs-dialect-map.
- **dimensions**: PI (primary), TC.

---

### 25. sql-indexing-and-sargability
- **type**: technique
- **one-line**: Portable indexing — B-tree mechanics, composite column order, covering indexes, sargability.
- **draft description**: Guides portable index design and the sargability rule — never wrap an indexed column in a function or expression (`WHERE LOWER(email)=...`, `WHERE DATE(ts)=...`, `WHERE col + 0 = ...`) or lead a `LIKE` with `%`, because it defeats the index. Covers B-tree mechanics, composite-index column order ("equality columns first, then the range/sort column"), covering indexes, that one index can serve `WHERE` + `ORDER BY` + `GROUP BY`, and the "index shotgun" antipattern of redundant single-column indexes. The highest-leverage performance skill, taught vendor-neutrally (Winand). Auto-invokes when writing or editing `CREATE INDEX`, slow `WHERE`/`ORDER BY`/`JOIN` predicates, functions applied to filtered columns, `LIKE` patterns, or on "why is this query slow" / "what index do I need" requests.
- **scope (IN)**: B-tree anatomy; sargable vs non-sargable predicates; composite column order; covering/included columns; index-supported ordering & grouping; index shotgun; DML cost of indexes.
- **scope boundaries (OUT)**: reading a plan → `sql-explain-and-set-based-thinking`; keyset pagination usage → `sql-pagination-and-keyset`; engine-specific index types (GIN/hash/bitmap) → vendor plugins / dialect note; join algorithms → `sql-explain-and-set-based-thinking`.
- **primary sources**: https://use-the-index-luke.com/sql/where-clause ; https://use-the-index-luke.com/sql/anatomy ; https://use-the-index-luke.com/sql/clustering ; https://use-the-index-luke.com/sql/sorting-grouping ; https://www.sqlite.org/queryplanner.html
- **cross-links**: foundation; → explain-and-set-based-thinking, pagination-and-keyset, joins, standard-vs-dialect-map.
- **dimensions**: TC, TR, PI (primary — performance).

---

### 26. sql-explain-and-set-based-thinking
- **type**: technique
- **one-line**: Reading a query plan as a concept and replacing N+1/RBAR loops with set-based queries.
- **draft description**: Guides measuring instead of guessing — read a query plan conceptually (scan vs index access, join order, estimated vs actual rows, cardinality/selectivity) and think in sets, not rows. Replaces application-side per-row query loops (N+1 / "RBAR") with a single set-based query (`JOIN`/`LATERAL`/`VALUES`), and explains how a stale or under-estimated cardinality drives a bad plan. Auto-invokes when writing or editing per-row query loops in application code, `EXPLAIN`/`EXPLAIN QUERY PLAN` output interpretation, performance-tuning a query, or on "why is this slow" / "optimize this query" requests.
- **scope (IN)**: `EXPLAIN` as a concept; scan vs seek; join-order & cardinality/selectivity; estimated vs actual rows; N+1 detection & set-based rewrite; RBAR vs set-based thinking; Karwin "Spaghetti Query".
- **scope boundaries (OUT)**: index choice → `sql-indexing-and-sargability`; per-engine plan format/internals (buffers, InnoDB) → vendor plugins; set-based rewrite tools (`LATERAL`, window, CTE) → their own skills.
- **primary sources**: https://www.postgresql.org/docs/current/using-explain.html ; https://www.sqlite.org/eqp.html ; https://use-the-index-luke.com/sql/testing-scalability
- **cross-links**: foundation; → indexing-and-sargability, lateral-and-correlated-derived, window-functions, set-operations.
- **dimensions**: TR (primary — proving performance), TC, PI.

---

### 27. sql-standard-vs-dialect-map  ← CROSS-CUTTING ANCHOR
- **type**: reference
- **one-line**: The portability index — standard feature → which readable engines have it and how they spell it.
- **draft description**: A portability reference mapping each standard SQL feature to its support and spelling across the readable engines (Postgres, SQLite, MySQL/MariaDB, plus notes on SQL Server/Oracle/DuckDB): `LIMIT` vs `OFFSET/FETCH`, `SERIAL`/`AUTO_INCREMENT` vs `GENERATED AS IDENTITY`, `||` vs `+` vs `CONCAT`, `IS DISTINCT FROM` vs `<=>` vs `IS`, `MERGE` vs `ON CONFLICT` vs `ON DUPLICATE KEY`, `INFORMATION_SCHEMA` vs `sqlite_master`/`PRAGMA`/`SHOW`, `LISTAGG`/`STRING_AGG`/`GROUP_CONCAT`, `LATERAL` vs `APPLY`, JSON standard functions vs `->`/`->>`, identifier quoting & `ANSI_QUOTES`, and which advanced features (temporal, `MATCH_RECOGNIZE`, SQL/PGQ) live where. The SQL analogue of a version/feature map — other skills route here for "who spells it how." Auto-invokes when targeting a specific engine, porting SQL between databases, choosing between a standard and a vendor spelling, or on "does Postgres/MySQL/SQLite support X" / "is this portable" questions.
- **scope (IN)**: feature-by-feature support/spelling matrix; "standard says X; Postgres ✓ / SQLite ~ / MySQL ✗ (spells Y)" blocks; pointers from every other skill's portability note.
- **scope boundaries (OUT)**: the *teaching* of each feature → its own skill (this is an index, not a tutorial); deep engine internals → vendor plugins.
- **primary sources**: https://modern-sql.com/standard (+ /2011, /2016, /2023) and per-feature compatibility tables ; engine docs (Postgres/SQLite/MySQL/MariaDB/DuckDB) cited per row.
- **cross-links**: foundation; linked FROM every skill's portability note; → all feature skills.
- **dimensions**: PI (primary), TC, HX.

---

### 28. sql-temporal-tables  (advanced — LOW PORTABILITY)
- **type**: reference
- **one-line**: SQL:2011 system-versioned & application-time period tables and `FOR SYSTEM_TIME` queries.
- **draft description**: Guides SQL:2011 temporal tables — system-versioned tables (`PERIOD FOR SYSTEM_TIME`, `WITH SYSTEM VERSIONING`, `FOR SYSTEM_TIME AS OF / FROM..TO / BETWEEN`) for automatic audit history, application-time period tables (`PERIOD FOR`, `UPDATE/DELETE ... FOR PORTION OF`) for valid-time, and bitemporal modeling — instead of hand-rolled trigger-based history tables. Flags low portability: supported in MariaDB/SQL Server/DB2/Oracle, NOT in Postgres/SQLite/MySQL. Auto-invokes when writing or editing audit-history/"as-of"/point-in-time queries, system- or valid-time period tables, slowly-changing-dimension history, or "track every change" requests. Low-portability standard feature — confirm engine support before recommending.
- **scope (IN)**: system-versioned DDL & temporal query clauses; application-time periods & `FOR PORTION OF`; system-time vs valid-time vs bitemporal; portability caveats.
- **scope boundaries (OUT)** — **HARD**: versioning/storage mechanics, snapshot semantics, MVCC time-travel internals → `mvcc-time-travel-queries`. Engine support matrix → `sql-standard-vs-dialect-map`. Hand-rolled history without the standard feature → `sql-schema-design-and-normalization`.
- **primary sources**: https://en.wikipedia.org/wiki/SQL:2011 ; https://modern-sql.com/standard/2011 ; https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables ; https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables
- **cross-links**: foundation; → datetime-and-intervals, schema-design-and-normalization, standard-vs-dialect-map; **MVCC**: mvcc-time-travel-queries.
- **dimensions**: TC, PI.

---

### 29. sql-match-recognize  (advanced — LOW PORTABILITY)
- **type**: reference
- **one-line**: `MATCH_RECOGNIZE` (SQL:2016) — regex-over-rows pattern recognition.
- **draft description**: Guides `MATCH_RECOGNIZE` (SQL:2016) — regex-style pattern matching across ordered rows for time-series problems (V-shapes, sessionization, threshold breaches) — instead of convoluted self-joins or window-function gymnastics. Covers `PARTITION BY`/`ORDER BY`, `PATTERN` quantifiers (greedy vs reluctant), `DEFINE`, `MEASURES`, `ONE ROW`/`ALL ROWS PER MATCH`, `AFTER MATCH SKIP`, and `RUNNING`/`FINAL` semantics. Flags low portability: Oracle/Trino/Snowflake/Flink/Vertica/DB2 only, NOT Postgres/SQLite/MySQL/MariaDB. Auto-invokes when writing or editing time-series pattern detection, row-sequence matching, complex sessionization, or "find this shape/sequence in rows" requests. Low-portability standard feature — confirm engine support and route to `sql-gaps-and-islands` for the portable fallback.
- **scope (IN)**: `MATCH_RECOGNIZE` full clause set; pattern quantifiers; `MEASURES` & running/final; skip modes; comparison to window/gaps-and-islands.
- **scope boundaries (OUT)**: portable window-based runs → `sql-gaps-and-islands`; window-function basics → `sql-window-functions`; engine support → `sql-standard-vs-dialect-map`.
- **primary sources**: https://modern-sql.com/feature/match_recognize ; https://link.springer.com/article/10.1007/s13222-022-00404-3 ; https://trino.io/blog/2021/05/19/row_pattern_matching.html
- **cross-links**: foundation; → gaps-and-islands, window-functions, standard-vs-dialect-map.
- **dimensions**: TC.

---

### 30. sql-property-graph-queries  (bleeding-edge — LOW PORTABILITY)
- **type**: reference
- **one-line**: SQL:2023 SQL/PGQ — `CREATE PROPERTY GRAPH` and `GRAPH_TABLE (... MATCH ...)` graph patterns.
- **draft description**: Guides SQL:2023 SQL/PGQ (Part 16) — define a property graph over existing relational tables with `CREATE PROPERTY GRAPH` and query it with `GRAPH_TABLE (graph MATCH (a)-[e]->(b) COLUMNS (...))` ASCII-art patterns — and its relationship to the standalone GQL language. A forward-looking reference for "graph queries without a graph database." Flags very low portability: Oracle 23ai and experimental/extension support only as of 2026. Auto-invokes when writing or editing graph-pattern queries over relational data, `GRAPH_TABLE`/`CREATE PROPERTY GRAPH`, or "can I do graph/Cypher-style queries in SQL" requests. Prevents hallucinating Cypher/Gremlin syntax where SQL/PGQ is meant. Bleeding-edge standard feature — confirm engine support before recommending.
- **scope (IN)**: `CREATE PROPERTY GRAPH` over tables; `GRAPH_TABLE`/`MATCH`/`COLUMNS`; vertex/edge ASCII patterns; relationship to GQL; portability reality.
- **scope boundaries (OUT)**: dedicated graph DBs (Neo4j/Cypher, Gremlin) → out of scope; recursive-CTE graph traversal (the portable alternative) → `sql-cte-and-recursion`; engine support → `sql-standard-vs-dialect-map`.
- **primary sources**: https://modern-sql.com/standard/2023 ; https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq ; https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard ; https://arxiv.org/pdf/2409.01102
- **cross-links**: foundation; → cte-and-recursion, standard-vs-dialect-map.
- **dimensions**: TC.

---

## 3. Build order (waves & batches)

The foundation must be drafted first so every other skill can cross-link it. Within each wave, batches of 2-3 are independent and safe to draft in parallel. The dialect map drafts late (Wave 2) so it can index features the other skills have already nailed down, but its sources are known now and it does not block anything.

### Batch 0 — Foundation (solo, blocks nothing-can-start-before-it)
- `sql-relational-and-null-discipline`

### Wave 1 — Technical-correctness core (the daily 80% + highest-stakes)
- **Batch 1**: `sql-select-and-query-processing`, `sql-joins`, `sql-subqueries-and-exists`
- **Batch 2**: `sql-aggregation-and-grouping`, `sql-set-operations`, `sql-window-functions`
- **Batch 3**: `sql-cte-and-recursion`, `sql-lateral-and-correlated-derived`, `sql-expressions-case-and-functions`
- **Batch 4**: `sql-datetime-and-intervals`, `sql-data-types-and-numerics`, `sql-constraints-and-integrity`
- **Batch 5**: `sql-injection-and-parameterization`, `sql-merge-and-upsert`, `sql-pagination-and-keyset`

### Wave 2 — Human-experience / process / advanced
- **Batch 6**: `sql-generated-and-identity-columns`, `sql-views-and-introspection`, `sql-json`
- **Batch 7**: `sql-schema-design-and-normalization`, `sql-style-and-naming`, `sql-gaps-and-islands`
- **Batch 8**: `sql-transactions-and-isolation`, `sql-privileges-and-access-control`
- **Batch 9**: `sql-indexing-and-sargability`, `sql-explain-and-set-based-thinking`
- **Batch 10**: `sql-standard-vs-dialect-map`  (draft after Batches 1-9 so it can index their settled spellings)
- **Batch 11**: `sql-temporal-tables`, `sql-match-recognize`, `sql-property-graph-queries`

Highest-quality-bar skills (per research §6 — most LLM-error-prone, spend extra care): foundation, `sql-window-functions`, `sql-injection-and-parameterization`, `sql-joins`, `sql-indexing-and-sargability`, `sql-pagination-and-keyset`, `sql-merge-and-upsert`.

---

## 4. Open questions for the user

1. **Skill count = 30** (1 foundation + 25 core + dialect map + 3 advanced). This is the "comprehensive, more than golang's 25, all the modern shit" reading. It is ~3 above the soft ~25-27 target. OK to ship 30, or trim? (Easiest trims: defer the 3 low-portability advanced skills → 27; or fold `sql-explain-and-set-based-thinking` into `sql-indexing-and-sargability` → 29.)
2. **Row-values fold (decision #5).** Plan folds `sql-row-values-and-comparisons` into `sql-pagination-and-keyset`. Un-folding for maximal granularity → 31 skills. Keep folded (recommended) or split out?
3. **The 3 advanced low-portability skills are IN** per your instruction. Confirm you want them as full SKILL.md files (not a single survey) given Postgres/SQLite/MySQL — the most common readable engines — lack all three. (They're flagged low-portability in-description and route to the portable fallback / MVCC where relevant.)
4. **Output path**: `plugins/sql/skills/<name>/SKILL.md` with `references/{common-mistakes.md,sources.yaml}` per skill, matching golang. Confirm.
5. **MVCC boundary**: plan routes all transaction/temporal *theory* to `~/code/mvcc-skills-plugin` rather than duplicating. Confirm that plugin is the intended sibling and the cross-link targets (`mvcc-isolation-levels-and-anomalies`, `mvcc-time-travel-queries`, etc.) are stable names to link to.
6. **Stored procedures / triggers / PL-dialects**: intentionally OUT of scope (too divergent to be "standard"). A thin `sql-triggers-and-routines` *concept* skill is possible if you want it (would make 31/32). Default: leave out. Confirm.

---

## 5. Plugin metadata

### `.claude-plugin/marketplace.json`
```json
{
  "name": "sql-skills-marketplace",
  "owner": {
    "name": "Q"
  },
  "metadata": {
    "description": "Agent skills for writing correct, portable, modern standard SQL (ISO/IEC 9075)"
  },
  "plugins": [
    {
      "name": "sql",
      "source": "./plugins/sql",
      "description": "Standard-SQL code-quality skills — NULL/relational discipline, joins, window functions, CTEs, JSON, constraints, indexing, parameterization, and portable modern features"
    }
  ]
}
```

### `plugins/sql/.claude-plugin/plugin.json`
```json
{
  "name": "sql",
  "description": "Standard-SQL code-quality skills — relational & NULL discipline, joins, aggregation, window functions, CTEs & recursion, JSON, MERGE/upsert, keyset pagination, constraints & schema design, indexing & sargability, parameterization & access control, transactions, and modern SQL:2011/2016/2023 features. Vendor-neutral; portability mapped per feature.",
  "version": "0.1.0",
  "author": {
    "name": "Q"
  }
}
```

### Per-skill `references/`
Each skill directory carries:
- `references/common-mistakes.md` — the concrete WRONG→RIGHT pairs and the LLM-failure list distilled from this spec's "what LLMs get wrong" notes.
- `references/sources.yaml` — the `primary sources` URLs from this skill's spec block, in the golang `sources.yaml` shape (`name`/`url`/`type`/`accessed`/`sections_used`).

Frontmatter for every skill: `allowed-tools: Read, Glob, Grep` and `compatibility: "Claude Code, Codex CLI, Gemini CLI"`.
