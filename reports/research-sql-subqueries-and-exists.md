# Research: sql-subqueries-and-exists

Research for skill #4 (`sql-subqueries-and-exists`) per skill-plan-sql.md lines 104-113.
Scope: scalar/correlated subqueries, IN/EXISTS semi-joins, the NOT IN + NULL trap and the
NOT EXISTS anti-join rewrite, ANY/ALL, subquery-vs-join tradeoffs. All sources accessed 2026-06-26.

The foundation skill `sql-relational-and-null-discipline` flags the NOT IN trap (§5) and explicitly
**defers the deep dive and the full rewrite to this skill**. This skill OWNS the NOT IN deep-dive.

---

## Source 1 — modern-sql.com — Three-Valued Logic
URL: https://modern-sql.com/concept/three-valued-logic

### (a) The NOT IN + NULL expansion — THE CENTERPIECE
Section: NOT IN with NULL.
Verbatim:
> "X NOT IN (NULL, …) ⇔ NOT(X IN(NULL, …)) ⇔ NOT(X = ANY(NULL, …))"
> "An = ANY predicate is only false (so that the negation becomes true) if all comparisons are false. This is however, not possible if there is a NULL comparison, which inevitably yields unknown."
> "The result of X NOT IN (NULL, …) is either false or unknown."
> "Don't allow null in not in lists."

Why it matters: This is the mechanical expansion that explains the trap. `NOT IN` is the negation of
`= ANY` (= IN). For the negation to be TRUE, every `=` comparison must be FALSE. A NULL comparison is
UNKNOWN, never FALSE, so the conjunction can never be TRUE — at best UNKNOWN, which WHERE drops. A
single NULL in the list/subquery collapses NOT IN to zero matching rows. This is the load-bearing
explanation for the WRONG/RIGHT pair in the skill.

### (b) NOT EXISTS is the safe alternative (unaffected by NULL)
Section: recommendation.
Verbatim:
> "When using a subquery, consider using not exists instead of not in or add a where condition to the subquery that removes possible null values."
> "EXISTS never returns unknown."

Why it matters: `EXISTS`/`NOT EXISTS` test row existence, not value equality, so NULLs in the subquery
rows do not produce UNKNOWN. `EXISTS` returns only TRUE or FALSE. This is the justification that
`NOT EXISTS` is the portable null-safe anti-join.

### (c) IN vs NOT IN asymmetry
Verbatim:
> "the result of not in predicates that contain a null value is never true."

Why it matters: `IN` can still return TRUE on a genuine match (forgiving); `NOT IN` with any NULL can
never be TRUE (unforgiving). Asymmetry to call out.

Note: the page does NOT use the terms "semi-join"/"anti-join" — those come from relational-algebra
vocabulary; cite Postgres/general usage for them, not this page.

---

## Source 2 — PostgreSQL — Subquery Expressions
URL: https://www.postgresql.org/docs/current/functions-subquery.html

### (a) EXISTS semantics + semi-join coding convention
Section: EXISTS.
Verbatim:
> "The argument of `EXISTS` is an arbitrary `SELECT` statement, or _subquery_. The subquery is evaluated to determine whether it returns any rows. If it returns at least one row, the result of `EXISTS` is \"true\"; if the subquery returns no rows, the result of `EXISTS` is \"false\"."
> "Since the result depends only on whether any rows are returned, and not on the contents of those rows, the output list of the subquery is normally unimportant. A common coding convention is to write all `EXISTS` tests in the form `EXISTS(SELECT 1 WHERE ...)`."

Why it matters: EXISTS is purely existential — the SELECT list is irrelevant, hence the `SELECT 1`
convention. Establishes EXISTS as a row-existence test (semi-join when EXISTS, anti-join when NOT EXISTS).

### (b) IN with NULL → NULL, not false
Section: IN.
Verbatim:
> "Note that if the left-hand expression yields null, or if there are no equal right-hand values and at least one right-hand row yields null, the result of the `IN` construct will be null, not false. This is in accordance with SQL's normal rules for Boolean combinations of null values."

Why it matters: Even IN is NULL-affected, but because WHERE only needs TRUE and IN can still return TRUE
on a real match, IN "usually does the right thing" for membership. The danger concentrates in NOT IN.

### (c) NOT IN with NULL → NULL, not true (THE verbatim spec quote)
Section: NOT IN.
Verbatim:
> "Note that if the left-hand expression yields null, or if there are no equal right-hand values and at least one right-hand row yields null, the result of the `NOT IN` construct will be null, not true. This is in accordance with SQL's normal rules for Boolean combinations of null values."

Why it matters: This is the authoritative standard-aligned statement of the trap. "null, not true"
means WHERE drops the row. Pair with the modern-sql.com expansion for the mechanism.

### (d) ANY/SOME — and `= ANY` ≡ IN
Section: ANY/SOME.
Verbatim:
> "`SOME` is a synonym for `ANY`. `IN` is equivalent to `= ANY`."
> "The result of `ANY` is \"true\" if any true result is obtained. The result is \"false\" if no true result is found (including the case where the subquery returns no rows)."

Why it matters: `IN` is sugar for `= ANY`. So anything true of IN is true of `= ANY`.

