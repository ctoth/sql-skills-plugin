# Common SQL Style & Naming Mistakes

Anti-patterns in LLM-generated SQL around quoting, casing, naming, and layout, each with
wrong/right code and a primary-source citation. The policy (`sql-style-and-naming`) states the
rules; this file holds the high-frequency failure modes. All RIGHT examples are standard/portable
SQL; dialect-specific spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Double-quoting a string literal (the correctness trap)

**The problem:** The model writes `"..."` for a text value (often translating from a language where
double quotes make strings). In standard SQL `"John"` is an *identifier* — "A delimited identifier is
always an identifier, never a key word" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)) — so PostgreSQL raises `column "John" does not exist`.

```sql
-- WRONG — "John" means "the column John"; ERROR on PostgreSQL/standard
SELECT * FROM users WHERE name = "John";

-- RIGHT — single quotes make a string value
SELECT * FROM users WHERE name = 'John';
```

*Source: [PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html). Depth: this skill, §1.*

---

## 2. Relying on MySQL's default double-quoted strings

**The problem:** The query "works" on the team's MySQL box because MySQL's default mode treats `"..."`
as a string, then breaks when ported to PostgreSQL or when `ANSI_QUOTES` is enabled. With `ANSI_QUOTES`,
"you cannot use double quotation marks to quote literal strings because they are interpreted as
identifiers" ([MySQL — Server SQL Modes](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)).

```sql
-- WRONG — silently works on default MySQL, breaks under ANSI_QUOTES / on PostgreSQL
INSERT INTO logs (level) VALUES ("error");

-- RIGHT — single quotes are a string literal everywhere
INSERT INTO logs (level) VALUES ('error');
```

*Source: [MySQL — String Literals](https://dev.mysql.com/doc/refman/8.0/en/string-literals.html); [MySQL — Server SQL Modes](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html). Depth: this skill, §1; dialect detail owned by `sql-standard-vs-dialect-map`.*

---

## 3. Escaping a quote with the wrong character

**The problem:** To put an apostrophe inside a string the model uses a backslash or a double quote,
not the standard doubled single quote. "To include a single-quote character within a string constant,
write two adjacent single quotes, e.g., `'Dianne''s horse'`" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)).

```sql
-- WRONG — backslash escaping is not portable; "..." is an identifier
SELECT 'Dianne\'s horse';
SELECT "Dianne's horse";

-- RIGHT — double the single quote
SELECT 'Dianne''s horse';
```

*Source: [PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html). Depth: this skill, §1.*

---

## 4. CamelCase identifiers that force permanent quoting

**The problem:** The model creates a column `"userName"`, pinning its exact case, because "unquoted
names are always folded to lower case" so an unquoted `userName` resolves to `username` and fails to
match ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)). Every future reference must keep the quotes.

```sql
-- WRONG — pins the case; userName now requires quotes in every query forever
CREATE TABLE accounts ("userName" text, "createdAt" timestamptz);

-- RIGHT — snake_case survives case-folding; never needs a quote
CREATE TABLE accounts (user_name text, created_at timestamptz);
```

*Source: [PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html); [SQL Style Guide](https://www.sqlstyle.guide/). Depth: this skill, §3.*

---

## 5. Naming a column after a reserved word

**The problem:** Names like `order`, `user`, `value`, `end` are reserved words, so they parse-error or
demand quoting everywhere. "Ensure the name is unique and does not exist as a reserved keyword"
([SQL Style Guide](https://www.sqlstyle.guide/)); you *can* quote them ([modern-sql.com](https://modern-sql.com/reserved-words-empirical-list)) but it is a permanent tax.

```sql
-- WRONG — reserved words need quoting in every reference
CREATE TABLE "order" ("user" text, "value" int);

-- RIGHT — non-reserved, descriptive names
CREATE TABLE purchase (customer text, amount int);
```

*Source: [SQL Style Guide](https://www.sqlstyle.guide/); [modern-sql.com — Troublesome words in SQL](https://modern-sql.com/reserved-words-empirical-list). Depth: this skill, §4.*

---

## 6. Lowercase keywords blurring into names

**The problem:** All-lowercase SQL makes keywords and identifiers visually indistinguishable. The
convention is "to write key words in upper case and names in lower case" ([PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)); "Always use uppercase for the reserved keywords" ([SQL Style Guide](https://www.sqlstyle.guide/)).

```sql
-- WRONG — where does the SQL end and the schema begin?
select id, status from orders where status = 'paid';

-- RIGHT — UPPER keywords frame lowercase names
SELECT id, status FROM orders WHERE status = 'paid';
```

*Source: [PostgreSQL — Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html); [SQL Style Guide](https://www.sqlstyle.guide/). Depth: this skill, §2.*

---

## 7. The single-line mega-query

**The problem:** The model emits the whole query on one line; any edit re-diffs the entire statement
and reviewers can't isolate the change. The "river" alignment exists so "the readers eye [can] scan
over the code and separate the keywords from the implementation detail" ([SQL Style Guide](https://www.sqlstyle.guide/)).

```sql
-- WRONG — one long line; un-reviewable diff
SELECT o.id, c.name FROM orders o JOIN customers c ON c.id=o.customer_id WHERE o.status='paid' AND o.total>100;

-- RIGHT — one clause/join/predicate per line
SELECT o.id,
       c.name
FROM orders AS o
JOIN customers AS c ON c.id = o.customer_id
WHERE o.status = 'paid'
  AND o.total > 100;
```

*Source: [SQL Style Guide](https://www.sqlstyle.guide/). Depth: this skill, §6.*

---

## 8. Inconsistent comma placement within a list

**The problem:** Mixing leading and trailing commas (or cramming the select list onto one line) makes
column-level diffs noisy. Either style is fine; consistency and one-column-per-line are the point.

```sql
-- WRONG — mixed styles, multiple columns per line
SELECT id, customer_id
    ,total, status FROM orders;

-- RIGHT — one column per line, consistent comma style
SELECT
    id,
    customer_id,
    total,
    status
FROM orders;
```

*Source: [SQL Style Guide](https://www.sqlstyle.guide/). Depth: this skill, §5.*
