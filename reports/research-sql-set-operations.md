# Research: sql-set-operations

Primary-source findings for the `sql-set-operations` skill. Each entry: URL, section, verbatim
quote, why it matters. Accessed 2026-06-26.

---

## Source A — PostgreSQL: Combining Queries (UNION, INTERSECT, EXCEPT)
URL: https://www.postgresql.org/docs/current/queries-union.html

### A1. UNION dedups + appends (the default-to-UNION cost trap)
Section: "7.4. Combining Queries (UNION, INTERSECT, EXCEPT)"
> "UNION effectively appends the result of query2 to the result of query1 (although there is no
> guarantee that this is the order in which the rows are actually returned). Furthermore, it
> eliminates duplicate rows from its result, in the same way as DISTINCT, unless UNION ALL is used."

Why it matters: This is the centerpiece trap. `UNION` does a `DISTINCT`-style dedup — which means a
sort or hash over the whole combined result. `UNION ALL` skips that work. LLMs default to `UNION`
reflexively; when duplicates are impossible or wanted, that is needless cost on prod. Also note "no
guarantee" of order — ties back to foundation §1 (a result is a set; order is undefined without
ORDER BY).

### A2. INTERSECT / INTERSECT ALL
> "INTERSECT returns all rows that are both in the result of query1 and in the result of query2.
> Duplicate rows are eliminated unless INTERSECT ALL is used."

Why it matters: Native set intersection — LLMs reinvent it with `IN`/`EXISTS`/joins. `[ALL]` is the
multiset form (keeps min(m,n) copies of each row instead of collapsing to one).

### A3. EXCEPT / EXCEPT ALL
> "EXCEPT returns all rows that are in the result of query1 but not in the result of query2. (This is
> sometimes called the difference between two queries.) Again, duplicates are eliminated unless
> EXCEPT ALL is used."

Why it matters: Native set difference / anti-join over whole rows. The set-based anti-join, distinct
from the predicate-based `NOT EXISTS` (which lives in subqueries-and-exists). `[ALL]` is the multiset
difference.

### A4. Union compatibility — column count + type, by position
> "In order to calculate the union, intersection, or difference of two queries, the two queries must
> be \"union compatible\", which means that they return the same number of columns and the
> corresponding columns have compatible data types."

Why it matters: Columns align by **position**, not by name. Same count + compatible types required.
Result column names come from the first query. Mismatched count = error; mismatched but
type-compatible columns silently line up by position (a footgun if the SELECT lists are reordered).

### A5. ORDER BY / LIMIT attach to the whole compound; parentheses isolate a branch
> "You can also surround an individual query with parentheses. This is important if the query needs
> to use any of the clauses discussed in following sections, such as LIMIT. Without parentheses,
> you'll get a syntax error, or else the clause will be understood as applying to the output of the
> set operation rather than one of its inputs."

Verbatim example:
> "SELECT a FROM b UNION SELECT x FROM y LIMIT 10
> is accepted, but it means
> (SELECT a FROM b UNION SELECT x FROM y) LIMIT 10
> not
> SELECT a FROM b UNION (SELECT x FROM y LIMIT 10)"

Why it matters: A trailing `ORDER BY`/`LIMIT`/`FETCH` binds to the FINAL compound result, never to one
branch. To order or limit a single branch you must parenthesize it. LLMs write per-branch ORDER BY
expecting it to survive — it does not.

### A6. Precedence — INTERSECT binds tighter than UNION/EXCEPT
> "Without parentheses, UNION and EXCEPT associate left-to-right, but INTERSECT binds more tightly
> than those two operators."

Verbatim example:
> "query1 UNION query2 INTERSECT query3
> means
> query1 UNION (query2 INTERSECT query3)"

Why it matters: `a UNION b INTERSECT c` is NOT `(a UNION b) INTERSECT c`. INTERSECT is the `*` to
UNION/EXCEPT's `+`. Get it wrong and you silently compute a different set. Parenthesize to be explicit.
NOTE: this is the STANDARD/PostgreSQL rule; SQLite deviates (see C5).

---

## Source B — PostgreSQL: VALUES
URL: https://www.postgresql.org/docs/current/sql-values.html

### B1. VALUES is a table constructor
> "VALUES computes a row value or set of row values specified by value expressions. It is most
> commonly used to generate a \"constant table\" within a larger command, but it can be used on its
> own."

Why it matters: `VALUES` is a standalone derived table — inline literal data without a temp table or a
chain of `SELECT ... UNION ALL SELECT ...`. Pairs naturally with set operations and `JOIN`.

### B2. Multi-row VALUES — same element count per row
> "When more than one row is specified, all the rows must have the same number of elements."
Syntax: `VALUES ( expression [, ...] ) [, ...]`  e.g. `VALUES (1, 'one'), (2, 'two'), (3, 'three');`

Why it matters: Multiple rows are comma-separated parenthesized tuples; column types combine "using
the same rules as for UNION."

### B3. Usable anywhere SELECT is
> "Within larger commands, VALUES is syntactically allowed anywhere that SELECT is."
> "VALUES can also be used where a sub-SELECT might be written, for example in a FROM clause"

