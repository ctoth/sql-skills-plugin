# Research: sql-style-and-naming

Research for SKILL `sql-style-and-naming` (skill #22 in `reports/skill-plan-sql.md`, lines 320-328).
Accessed: 2026-06-26. Every entry: URL, section, verbatim quote, why it matters.

The centerpiece is the **correctness trap** in quoting: `'...'` is a string literal,
`"..."` is a delimited identifier per the SQL standard. Writing `"value"` where you meant
`'value'` references a column/identifier and usually errors — or silently misbehaves on
MySQL, which by default treats `"..."` as a string until `ANSI_QUOTES` flips it.

---

## 1. PostgreSQL — Lexical Structure (THE KEY SOURCE)

URL: https://www.postgresql.org/docs/current/sql-syntax-lexical.html

### 1a. String constants = single quotes
Section: "4.1.2.1. String Constants"
> "A string constant in SQL is an arbitrary sequence of characters bounded by single quotes
> (`'`), for example `'This is a string'`. To include a single-quote character within a string
> constant, write two adjacent single quotes, e.g., `'Dianne''s horse'`. Note that this is _not_
> the same as a double-quote character (`"`)."

WHY: Establishes that `'...'` is the value side. The closing sentence explicitly warns the two
quote characters are different things — the seed of the whole correctness trap.

### 1b. Double quotes = delimited identifiers, never keywords/strings
Section: "4.1.1. Identifiers and Key Words"
> "There is a second kind of identifier: the _delimited identifier_ or _quoted identifier_. It is
> formed by enclosing an arbitrary sequence of characters in double-quotes (`"`). A delimited
> identifier is always an identifier, never a key word. So `"select"` could be used to refer to a
> column or table named "select", whereas an unquoted `select` would be taken as a key word and
> would therefore provoke a parse error when used where a table or column name is expected."

WHY: `"..."` is the *name* side, "always an identifier, never a key word." Therefore `"John"` is
a reference to a thing named John, not the text John. This is the literal-vs-identifier rule, from
the standard's own implementation.

### 1c. Case-folding: PG lowercases, standard uppercases
Section: "4.1.1. Identifiers and Key Words"
> "Key words and unquoted identifiers are case-insensitive. ... Quoting an identifier also makes it
> case-sensitive, whereas unquoted names are always folded to lower case. For example, the
> identifiers `FOO`, `foo`, and `"foo"` are considered the same by PostgreSQL, but `"Foo"` and
> `"FOO"` are different from these three and each other. (The folding of unquoted names to lower
> case in PostgreSQL is incompatible with the SQL standard, which says that unquoted names should
> be folded to upper case. Thus, `foo` should be equivalent to `"FOO"` not `"foo"` according to the
> standard.)"

WHY: This is why a CamelCase identifier forces permanent quoting. If you `CREATE TABLE ... ("userName" ...)`,
the column is pinned to exact-case `userName`; an unquoted `userName` folds to `username` and won't
match, so every future reference must keep the double quotes. And the fold direction differs by
engine (PG→lower, standard→upper), so an unquoted CamelCase name is not even portable. Routes to
`sql-standard-vs-dialect-map`.

### 1d. Keyword-casing convention
Section: "4.1.1. Identifiers and Key Words"
> "A convention often used is to write key words in upper case and names in lower case, e.g.:
> `UPDATE my_table SET a = 5;`"

WHY: Primary-source backing for the UPPER keywords / lower identifiers convention. Note `my_table`
in the canonical example — snake_case, lowercase.

---

## 2. SQL Style Guide (Simon Holywell)

URL: https://www.sqlstyle.guide/

### 2a. snake_case + lowercase
Section: "Naming conventions / General"
> "Use underscores where you would naturally include a space in the name (first name becomes
> `first_name`)."
> "Always use lowercase except where it may make sense not to such as proper nouns."

WHY: Backs snake_case + lowercase identifiers — the convention that means you never need permanent
double-quoting (lowercase snake_case survives case-folding unchanged in every engine).

### 2b. Avoid reserved words
Section: "Naming conventions / General"
> "Ensure the name is unique and does not exist as a reserved keyword."

WHY: Backs the reserved-word-collision section: pick names that aren't keywords so you never need
to quote them defensively.

### 2c. Identifier rules
Section: "Naming conventions / General"
> "Names must begin with a letter and may not end with an underscore."
> "Only use letters, numbers and underscores in names."
> "Keep the length to a maximum of 30 bytes—in practice this is 30 characters unless you are using
> a multi-byte character set."

