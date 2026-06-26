# Research — sql-select-and-query-processing

Accessed: 2026-06-26. Primary sources for the skill on SELECT fundamentals and the
logical clause-evaluation order. Each finding records URL, section, a verbatim quote,
and why it matters for the skill.

---

## Source A — PostgreSQL: SELECT statement reference
URL: https://www.postgresql.org/docs/current/sql-select.html

### A1 — Logical evaluation order ("general processing of SELECT")
Section: **Description**
> "SELECT retrieves rows from zero or more tables. The general processing of SELECT is as follows:
> 1. All queries in the WITH list are computed...
> 2. All elements in the FROM list are computed...
> 3. If the WHERE clause is specified, all rows that do not satisfy the condition are eliminated from the output.
> 4. If the GROUP BY clause is specified, or if there are aggregate function calls, the output is combined into groups of rows that match on one or more values, and the results of aggregate functions are computed. If the HAVING clause is present, it eliminates groups that do not satisfy the given condition...
> 5. The actual output rows are computed using the SELECT output expressions for each selected row or row group.
> 6. SELECT DISTINCT eliminates duplicate rows from the result...
> 7. Using the operators UNION, INTERSECT, and EXCEPT, the output of more than one SELECT statement can be combined to form a single result set...
> 8. If the ORDER BY clause is specified, the returned rows are sorted in the specified order...
> 9. If the LIMIT (or FETCH FIRST) or OFFSET clause is specified, the SELECT statement only returns a subset of the result rows...
> 10. If FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE or FOR KEY SHARE is specified, the SELECT statement locks the selected rows against concurrent updates."

WHY: This is the authoritative ordered list. It is the backbone of the skill: the
processing order is FROM → WHERE → GROUP BY/aggregates → HAVING → SELECT-list →
DISTINCT → UNION/set ops → ORDER BY → LIMIT/OFFSET. It is NOT the written/lexical order
(SELECT is written first but computed at step 5). This single list explains nearly every
"why is this column not allowed here" error.

### A2 — Alias visibility: legal in ORDER BY/GROUP BY, illegal in WHERE/HAVING
Section: **SELECT List**
> "An output column's name can be used to refer to the column's value in ORDER BY and
> GROUP BY clauses, but not in the WHERE or HAVING clauses; there you must write out the
> expression instead."

WHY: The exact rule for alias visibility. Confirms the task's core question. Note the
NUANCE: PostgreSQL allows aliases in BOTH ORDER BY and GROUP BY. The SQL standard only
guarantees aliases in ORDER BY; PostgreSQL's GROUP BY-by-alias is a documented extension
(see A3). WHERE and HAVING never see aliases — you must repeat the expression. This is
because step 5 (compute SELECT output expressions) runs AFTER step 3 (WHERE) and the
group/HAVING phase (step 4).

### A3 — GROUP BY can reference output columns (the extension)
Section: **GROUP BY Clause**
> "Although query output columns are nominally computed in the next step, they can also be
> referenced (by name or ordinal number) in the GROUP BY clause."

WHY: Explicitly frames GROUP BY-by-alias as referencing something "nominally computed in
the next step" — i.e., an extension PostgreSQL chooses to allow. The portable/standard rule
is stricter (only ORDER BY sees aliases), so the skill flags GROUP BY-by-alias as a
portability gray area and recommends writing the expression for portability.

### A4 — ORDER BY: output column name or ordinal
Section: **ORDER BY Clause**
> "Each expression can be the name or ordinal number of an output column (SELECT list item),
> or it can be an arbitrary expression formed from input-column values."
> "The ordinal number refers to the ordinal (left-to-right) position of the output column."
> Example: "SELECT * FROM distributors ORDER BY name;" and "SELECT * FROM distributors ORDER BY 2;"

WHY: Confirms ORDER BY uniquely sees SELECT-list names AND supports ordinal positions. Backs
the "ORDER BY ordinals/expressions" section.

