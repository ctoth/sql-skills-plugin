---
name: sql-injection-and-parameterization
description: Guides the one non-negotiable rule for combining SQL with data — pass every value through a bind parameter / prepared statement, and never build SQL by string interpolation, concatenation, f-strings, or `format`. Explains why this beats hand-escaping (OWASP ranks escaping last and "STRONGLY DISCOURAGED," and "CANNOT guarantee" it works), why identifiers (table/column names, ASC/DESC) cannot be parameterized and must be allow-listed against a fixed set, and that placeholder syntax (`?`, `$1`, `:name`, `%s`) is a driver convention, not a SQL dialect feature. Covers dynamic WHERE/ORDER BY/LIMIT done safely and the standard `PREPARE`/`EXECUTE` surface. Treats "just for this example" interpolation of user input as a defect, not a shortcut. Auto-invokes when writing or editing any query that embeds a variable or user input, string-building of SQL, dynamic `WHERE`/`ORDER BY`/table or column names, ORM raw-query escape hatches, or on "build this query from input" / "is this safe from injection" / "escape this value" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Injection and Parameterization

> "A SQL injection attack consists of insertion or 'injection' of a SQL query via the input data from the client to the application."
> — [OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)

> "Parameterized queries force the developer to define all SQL code first and pass in each parameter to the query later. ... prepared statements ensure that an attacker cannot change the intent of a query, even if SQL commands are inserted by an attacker."
> — [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

There is exactly one correct way to combine a SQL statement with runtime data: define the SQL text first with placeholders, then hand the values to the driver separately as **bind parameters**. Every other path — string concatenation, f-strings, `printf`/`format`, "just escaping the quotes" — is a defect. Not a style preference, not a performance trade-off: a security defect that hands an attacker the ability to rewrite your query. This skill builds on the set-and-three-valued-logic floor in `sql-relational-and-null-discipline` (a query is code that operates on data); the rule here is what keeps untrusted *data* from ever becoming *code*.

The whole skill collapses to one sentence: **the value never enters the SQL text.** Hold that and the rest is mechanics.

---

## 1. The Rule — Bind Parameters, Never Interpolate (the centerpiece)

A SQL injection happens "when an unintended data enters a program from an untrusted source [and] the data is used to dynamically construct a SQL query" ([OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)). The instant you build query *text* out of a value, the value can carry SQL of its own. The classic payload: a login field set to `' OR '1'='1` turns `... WHERE owner = 'wiley' AND itemname = 'name' OR 'a'='a'` ([OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)) — the trailing always-true `OR` defeats the filter and returns every row.

Parameterization removes the possibility at the root: it "force[s] the developer to define all SQL code first and pass in each parameter to the query later," so "an attacker cannot change the intent of a query, even if SQL commands are inserted" ([OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)). The structure is fixed before any value is seen; the value is delivered out-of-band and is only ever treated as a single scalar.

```python
# WRONG — Python f-string: the value becomes part of the SQL text
name = request.args["name"]                       # e.g.  ' OR '1'='1
cur.execute(f"SELECT * FROM items WHERE name = '{name}'")
# -> SELECT * FROM items WHERE name = '' OR '1'='1   -> returns every row

# WRONG — same defect, written with concatenation or % formatting
cur.execute("SELECT * FROM items WHERE name = '" + name + "'")
cur.execute("SELECT * FROM items WHERE name = '%s'" % name)   # NOT a bind param!
```

```python
# RIGHT — DB-API placeholder; the driver sends SQL and values on separate channels
cur.execute("SELECT * FROM items WHERE name = ?", (name,))    # sqlite3, many drivers

# RIGHT — psycopg (PostgreSQL): %s here IS a bind placeholder, value passed separately
cur.execute("SELECT * FROM items WHERE name = %s", (name,))
```

The dangerous trap above is `"... = '%s'" % name`: Python's `%` operator builds the string *before* the driver ever sees it — identical to an f-string. A real bind parameter is the placeholder left **in** the query string and the value passed as a separate argument. Whether the placeholder is `?` or `%s` is the driver's business (§5); the discipline is the same everywhere.

```sql
-- Language-agnostic shape of the rule:
-- 1. SQL text, fixed, with placeholders:   SELECT * FROM items WHERE name = ?
-- 2. values, separate:                     [name]
-- The engine never re-parses the value as SQL. That is the entire defense.
```

---

## 2. Why This Beats Escaping (escaping is the last resort, not a fix)

The tempting "fix" is to escape the quotes in the input and keep concatenating. OWASP lists escaping **dead last** of four defenses and labels it "STRONGLY DISCOURAGED," because "this methodology is fragile compared to other defenses, and we CANNOT guarantee that this option will prevent all SQL injections in all situations" ([OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)). Hand-escaping is a denylist, and denylists lose: multi-byte character sets, alternate encodings, backslash-mode differences across engines, and numeric contexts (an unquoted `id = $value` needs no quote to break) all defeat naive quote-doubling.

```python
# WRONG — escaping by hand; brittle, charset-dependent, and easy to get subtly wrong
safe = name.replace("'", "''")
cur.execute(f"SELECT * FROM items WHERE name = '{safe}'")   # still building SQL from data

# RIGHT — bind it; there is nothing to escape because the value never enters the text
cur.execute("SELECT * FROM items WHERE name = ?", (name,))
```

A bind parameter sidesteps escaping entirely: the value is transmitted as data, so quotes, backslashes, and encodings are simply bytes — never syntax. Prefer the structural fix over the textual patch every time.

---

## 3. Identifiers Cannot Be Parameterized — Allow-List Them

Bind parameters carry **values**, not SQL structure. A placeholder can stand in for `WHERE id = ?` but never for a table name, a column name, or a sort direction — those are part of the query's grammar and must be known when the statement is prepared. OWASP is explicit: "If you are faced with parts of SQL queries that can't use bind variables, such as table names, column names, or sort order indicators (ASC or DESC), input validation or query redesign is the most appropriate defense" ([OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)).

The only safe way to make an identifier dynamic is to **map untrusted input through an allow-list** of values you control — never to interpolate the raw string.

```python
# WRONG — interpolating a table/column name straight from input: injection wide open
table = request.args["table"]
cur.execute(f"SELECT * FROM {table}")            # ?table=users;DROP TABLE users--

# RIGHT — the input only ever selects from a fixed, code-defined set
ALLOWED_TABLES = {"users": "users", "orders": "orders"}
table = ALLOWED_TABLES[request.args["table"]]    # KeyError on anything else -> rejected
cur.execute(f"SELECT * FROM {table}")            # interpolated value is now trusted
```

The input chooses *which* of your known identifiers to use; it never supplies the identifier text. If the key is absent, reject the request. (Some drivers offer an identifier-quoting helper, e.g. psycopg's `sql.Identifier`; even then, validate against an allow-list — quoting an identifier does not make an arbitrary one *intended*.)

---

## 4. Dynamic WHERE / ORDER BY / LIMIT, Done Safely

Real apps build queries from optional filters and user-chosen sorting. Each piece splits cleanly by kind: **values** get bound; **identifiers and keywords** get allow-listed.

```python
# RIGHT — dynamic WHERE: build the placeholder skeleton, collect values to bind
clauses, params = [], []
if status is not None:
    clauses.append("status = ?"); params.append(status)
if min_total is not None:
    clauses.append("total >= ?"); params.append(min_total)
where = (" WHERE " + " AND ".join(clauses)) if clauses else ""
cur.execute(f"SELECT * FROM orders{where}", params)   # only ?-placeholders are interpolated
```

ORDER BY is the trap: the column and direction are identifiers/keywords, so they **cannot** be bound — allow-list them. The LIMIT/OFFSET, by contrast, are **values** — bind them.

```python
# WRONG — sort column and direction interpolated from input -> injection
cur.execute(f"SELECT * FROM orders ORDER BY {col} {dir} LIMIT {n}")

# RIGHT — column and direction validated against fixed sets; LIMIT bound as a value
SORT_COLS = {"created", "total", "id"}
SORT_DIR  = {"asc": "ASC", "desc": "DESC"}
col = col if col in SORT_COLS else "id"
dir = SORT_DIR.get(dir, "ASC")
cur.execute(f"SELECT * FROM orders ORDER BY {col} {dir} LIMIT ?", (n,))
```

Note `LIMIT ?` — the row count is data and binds normally; only the validated `col`/`dir` are interpolated, and they came from a set you defined, not from the request.

---

## 5. Placeholder Syntax Is Driver-Specific, Not a Dialect Feature

The *spelling* of a placeholder is a property of the driver/binding API, not of the SQL dialect. The binding model is universal; the punctuation is not. Do not infer a database from a placeholder, and do not switch placeholder styles to "fix" a binding bug — fix the call instead.

| Style | Where you see it | Example |
|---|---|---|
| `?` (positional) | SQLite, JDBC, ODBC, Python DB-API `qmark` | `WHERE id = ?` |
| `$1`, `$2` (numbered) | PostgreSQL native / `PREPARE`, some drivers | `WHERE id = $1` |
| `:name` (named) | Oracle, named-param APIs, SQLAlchemy `text()` | `WHERE id = :id` |
| `%s`, `%(name)s` | psycopg / Python `pyformat` (a *bind* marker, not `printf`) | `WHERE id = %s` |

`%s` in psycopg is the single most misread marker: it is a **bind placeholder**, and you must pass the value as a separate argument — never `query % value`. The full spelling table and per-driver `paramstyle` notes live in **`sql-standard-vs-dialect-map`**; this skill only teaches that the *style varies and the rule does not*.

---

## 6. `PREPARE` / `EXECUTE` — the Standard Surface Underneath

Driver placeholders sit on top of the SQL standard's own prepared-statement facility. `PREPARE` "creates a prepared statement ... a server-side object," and "prepared statements can take parameters: values that are substituted into the statement when it is executed ... refer to parameters by position, using `$1`, `$2`, etc." The values are supplied later: "when executing the statement, specify the actual values for these parameters in the `EXECUTE` statement" ([PostgreSQL — PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html)).

```sql
-- The value/code separation, visible at the SQL level:
PREPARE fooplan (int, text, bool, numeric) AS
    INSERT INTO foo VALUES($1, $2, $3, $4);
EXECUTE fooplan(1, 'Hunter Valley', 't', 200.00);   -- $1..$4 are VALUES, never re-parsed
```

`$1..$4` are positions filled with values at `EXECUTE` time; the engine never treats them as SQL text. That is the same guarantee your driver's `?`/`%s` gives — it is parameterization made explicit in the language. (Most app code lets the driver prepare implicitly; you rarely write `PREPARE` by hand, but it shows what "bind" means.)

---

## 7. "Just for This Example" Is the Defect

The most common way an injection ships is a model (or a developer) writing an f-string query "just to illustrate," "just for the demo," or "just for this internal tool," intending to fix it later. Later does not come; the example gets copied into production. There is no tier of code where string-built SQL is acceptable — examples, tests, migrations, and admin scripts all run real SQL against real engines. Write the bind-parameter version the first time, every time. The parameterized form is not longer or harder (§1) — it is the same line with the value moved one argument to the right.

If you catch yourself thinking "the input here is trusted / internal / already validated," parameterize anyway: trust boundaries move, validation gets bypassed, and the cost of binding is zero. Parameterization is unconditional.

---

## 8. Portability

| Concern | Standard / portable | Notes |
|---|---|---|
| Binding model (value substituted at execute, never re-parsed) | universal | The rule in §1 holds on every engine and driver. |
| `PREPARE` / `EXECUTE` with positional params | SQL standard surface | PostgreSQL uses `$1`; other engines vary; route spellings to dialect map. |
| Placeholder spelling (`?` / `$1` / `:name` / `%s`) | **not portable** | A driver/`paramstyle` property, not a dialect feature (§5). |
| Identifier quoting helper (e.g. `sql.Identifier`) | driver-specific | Still allow-list; quoting ≠ intending (§3). |
| Escaping function (`quote_literal`, `mysql_real_escape_string`, …) | engine-specific, discouraged | OWASP "CANNOT guarantee" it works; do not rely on it (§2). |

The lesson is dialect-neutral: **bind values, allow-list identifiers, never build SQL from data.** Only the placeholder punctuation changes across drivers, and that is a lookup, not a decision — see **`sql-standard-vs-dialect-map`**.

---

## 9. Who Suffers When SQL Is Built From Data

Injection never throws at write time — it compiles, it passes the demo, and the bill arrives later:

- The **user whose PII leaked.** A single interpolated parameter let an attacker run `UNION SELECT` against the users table; OWASP notes SQL injection allows "the complete disclosure of all data on the system" ([OWASP](https://owasp.org/www-community/attacks/SQL_Injection)). Their email, address, and password hash are now in a breach dump, forever — they did nothing wrong and cannot undo it.
- The **on-call engineer during an active exploit.** Logs show thousands of malformed queries; rows are being exfiltrated or deleted in real time ("destroy the data or make it otherwise unavailable" — [OWASP](https://owasp.org/www-community/attacks/SQL_Injection)). They are choosing between pulling the database offline and letting the bleed continue, at 3 a.m., because of one f-string.
- The **junior who shipped it.** They asked an assistant for "a quick query from this input," got an f-string "for simplicity," and merged it — the tool that should have known better taught them the defect. The fix was always one argument away (§1); the assistant's job was to write the bind-parameter version unprompted.

Parameterizing is empathy in code: the `?` you wrote instead of the f-string is the breach that never happened.

---

## 10. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics and three-valued logic, the policy root every SQL skill links back to.
- `sql-privileges-and-access-control` — least privilege limits the blast radius *if* an injection (or any compromise) still lands; defense in depth beneath this rule.
- `sql-transactions-and-isolation` — wrapping the data-changing statements you parameterize in correct transaction boundaries.
- `sql-standard-vs-dialect-map` — the placeholder-spelling table (`?` / `$1` / `:name` / `%s`), `paramstyle`, and per-engine escaping/quoting functions.

---

## 11. Reference Files

High-frequency injection/parameterization anti-patterns in LLM-generated code, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
