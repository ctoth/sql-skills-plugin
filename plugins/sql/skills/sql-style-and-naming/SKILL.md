---
name: sql-style-and-naming
description: Guides readable, reviewable SQL and the one genuine correctness trap in formatting — `'single quotes'` are string literals and `"double quotes"` are delimited identifiers per the SQL standard, so `WHERE name = "John"` references a column named John (error on PostgreSQL, silent misbehavior on MySQL until ANSI_QUOTES flips it). Covers keyword casing (UPPER keywords, lower names), snake_case identifier naming so columns never need permanent double-quoting, why a CamelCase name forces forever-quoting via case-folding, reserved-word collisions, leading-vs-trailing comma style, and multi-line layout instead of single-line mega-queries. Auto-invokes when writing or editing SQL with string-vs-identifier quoting, choosing identifier names, formatting/laying out a query, or on "clean up / format this SQL" requests. A foundation-level style policy.
allowed-tools: Read, Glob, Grep
---

# SQL Style and Naming

> "A string constant in SQL is an arbitrary sequence of characters bounded by single quotes (`'`) ... Note that this is _not_ the same as a double-quote character (`"`)."
> — [PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)

> "A delimited identifier is always an identifier, never a key word. So `"select"` could be used to refer to a column or table named "select" ... A convention often used is to write key words in upper case and names in lower case."
> — [PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)

Most "style" is taste, but one formatting choice in SQL is a hard correctness rule: **`'...'` and `"..."` are not interchangeable**. Single quotes make a *value*; double quotes make a *name*. Confuse them and your query references a column that does not exist — and on MySQL it may "work" silently until a server flag changes underneath you. Everything else here (casing, naming, layout) is about making SQL diffable and reviewable so the next reader can spot the bug. This skill builds on the set/NULL semantics in `sql-relational-and-null-discipline`; clause *logical order* is owned by `sql-select-and-query-processing`.

---

## 1. `'literal'` vs `"identifier"` — the Correctness Trap (the centerpiece)

This is the one rule in this skill that is not a preference. In standard SQL the two quote characters mean different things:

- **Single quotes delimit a string constant (a value).** "A string constant in SQL is an arbitrary sequence of characters bounded by single quotes (`'`) ... Note that this is _not_ the same as a double-quote character (`"`)" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)).
- **Double quotes delimit an identifier (a name).** "A delimited identifier is always an identifier, never a key word. So `"select"` could be used to refer to a column or table named "select"" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)).

So `"John"` is a reference to *a column/table named John*, not the text `John`. Compare against a string with single quotes:

```sql
-- WRONG — "John" is an identifier; this means "the column John"
-- On PostgreSQL/standard: ERROR: column "John" does not exist
SELECT * FROM users WHERE name = "John";

-- RIGHT — 'John' is a string literal (a value)
SELECT * FROM users WHERE name = 'John';
```

To put a single quote *inside* a string, double it — `'Dianne''s horse'` — never reach for `"` to escape it ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)).

### The MySQL `ANSI_QUOTES` landmine

