# Research — sql-expressions-case-and-functions

Research backing the skill `sql-expressions-case-and-functions`. Each entry: URL, section, verbatim
quote, and why it matters for the skill. Accessed 2026-06-26.

---

## Source 1 — PostgreSQL: Conditional Expressions

URL: https://www.postgresql.org/docs/current/functions-conditional.html

### 9.18.1 CASE — simple and searched forms

> "The SQL `CASE` expression is a generic conditional expression, similar to if/else statements in
> other programming languages"

Searched form:
> "CASE WHEN condition THEN result [WHEN ...] [ELSE result] END"

Simple form:
> "CASE expression WHEN value THEN result [WHEN ...] [ELSE result] END"

**Why it matters:** Establishes the two standard forms. Simple CASE compares one expression against
values (equality); searched CASE evaluates arbitrary boolean conditions. The skill's first section.

### 9.18.1 CASE — short-circuit

> "A `CASE` expression does not evaluate any subexpressions that are not needed to determine the result."

**Why it matters:** CASE short-circuits — the canonical pattern for guarding a divide-by-zero or an
expensive/erroring subexpression. WHEN conditions are evaluated in order; the first TRUE wins. No
fall-through (unlike a C switch). Load-bearing for the "guard" idiom.

### 9.18.1 CASE — ELSE / implicit NULL default

> "If no `WHEN` condition yields true, the value of the `CASE` expression is the result of the `ELSE`
> clause. If the `ELSE` clause is omitted and no condition is true, the result is null."

**Why it matters:** The silent-NULL default. A CASE with no ELSE returns NULL for unmatched rows —
a frequent surprise that feeds NULL into downstream arithmetic/concat. The skill must flag "always
consider the implicit ELSE NULL."

### 9.18.2 COALESCE

> "The `COALESCE` function returns the first of its arguments that is not null. Null is returned only
> if all arguments are null."

> "Like a `CASE` expression, `COALESCE` only evaluates the arguments that are needed to determine the
> result; that is, arguments to the right of the first non-null argument are not evaluated."

**Why it matters:** COALESCE is the standard, portable null-substitution function. Short-circuits like
CASE. The replacement for vendor IFNULL/NVL/ISNULL/IIF. Core section.

### 9.18.3 NULLIF

> "The `NULLIF` function returns a null value if value1 equals value2; otherwise it returns value1."

**Why it matters:** NULLIF is the inverse tool — it manufactures a NULL on a sentinel match. The
canonical use is the divide-by-zero guard `x / NULLIF(y, 0)`: when y is 0, NULLIF yields NULL, and
division by NULL yields NULL instead of raising. Core section.

### 9.18.4 GREATEST and LEAST + the NULL deviation

> "The `GREATEST` and `LEAST` functions select the largest or smallest value from a list of any number
> of expressions."

> "NULL values in the argument list are ignored. The result will be NULL only if all the expressions
> evaluate to NULL. (This is a deviation from the SQL standard. According to the standard, the return
> value is NULL if any argument is NULL.)"

**Why it matters:** This is the single most important portability fact in the GREATEST/LEAST section.
PostgreSQL (and MySQL differs again) IGNORE NULLs; the SQL standard says the result is NULL if ANY
argument is NULL. Same call, different answer across engines. The skill must show both behaviors and
route the divergence.

---

## Source 2 — PostgreSQL: String Functions and Operators

URL: https://www.postgresql.org/docs/current/functions-string.html

### Concatenation operator `||`

> "text || text → text" — "Concatenates two strings." Example: `'Post' || 'greSQL' → PostgreSQL`

> "text || anynonarray → text" / "anynonarray || text → text" — "Converts the non-string input to
> text, then concatenates the two strings."

