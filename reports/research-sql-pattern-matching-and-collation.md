# Research: sql-pattern-matching-and-collation

Research for a NEW standard-SQL skill on text matching and collation. Gap found in audit:
no skill covered `LIKE`, the SQL-standard `SIMILAR TO` regex, non-standard vendor regex
(`~`, `REGEXP`/`RLIKE`), `ILIKE`, and — the centerpiece — the fact that `LIKE` case
sensitivity is **collation-dependent** and therefore silently differs across engines.

Accessed: 2026-06-26. Each entry: URL, section, verbatim quote, why it matters.

---

## Source 1 — PostgreSQL: Pattern Matching

URL: https://www.postgresql.org/docs/current/functions-matching.html

### LIKE wildcards (% and _)

- Section **9.7.1. LIKE**
  - Quote: "An underscore (`_`) in *pattern* stands for (matches) any single character; a
    percent sign (`%`) matches any sequence of zero or more characters."
  - Why it matters: defines the two LIKE wildcards. `_` = exactly one char, `%` = zero-or-more
    chars. This is the portable, universal core of LIKE.

### ESCAPE clause (literal % and _)

- Section **9.7.1. LIKE**
  - Quote: "To match a literal underscore or percent sign without matching other characters,
    the respective character in *pattern* must be preceded by the escape character. The default
    escape character is the backslash but a different one can be selected by using the `ESCAPE`
    clause. To match the escape character itself, write two escape characters."
  - Why it matters: a user searching for a literal `%` or `_` (e.g. `100%`, `first_name`) must
    escape it or the wildcard meaning silently kicks in. ESCAPE picks the escape char.

### LIKE is case-sensitive (in PG / the standard)

- Section **9.7.1. LIKE**
  - Quote: "The key word `ILIKE` can be used instead of `LIKE` to make the match
    case-insensitive according to the active locale."
  - Why it matters: PG only provides a SEPARATE keyword (ILIKE) for case-insensitive matching,
    which establishes that bare LIKE is case-SENSITIVE in PostgreSQL / the standard. This is the
    pivot of the portability trap: other engines make bare LIKE case-INSENSITIVE by default.

### SIMILAR TO = the SQL-standard regular expression operator

- Section **9.7.2. SIMILAR TO Regular Expressions**
  - Quote: "The `SIMILAR TO` operator returns true or false depending on whether its pattern
    matches the given string. It is similar to `LIKE`, except that it interprets the pattern
    using the SQL standard's definition of a regular expression."
  - Quote: "Like `LIKE`, the `SIMILAR TO` operator succeeds only if its pattern matches the
    entire string; this is unlike common regular expression behavior where the pattern can match
    any part of the string."
  - Why it matters: SIMILAR TO is THE SQL-standard regex operator — verbatim "the SQL standard's
    definition of a regular expression." It is whole-string anchored (unlike POSIX). It is rarely
    used and unevenly supported, but it is the only *standard* regex.

### POSIX ~ operators are non-standard (PostgreSQL-specific)

- Section **9.7.3. POSIX Regular Expressions**
  - Quote: "The operator `~~` is equivalent to `LIKE`, and `~~*` corresponds to `ILIKE`. There are
    also `!~~` and `!~~*` operators that represent `NOT LIKE` and `NOT ILIKE`, respectively. All
    of these operators are PostgreSQL-specific."
  - Why it matters: nails the standard-vs-nonstandard distinction verbatim. The `~` family
    (POSIX regex match operators) is "PostgreSQL-specific" — not in the SQL standard. Use them
    only when PG-locked.

### ILIKE is a PostgreSQL extension (non-standard)

- Section **9.7.1. LIKE**
  - Quote: "The key word `ILIKE` can be used instead of `LIKE` to make the match case-insensitive
    according to the active locale. (But this does not support nondeterministic collations.) This
    is not in the SQL standard but is a PostgreSQL extension."
  - Why it matters: ILIKE is explicitly "not in the SQL standard but is a PostgreSQL extension."
    Portable case-insensitive matching must use COLLATE or LOWER(), not ILIKE.

---

## Source 2 — PostgreSQL: Collation Support

URL: https://www.postgresql.org/docs/current/collation.html

### What COLLATE does

- Section **23.2. Collation Support**
  - Quote: "The collation feature allows specifying the sort order and character classification
    behavior of data per-column, or even per-operation."
  - Why it matters: COLLATE is the standard lever that controls BOTH sort order AND character
    classification (which includes case/accent treatment), and it can be applied per-operation —
    i.e. attached to a single comparison or LIKE.