### A5 — DISTINCT and DISTINCT ON
Section: **DISTINCT Clause**
> "If SELECT DISTINCT is specified, all duplicate rows are removed from the result set (one
> row is kept from each group of duplicates). SELECT ALL specifies the opposite: all rows are
> kept; that is the default."
> "SELECT DISTINCT ON ( expression [, ...] ) keeps only the first row of each set of rows
> where the given expressions evaluate to equal. The DISTINCT ON expressions are interpreted
> using the same rules as for ORDER BY (see above). Note that the 'first row' of each set is
> unpredictable unless ORDER BY is used to ensure that the desired row appears first."

WHY: Two key facts. (1) DISTINCT operates on the WHOLE result row, not a single column —
kills the "DISTINCT col1, col2 dedups only col1" misconception. (2) DISTINCT ON is a
PostgreSQL extension (non-standard) and is order-dependent — route to dialect-map. The
"unpredictable unless ORDER BY" line also reinforces the foundation skill's set-semantics rule.

---

## Source B — PostgreSQL: Query Language (Overview)
URL: https://www.postgresql.org/docs/current/queries-overview.html

Section: **7.1 Overview**
> "The general syntax of the SELECT command is
>   [WITH with_queries] SELECT select_list FROM table_expression [sort_specification]
> The following sections describe the details of the select list, the table expression, and
> the sort specification."

WHY: Limited. This page is a syntax overview that defers detail to later chapters (7.2 table
expressions, 7.3 select lists, sorting). It establishes the three macro-parts (select list,
table expression, sort spec) but does NOT itself enumerate the evaluation order — that lives
in sql-select.html (Source A1). Useful only as a structural cross-reference; the load-bearing
quotes come from Source A.

---

