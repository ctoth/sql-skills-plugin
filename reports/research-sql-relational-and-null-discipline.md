# Research: `sql-relational-and-null-discipline` (FOUNDATION skill)

Primary-source research for the standard-SQL skills plugin. Each finding records the source URL, the section/heading, a verbatim quote fragment, and why it matters for the skill.

---

## 1. A relation is a SET of rows — unordered; result order is undefined without ORDER BY

### Codd 1970, "A Relational Model of Data for Large Shared Data Banks"
- URL: https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf (redirects to https://www.engineering.upenn.edu/~zives/03f/cis550/codd.pdf; the canonical PDF is image/encrypted and not machine-readable, so quotes below are corroborated via a clean secondary transcription)
- Section: 1.3 "A Relational View of Data" (p. 379), properties of a relation R that is a subset of a Cartesian product of domains.
- The four classic properties Codd lists for such an array: each row represents an n-tuple of R; **"the ordering of rows is immaterial"**; **"all rows are distinct"**; the ordering of columns is significant and corresponds to the ordering of the domains.
- Mathematically a relation is a *set* of n-tuples drawn from n domains (a subset of the Cartesian product); being a set, it forbids duplicate tuples and carries no inherent order.
- Corroborating source (clean text): https://users.dimi.uniud.it/~massimo.franceschet/ds/syllabus/learn/database/RM.html — "The order of the rows and that of the columns in the table is not relevant" and "every tuple of the relation is distinct from the others."
- WHY IT MATTERS: This is the bedrock. SQL tables are (logically) sets; a query result has **no defined order** unless you ask for one. Any reliance on "natural" order, insertion order, or physical order is a bug.

### PostgreSQL — ORDER BY (queries-order)
- URL: https://www.postgresql.org/docs/current/queries-order.html
- Section: "Sorting Rows (ORDER BY)"
- Verbatim: **"If sorting is not chosen, the rows will be returned in an unspecified order. The actual order in that case will depend on the scan and join plan types and the order on disk, but it must not be relied on."**
- WHY IT MATTERS: The authoritative engine statement that result order is undefined without `ORDER BY`. The phrase "must not be relied on" is the rule the skill enforces.

---

## 2. Three-valued logic: comparisons with NULL yield UNKNOWN; WHERE/HAVING/ON keep only TRUE; CHECK is the asymmetric exception

### modern-sql.com — Three-Valued Logic
- URL: https://modern-sql.com/concept/three-valued-logic
- Verbatim: **"SQL uses a _three-valued logic_: besides _true_ and _false_, the result of logical expressions can also be _unknown_."**
- Verbatim: **"The logical value _unknown_ indicates that a result _actually_ depends on a `null` value."**
- Section "General Rule: `where`, `having`, `when`, etc.": these clauses **"require _true_ conditions. It is not enough that a condition is not _false_."** Example: `WHERE col = NULL` returns the empty set because "the result of the equals comparison to `null` is _always_ _unknown_. The `where` clause thus rejects _all_ rows."
- Section "Exception: Check Constraints": **"`Check` constraints follow the reverse logic: they reject _false_, rather than accepting _true_."** Consequently **"check constraints accept _true_ and _unknown_."**
- Truth-table behavior described (no formal grid on the page): AND is false as soon as any operand is false, otherwise unknown operands make the result unknown; OR is true as soon as any operand is true, otherwise unknown; **"not(unknown) is also unknown."**
- WHY IT MATTERS: This is the central asymmetry the skill must nail. `WHERE`/`HAVING`/`ON` keep only rows that evaluate to **TRUE** (UNKNOWN is dropped, like FALSE). A `CHECK` constraint does the opposite: it is **satisfied unless it evaluates to FALSE**, so UNKNOWN *passes*. Same row, same predicate, opposite outcome depending on clause.

### Oracle corroboration on UNKNOWN ≈ FALSE but not identical
- URL: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html
- A condition evaluating to `UNKNOWN` "acts almost like `FALSE`" but differs in that **`NOT UNKNOWN` evaluates to `UNKNOWN`** (whereas `NOT FALSE` evaluates to `TRUE`).
- WHY IT MATTERS: Reinforces that you cannot simply negate to "recover" NULL rows; `NOT (x = y)` does not flip UNKNOWN to TRUE.

---

## 3. `x = NULL` is always UNKNOWN — use `IS NULL`; `col <> 'a'` silently drops NULL rows

### PostgreSQL — Comparison Functions and Operators
- URL: https://www.postgresql.org/docs/current/functions-comparison.html
- Verbatim: **"Ordinary comparison operators yield null (signifying 'unknown'), not true or false, when either input is null. For example, `7 = NULL` yields null, as does `7 <> NULL`."**
- Verbatim: **"Do _not_ write `expression = NULL` because `NULL` is not 'equal to' `NULL`. (The null value represents an unknown value, and it is not known whether two unknown values are equal.)"**
- Test predicates: `expression IS NULL` / `expression IS NOT NULL` (nonstandard equivalents `ISNULL` / `NOTNULL`).

### modern-sql.com — NULL
- URL: https://modern-sql.com/concept/null
- Section "Comparisons Involving `null`": **"Comparisons (`<`, `>`, `=`, …) to `null` are neither _true_ nor _false_ but instead return the third logical value of SQL: _unknown_."**
- Memorable framing (three-valued-logic page): **"Nothing _equals_ `null`. Not even `null` equals `null` because each `null` could be different."**
- WHY IT MATTERS: Because both `=` and `<>` against NULL yield UNKNOWN, a predicate like `col <> 'a'` returns **only** rows where `col` is a non-NULL value different from 'a' — every NULL row is silently excluded. The fix is `IS NULL` / `IS NOT NULL` (and, for "different OR null", `col IS DISTINCT FROM 'a'`).

---

## 4. The `NOT IN (... NULL ...)` trap — the canonical footgun

### modern-sql.com — NULL / Three-Valued Logic
- URLs: https://modern-sql.com/concept/null and https://modern-sql.com/concept/three-valued-logic
- Mechanism: `x NOT IN (a, b, NULL)` expands to `x <> a AND x <> b AND x <> NULL`. The final conjunct `x <> NULL` is **UNKNOWN**, so the whole AND can never be TRUE — it is at best UNKNOWN, which `WHERE` drops. Result: a single NULL in the IN-list/subquery collapses `NOT IN` to **zero matching rows**.
- The pages establish each constituent fact: comparisons to null are unknown, AND with unknown stays unknown (unless an operand is false), and WHERE keeps only TRUE.
- WHY IT MATTERS: This is *the* portability/correctness landmine. `IN` with a NULL is comparatively forgiving (can still return TRUE on a match), but `NOT IN` against a subquery that yields even one NULL silently returns nothing. The skill should mandate `NOT EXISTS` (or filtering NULLs out of the subquery, or `IS DISTINCT FROM`) instead of `NOT IN` whenever the operand can be NULL.

---

## 5. NULL propagates through arithmetic and string concatenation

### modern-sql.com — NULL
- URL: https://modern-sql.com/concept/null
- Section "`Null` Propagates Through Expressions": **"Expressions and functions that process a `null` value generally return the `null` value."**
- Examples: `1 + NULL` → NULL; `'foo ' || NULL || 'bar'` → NULL; `SUBSTRING('foo bar' FROM 4 FOR NULL)` → NULL.
- Exceptions: aggregates and the logical operators — e.g. `null AND false` is **false**, `null OR true` is **true**.

### Oracle — Nulls
- URL: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html
- Verbatim: **"Any arithmetic expression containing a null always evaluates to null. For example, null added to 10 is null."**
- Verbatim: **"All operators (except concatenation) return null when given a null operand."**
- WHY IT MATTERS: NULL is contagious in scalar expressions, including string `||`. **Portability caveat:** Oracle is the deviation — its `||` treats NULL as the empty string (because Oracle treats `''` as NULL; see §9), so `'a' || NULL` is `'a'` in Oracle but `NULL` in standard SQL / PostgreSQL. The skill should flag concatenation as engine-divergent and recommend `COALESCE`/`CONCAT` semantics explicitly.

---

## 6. COUNT(*) counts rows; COUNT(col)/COUNT(DISTINCT col) ignore NULLs; SUM/AVG/MIN/MAX skip NULLs

### PostgreSQL — Aggregate Functions
- URL: https://www.postgresql.org/docs/current/functions-aggregate.html
- `count(*)` — verbatim: **"Computes the number of input rows."**
- `count(expression)` — verbatim: **"Computes the number of input rows in which the input value is not null."**
- `sum` — "Computes the sum of the non-null input values." `avg` — "the average (arithmetic mean) of all the non-null input values." `min`/`max` — "the minimum/maximum of the non-null input values."
- Verbatim on empty inputs: **"except for `count`, these functions return a null value when no rows are selected. In particular, `sum` of no rows returns null, not zero as one might expect."**
- WHY IT MATTERS: Two traps. (a) `COUNT(col)` and `COUNT(DISTINCT col)` silently undercount versus `COUNT(*)` whenever `col` has NULLs — they are different questions. (b) `AVG(col)` divides the sum of non-null values by the *count of non-null values*; it is **not** the same as treating NULL as 0 (which would inflate the denominator and lower the average). `SUM` over an all-NULL/empty set returns NULL, not 0 — wrap in `COALESCE(SUM(x),0)` if you need 0.

---

## 7. `IS [NOT] DISTINCT FROM` — null-safe equality

### PostgreSQL — Comparison
- URL: https://www.postgresql.org/docs/current/functions-comparison.html
- Verbatim: **"For non-null inputs, `IS DISTINCT FROM` is the same as the `<>` operator. However, if both inputs are null it returns false, and if only one input is null it returns true."**
- Table 9.2 results: `1 IS DISTINCT FROM NULL` → `t`; `NULL IS DISTINCT FROM NULL` → `f`; `1 IS NOT DISTINCT FROM NULL` → `f`; `NULL IS NOT DISTINCT FROM NULL` → `t`. All return true/false, **never NULL**.

### modern-sql.com — NULL
- URL: https://modern-sql.com/concept/null
- `IS DISTINCT FROM` provides a **"null-aware equals comparison"** where **"two `null` values are not distinct from each other"**; an optional SQL:1999/2003 feature, "still not widely supported."

### MySQL — `<=>` spaceship operator
- URL: https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html
- Verbatim: `<=>` is **"NULL-safe equal. This operator performs an equality comparison like the `=` operator, but returns `1` rather than `NULL` if both operands are `NULL`, and `0` rather than `NULL` if one operand is `NULL`."**
- Verbatim: **"The `<=>` operator is equivalent to the standard SQL `IS NOT DISTINCT FROM` operator."**
- Contrast: `SELECT NULL = NULL` → `NULL`; `SELECT NULL <=> NULL` → `1`.
- WHY IT MATTERS: `IS NOT DISTINCT FROM` is the correct, portable way to express "equal, treating two NULLs as equal." MySQL's `<=>` is the non-standard spelling of the same thing. Use it for null-safe joins/merges and to avoid the `<>`-drops-NULL problem in §3.

---

## 8. NULL in GROUP BY, ORDER BY, and UNIQUE

### GROUP BY — NULLs form one group
- `GROUP BY` collapses all NULLs into a single group (treated as "not distinct" for grouping), which is the *opposite* of how a default `UNIQUE` constraint treats them. (See SQLite matrix below: "nulls are distinct in SELECT DISTINCT" → **No** for every engine, i.e. DISTINCT/GROUP BY fold NULLs together.)

### ORDER BY — NULLS FIRST / NULLS LAST and engine-default divergence
- URL: https://www.postgresql.org/docs/current/queries-order.html
- Verbatim: **"By default, null values sort as if larger than any non-null value; that is, `NULLS FIRST` is the default for `DESC` order, and `NULLS LAST` otherwise."**
- WHY IT MATTERS: Default NULL placement differs across engines (PostgreSQL/Oracle sort NULLs as largest → last on ASC; MySQL/SQLite sort NULLs as smallest → first on ASC). For deterministic, portable ordering, always specify `NULLS FIRST` / `NULLS LAST` explicitly (note: MySQL historically lacks the `NULLS FIRST/LAST` syntax and needs `ORDER BY col IS NULL, col`).

### UNIQUE — multiple NULLs allowed by default; SQL:2023 `NULLS NOT DISTINCT`
- URL: https://www.postgresql.org/docs/current/sql-createtable.html
- Verbatim: **"For the purpose of a unique constraint, null values are not considered equal, unless `NULLS NOT DISTINCT` is specified."**
- Syntax verbatim: `UNIQUE [ NULLS [ NOT ] DISTINCT ]`.
- Default (`NULLS DISTINCT`): a UNIQUE column permits **multiple** rows with NULL. `NULLS NOT DISTINCT` (PostgreSQL 15+, aligning with **SQL:2023**) permits at most **one** NULL.
- SQL:2023 feature reference: https://modern-sql.com/standard/2023 lists **F292 "UNIQUE null treatment"** and the open issue of "unique with nulls distinct, or unique with nulls not distinct" — confirming this is a standardized 2023 capability.
- WHY IT MATTERS: The grouping/uniqueness duality is a classic source of confusion (also flagged by SQLite below): NULLs are folded together for `GROUP BY`/`DISTINCT`/`UNION` but kept distinct for a default `UNIQUE` constraint. The skill must teach that a default UNIQUE column does **not** enforce single-NULL, and that `NULLS NOT DISTINCT` is the modern lever to change that.

---

## 9. Portability deviations across engines

### SQLite — "NULL Handling in SQLite" comparison matrix
- URL: https://www.sqlite.org/nulls.html
- Captured matrix (verbatim cell values), engines: SQLite / PostgreSQL / Oracle / Informix / DB2 / MS-SQL / OCELOT:

| Behavior | SQLite | PostgreSQL | Oracle | Informix | DB2 | MS-SQL | OCELOT |
|---|---|---|---|---|---|---|---|
| Adding anything to null gives null | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Multiplying null by zero gives null | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| nulls are distinct in a UNIQUE column | Yes | Yes | Yes | No | No | Yes | No |
| nulls are distinct in SELECT DISTINCT | No | No | No | No | No | No | No |
| nulls are distinct in a UNION | No | No | No | No | No | No | No |
| "CASE WHEN null THEN 1 ELSE 0 END" is 0? | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| "null OR true" is true | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| "not (null AND false)" is true | Yes | Yes | Yes | Yes | Yes | Yes | Yes |

- Notes: **Note 4** — "DB2, SQL Anywhere, and Borland Interbase do not allow NULLs in a UNIQUE column." **Note 1** — "Older versions of firebird omit all NULLs from SELECT DISTINCT and from UNION."
- Verbatim editorial: **"The fact that NULLs are distinct for UNIQUE columns but are indistinct for SELECT DISTINCT and UNION continues to be puzzling. It seems that NULLs should be either distinct everywhere or nowhere."**
- WHY IT MATTERS: This is the single best portability source — it shows the consensus (arithmetic propagation, DISTINCT/UNION fold NULLs) and the divergence (UNIQUE-column NULL distinctness varies; Informix/DB2/OCELOT say NULLs are *not* distinct there, and some engines forbid NULL in UNIQUE entirely).

### Oracle — empty string `=` NULL (the big deviation)
- URL: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html
- Verbatim: **"Oracle Database currently treats a character value with a length of zero as null. However, this may not continue to be true in future releases, and Oracle recommends that you do not treat empty strings the same as nulls."**
- Verbatim: **"Because null represents a lack of data, a null cannot be equal or unequal to any value or to another null."**
- Verbatim: **"To test for nulls, use only the comparison conditions `IS NULL` and `IS NOT NULL`."**
- WHY IT MATTERS: Oracle's `'' = NULL` equivalence is the most damaging cross-engine difference: an empty string round-trips as NULL, breaks `NOT NULL` assumptions, and (combined with Oracle's NULL-as-empty-string concatenation) makes `||` behave unlike standard SQL. Code that distinguishes `''` from NULL is non-portable to/from Oracle.

