---
name: sql-pattern-matching-and-collation
description: Guides correct, portable text matching in SQL — LIKE with its `%` (any sequence) and `_` (any single character) wildcards, the ESCAPE clause for literal `%`/`_`, and the central trap that LIKE's case sensitivity is decided by the active COLLATION and therefore silently differs across engines (case-SENSITIVE in standard SQL and PostgreSQL, case-INSENSITIVE under the common MySQL/SQL Server default collations, case-insensitive for ASCII only in SQLite) so the identical query returns different rows on different databases. Teaches the explicit fix — COLLATE per-operation or LOWER() on both sides — plus SIMILAR TO as the only SQL-standard regular-expression operator versus the non-standard vendor regex (`~`/`~*`, REGEXP/RLIKE), ILIKE as a non-standard PostgreSQL extension, and COLLATE for deterministic ordering and accent sensitivity. Auto-invokes when writing or editing a LIKE/ILIKE/SIMILAR TO/REGEXP/`~`/GLOB predicate, a COLLATE clause, case-insensitive or accent-insensitive search, or on "case-insensitive search", "find regardless of accents", "LIKE doesn't match" / "matched different rows on MySQL vs Postgres" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Pattern Matching and Collation

> "It is similar to `LIKE`, except that it interprets the pattern using the SQL standard's definition of a regular expression."
> — [PostgreSQL — SIMILAR TO](https://www.postgresql.org/docs/current/functions-matching.html)

> "Nondeterministic comparison may determine strings to be equal even if they consist of different bytes. Typical situations include case-insensitive comparison, accent-insensitive comparison..."
> — [PostgreSQL — Collation Support](https://www.postgresql.org/docs/current/collation.html)

Text matching looks like the simplest thing in SQL and is one of the least portable. `LIKE` itself is universal, but *whether it cares about case* is not a property of the operator — it is a property of the **collation** in force, and that default differs from engine to engine. The same `WHERE name LIKE 'abc'` matches `'ABC'` on MySQL and SQLite but not on PostgreSQL. This skill builds on the set-and-three-valued-logic floor in `sql-relational-and-null-discipline` (a `LIKE` against a NULL column is UNKNOWN and dropped by `WHERE`, exactly like any other predicate) and routes string-function and index-cost concerns to their owners.

---

## 1. `LIKE` Wildcards `%` and `_`, and `ESCAPE`

`LIKE` has exactly two wildcards. "An underscore (`_`) in *pattern* stands for (matches) any single character; a percent sign (`%`) matches any sequence of zero or more characters" ([PostgreSQL — LIKE](https://www.postgresql.org/docs/current/functions-matching.html)). Every major engine agrees: in MySQL "`%` matches any number of characters, even zero characters. `_` matches exactly one character" ([MySQL — String Comparison](https://dev.mysql.com/doc/refman/8.4/en/string-comparison-functions.html)).

To match a **literal** `%` or `_`, escape it. "To match a literal underscore or percent sign without matching other characters, the respective character in *pattern* must be preceded by the escape character. The default escape character is the backslash but a different one can be selected by using the `ESCAPE` clause" ([PostgreSQL — LIKE](https://www.postgresql.org/docs/current/functions-matching.html)).

```sql
-- WRONG — the _ is a wildcard; this matches 'first_name', 'firstXname', 'first name'…
SELECT * FROM columns_meta WHERE name LIKE 'first_name';

-- WRONG — searching for a literal 90% discount code; % matches anything, so every row matches
SELECT * FROM codes WHERE code LIKE '90%';

-- RIGHT — escape the wildcard and declare the escape char explicitly
SELECT * FROM columns_meta WHERE name LIKE 'first\_name' ESCAPE '\';
SELECT * FROM codes        WHERE code LIKE '90\%'        ESCAPE '\';
```

The default escape character is **not** portable: PostgreSQL and MySQL default it to backslash, but SQLite has no default — "If the optional ESCAPE clause is present..." is the only way to get one there ([SQLite — LIKE](https://www.sqlite.org/lang_expr.html)). Always write `ESCAPE` explicitly when matching literal wildcard characters.

---

## 2. The Centerpiece: `LIKE` Case Sensitivity Is Collation-Dependent and Not Portable

This is the single most damaging misconception about `LIKE`. Whether `LIKE` is case-sensitive is decided by the **collation**, and collations "are taken into account... by pattern matching operators" ([PostgreSQL — Collation](https://www.postgresql.org/docs/current/collation.html)). Because each engine ships a different default collation, the *same* query returns *different rows*:

| Engine | Default `LIKE` case behavior | Source |
|---|---|---|
| SQL standard / **PostgreSQL** | case-**SENSITIVE** (separate `ILIKE` needed for insensitive) | [PG — LIKE](https://www.postgresql.org/docs/current/functions-matching.html) |
| **MySQL** / SQL Server | case-**INSENSITIVE** under the common default collations | [MySQL — String Comparison](https://dev.mysql.com/doc/refman/8.4/en/string-comparison-functions.html) |
| **SQLite** | case-**INSENSITIVE for ASCII only** (case-sensitive beyond ASCII) | [SQLite — LIKE](https://www.sqlite.org/lang_expr.html) |

PostgreSQL exposes a separate keyword precisely because bare `LIKE` is case-sensitive there: "The key word `ILIKE` can be used instead of `LIKE` to make the match case-insensitive" ([PG — LIKE](https://www.postgresql.org/docs/current/functions-matching.html)). MySQL is the opposite by default: "string comparisons are not case-sensitive unless one of the operands... uses a case-sensitive collation or is a binary string" — `SELECT 'abc' LIKE 'ABC'` returns `1` ([MySQL](https://dev.mysql.com/doc/refman/8.4/en/string-comparison-functions.html)). SQLite folds case only for ASCII: "`'a' LIKE 'A'` is TRUE but `'æ' LIKE 'Æ'` is FALSE" ([SQLite](https://www.sqlite.org/lang_expr.html)).

```sql
-- WRONG — relies on an UNSTATED case behavior; returns different rows per engine.
-- Matches 'José','JOSÉ','josé' on MySQL; only the exact case on PostgreSQL.
SELECT * FROM people WHERE name LIKE 'jose%';
```

Make the intent explicit instead of inheriting the engine default. Two portable levers:

```sql
-- RIGHT (a) — fold both sides with LOWER() so case is irrelevant on every engine
SELECT * FROM people WHERE LOWER(name) LIKE LOWER('jose%');

-- RIGHT (b) — attach a collation to the operation so the rule is stated in the query
SELECT * FROM people WHERE name COLLATE "und-x-icu" LIKE 'jose%';   -- PG/ICU spelling
```

`COLLATE` "allows specifying the sort order and character classification behavior of data per-column, or even per-operation" ([PG — Collation](https://www.postgresql.org/docs/current/collation.html)) — so you can pin case behavior on a single comparison. **Index cost:** wrapping the column in `LOWER(name)` (option a) makes the predicate non-sargable — a plain B-tree index on `name` can no longer be used, forcing a scan unless you build a matching expression/functional index. That trade-off and its fix (functional index on `LOWER(name)`, or a case-insensitive collation on the column) are owned by **`sql-indexing-and-sargability`** — route there.

---

## 3. `SIMILAR TO` Is the SQL-Standard Regex; `~` / `REGEXP` / `RLIKE` Are Not

When `LIKE`'s two wildcards aren't enough, the *standard* escalation is `SIMILAR TO`: "It is similar to `LIKE`, except that it interprets the pattern using the SQL standard's definition of a regular expression" ([PG — SIMILAR TO](https://www.postgresql.org/docs/current/functions-matching.html)). Like `LIKE`, it is whole-string anchored: "the `SIMILAR TO` operator succeeds only if its pattern matches the entire string; this is unlike common regular expression behavior where the pattern can match any part of the string" ([PG](https://www.postgresql.org/docs/current/functions-matching.html)).

```sql
-- RIGHT (standard regex) — whole-string match: a 3-letter code of A–Z
SELECT * FROM items WHERE code SIMILAR TO '[A-Z]{3}';

-- NON-STANDARD (PostgreSQL only) — POSIX regex, substring match, NOT in the SQL standard
SELECT * FROM items WHERE code ~ '[A-Z]{3}';
```

Everything regex-flavored that isn't `SIMILAR TO` is a vendor extension. PostgreSQL's POSIX operators are explicit about it: "The operator `~~` is equivalent to `LIKE`, and `~~*` corresponds to `ILIKE`... All of these operators are PostgreSQL-specific" ([PG — POSIX](https://www.postgresql.org/docs/current/functions-matching.html)). MySQL and SQLite instead spell regex as `REGEXP` / `RLIKE`. None of these is portable; `SIMILAR TO` itself is supported only on some engines (notably PostgreSQL) and is rarely seen in practice. Route dialect spellings to **`sql-standard-vs-dialect-map`**.

---

## 4. `ILIKE` Is a Non-Standard PostgreSQL Extension

`ILIKE` is convenient and not portable. The documentation is explicit: case-insensitive `ILIKE` "is not in the SQL standard but is a PostgreSQL extension" ([PG — LIKE](https://www.postgresql.org/docs/current/functions-matching.html)). It also "does not support nondeterministic collations." Reach for the portable levers from §2 instead.

```sql
-- WRONG — ILIKE is PostgreSQL-only; fails to parse on MySQL, SQLite, Oracle
SELECT * FROM people WHERE name ILIKE 'jose%';

-- RIGHT — portable case-insensitive matching
SELECT * FROM people WHERE LOWER(name) LIKE 'jose%';
```

---

## 5. `COLLATE` for Comparison, Ordering, and Accent Sensitivity

Collation governs more than case — it decides accent sensitivity and the sort order of every comparison. "When the database system has to perform an ordering or a character classification, it uses the collation of the input expression" ([PG — Collation](https://www.postgresql.org/docs/current/collation.html)). Case-insensitive and accent-insensitive matching are realized through a **nondeterministic** collation, which "may determine strings to be equal even if they consist of different bytes... case-insensitive comparison, accent-insensitive comparison" ([PG — Collation](https://www.postgresql.org/docs/current/collation.html)).

The accent trap bites searches over names: under an accent-*sensitive* collation, `'José'` and `'Jose'` are different strings, so a user typing `Jose` finds nothing.

```sql
-- WRONG — accent-sensitive default collation: 'Jose' never matches stored 'José'
SELECT * FROM people WHERE name = 'Jose';

-- RIGHT — compare under an accent- and case-insensitive (nondeterministic) collation
SELECT * FROM people WHERE name = 'Jose' COLLATE "und-u-ks-level1";
```

`COLLATE` also pins a deterministic, reproducible `ORDER BY` instead of inheriting whatever the column/database default happens to be — important when output ordering must be stable across environments (a report sorted under `en_US` and another under `C`/byte-order disagree on where punctuation and case fall).

```sql
-- WRONG — sort order depends on the column's default collation; differs across DBs
SELECT name FROM people ORDER BY name;

-- RIGHT — state the collation so the ordering is reproducible everywhere
SELECT name FROM people ORDER BY name COLLATE "en-x-icu";
```

The base sort-order and character-classification concepts are deepened in `sql-relational-and-null-discipline` (ordering) and the dialect-specific collation *names* live in `sql-standard-vs-dialect-map`.

---

## 6. Leading-Wildcard `LIKE '%term%'` Cannot Use an Index

A `LIKE` whose pattern starts with a wildcard — `'%term'` or `'%term%'` — cannot use an ordinary B-tree index, because the index is ordered by leading characters and a leading `%` leaves no prefix to seek on. The result is a full table (or index) scan that degrades badly as the table grows. A *trailing*-only wildcard (`'term%'`) is sargable and can range-scan an index.

```sql
-- SLOW — leading wildcard forces a scan of every row
SELECT * FROM articles WHERE body LIKE '%postgres%';

-- SARGABLE — anchored prefix can use a B-tree index on body
SELECT * FROM articles WHERE title LIKE 'postgres%';
```

This is a sargability problem, not a correctness one. The fixes — prefix-anchored patterns, trigram/GIN indexes, or a real full-text search index — are owned by **`sql-indexing-and-sargability`**; route there for the substring-search performance pattern.

---

## 7. Portability Snapshot

`LIKE` and its `%`/`_` wildcards are universal; almost everything else here varies:

| Feature | PostgreSQL | MySQL | SQLite | Standard? |
|---|---|---|---|---|
| `LIKE` + `%`/`_` | yes | yes | yes | **yes** |
| `LIKE` default case behavior | sensitive | **insensitive** | insensitive (ASCII only) | sensitive |
| `ESCAPE` clause | yes (default `\`) | yes (default `\`) | yes (**no default**) | yes |
| `SIMILAR TO` (standard regex) | yes | no | no | **yes** (optional feature) |
| `~` / `~*` POSIX regex | yes | no | no | no (PG-specific) |
| `REGEXP` / `RLIKE` | no | yes | yes (if enabled) | no |
| `ILIKE` (case-insensitive) | yes | no | no | no (PG extension) |
| `GLOB` (`*`/`?`, case-sensitive) | no | no | yes | no (SQLite) |
| `COLLATE` clause | yes | yes | yes (named collations) | yes |

The portable kit: `LIKE` with explicit `ESCAPE`; `LOWER()` on both sides (or a stated `COLLATE`) for case-insensitive search; `SIMILAR TO` only if your engine supports it. For every dialect spelling — `ILIKE`, `~`/`~*`, `REGEXP`/`RLIKE`, `GLOB`, collation names — route to **`sql-standard-vs-dialect-map`**.

---

## 8. Who Suffers When Matching Is Mishandled

A matching bug rarely errors — it returns the wrong rows, and someone downstream pays:

- The **engineer** who tested `WHERE name LIKE 'admin%'` on a local SQLite database, watched it match `'Admin'`, shipped it, and then saw it miss every capitalized name in PostgreSQL production — the case behavior silently flipped between engines (§2). The query was "valid" on both; it just meant different things.
- The **support agent** whose customer "José" can't find his own account because the search box runs `name = 'Jose'` under an accent-sensitive collation, and `'Jose' ≠ 'José'` (§5). No error, no result, an angry customer.
- The **analyst** whose `LIKE '%term%'` report ran in 40 ms against ten thousand rows in staging and times out against ten million in production, because the leading wildcard forced a full scan that no index could rescue (§6).
- The **maintainer** who copied a working `ILIKE` query from a PostgreSQL service into a MySQL one and got a syntax error at deploy time (§4).

Explicit matching is empathy in code: the `LOWER()` on both sides, the stated `COLLATE`, the escaped literal `%`, and the anchored prefix are all gifts to whoever runs this query on the next engine.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics and three-valued logic; a `LIKE` against NULL is UNKNOWN and dropped by `WHERE`, and `ORDER BY`/collation ordering basics.
- `sql-expressions-case-and-functions` — general string functions (`LOWER`, `UPPER`, `TRIM`, `SUBSTRING`, `||`/`CONCAT`) used to build and normalize the strings you match on.
- `sql-indexing-and-sargability` — why `LOWER(col)` and leading-wildcard `LIKE '%x%'` are non-sargable, and the functional-index / trigram / full-text fixes (§2, §6).
- `sql-data-types-and-numerics` — `CHAR` vs `VARCHAR` and trailing-space/padding semantics that interact with comparison and matching.
- `sql-standard-vs-dialect-map` — dialect spellings: `ILIKE`, POSIX `~`/`~*`, `REGEXP`/`RLIKE`, `GLOB`, and collation names per engine.

---

## 10. Reference Files

High-frequency pattern-matching and collation anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