### Collation drives comparison and pattern matching

- Section **23.2.1. Concepts**
  - Quote: "When the database system has to perform an ordering or a character classification, it
    uses the collation of the input expression."
  - Quote: "In addition to comparison operators, collations are taken into account by functions
    that convert between lower and upper case letters, such as `lower`, `upper`, and `initcap`; by
    pattern matching operators; and by `to_char` and related functions."
  - Why it matters: explicitly states "pattern matching operators" (LIKE/SIMILAR TO) are governed
    by collation — so the SAME LIKE expression behaves differently under different collations.
    This is the mechanism behind the portability trap.

### Nondeterministic collations = case/accent insensitivity

- Section **23.2.2.4. Nondeterministic Collations**
  - Quote: "A collation is either deterministic or nondeterministic. A deterministic collation
    uses deterministic comparisons, which means that it considers strings to be equal only if they
    consist of the same byte sequence. Nondeterministic comparison may determine strings to be
    equal even if they consist of different bytes. Typical situations include case-insensitive
    comparison, accent-insensitive comparison, as well as comparison of strings in different
    Unicode normal forms."
  - Why it matters: defines how case-insensitive and accent-insensitive comparison are realized —
    via a nondeterministic collation. Equality and matching of `'José'` vs `'jose'` depends
    entirely on which collation is in force.

- Section **23.2.3. Managing Collations** (example)
  - Quote (example): `CREATE COLLATION ignore_accent_case (provider = icu, deterministic = false,
    locale = 'und-u-ks-level1');` then `SELECT 'Å' = 'A' COLLATE ignore_accent_case;` returns
    `true` and `SELECT 'z' = 'Z' COLLATE ignore_accent_case;` returns `true`.
  - Why it matters: concrete proof that COLLATE on a single comparison flips case/accent
    sensitivity — the explicit, portable-in-intent control to reach for.

### ICU comparison levels (case/accent are "levels")

- Section **23.2.3.1. ICU Comparison Levels** (Table 23.1)
  - Facts: level1 (base character) makes `'n' = 'ñ'` true (accent-insensitive); level2 adds
    accents so `'n' = 'ñ'` is false; level3 adds case so `'g' = 'G'` is false (case-sensitive).
  - Why it matters: shows case sensitivity and accent sensitivity are independent, tunable dials
    of the collation — not fixed properties of the operator.

---

## Source 3 — SQLite: Expression / LIKE & GLOB

URL: https://www.sqlite.org/lang_expr.html  (section 5, "The LIKE, GLOB, REGEXP, MATCH, and GLOB operators")

### LIKE is case-insensitive for ASCII by default (a deviation)

- Quote: "Any other character matches itself or its lower/upper case equivalent (i.e.
  case-insensitive matching)."
  - Why it matters: SQLite's bare LIKE is case-INSENSITIVE by default — the opposite of PG/the
    standard. The same `WHERE name LIKE 'abc'` matches `'ABC'` in SQLite but not in PG.

### Case-insensitivity is ASCII-only (a second trap)

- Quote: "Important Note: SQLite only understands upper/lower case for ASCII characters by
  default. The LIKE operator is case sensitive by default for unicode characters that are beyond
  the ASCII range. For example, the expression `'a' LIKE 'A'` is TRUE but `'æ' LIKE 'Æ'` is
  FALSE."
  - Why it matters: SQLite's default case-folding stops at ASCII, so non-ASCII text behaves
    case-SENSITIVELY even though ASCII text behaves case-insensitively — a within-engine
    inconsistency on top of the cross-engine one.

### % and _ wildcards

