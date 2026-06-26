# Research: sql-injection-and-parameterization

Skill #14 in the standard-SQL skills plugin (spec: `reports/skill-plan-sql.md` lines 224-232).
Highest-stakes skill in the set. Every claim below is sourced to a primary URL with the section it
came from, a verbatim (or near-verbatim, as returned by fetch) quote, and why it matters for the skill.

Accessed: 2026-06-26.

---

## Source 1 — OWASP: SQL Injection (the attack)

URL: https://owasp.org/www-community/attacks/SQL_Injection

### What it is (Overview / Description)
> "A SQL injection attack consists of insertion or 'injection' of a SQL query via the input data from
> the client to the application."

> "SQL injection attack occurs when: An unintended data enters a program from an untrusted source. The
> data is used to dynamically construct a SQL query"

**Why it matters:** Defines the failure precisely — injection is what happens when *data* is allowed to
become *code* because the query was built by string construction. This is the framing for "THE RULE":
data must stay data. It anchors the whole skill in the root cause, not the symptom.

### Impact (Threat Modeling)
> "SQL injection attacks allow attackers to spoof identity, tamper with existing data, cause repudiation
> issues such as voiding transactions or changing balances, allow the complete disclosure of all data on
> the system, destroy the data or make it otherwise unavailable, or become administrators of the database
> server."

> (Confidentiality) "Since SQL databases generally hold sensitive data, loss of confidentiality is a
> frequent problem with SQL Injection vulnerabilities."

> (Authentication) "If poor SQL commands are used to check user names and passwords, it may be possible to
> connect to a system as another user with no previous knowledge of the password."

> (Integrity) "It is also possible to make changes or even delete this information with a SQL Injection
> attack."

**Why it matters:** Supplies the stakes for the pull-quote and the "Who suffers" section — full data
disclosure, auth bypass, deletion, DB takeover. Justifies "highest-stakes skill" framing.

### The classic `' OR '1'='1` example (Example 2)
> "If an attacker with the user name wiley enters the string 'name' OR 'a'='a' for itemName, then the
> query becomes the following: SELECT * FROM items WHERE owner = 'wiley' AND itemname = 'name' OR 'a'='a';"

**Why it matters:** The canonical demonstration that interpolated input changes query *structure* (an
always-true `OR` defeats the `WHERE`). Use as the WRONG example's payload to make the abstract rule
concrete.

---

## Source 2 — OWASP: SQL Injection Prevention Cheat Sheet (the fix)

URL: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

### Primary Defense = prepared statements / parameterized queries (Defense Option 1)
> "Prepared statements are simple to write and easier to understand than dynamic queries, and
> parameterized queries force the developer to define all SQL code first and pass in each parameter to the
> query later."

> "Also, prepared statements ensure that an attacker cannot change the intent of a query, even if SQL
> commands are inserted by an attacker."

**Why it matters:** This is the centerpiece sentence the spec demands verbatim — the mechanism
(SQL defined first, parameters bound later) is *why* parameterization works: structure is fixed before
any data is seen, so data can never become code. The "cannot change the intent of a query" clause is the
strongest single justification in the skill.

### Defense ranking (Primary Defenses, ordered list)
Presented in this order:
1. Prepared Statements (with Parameterized Queries)
2. Properly Constructed Stored Procedures
3. Allow-list Input Validation
4. Escaping All User-Supplied Input

**Why it matters:** Establishes that escaping is dead last and allow-listing is the tool for the parts you
*cannot* parameterize. This ordering drives the skill's structure: bind first, allowlist identifiers,
escaping is not on the table.

### Escaping is the weakest, discouraged option (Defense Option 4)
> "STRONGLY DISCOURAGED: Escaping All User-Supplied Input"

> "This methodology is fragile compared to other defenses, and we CANNOT guarantee that this option will
> prevent all SQL injections in all situations."

**Why it matters:** Direct support for "why this beats escaping" — OWASP itself will not guarantee escaping
works. Charset/encoding edge cases make hand-escaping unreliable; the source says so in its own words.

