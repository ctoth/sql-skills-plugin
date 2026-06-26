# Research — `sql-aggregation-and-grouping`

Primary-source research for the standard-SQL skill on aggregation and grouping. Each entry: URL,
section, verbatim quote, and why it matters for the skill. Accessed 2026-06-26.

---

## Source A — PostgreSQL: Aggregate Functions
URL: https://www.postgresql.org/docs/current/functions-aggregate.html

### A1. COUNT(*) vs COUNT(expression) — Table 9.62 General-Purpose Aggregate Functions
> `count ( * ) → bigint` — "Computes the number of input rows."
> `count ( "any" ) → bigint` — "Computes the number of input rows in which the input value is not null."

**Why it matters:** The single most common aggregate confusion. `COUNT(col)` silently undercounts vs
`COUNT(*)` whenever the column has NULLs. Skill §4 (COUNT variants). Deep NULL theory routes to
foundation `sql-relational-and-null-discipline` §7.

### A2. SUM/AVG/MIN/MAX skip NULL; SUM of no rows is NULL — §9.21 + Table 9.62
> `avg` — "Computes the average (arithmetic mean) of all the non-null input values."
> `sum` — "Computes the sum of the non-null input values."
> "It should be noted that except for `count`, these functions return a null value when no rows are
> selected. In particular, `sum` of no rows returns null, not zero as one might expect, and
> `array_agg` returns null rather than an empty array when there are no input rows."

**Why it matters:** Aggregates skip NULL rather than propagate it; SUM/AVG over zero rows is NULL not 0.
Skill §4 mentions, but routes the depth to foundation §7 to avoid duplication.

### A3. array_agg / string_agg — Table 9.62
> `array_agg ( anynonarray ORDER BY input_sort_columns ) → anyarray` — "Collects all the input values,
> including nulls, into an array."
> `string_agg ( value text, delimiter text ) → text` — "Concatenates the non-null input values into a
> string. Each value after the first is preceded by the corresponding `delimiter` (if it's not null)."

**Why it matters:** Ordered-set / list aggregation. Note the asymmetry: `array_agg` keeps nulls,
`string_agg` skips them. Skill §6.

### A4. Ordered-set aggregates & WITHIN GROUP — Table 9.64
> "shows some aggregate functions that use the _ordered-set aggregate_ syntax. These functions are
> sometimes referred to as 'inverse distribution' functions. Their aggregated input is introduced by
> `ORDER BY`, and they may also take a _direct argument_ that is not aggregated, but is computed only
> once."
> `mode () WITHIN GROUP ( ORDER BY anyelement )` — "Computes the _mode_, the most frequent value..."
> `percentile_cont ( fraction ) WITHIN GROUP ( ORDER BY double precision )` — "Computes the
> _continuous percentile_... This will interpolate between adjacent input items if needed."

**Why it matters:** WITHIN GROUP is the standard syntax for order-sensitive aggregates (percentiles,
mode, LISTAGG). Skill §6.

### A5. GROUPING() bit mask — Table 9.66 Grouping Operations
> "Returns a bit mask indicating which `GROUP BY` expressions are not included in the current grouping
> set. Bits are assigned with the rightmost argument corresponding to the least-significant bit; each
> bit is 0 if the corresponding expression is included in the grouping criteria of the grouping set
> generating the current result row, and 1 if it is not included."

**Why it matters:** GROUPING() is how you tell a NULL produced by a ROLLUP subtotal/grand-total row
apart from a real NULL in the data. Skill §5.

---

## Source B — PostgreSQL: Aggregate Expressions (syntax) §4.2.7
URL: https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES

### B1. DISTINCT in aggregates
> "The third form invokes the aggregate once for each distinct value of the expression (or distinct
> set of values, for multiple expressions) found in the input rows."

**Why it matters:** `COUNT(DISTINCT x)` counts distinct non-null values. Skill §4.

### B2. FILTER clause — definitive semantics
> "If `FILTER` is specified, then only the input rows for which the _filter_clause_ evaluates to true
> are fed to the aggregate function; other rows are discarded."

Example: `count(*) FILTER (WHERE i < 5)` returns 4 of 10 rows.

**Why it matters:** FILTER scopes one aggregate to a subset without touching the WHERE clause. Skill §3.

### B3. WITHIN GROUP placement
> "There is a subclass of aggregate functions called _ordered-set aggregates_ for which an
> _order_by_clause_ is _required_..."
> "For an ordered-set aggregate, the _order_by_clause_ is written inside `WITHIN GROUP (...)`..."

**Why it matters:** Confirms exact placement of ORDER BY inside WITHIN GROUP. Skill §6.

---

## Source C — modern-sql.com: The FILTER clause
URL: https://modern-sql.com/feature/filter

### C1. What it is + standard status
> "The `filter` clause extends aggregate functions (`sum`, `avg`, `count`, …) by an additional `where`
> clause."
> "SQL:2003 introduced the `filter` clause as part of the optional feature 'Advanced OLAP operations'
> (T612)."

**Why it matters:** FILTER is the *standard* SQL:2003 way to do conditional aggregation — not a
Postgres extension. Skill §3.

### C2. FILTER vs CASE equivalence
> "The following two expressions are equivalent: `SUM(<expression>) FILTER(WHERE <condition>)` and
> `SUM(CASE WHEN <condition> THEN <expression> END)`"
> "Because aggregate functions generally skip over `null` values, the implicit `else null` clause is
> enough to ignore non-matching rows."

**Why it matters:** CASE-inside-aggregate is the portable emulation; it works *because* aggregates skip
NULL (ties back to foundation). Skill §3 + portability block.

