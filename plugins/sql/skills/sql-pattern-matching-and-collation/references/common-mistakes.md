# Common SQL Pattern-Matching & Collation Mistakes
## Contents

- [1. Assuming LIKE case behavior is portable](#1-assuming-like-case-behavior-is-portable)
- [2. Unescaped % or  when matching a literal wildcard](#2-unescaped-or-when-matching-a-literal-wildcard)
- [3. Using ILIKE for portable case-insensitive matching](#3-using-ilike-for-portable-case-insensitive-matching)
- [4. Reaching for ~ / REGEXP / RLIKE when standard regex (or LIKE) would do](#4-reaching-for-regexp-rlike-when-standard-regex-or-like-would-do)
- [5. Accent-sensitive collation hiding rows from search](#5-accent-sensitive-collation-hiding-rows-from-search)
- [6. Leading-wildcard LIKE '%term%' forcing a table scan](#6-leading-wildcard-like-term-forcing-a-table-scan)
- [7. Relying on SQLite's ASCII-only case folding](#7-relying-on-sqlites-ascii-only-case-folding)


Anti-patterns in LLM-generated SQL around text matching and collation, each with wrong/right code
and a primary-source citation. The skill (`sql-pattern-matching-and-collation`) states the rules;
this file holds the high-frequency failure modes. All RIGHT examples favor standard/portable SQL;
non-standard spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Assuming `LIKE` case behavior is portable

**The problem:** The model writes `WHERE name LIKE 'jose%'` and assumes one case behavior. But
case sensitivity is collation-driven: PostgreSQL/standard is case-SENSITIVE, while "string
comparisons are not case-sensitive unless one of the operands... uses a case-sensitive collation"
in MySQL — `SELECT 'abc' LIKE 'ABC'` returns `1` ([MySQL — String Comparison](https://dev.mysql.com/doc/refman/8.4/en/string-comparison-functions.html)). The identical query returns different rows on different engines.

```sql
-- WRONG — inherits the engine default; matches 'JOSÉ' on MySQL, not on PostgreSQL
SELECT * FROM people WHERE name LIKE 'jose%';

-- RIGHT — fold both sides so case is irrelevant everywhere
SELECT * FROM people WHERE LOWER(name) LIKE LOWER('jose%');
```

*Source: [MySQL — String Comparison](https://dev.mysql.com/doc/refman/8.4/en/string-comparison-functions.html); [PostgreSQL — LIKE](https://www.postgresql.org/docs/current/functions-matching.html). Depth: this skill, §2. Index cost owned by `sql-indexing-and-sargability`.*

---

## 2. Unescaped `%` or `_` when matching a literal wildcard

**The problem:** The model searches for a value that contains `%` or `_` literally (a discount
code `90%`, a column named `first_name`) and writes it straight into the pattern, where it acts as
a wildcard. "To match a literal underscore or percent sign... the respective character in *pattern*
must be preceded by the escape character" ([PostgreSQL — LIKE](https://www.postgresql.org/docs/current/functions-matching.html)).

```sql
-- WRONG — _ matches any char; 'firstXname' and 'first name' also match
SELECT * FROM cols WHERE name LIKE 'first_name';

-- RIGHT — escape the literal and state the escape char (SQLite has no default)
SELECT * FROM cols WHERE name LIKE 'first\_name' ESCAPE '\';
```

*Source: [PostgreSQL — LIKE](https://www.postgresql.org/docs/current/functions-matching.html); [SQLite — LIKE](https://www.sqlite.org/lang_expr.html). Depth: this skill, §1.*

---

## 3. Using `ILIKE` for portable case-insensitive matching

**The problem:** The model reaches for PostgreSQL's `ILIKE`, which is convenient but not portable.
Case-insensitive `ILIKE` "is not in the SQL standard but is a PostgreSQL extension" ([PostgreSQL — LIKE](https://www.postgresql.org/docs/current/functions-matching.html)) — it is a parse error on MySQL, SQLite, and Oracle.

```sql
-- WRONG — PostgreSQL-only; fails to parse elsewhere
SELECT * FROM people WHERE name ILIKE 'jose%';

-- RIGHT — portable equivalent
SELECT * FROM people WHERE LOWER(name) LIKE 'jose%';
```

*Source: [PostgreSQL — Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html). Depth: this skill, §4; dialect spellings owned by `sql-standard-vs-dialect-map`.*

---

## 4. Reaching for `~` / `REGEXP` / `RLIKE` when standard regex (or LIKE) would do

**The problem:** The model writes vendor regex without realizing it locks the query to one engine.
PostgreSQL's POSIX operators are "PostgreSQL-specific" ([PostgreSQL — POSIX Regex](https://www.postgresql.org/docs/current/functions-matching.html)); MySQL/SQLite use `REGEXP`/`RLIKE`. The SQL-standard regex operator is `SIMILAR TO`, which "interprets the pattern using the SQL standard's definition of a regular expression" ([PostgreSQL — SIMILAR TO](https://www.postgresql.org/docs/current/functions-matching.html)).

```sql
-- WRONG — POSIX ~ is PostgreSQL-only and non-standard
SELECT * FROM items WHERE code ~ '^[A-Z]{3}$';

-- RIGHT (standard, where supported) — SIMILAR TO is whole-string anchored
SELECT * FROM items WHERE code SIMILAR TO '[A-Z]{3}';

-- RIGHT (simplest) — if you only need a prefix, LIKE is universal and sargable
SELECT * FROM items WHERE code LIKE 'ABC%';
```

*Source: [PostgreSQL — Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html). Depth: this skill, §3; dialect spellings owned by `sql-standard-vs-dialect-map`.*

---

## 5. Accent-sensitive collation hiding rows from search

**The problem:** The model compares user input to stored names under an accent-sensitive collation,
so `'Jose'` never matches `'José'`. A nondeterministic collation is what makes accent- and
case-insensitive comparison possible: it "may determine strings to be equal even if they consist of
different bytes... accent-insensitive comparison" ([PostgreSQL — Collation](https://www.postgresql.org/docs/current/collation.html)).

```sql
-- WRONG — accent-sensitive: 'Jose' ≠ 'José', the user finds nothing
SELECT * FROM people WHERE name = 'Jose';

-- RIGHT — compare under an accent/case-insensitive (nondeterministic) collation
SELECT * FROM people WHERE name = 'Jose' COLLATE "und-u-ks-level1";
```

*Source: [PostgreSQL — Collation Support](https://www.postgresql.org/docs/current/collation.html). Depth: this skill, §5; collation names owned by `sql-standard-vs-dialect-map`.*

---

## 6. Leading-wildcard `LIKE '%term%'` forcing a table scan

**The problem:** The model implements substring search with a leading wildcard, which no B-tree
index can satisfy — the index is ordered by leading characters and a leading `%` leaves no prefix
to seek. It works in testing and times out in production.

```sql
-- SLOW — leading wildcard scans every row
SELECT * FROM articles WHERE body LIKE '%postgres%';

-- SARGABLE — an anchored prefix can range-scan a B-tree index on title
SELECT * FROM articles WHERE title LIKE 'postgres%';
```

*Source: this skill, §6. The substring-search fixes (trigram/GIN index, full-text search) are owned by `sql-indexing-and-sargability`.*

---

## 7. Relying on SQLite's ASCII-only case folding

**The problem:** The model assumes SQLite's case-insensitive `LIKE` covers all text. It folds case
only for ASCII: "SQLite only understands upper/lower case for ASCII characters by default... `'a'
LIKE 'A'` is TRUE but `'æ' LIKE 'Æ'` is FALSE" ([SQLite — LIKE](https://www.sqlite.org/lang_expr.html)). Non-ASCII text silently behaves case-sensitively.

```sql
-- WRONG (SQLite) — assumes 'æ' folds like ASCII; 'æ' LIKE 'Æ' is FALSE
SELECT * FROM terms WHERE word LIKE 'Æ%';

-- RIGHT — normalize case explicitly rather than trusting the default folding
SELECT * FROM terms WHERE LOWER(word) LIKE LOWER('Æ%');
```

*Source: [SQLite — LIKE](https://www.sqlite.org/lang_expr.html). Depth: this skill, §2.*