### (e) ALL — and `NOT IN` ≡ `<> ALL` (inherits the trap)
Section: ALL.
Verbatim:
> "`NOT IN` is equivalent to `<> ALL`."
> "The result of `ALL` is \"true\" if all rows yield true (including the case where the subquery returns no rows). The result is \"false\" if any false result is found."

Why it matters: Because `<> ALL` ≡ `NOT IN`, it inherits the exact same NULL trap. A NULL right-hand
row makes one `<>` comparison UNKNOWN, so ALL is never TRUE. Developers who "avoid NOT IN" by writing
`<> ALL` have not escaped anything.

### (f) Scalar subquery: >1 row = error, 0 rows = NULL
Section: scalar subquery (intro / row-comparison).
Verbatim:
> "Furthermore, the subquery cannot return more than one row. (If it returns zero rows, the result is taken to be null.)"

Why it matters: A scalar subquery used where a single value is expected raises a runtime error
("more than one row returned by a subquery used as an expression") if it returns >1 row, and silently
yields NULL if it returns 0 rows — which then propagates NULL through the surrounding expression.

---

## Source 3 — SQLite — Expression / EXISTS / IN / scalar subquery
URL: https://www.sqlite.org/lang_expr.html#the_exists_operator

### (a) EXISTS — NULL rows not special
Section: The EXISTS Operator.
Verbatim:
> "The EXISTS operator always evaluates to one of the integer values 0 and 1. If executing the SELECT statement specified as the right-hand operand of the EXISTS operator would return one or more rows, then the EXISTS operator evaluates to 1. If executing the SELECT would return no rows at all, then the EXISTS operator evaluates to 0."
> "The number of columns in each row returned by the SELECT statement (if any) and the specific values returned have no effect on the results of the EXISTS operator. In particular, rows containing NULL values are not handled any differently from rows without NULL values."

Why it matters: Confirms cross-engine that EXISTS is NULL-immune — "rows containing NULL values are not
handled any differently." Reinforces NOT EXISTS as the portable anti-join.

### (b) IN / NOT IN with NULL — same trap, plus empty-set special case
Section: The IN and NOT IN Operators.
Verbatim:
> "The IN and NOT IN operators take an expression on the left and a list of values or a subquery on the right."
> "When the right operand of an IN or NOT IN operator is a list of values, each of those values must be scalars and the left expression must also be a scalar."
> "When the right operand is an empty set, the result of IN is false and the result of NOT IN is true, regardless of the left operand and even if the left operand is NULL."

And per the results matrix: when the right operand contains NULL and the left operand is not found among
the right values, the result of IN is NULL and the result of NOT IN is also NULL.

Why it matters: SQLite agrees with Postgres — NOT IN yields NULL (→ dropped) when a NULL is present and
no match. The empty-set case is the one safe NOT IN case (NOT IN over an empty set is TRUE). Confirms
portability of the trap across engines.

### (c) Scalar subquery — SQLite divergence (lenient, takes first row)
Section: scalar subquery.
Verbatim:
> "The value of a subquery expression is the first row of the result from the enclosed SELECT statement. The value of a subquery expression is NULL if the enclosed SELECT statement returns no rows."

Why it matters: SQLite does NOT raise an error on a multi-row scalar subquery — it silently takes the
**first row** (and the row is undefined without ORDER BY). This diverges from standard SQL / PostgreSQL,
which raise a runtime error. The 0-rows→NULL behavior is consistent. Portability note for the skill:
the >1-row case is a hard error in Postgres/Oracle/SQL Server but a silent first-row pick in SQLite —
arguably worse, because it hides the bug.

---

## Synthesis — points the skill must nail
1. **NOT IN + NULL trap (centerpiece):** `x NOT IN (...NULL...)` ⇔ `NOT(x = ANY(...))`; expands to
   `x<>a AND x<>b AND x<>NULL`; the NULL conjunct is UNKNOWN so the AND is never TRUE → WHERE drops all
   rows. Spec: result is "null, not true." A single NULL collapses the whole result to empty, silently.
2. **NOT EXISTS is the safe portable anti-join:** EXISTS tests row existence, never returns UNKNOWN,
   "rows containing NULL values are not handled any differently." Correct rewrite of the orphans query.
3. **Scalar subquery:** >1 row = runtime error (Postgres/standard) vs silent first-row (SQLite); 0 rows
   = NULL (universal) that then propagates.
4. **Correlated vs uncorrelated:** uncorrelated runs once; correlated references the outer row and is
   (conceptually) re-evaluated per outer row. EXISTS/NOT EXISTS anti/semi-joins are correlated.
5. **EXISTS/IN semi-join equivalence:** both express a semi-join; EXISTS is null-safe and the SELECT
   list is ignored (`SELECT 1`).
6. **ANY/ALL:** `IN` ≡ `= ANY`; `NOT IN` ≡ `<> ALL`; therefore `<> ALL` inherits the identical NULL trap.
7. **Subquery vs join tradeoffs:** clarity + planner behavior; route plan detail to
   `sql-explain-and-set-based-thinking`.
8. **Portability:** EXISTS / IN / NOT IN / ANY / ALL are all standard SQL; spellings/dialect notes route
   to `sql-standard-vs-dialect-map`. The scalar-subquery >1-row divergence is the one engine wart.