WHY: Concrete naming rules.

### 2d. UPPERCASE keywords
Section: "Reserved keywords"
> "Always use uppercase for the reserved keywords like `SELECT` and `WHERE`."

WHY: Second independent source for keyword casing (pairs with PG 1d).

### 2e. The "river" alignment
Section: "Spaces"
> "Spaces should be used to line up the code so that the root keywords all end on the same character
> boundary. This forms a river down the middle making it easy for the readers eye to scan over the
> code and separate the keywords from the implementation detail."

WHY: Backs the multi-line layout / reviewability section — alignment is for the reader, not the parser.

### 2f. Space rules (incl. apostrophes around literals)
Section: "Spaces"
> "Always include spaces: before and after equals (`=`); after commas (`,`); surrounding apostrophes (`'`)."

WHY: Minor layout backing; reinforces `'...'` = the literal.

---

## 3. MySQL — String Literals (the deviation)

URL: https://dev.mysql.com/doc/refman/8.0/en/string-literals.html

Section: opening + ANSI_QUOTES note
> "A string is a sequence of bytes or characters, enclosed within either single quote (`'`) or
> double quote (`"`) characters."

> "If the `ANSI_QUOTES` SQL mode is enabled, string literals can be quoted only within single
> quotation marks because a string quoted within double quotation marks is interpreted as an
> identifier."

WHY: THE LANDMINE. MySQL's *default* accepts `"abc"` as a string, so `WHERE name = "John"` silently
works on MySQL — until someone enables `ANSI_QUOTES` (or the code is ported to PG/standard), at which
point `"John"` becomes an identifier reference and the query breaks. Same text, opposite meaning,
depending on a server flag.

---

## 4. MySQL — Server SQL Modes (ANSI_QUOTES definition)

URL: https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html

Section: "ANSI_QUOTES"
> "Treat `"` as an identifier quote character (like the `` ` `` quote character) and not as a string
> quote character. You can still use `` ` `` to quote identifiers with this mode enabled. With
> `ANSI_QUOTES` enabled, you cannot use double quotation marks to quote literal strings because they
> are interpreted as identifiers."

WHY: Exact definition of the flag that flips MySQL onto the standard behavior. Also documents the
backtick (`` ` ``) as MySQL's native identifier quote — a dialect spelling for `sql-standard-vs-dialect-map`.

---

## 5. modern-sql.com — Troublesome words in SQL (reserved words)

URL: https://modern-sql.com/reserved-words-empirical-list

Section: intro
> "Note that you can still use these words as identifiers by putting them under double quotes (`"`)."

WHY: Confirms double quotes are the standard escape for using a reserved word as an identifier — and
implicitly, the discipline is to *avoid* needing it by not naming things after keywords.

---

## Synthesis → skill sections

1. **CENTERPIECE — `'literal'` vs `"identifier"`**: 1a + 1b (standard rule) + 3 + 4 (MySQL landmine).
   WRONG `WHERE name = "John"` → on standard/PG: `column "John" does not exist`; on default MySQL:
   silently works, breaks under ANSI_QUOTES. RIGHT `WHERE name = 'John'`.
2. **Keyword casing**: 1d + 2d. UPPER keywords, lower identifiers.
3. **Identifier naming (snake_case)**: 2a + 2c + 1c. snake_case lowercase so case-folding never bites;
   CamelCase forces permanent quoting via case-folding (1c).
4. **Reserved words**: 2b + 5. Don't name things after keywords; if forced, double-quote.
5. **Comma style**: leading vs trailing (style preference; no hard source — present as convention,
   tie to reviewability/diff).
6. **Multi-line layout**: 2e (river) + 2f. One clause per line, JOIN/CTE/predicate per line, for diffs.
7. **Portability block**: quote rule is SQL-92 standard (1a/1b); MySQL deviates without ANSI_QUOTES
   (3/4); case-folding PG-lower vs standard-upper (1c) → route to `sql-standard-vs-dialect-map`.
8. **Who suffers**: reviewer who can't diff a single-line 300-char query (2e); newcomer baffled by
   `column "John" does not exist` (1b); column forever needing quotes because someone named it
   `"userName"` (1c).
9. **Routing**: foundation (`sql-relational-and-null-discipline`), `sql-select-and-query-processing`
   (clause order), `sql-standard-vs-dialect-map` (ANSI_QUOTES, backticks, fold direction).