### Allow-list for non-parameterizable elements (Defense Option 3)
> "If you are faced with parts of SQL queries that can't use bind variables, such as table names, column
> names, or sort order indicators (ASC or DESC), input validation or query redesign is the most
> appropriate defense."

**Why it matters:** The authoritative statement that identifiers (table/column names) and sort direction
*cannot* be bound, so they require an allow-list mapped against a fixed set. Drives the identifiers section
and the dynamic ORDER BY section.

---

## Source 3 — PostgreSQL: PREPARE (the standard surface)

URL: https://www.postgresql.org/docs/current/sql-prepare.html

### PREPARE creates a server-side prepared statement
> "PREPARE creates a prepared statement. A prepared statement is a server-side object that can be used to
> optimize performance."

### Parameters are VALUES substituted at execution, referenced by position
> "Prepared statements can take parameters: values that are substituted into the statement when it is
> executed. When creating the prepared statement, refer to parameters by position, using `$1`, `$2`, etc."

> "When executing the statement, specify the actual values for these parameters in the `EXECUTE`
> statement."

### Syntax and example
> "PREPARE _name_ [ ( _data_type_ [, ...] ) ] AS _statement_"

> "PREPARE fooplan (int, text, bool, numeric) AS
>     INSERT INTO foo VALUES($1, $2, $3, $4);
> EXECUTE fooplan(1, 'Hunter Valley', 't', 200.00);"

### Typing
> "A corresponding list of parameter data types can optionally be specified. When a parameter's data type
> is not specified or is declared as `unknown`, the type is inferred from the context in which the
> parameter is first referenced (if possible)."

**Why it matters:** The standard SQL `PREPARE`/`EXECUTE` surface proves the value/code separation at the
engine level: `$1..$4` are *values* the engine substitutes, never re-parsed as SQL. This is the
dialect-neutral demonstration of the binding model that driver placeholders (`?`, `:name`, `%s`) sit on
top of. It also shows the placeholder spelling (`$1`) is a surface detail, routed to the dialect map.

---

## Synthesis — what the skill must nail (from spec lines 224-232)

- **(a) THE RULE — centerpiece.** Combine SQL with data *only* via bind parameters / prepared statements;
  never string interpolation, concatenation, or `format`/f-strings. Mechanism: SQL code defined first,
  parameters bound later (Source 2), so an attacker "cannot change the intent of a query" (Source 2).
  Concrete failure: `' OR '1'='1` rewrites the query (Source 1).
- **(b) Why escaping is NOT a reliable fix.** OWASP ranks it last and "STRONGLY DISCOURAGED," and
  "CANNOT guarantee" it prevents injection (Source 2). Charset/encoding/edge cases. Bind parameters
  sidestep escaping entirely because the value never enters the SQL text.
- **(c) Identifiers cannot be parameterized.** Table/column names and sort direction can't use bind
  variables (Source 2) → allow-list against a fixed set, never interpolate raw input.
- **(d) Placeholder syntax is driver-specific.** `?` (DB-API / JDBC / SQLite), `$1` (PostgreSQL native /
  psycopg numeric), `:name` (named), `%s` (psycopg / Python format-style). A binding *convention*, not a
  SQL dialect feature. The PREPARE `$1` form (Source 3) is the standard surface. Route spellings to
  `sql-standard-vs-dialect-map`.
- **(e) Dynamic ORDER BY / LIMIT safely.** ORDER BY column + direction → validate against an allow-list of
  permitted columns/directions (Source 2). LIMIT/OFFSET are *values* → bind them as parameters, do not
  interpolate.
- Show language-agnostic pseudocode + a couple concrete driver examples (Python DB-API `?`, psycopg
  `%s`/`$1`) but keep the LESSON portable.

## Cross-links (routing)
- Foundation: `sql-relational-and-null-discipline` (policy root every SQL skill links back to).
- `sql-privileges-and-access-control` — least privilege limits blast radius if injection still happens
  (defense in depth; out of scope here per spec).
- `sql-transactions-and-isolation` — transaction wrapping (out of scope here per spec).
- `sql-standard-vs-dialect-map` — placeholder spelling table.