Why it matters: In `FROM` (with an alias + column names) it is a derived table you can join against;
as the source of an `INSERT`; as a branch of a compound query.

### B4. Default column names
> "The default column names for VALUES are column1, column2, etc. in PostgreSQL, but these names
> might be different in other database systems."

Why it matters: Always supply names via `AS t(col1, col2)` in FROM — the defaults are unportable.

---

## Source C — SQLite: Compound SELECT Statements
URL: https://www.sqlite.org/lang_select.html#compound_select_statements
(Full prose extracted from raw HTML; WebFetch truncates this page.)

### C1. Same number of result columns
> "In a compound SELECT, all the constituent SELECTs must return the same number of result columns."

### C2. UNION ALL vs UNION; INTERSECT/EXCEPT always dedup
> "A compound SELECT created using the UNION ALL operator returns all the rows from the SELECT to the
> left of the UNION ALL operator, and all the rows from the SELECT to the right of it. The UNION
> operator works the same way as UNION ALL, except that duplicate rows are removed from the final
> result set. The INTERSECT operator returns the intersection of the results of the left and right
> SELECTs. The EXCEPT operator returns the subset of rows returned by the left SELECT that are not
> also returned by the right-hand SELECT. Duplicate rows are removed from the results of INTERSECT
> and EXCEPT operators before the result set is returned."

### C3. SQLite LACKS [ALL] for INTERSECT and EXCEPT
The compound-operator grammar is `UNION | UNION ALL | INTERSECT | EXCEPT` — `ALL` attaches only to
UNION. Confirmed in prose: INTERSECT and EXCEPT always remove duplicates ("Duplicate rows are removed
from the results of INTERSECT and EXCEPT operators"); there is no `INTERSECT ALL` / `EXCEPT ALL`.

Why it matters: A portability deviation. Standard SQL + PostgreSQL support `INTERSECT ALL` /
`EXCEPT ALL` (multiset); SQLite does not. Route the spelling gap to standard-vs-dialect-map.

### C4. NULLs are NOT distinct in set ops (opposite of `=`)
> "For the purposes of determining duplicate rows for the results of compound SELECT operators, NULL
> values are considered equal to other NULL values and distinct from all non-NULL values."

Why it matters: This is the NULL-as-not-distinct rule. In a `WHERE`, `NULL = NULL` is UNKNOWN; but in
`UNION`/`INTERSECT`/`EXCEPT` dedup two NULL rows collapse to one. So set ops fold NULLs together — the
exact opposite of equality comparison. Theory lives in the foundation skill; this skill cites it and
routes back.

### C5. SQLite precedence — ALL operators group left-to-right (deviation)
> "When three or more simple SELECTs are connected into a compound SELECT, they group from left to
> right. In other words, if \"A\", \"B\" and \"C\" are all simple SELECT statements, (A op B op C) is
> processed as ((A op B) op C)."

Why it matters: SQLite gives INTERSECT NO higher precedence — everything is left-to-right. PostgreSQL
and standard SQL bind INTERSECT tighter (A6). So `a UNION b INTERSECT c` means different things in
SQLite vs PostgreSQL. Strong argument to always parenthesize mixed compounds. Route to
standard-vs-dialect-map.

### C6. ORDER BY / LIMIT only on the final compound; not on a VALUES tail
> "As the components of a compound SELECT must be simple SELECT statements, they may not contain ORDER
> BY or LIMIT clauses. ORDER BY and LIMIT clauses may only occur at the end of the entire compound
> SELECT, and then only if the final element of the compound is not a VALUES clause."
> "In a compound SELECT statement, only the last or right-most simple SELECT may have an ORDER BY
> clause. That ORDER BY clause will apply across all elements of the compound."

Why it matters: Confirms A5 cross-engine — ORDER BY/LIMIT bind to the whole compound, not a branch.

---

## Synthesis — the nails for the skill
- (a) UNION dedups+sorts (cost) vs UNION ALL appends — default-to-UNION trap [A1, C2].
- (b) Set ops treat NULLs as NOT distinct (two NULLs collapse) — opposite of `=` [C4]; theory →
  foundation.
- (c) INTERSECT/EXCEPT native; `[ALL]` = multiset semantics [A2, A3]; SQLite has no INTERSECT/EXCEPT
  ALL [C3].
- (d) ORDER BY/FETCH legal only on the final query; parentheses isolate a branch [A5, C6].
- (e) INTERSECT precedence higher than UNION/EXCEPT in standard/PG [A6]; SQLite is left-to-right [C5].
- (f) VALUES as a derived table for inline data [B1–B4].

## Portability deviations → standard-vs-dialect-map
- SQLite: no `INTERSECT ALL` / `EXCEPT ALL`; all compound ops left-to-right (no INTERSECT precedence).
- Oracle: spells set difference `MINUS` (not `EXCEPT`); historically no `[ALL]` multiset variants.
- MySQL: `INTERSECT` / `EXCEPT` only in 8.0.31+ (and MariaDB 10.3+); older versions lack them entirely.