- Quote: "A percent symbol (\"%\") in the LIKE pattern matches any sequence of zero or more
  characters in the string. An underscore (\"_\") in the LIKE pattern matches any single
  character in the string."
  - Why it matters: confirms the universal wildcard semantics carry to SQLite.

### GLOB is case-sensitive, Unix-glob syntax

- Quote: "The GLOB operator is similar to LIKE but uses the Unix file globbing syntax for its
  wildcards. Also, GLOB is case sensitive, unlike LIKE."
  - Why it matters: GLOB is SQLite-only, case-sensitive, and uses `*`/`?` (not `%`/`_`). Route
    dialect spelling to the dialect map.

### ESCAPE clause

- Quote: "If the optional ESCAPE clause is present, then the expression following the ESCAPE
  keyword must evaluate to a string consisting of a single character. This character may be used
  in the LIKE pattern to include literal percent or underscore characters."
  - Why it matters: confirms ESCAPE is portable to SQLite. Note SQLite has no default escape
    char — you must supply ESCAPE.

---

## Source 4 — MySQL 8.4: String Comparison Functions (LIKE)

URL: https://dev.mysql.com/doc/refman/8.4/en/string-comparison-functions.html

### % and _ wildcards

- Quote: "With LIKE you can use the following two wildcard characters in the pattern: `%` matches
  any number of characters, even zero characters. `_` matches exactly one character."
  - Why it matters: same wildcard semantics; universal.

### LIKE case sensitivity is collation-driven (case-insensitive by default)

- Quote: "The following statements illustrate that string comparisons are not case-sensitive
  unless one of the operands is case-sensitive (uses a case-sensitive collation or is a binary
  string):"
  - Example facts: `SELECT 'abc' LIKE 'ABC';` returns `1` (match) under MySQL's default
    collation; `SELECT 'abc' LIKE _utf8mb4 'ABC' COLLATE utf8mb4_0900_as_cs;` returns `0`;
    `SELECT 'abc' LIKE BINARY 'ABC';` returns `0`.
  - Why it matters: MySQL's default LIKE is case-INSENSITIVE (like SQLite, unlike PG/standard) —
    and you flip it with a `COLLATE ... _cs/_bin` clause or `BINARY`. This is the centerpiece:
    the same query returns different rows on MySQL vs PostgreSQL purely because of the default
    collation.

### LIKE matches per-character (standard) — can differ from =

- Quote: "Per the SQL standard, LIKE performs matching on a per-character basis, thus it can
  produce results different from the `=` comparison operator."
  - Why it matters: even within one engine, LIKE and `=` can disagree because LIKE is
    per-character and `=` may apply trailing-space / collation rules differently.

### ESCAPE clause

- Quote: "To test for literal instances of a wildcard character, precede it by the escape
  character. If you do not specify the `ESCAPE` character, `\` is assumed, unless the
  NO_BACKSLASH_ESCAPES SQL mode is enabled." And: `SELECT 'David_' LIKE 'David|_' ESCAPE '|';`
  returns `1`.
  - Why it matters: MySQL defaults the escape char to backslash (PG also backslash; SQLite has
    none) — another portability wrinkle: the default ESCAPE char is not universal, so always
    specify ESCAPE explicitly for portable literal-wildcard matching.

---

## Synthesis — what the skill must nail

1. **LIKE wildcards + ESCAPE** — `%` = any string (incl. empty), `_` = any single char;
   ESCAPE to match a literal `%`/`_`. Default escape char differs (PG/MySQL backslash, SQLite
   none), so specify ESCAPE explicitly.

2. **THE PORTABILITY TRAP (centerpiece)** — LIKE case sensitivity is collation-dependent and
   differs across engines:
   - PostgreSQL / SQL standard: case-SENSITIVE.
   - MySQL / SQL Server: case-INSENSITIVE under the common default collations.
   - SQLite: case-INSENSITIVE for ASCII only (case-sensitive for non-ASCII).
   The SAME query returns different rows. Control it explicitly with `COLLATE` (per-operation)
   or `LOWER()` on both sides. Note `LOWER(col)` on the column kills index usage — route the
   sargability cost to `sql-indexing-and-sargability`.

3. **SIMILAR TO** = the SQL-standard regex operator (verbatim "the SQL standard's definition of a
   regular expression"); whole-string anchored; rarely used / unevenly supported. Non-standard
   vendor regex: PG POSIX `~`/`~*` (PostgreSQL-specific), MySQL/SQLite `REGEXP`/`RLIKE`.

4. **ILIKE** = "not in the SQL standard but is a PostgreSQL extension" — don't use for portable
   case-insensitive matching.

5. **COLLATE** controls sort order AND character classification (case/accent), per-column or
   per-operation; case/accent insensitivity is realized via nondeterministic collations.

6. **Leading-wildcard LIKE** `'%term%'` / `'%x'` can't use a B-tree index → table scan. Route the
   index death to `sql-indexing-and-sargability`.

### Routing
- foundation: `sql-relational-and-null-discipline`
- general string functions (LOWER/UPPER/TRIM/SUBSTRING): `sql-expressions-case-and-functions`
- sargability / leading-wildcard / functional-index cost: `sql-indexing-and-sargability`
- CHAR vs VARCHAR, trailing-space/padding semantics: `sql-data-types-and-numerics`
- dialect spellings (ILIKE, `~`, REGEXP/RLIKE, GLOB, COLLATE names): `sql-standard-vs-dialect-map`