### MySQL — `<=>` null-safe equal (see §7)
- URL: https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html
- MySQL adds the non-standard `<=>` operator (= `IS NOT DISTINCT FROM`); also note MySQL's historical lack of `NULLS FIRST/LAST` ordering syntax.

### PostgreSQL — boolean test predicates that never return NULL
- URL: https://www.postgresql.org/docs/current/functions-comparison.html
- `IS [NOT] TRUE / FALSE / UNKNOWN` "always return true or false, never null." Verbatim: **"A null input is treated as the logical value 'unknown'. Notice that `IS UNKNOWN` and `IS NOT UNKNOWN` are effectively the same as `IS NULL` and `IS NOT NULL`, respectively, except that the input expression must be of Boolean type."**
- WHY IT MATTERS: These predicates are the escape hatch from three-valued logic when you need a definite TRUE/FALSE branch (e.g. `WHERE (a = b) IS NOT TRUE` to *include* the NULL/UNKNOWN rows that a plain `a <> b` would drop).

---

## 10. Antipattern framing — Karwin, "Fear of the Unknown"

- Source: Bill Karwin, *SQL Antipatterns* — https://pragprog.com/titles/bksqla/sql-antipatterns/
- The "Fear of the Unknown" chapter argues against avoiding NULL by substituting a sentinel value (e.g. `0`, `-1`, `''`, `'N/A'`, `'unknown'`). NULL is **not** "nothing" and is not a zero/empty string — it is the marker for *absent/unknown* data. Sentinels poison aggregates (a `-1` placeholder corrupts `SUM`/`AVG`/`MIN`), defeat `IS NULL` testing, collide with legitimate values, and force every query to special-case the magic value.
- The book's prescribed solution: use NULL properly, understand three-valued logic, use `IS NULL` / `IS DISTINCT FROM`, and declare columns `NOT NULL` only when absence is genuinely impossible.
- WHY IT MATTERS: This is the conceptual posture the foundation skill should adopt — embrace NULL with disciplined three-valued-logic handling rather than fearing it into sentinel hacks.

---

## Source URL index
- https://modern-sql.com/concept/three-valued-logic
- https://modern-sql.com/concept/null
- https://www.sqlite.org/nulls.html
- https://www.postgresql.org/docs/current/functions-comparison.html
- https://www.postgresql.org/docs/current/functions-aggregate.html
- https://www.postgresql.org/docs/current/queries-order.html
- https://www.postgresql.org/docs/current/sql-createtable.html
- https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Nulls.html
- https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html
- https://modern-sql.com/standard/2023
- https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf (Codd 1970; image PDF — corroborated via https://users.dimi.uniud.it/~massimo.franceschet/ds/syllabus/learn/database/RM.html)
- https://pragprog.com/titles/bksqla/sql-antipatterns/ (Karwin, "Fear of the Unknown")