The trap is worse than a clean error, because MySQL's **default** mode accepts double quotes as a string delimiter: "A string is a sequence of bytes or characters, enclosed within either single quote (`'`) or double quote (`"`) characters" ([MySQL — String Literals](https://dev.mysql.com/doc/refman/8.0/en/string-literals.html)). So `WHERE name = "John"` *silently works* on a default MySQL — masking the bug — until someone enables the `ANSI_QUOTES` mode, which makes MySQL "treat `"` as an identifier quote character ... and not as a string quote character," after which "you cannot use double quotation marks to quote literal strings because they are interpreted as identifiers" ([MySQL — Server SQL Modes](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)). The same query then breaks, or worse, resolves to a same-named column. **Always single-quote strings; never depend on MySQL's lax default.** The `ANSI_QUOTES`/backtick dialect detail is owned by `sql-standard-vs-dialect-map`.

---

## 2. Keyword Casing — UPPER Keywords, lower Names

The widely-followed convention is "to write key words in upper case and names in lower case" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)); the SQL Style Guide states it as a rule: "Always use uppercase for the reserved keywords like `SELECT` and `WHERE`" ([SQL Style Guide](https://www.sqlstyle.guide/)). Casing has no effect on execution (keywords are case-insensitive), but the visual contrast lets a reader separate the SQL skeleton from your table and column names at a glance.

```sql
-- WRONG — keywords and names blur together
select id, total from orders where status = 'paid' order by created_at;

-- RIGHT — UPPER keywords frame the lowercase names
SELECT id, total FROM orders WHERE status = 'paid' ORDER BY created_at;
```

---

## 3. Identifier Naming — `snake_case`, and Why CamelCase Forces Permanent Quoting

Name identifiers in lowercase `snake_case`: "Use underscores where you would naturally include a space in the name (first name becomes `first_name`)" and "Always use lowercase except where it may make sense not to such as proper nouns" ([SQL Style Guide](https://www.sqlstyle.guide/)). This is not only aesthetic — it sidesteps **case-folding**.

Unquoted identifiers are case-insensitive and get folded to a canonical case: "unquoted names are always folded to lower case ... the identifiers `FOO`, `foo`, and `"foo"` are considered the same by PostgreSQL, but `"Foo"` and `"FOO"` are different" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)). The instant you create a column with double quotes around a mixed-case name, you pin its case forever — and every future reference must keep the quotes:

```sql
-- WRONG — the quotes pin the exact case; userName is now FOREVER "userName"
CREATE TABLE accounts ("userName" text);
SELECT userName FROM accounts;     -- folds to "username" -> ERROR, no such column
SELECT "userName" FROM accounts;   -- the ONLY way to reference it, forever

-- RIGHT — snake_case survives folding; never needs a quote again
CREATE TABLE accounts (user_name text);
SELECT user_name FROM accounts;    -- works unquoted, in any case
```

The fold *direction* is itself non-portable — PostgreSQL folds unquoted names to lower case, but "the SQL standard ... says that unquoted names should be folded to upper case" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)). So an unquoted mixed-case name is doubly fragile. Lowercase `snake_case` is the only spelling that means the same thing everywhere without quotes.

---

## 4. Reserved Words — Don't Name Things After Keywords

