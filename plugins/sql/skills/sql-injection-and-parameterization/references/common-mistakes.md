# Common SQL Injection & Parameterization Mistakes

Anti-patterns in LLM-generated code around combining SQL with data, each with wrong/right code and a
primary-source citation. The skill (`sql-injection-and-parameterization`) states the rule; this file
holds the high-frequency failure modes. The lesson is dialect-neutral — placeholder *spellings* are
driver-specific and routed to `sql-standard-vs-dialect-map`.

---

## 1. f-string / template-literal query built from user input

**The problem:** The model interpolates input directly into the SQL text. The value can carry SQL of its
own — a login field of `' OR '1'='1` rewrites the `WHERE` into an always-true clause and returns every
row. Injection "occurs when ... data is used to dynamically construct a SQL query" ([OWASP](https://owasp.org/www-community/attacks/SQL_Injection)).

```python
# WRONG — the value becomes part of the query text
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")

# RIGHT — placeholder in the text, value passed separately
cur.execute("SELECT * FROM users WHERE email = ?", (email,))
```

*Source: [OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection). Depth: skill §1.*

---

## 2. `%` / `.format()` mistaken for a bind parameter

**The problem:** The model writes `"... = '%s'" % value` or `"...".format(value)` and believes it is
parameterizing. Python's `%`/`format` build the string *before* the driver sees it — identical to an
f-string. A real bind leaves the marker in the string and passes the value as a separate argument.

```python
# WRONG — % formats the string first; this is interpolation, not binding
cur.execute("SELECT * FROM users WHERE id = '%s'" % user_id)

# RIGHT — psycopg: %s stays in the query, value passed separately
cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

*Source: [OWASP Cheat Sheet — Prepared Statements](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §1, §5.*

---

## 3. "Escaping the quotes" instead of binding

**The problem:** The model patches the symptom by doubling/escaping quotes and keeps concatenating.
OWASP ranks escaping last and "STRONGLY DISCOURAGED," and "CANNOT guarantee that this option will prevent
all SQL injections" ([OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)) — multi-byte charsets, encodings, and numeric contexts defeat it.

```python
# WRONG — brittle denylist; still building SQL from data
safe = value.replace("'", "''")
cur.execute(f"SELECT * FROM t WHERE c = '{safe}'")

# RIGHT — bind it; there is nothing to escape
cur.execute("SELECT * FROM t WHERE c = ?", (value,))
```

*Source: [OWASP Cheat Sheet — Defense Option 4](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §2.*

---

## 4. Interpolating a table or column name

**The problem:** The model tries to bind an identifier, or interpolates it raw. Bind variables carry
values, not structure: "parts of SQL queries that can't use bind variables, such as table names, column
names, or sort order indicators (ASC or DESC)" need "input validation" instead ([OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)).

```python
# WRONG — raw identifier from input: ?table=users;DROP TABLE users--
cur.execute(f"SELECT * FROM {request.args['table']}")

# RIGHT — input only selects from a fixed, code-defined set
ALLOWED = {"users": "users", "orders": "orders"}
cur.execute(f"SELECT * FROM {ALLOWED[request.args['table']]}")  # KeyError = rejected
```

*Source: [OWASP Cheat Sheet — Defense Option 3](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §3.*

---

## 5. Dynamic ORDER BY interpolated from input

**The problem:** The model interpolates the sort column and/or direction from the request. Both are
identifiers/keywords and cannot be bound — they must be allow-listed. (The LIMIT, by contrast, is a value
and binds normally.)

```python
# WRONG — col and dir come straight from input
cur.execute(f"SELECT * FROM t ORDER BY {col} {dir} LIMIT {n}")

# RIGHT — validate col/dir against fixed sets; bind the LIMIT value
SORT_COLS = {"created", "total", "id"}; SORT_DIR = {"asc": "ASC", "desc": "DESC"}
col = col if col in SORT_COLS else "id"
dir = SORT_DIR.get(dir, "ASC")
cur.execute(f"SELECT * FROM t ORDER BY {col} {dir} LIMIT ?", (n,))
```

*Source: [OWASP Cheat Sheet — Defense Option 3](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §4.*

---

## 6. Interpolating LIMIT / OFFSET instead of binding

**The problem:** The model treats the row count as structure and interpolates it. LIMIT/OFFSET are
**values** — bind them like any other parameter. Parameters are "values that are substituted into the
statement when it is executed" ([PostgreSQL — PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html)).

```python
# WRONG — page size from input, interpolated
cur.execute(f"SELECT * FROM t LIMIT {limit} OFFSET {offset}")

# RIGHT — bind the numeric values
cur.execute("SELECT * FROM t LIMIT ? OFFSET ?", (limit, offset))
```

*Source: [PostgreSQL — PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html). Depth: skill §4.*

---

## 7. Building an `IN (...)` list by joining strings

**The problem:** The model concatenates a comma-joined list of values into `IN (...)`. Every element is
interpolated and injectable. Generate one placeholder per value and bind them all.

```python
# WRONG — joins raw values into the IN list
ids = ",".join(str(i) for i in id_list)
cur.execute(f"SELECT * FROM t WHERE id IN ({ids})")

# RIGHT — one placeholder per value, all bound
marks = ",".join(["?"] * len(id_list))
cur.execute(f"SELECT * FROM t WHERE id IN ({marks})", id_list)
```

*Source: [OWASP Cheat Sheet — Prepared Statements](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §1, §4.*

---

## 8. "Just for this example / it's internal" interpolation

**The problem:** The model writes an f-string query "for simplicity" in an example, test, migration, or
internal tool, planning to fix it later. There is no tier of code where string-built SQL is safe; the
example gets copied to production. The bind-parameter form is the same length.

```python
# WRONG — "demo only" string-built query (ships anyway)
db.execute(f"DELETE FROM sessions WHERE user = '{user}'")

# RIGHT — parameterize the first time, every time
db.execute("DELETE FROM sessions WHERE user = ?", (user,))
```

*Source: [OWASP Cheat Sheet — Prepared Statements](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §7.*

---

## 9. ORM raw-query escape hatch with interpolation

**The problem:** The model drops to a raw-SQL escape hatch (`.raw()`, `text()`, `execute()`) and
interpolates input into it, bypassing the ORM's parameter binding. Raw hatches still accept bind
parameters — use them.

```python
# WRONG — f-string inside a raw escape hatch defeats the ORM's protection
session.execute(text(f"SELECT * FROM t WHERE name = '{name}'"))

# RIGHT — named bind parameter passed to the raw hatch
session.execute(text("SELECT * FROM t WHERE name = :name"), {"name": name})
```

*Source: [OWASP Cheat Sheet — Prepared Statements](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html). Depth: skill §1, §5.*