**NULL behavior note:** The `||` operator entries do not restate NULL handling, but the standard rule
(documented in the relational/NULL foundation and Oracle's docs) is that `||` propagates NULL: any
NULL operand makes the whole concatenation NULL. PostgreSQL's separate `concat()` FUNCTION instead
"ignores" NULL arguments — a different behavior, which is exactly why `||` is the predictable standard
choice and `concat()` is a vendor convenience.

**Why it matters:** `||` is the SQL-standard concatenation operator (vs SQL Server's `+`). The
silent-NULL-concat trap: `first || ' ' || last` becomes NULL if `last` is NULL, blanking a report
field. The fix is COALESCE on each nullable operand (or `concat()`/`concat_ws()` where the
NULL-ignoring behavior is wanted, noting it is non-standard).

### SUBSTRING — standard FROM/FOR form

> "SUBSTRING(string text [FROM start integer] [FOR count integer]) → text" — "Extracts the substring
> of string starting at the start'th character ... and stopping after count characters..."
> Examples: `substring('Thomas' from 2 for 3) → hom`, `substring('Thomas' from 3) → omas`,
> `substring('Thomas' for 2) → Th`

**Why it matters:** Standard keyword form. Vendor alias `SUBSTR(string, start [, count])` is the
positional form most engines also accept; the skill prefers the standard `FROM ... FOR ...`.

### TRIM — standard form

> "TRIM([LEADING | TRAILING | BOTH] [characters text] FROM string text) → text" — "Removes the longest
> string containing only characters in characters (a space by default) from the start, end, or both
> ends ... of string." Example: `trim(both 'xyz' from 'yxTomxx') → Tom`

Non-standard alternative documented alongside:
> "TRIM([LEADING | TRAILING | BOTH] [FROM] string text [, characters text]) → text"

**Why it matters:** Standard TRIM uses the `[LEADING|TRAILING|BOTH] chars FROM string` keyword form.
The comma form is non-standard. SQL:2023 standardized multi-character TRIM (feature T056).

### POSITION

> "POSITION(substring text IN string text) → integer" — "Returns first starting index of the specified
> substring within string, or zero if it's not present." Example: `position('om' in 'Thomas') → 3`

**Why it matters:** Standard form is `POSITION(sub IN str)`. Vendor spellings: `STRPOS(str, sub)`
(note reversed args) and Oracle/others' `INSTR`. The reversed argument order is a real bug source.

### OVERLAY

> "OVERLAY(string text PLACING newsubstring text FROM start integer [FOR count integer]) → text" —
> "Replaces the substring of string that starts at the start'th character and extends for count
> characters with newsubstring." Example: `overlay('Txxxxas' placing 'hom' from 2 for 4) → Thomas`

**Why it matters:** Standard string-splice. Rarely used but fully portable; vendor code reaches for
custom substring concatenation instead.

### CHAR_LENGTH / CHARACTER_LENGTH

> "CHAR_LENGTH(text) → integer" / "CHARACTER_LENGTH(text) → integer" — "Returns number of characters
> in the string." Example: `char_length('josé') → 4`

**Why it matters:** Standard length function (counts characters, not bytes). Vendor alias `LENGTH()`.

### UPPER / LOWER

> "UPPER(text) → text" — "Converts the string to all upper case, according to the rules of the
> database's locale." "LOWER(text) → text" — analogous.

**Why it matters:** Both are standard. Locale-dependent — relevant when case-folding for comparison.

### LPAD / RPAD

> "LPAD(string text, length integer [, fill text]) → text" — "Extends the string to length length by
> prepending the characters fill (a space by default)..." Example: `lpad('hi', 5, 'xy') → xyxhi`

> "RPAD(string text, length integer [, fill text]) → text" — appends. Example: `rpad('hi', 5, 'xy') → hixyx`

**Why it matters:** Standardized in SQL:2023 (T055). Previously vendor-only; the skill notes the
version gate.

### Vendor alternative forms (documented contrast)

Standard vs vendor mapping the docs make explicit:
- `SUBSTRING(... FROM ... FOR ...)` vs `SUBSTR(string, start [, count])`
- `CHAR_LENGTH(text)` vs `LENGTH(text)`
- `POSITION(sub IN str)` vs `STRPOS(str, sub)` (reversed args)
- `LEFT(string, n)` / `RIGHT(string, n)` — vendor-specific, no standard keyword equivalent

**Why it matters:** Directly feeds the portability table routing vendor spellings to
`sql-standard-vs-dialect-map`.

---

## Source 3 — modern-sql.com: SQL:2023

URL: https://modern-sql.com/standard/2023

(Page is a frequently-updated stub; it lists the feature IDs for the new scalar functions.)

> "GREATEST and LEAST" — Feature **T054**.

> "String padding functions" (LPAD / RPAD) — Feature **T055**.

> "Multi-character TRIM function" — Feature **T056**.

> "ANY_VALUE" — Feature **T626** (new reserved word; aggregate that returns an arbitrary value from
> the group).

**Why it matters:** Confirms that GREATEST/LEAST, LPAD/RPAD, and multi-character TRIM were only
*standardized* in SQL:2023 — before that they were widely implemented but non-standard. The skill must
present them with that version caveat (portable in intent, but availability and NULL semantics for
GREATEST/LEAST still diverge per engine; see Source 1's deviation note). ANY_VALUE is mentioned as a
related SQL:2023 scalar/aggregate addition but its depth belongs to the aggregation skill — note only.

---

## Synthesis — what the skill must nail

1. **Prefer standard scalar expressions over vendor spellings.** CASE/COALESCE/NULLIF/`||` over
   IFNULL/ISNULL/NVL/IIF/`+`. (Sources 1, 2.)
2. **`||` and arithmetic propagate NULL** — one NULL operand nullifies the result; the silent-blank
   trap. Oracle deviates (treats NULL as `''` in `||`). NULL theory itself lives in the foundation
   skill; route there. (Source 2 + foundation.)
3. **NULLIF(a,b) → NULL when equal** — divide-by-zero guard `x / NULLIF(y,0)`. (Source 1.)
4. **CASE: short-circuits, no fall-through, implicit ELSE NULL.** (Source 1.)
5. **Standard string functions** SUBSTRING/TRIM/POSITION/OVERLAY/CHAR_LENGTH vs vendor
   SUBSTR/INSTR/LEFT/RIGHT/LENGTH/STRPOS. (Source 2.)
6. **CAST(x AS type) is the portable conversion** (vs `::` or CONVERT); implicit coercion differs per
   engine. (Foundation/general SQL knowledge; CAST is standard.)
7. **GREATEST/LEAST standardized in SQL:2023 (T054) + NULL divergence** — PG ignores NULL, standard
   says NULL if any arg NULL. (Sources 1, 3.)