Picking a name that is a reserved word forces you to double-quote it everywhere (or hit a parse error). The fix is to not do it: "Ensure the name is unique and does not exist as a reserved keyword" ([SQL Style Guide](https://www.sqlstyle.guide/)). You *can* quote your way out — "you can still use these words as identifiers by putting them under double quotes (`"`)" ([modern-sql.com — Troublesome words in SQL](https://modern-sql.com/reserved-words-empirical-list)) — but a column you must quote in every query is a permanent tax.

```sql
-- WRONG — order, user, value, end are reserved; need quoting forever
CREATE TABLE "order" ("user" text, "value" int, "end" date);

-- RIGHT — non-reserved, descriptive names need no quoting
CREATE TABLE purchase (customer text, amount int, ended_on date);
```

The set of reserved words differs by engine; the portable habit is to avoid the common ones (`order`, `user`, `value`, `end`, `select`, `table`, `group`) entirely. Route per-engine reserved-word lists to `sql-standard-vs-dialect-map`.

---

## 5. Comma Style — Pick One, Make Diffs Clean

Standard SQL allows commas only *between* items (no trailing comma before `FROM`), so the real choice is leading vs trailing. Either is fine; consistency is the point. Leading commas make it obvious when the final item lacks a separator and keep each added line a clean one-line diff:

```sql
-- Trailing-comma style
SELECT
    id,
    customer_id,
    total
FROM orders;

-- Leading-comma style — adding a column touches exactly one line
SELECT
      id
    , customer_id
    , total
FROM orders;
```

What matters: **one column per line** once the list is non-trivial, so a code review shows exactly which column was added or removed rather than a re-wrapped blob.

---

## 6. Multi-line Layout — for the Reviewer, Not the Parser

The parser does not care about whitespace; the human diffing your query does. The SQL Style Guide's "river" idea is to align root keywords: "Spaces should be used to line up the code so that the root keywords all end on the same character boundary. This forms a river down the middle making it easy for the readers eye to scan over the code" ([SQL Style Guide](https://www.sqlstyle.guide/)). Put each clause, each `JOIN`, and each `AND`/`OR` predicate on its own line:

```sql
-- WRONG — a 300-char single line; a one-token change re-diffs the whole thing
SELECT o.id, c.name, SUM(li.qty*li.price) AS total FROM orders o JOIN customers c ON c.id=o.customer_id JOIN line_items li ON li.order_id=o.id WHERE o.status='paid' AND o.created_at>='2026-01-01' GROUP BY o.id, c.name;

-- RIGHT — one clause/join/predicate per line; diffs point at the exact change
SELECT o.id,
       c.name,
       SUM(li.qty * li.price) AS total
FROM orders AS o
JOIN customers AS c   ON c.id = o.customer_id
JOIN line_items AS li ON li.order_id = o.id
WHERE o.status = 'paid'
  AND o.created_at >= '2026-01-01'
GROUP BY o.id, c.name;
```

The same applies to CTEs: give each `WITH` subquery its own block so the query reads as named, reviewable steps. CTE mechanics live in `sql-cte-and-recursion`; here the rule is just *break it onto lines*.

---

## 7. Portability Snapshot

The quote rule is SQL-92 standard and consistent where it matters; the deviations to watch are about *who breaks your query and when*:

| Behavior | Standard / PostgreSQL | MySQL (default) | MySQL (`ANSI_QUOTES`) |
|---|---|---|---|
| `'abc'` | string literal | string literal | string literal |
| `"abc"` | **identifier** (name) | **string literal** (deviation) | identifier (name) |
| Native identifier quote | `"..."` | `` `...` `` (backtick) | `"..."` or `` `...` `` |
| Unquoted name case-folds to | lower (PG) / upper (standard) | (case-sensitivity is filesystem-dependent) | same as default |

The dangerous cell is MySQL's default `"abc"` = string: it lets `= "John"` pass review and CI on MySQL, then fail on PostgreSQL or under `ANSI_QUOTES`. The case-fold split (PG lower vs standard upper) means an unquoted CamelCase name is not portable either. **Write to the standard: single-quote every string, lowercase-snake_case every name, and you never touch either deviation.** Per-engine quoting modes, backticks, and fold direction are owned by `sql-standard-vs-dialect-map`.

---

## 8. Who Suffers When Style Is Sloppy

Bad SQL style rarely errors at write time — it costs someone else later:

- The **reviewer** staring at a 300-character single-line query in a pull request, unable to tell which of twelve predicates changed because the whole line re-diffs on any edit (§6). One clause per line would have made the change a single highlighted row.
- The **newcomer** who wrote `WHERE name = "John"`, watched it work on the team's MySQL box, and then spent an afternoon baffled by `ERROR: column "John" does not exist` after it shipped to the PostgreSQL reporting replica (§1). Nothing in the code looked wrong; the quote character was the bug.
- The **maintainer** who inherits a column named `"userName"` and must double-quote it in every query, every view, every migration — forever — because someone created it CamelCase and the case got pinned (§3). A `user_name` column would have just worked.
- The **on-call engineer** who can't grep the codebase for a table because half the queries call it `Orders`, half `"order"`, and the keyword collision means tooling chokes on it (§4).

Consistent style is empathy in code: the `'string'` you single-quoted, the `snake_case` name you chose, and the one-clause-per-line layout are all gifts to whoever debugs this under load.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation/policy root: set semantics and three-valued logic that all SQL assumes.
- `sql-select-and-query-processing` — the *logical* clause evaluation order (FROM → WHERE → GROUP BY → … → ORDER BY); this skill governs how you *lay out* those clauses, not how they execute.
- `sql-standard-vs-dialect-map` — `ANSI_QUOTES`, MySQL backtick identifiers, per-engine reserved-word lists, and the case-fold direction (PG-lower vs standard-upper).

---

## 10. Reference Files

High-frequency style and quoting anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