## Source C — SQLite: SELECT
URL: https://www.sqlite.org/lang_select.html
(verbatim ORDER BY/DISTINCT text confirmed via mirror
https://system.data.sqlite.org/home/doc/8e13c43294410407/Doc/Extra/Core/lang_select.html)

### C1 — Processing described as a series of steps
Section: **2. Simple Select Processing**
> "The SELECT statement is the most complicated command in the SQL language. To make the
> description easier to follow, some of the passages below describe the way the data returned
> by a SELECT statement is determined as a series of steps."

Steps as documented (section headings): 2.1 Determination of input data (FROM clause
processing) → 2.3 WHERE clause filtering → result-row generation / GROUP BY → 2.6 Removal of
duplicate rows (DISTINCT processing) → 4. ORDER BY clause → 5. LIMIT clause.

WHY: SQLite documents essentially the same logical order as PostgreSQL: FROM → WHERE →
GROUP BY/result rows → DISTINCT → ORDER BY → LIMIT. Confirms the order is not Postgres-specific.

### C2 — ORDER BY resolves against output-column ALIASES (the deviation)
Section: **4. The ORDER BY clause**
> "If the ORDER BY expression is a constant integer K then the expression is considered an
> alias for the K-th column of the result set (columns are numbered from left to right starting
> with 1)."
> "If the ORDER BY expression is an identifier that corresponds to the alias of one of the
> output columns, then the expression is considered an alias for that column."

WHY: SQLite matches the standard here — ORDER BY sees output aliases and ordinals. The broader
SQLite deviation is that SQLite is *more permissive* about resolving result-column aliases
earlier than the standard allows (it will resolve a bare alias name in GROUP BY/HAVING/ORDER BY
against the result set). This is the documented basis for "SQLite lets aliases be visible
earlier" — a portability hazard, because the same alias reference in WHERE/HAVING that SQLite
may accept is rejected by PostgreSQL. The skill flags this and routes spellings to dialect-map.

### C3 — DISTINCT vs ALL
Section: **2. Simple Select Processing** (result rows / DISTINCT)
> "If the simple SELECT is a SELECT ALL, then the entire set of result rows are returned by the
> SELECT. If neither ALL or DISTINCT are present, then the behavior is as if ALL were specified."

WHY: Confirms ALL is the default in SQLite too, matching PostgreSQL A5. DISTINCT is opt-in and
whole-row.

### C4 — Result expression list / SELECT * substitution
Section: **2.4 Generation of the set of result rows**
> "The list of expressions between SELECT and FROM keywords is known as the result expression
> list. If a result expression is '*' then all columns in the input data are substituted."

WHY: Documents that `*` is expanded to all input columns at result-generation time — the
mechanical basis for the SELECT * fragility argument (the expansion is resolved against the
current schema, so adding/reordering a column silently changes what `*` returns).

---

## Source D — PostgreSQL: Sorting Rows (ORDER BY)
URL: https://www.postgresql.org/docs/current/queries-order.html

### D1 — Output column label or number; the one place aliases ARE visible
> "A sort_expression can also be the column label or number of an output column, as in:
>   SELECT a + b AS sum, c FROM table1 ORDER BY sum;
>   SELECT a, max(b) FROM table1 GROUP BY a ORDER BY 1;
> both of which sort by the first output column."

WHY: The canonical demonstration that an alias defined in the SELECT list (`AS sum`) is legal
in ORDER BY — and only there among the filter/group clauses. Directly answers the task's
confirmation question.

### D2 — Output-name-vs-input-name ambiguity rule
> "There is still ambiguity if an ORDER BY item is a simple name that could match either an
> output column name or a column from the table expression. The output column is used in such
> cases. This would only cause confusion if you use AS to rename an output column to match some
> other table column's name."

WHY: A subtle gotcha — in ORDER BY, a bare name binds to the OUTPUT column first, not the input
column. Worth a one-line caution.

### D3 — Alias must stand alone in ORDER BY
> "Note that an output column name has to stand alone, that is, it cannot be used in an
> expression — for example, this is not correct:
>   SELECT a + b AS sum, c FROM table1 ORDER BY sum + c;          -- wrong"

WHY: The alias is usable as a whole ORDER BY term but you cannot build a larger expression on
it (because it is not an input value). Important precision for the ORDER BY section.

### D4 — NULLS FIRST / NULLS LAST and defaults
> "The NULLS FIRST and NULLS LAST options can be used to determine whether nulls appear before
> or after non-null values in the sort ordering. By default, null values sort as if larger than
> any non-null value; that is, NULLS FIRST is the default for DESC order, and NULLS LAST
> otherwise."

> "ASC order is the default. Ascending order puts smaller values first, where 'smaller' is
> defined in terms of the < operator."

WHY: Backs the NULLS FIRST/LAST and ASC/DESC defaults content. (Deep NULL-ordering treatment
lives in the foundation skill; this skill links there.)

---

## Confirmation of the task's key question

CONFIRMED: The SQL standard / PostgreSQL allow ORDER BY to reference a SELECT-list alias, but
WHERE and HAVING do NOT.
- ORDER BY legal: A2, A4, D1 — "An output column's name can be used to refer to the column's
  value in ORDER BY and GROUP BY clauses".
- WHERE/HAVING illegal: A2 — "...but not in the WHERE or HAVING clauses; there you must write
  out the expression instead."
- Reason: evaluation order (A1) — SELECT output expressions are computed at step 5, AFTER WHERE
  (step 3) and the GROUP BY/HAVING phase (step 4); ORDER BY (step 8) runs after step 5 and so
  can see the computed output names.

NUANCE worth flagging in the skill: PostgreSQL additionally permits aliases in GROUP BY (A2, A3)
as a documented extension beyond the strict standard, and SQLite is more permissive still about
resolving result-column aliases early (C2). The portable rule to teach: aliases are reliably
visible ONLY in ORDER BY; repeat the expression (or use a subquery/CTE) anywhere else.