### C3. Engine support + portability recommendation
> Native FILTER: "BigQuery ..., Db2 (LUW) ..., DuckDB ..., H2 ..., MariaDB ..., MySQL 9.7.0, Oracle DB
> 23.26.2, PostgreSQL 18, SQL Server 2025, SQLite 3.53.0" (recent versions).
> "As the above described alternative with the `case` expression is very widely supported I recommend
> using that approach rather than the non-standard alternatives."

**Why it matters:** FILTER is reliably available on PostgreSQL and SQLite (and recent others); older
MySQL/SQL Server/Oracle need the CASE form. For maximum portability, CASE. Skill §3 + portability.

---

## Source D — PostgreSQL: GROUPING SETS / ROLLUP / CUBE §7.2.4
URL: https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS

### D1. GROUPING SETS
> "More complex grouping operations than those described above are possible using the concept of
> _grouping sets_. The data selected by the `FROM` and `WHERE` clauses is grouped separately by each
> specified grouping set, aggregates computed for each group just as for simple `GROUP BY` clauses, and
> then the results returned."
> "An empty grouping set means that all rows are aggregated down to a single group (which is output
> even if no input rows were present)..."

**Why it matters:** One query computes several grouping levels — replaces hand-UNION-ing. Skill §5.

### D2. ROLLUP
> "A clause of the form `ROLLUP ( e1, e2, e3, ... )` represents the given list of expressions and all
> prefixes of the list including the empty list; thus it is equivalent to `GROUPING SETS ( (e1,e2,e3),
> ..., (e1,e2), (e1), () )`. This is commonly used for analysis over hierarchical data; e.g., total
> salary by department, division, and company-wide total."

**Why it matters:** ROLLUP = subtotals up a hierarchy + grand total. Skill §5.

### D3. CUBE
> "A clause of the form `CUBE ( e1, e2, ... )` represents the given list and all of its possible
> subsets (i.e., the power set)."

**Why it matters:** CUBE = all 2^n cross-tab subtotals. Skill §5.

### D4. NULLs in grouping-set rows
> "References to the grouping columns or expressions are replaced by null values in result rows for
> grouping sets in which those columns do not appear. To distinguish which grouping a particular output
> row resulted from, see [Table 9.66]..."

**Why it matters:** This is the exact reason GROUPING() exists — subtotal rows carry NULL in the
columns they don't group by, indistinguishable from real NULLs without GROUPING(). Skill §5.

---

## Source E — modern-sql.com: What's new in SQL:2016 (LISTAGG)
URL: https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016

### E1. LISTAGG
> "Listagg is a new _ordered set function_ that resembles the `group_concat` and `string_agg`
> functions offered by some databases. It transforms values from a group of rows into a delimited
> string."
> "The minimal syntax is: `LISTAGG(<expr>, <separator>) WITHIN GROUP(ORDER BY …)`"
> "Listagg accepts the optional `on overflow` clause to define the behavior if the result becomes too
> long." Default is `on overflow error`.

**Why it matters:** LISTAGG is the SQL:2016 standard string-aggregation; it is an ordered-set function
(WITHIN GROUP). Skill §6.

### E2. Engine support (the portability wart)
> LISTAGG: Oracle (full incl. `on overflow`), SQL Server 2025 (partial, no `on overflow`); PostgreSQL,
> DuckDB, MySQL, MariaDB, SQLite, Db2, H2, BigQuery — no LISTAGG.

**Why it matters:** The standard spelling (LISTAGG) is *least* supported; PostgreSQL/SQLite use
`string_agg`, MySQL/SQLite use `GROUP_CONCAT`. String aggregation is heavily dialect-divergent → route
to `sql-standard-vs-dialect-map`. Skill §6 + portability.

---

## Synthesis — what the skill must nail

1. **Functional-dependency rule (the antipattern):** every SELECT-list column must be either in
   GROUP BY or wrapped in an aggregate. Standard SQL and most engines reject violations; MySQL with
   `ONLY_FULL_GROUP_BY` disabled silently returns an *arbitrary* row's value for the ungrouped column.
   This is "ambiguous groups." WRONG/RIGHT in §1. (Note: ONLY_FULL_GROUP_BY is on by default since
   MySQL 5.7.5; the danger is when it is turned off or on legacy configs.)
2. **WHERE vs HAVING:** WHERE filters rows *before* grouping (cannot reference aggregates); HAVING
   filters groups *after* (can). Clause-evaluation order is owned by `sql-select-and-query-processing`.
3. **FILTER vs CASE:** FILTER (WHERE …) is the SQL:2003 standard conditional-aggregation clause;
   CASE-inside-aggregate is the portable emulation that works because aggregates skip NULL.
4. **COUNT variants & NULL:** `COUNT(*)` rows, `COUNT(col)` non-null values, `COUNT(DISTINCT col)`
   distinct non-null values; SUM/AVG skip NULL, SUM of no rows = NULL. Depth → foundation.
5. **GROUPING SETS / ROLLUP / CUBE + GROUPING():** one query for multi-level subtotals instead of
   hand-UNIONed levels that drift; GROUPING() distinguishes a subtotal NULL from a data NULL.
6. **Ordered-set aggregates:** LISTAGG / ARRAY_AGG / percentile_cont / mode with `WITHIN GROUP
   (ORDER BY …)`. LISTAGG is the standard but least-supported spelling → dialect map.

Cross-links: foundation (`sql-relational-and-null-discipline`), `sql-window-functions` (aggregate as
window via OVER), `sql-select-and-query-processing` (clause order), `sql-standard-vs-dialect-map`
(FILTER availability, LISTAGG/STRING_AGG/GROUP_CONCAT).
