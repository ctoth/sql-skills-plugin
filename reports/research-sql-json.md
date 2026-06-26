# Research: sql-json (skill #19)

Research for the `sql-json` skill in the standard-SQL skills plugin. Each entry: URL, section,
verbatim quote, why it matters. Accessed 2026-06-26.

Note: `https://modern-sql.com/feature/json_table` (listed in the spec) returns **HTTP 404** as of
2026-06-26 — the dedicated feature page is gone. JSON_TABLE coverage is sourced instead from the
modern-sql SQL:2016 overview (which documents JSON_TABLE) and from the PostgreSQL JSON_TABLE docs
(Section 9.16.4), which give the full COLUMNS / NESTED PATH semantics.

---

## 1. modern-sql.com — What's new in SQL:2016 (SQL/JSON)
URL: https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016

### SQL/JSON introduced in SQL:2016 — the function set
- Constructors: `JSON_OBJECT([key] <expr> value <expr>)`, `JSON_ARRAY([<expr>,…])`,
  `JSON_ARRAYAGG(<expr> [order by …])`, `JSON_OBJECTAGG([key] <expr> value <expr>)`.
- Accessors:
  - `JSON_EXISTS(<json>, <path>)` — "Tests whether a specific path exists in JSON document".
  - `JSON_VALUE(<json>, <path>)` — "Extracts a scalar JSON value—everything except object and array".
  - `JSON_QUERY(<json>, <path>)` — "Extracts a part out of JSON document and returns it as a JSON string".
  - `JSON_TABLE(<json>, <path> columns …)` — transforms JSON into relational table rows.
- Validation predicate: `IS [NOT] JSON [value | array | object | scalar]` — "Tests whether expression contains valid JSON".

**Why it matters:** establishes the canonical standard surface. These functions are the portable,
vendor-neutral spelling the skill should prefer over `->`/`->>`/`JSON_EXTRACT`.

### No native JSON type in SQL:2016
> "does not define a native JSON type like it does for XML. Instead, the standard uses strings to store JSON data."

**Why it matters:** explains why JSON is stored as character strings in SQL:2016, sets up the SQL:2023
native `JSON` type as the later addition, and why `IS JSON` validation exists (the string isn't typed).

### Error handling: ON ERROR / ON EMPTY
> The standard defines an `ON ERROR` clause for functions interpreting JSON strings. Functions also
> support `ON EMPTY` clauses.

**Why it matters:** the path may not match — every accessor takes ON ERROR/ON EMPTY to choose error
vs NULL vs default. This is the path-mismatch discipline.

### SQL/JSON path language + lax vs strict
- Path syntax: `$` context element, `.` object member, `[]` array element, `?()` filter (with `@`
  current context). Examples: `$.name`, `$[0]`, `$.events[last]`, `$.*`, `$.* ?(@.type()=="number")`.
- Lax (default): "suppresses these errors...unwraps arrays or wraps scalar values".
- Strict: "triggers error handling for all errors...accessing non-existing object members".

**Why it matters:** the path language is the second half of SQL/JSON; lax-vs-strict decides whether a
structural mismatch is silently ignored or surfaced.

### JSON_VALUE vs JSON_QUERY (the wrong-function trap)
> `JSON_VALUE` returns scalars only; `JSON_QUERY` can return any JSON type and return multiple matches
> using optional `WITH [CONDITIONAL | UNCONDITIONAL] [ARRAY] WRAPPER` clause.

**Why it matters:** the centerpiece distinction. Use JSON_VALUE for a scalar; use JSON_QUERY for an
object/array. Picking the wrong one is the classic bug (quoted-string-where-a-scalar-belonged).

---

## 2. PostgreSQL — JSON Functions and Operators
URL: https://www.postgresql.org/docs/current/functions-json.html

### Standard SQL/JSON query functions (added in PG 17)
> "SQL/JSON functions `JSON_EXISTS()`, `JSON_QUERY()`, and `JSON_VALUE()` ... can be used to query JSON documents."

> "`JSON_TABLE` is an SQL/JSON function which queries JSON data and presents the results as a relational view, which can be accessed as a regular SQL table."

(PostgreSQL 17, released 2024-09, is the version that added the standard JSON_TABLE/JSON_VALUE/
JSON_QUERY/JSON_EXISTS functions; `jsonb`/`json` types and the `->`/`->>` operators predate them.)

**Why it matters:** confirms the standard functions exist in a major open-source engine and that
PostgreSQL had a vendor JSON story (json/jsonb + operators) long before the standard caught up.

### JSON_VALUE vs JSON_QUERY return types (the centerpiece)
JSON_VALUE:
> "Only use `JSON_VALUE()` if the extracted value is expected to be a single SQL/JSON scalar item;
> getting multiple values will be treated as an error. If you expect that extracted value might be an
> object or an array, use the `JSON_QUERY` function instead. By default, the result, which must be a
> single scalar value, is returned as a value of type `text`".

JSON_QUERY:
> "By default, the result is returned as a value of type `jsonb`".

> "Note that scalar strings returned by `JSON_VALUE` always have their quotes removed, equivalent to
> specifying `OMIT QUOTES` in `JSON_QUERY`."

> "`JSON_VALUE()` returns an SQL NULL if path_expression returns a JSON `null`, whereas `JSON_QUERY()`
> returns the JSON `null` as is."

**Why it matters:** nails the WRONG/RIGHT centerpiece — JSON_QUERY on a scalar yields a *quoted* JSON
string (`"abc"`), JSON_VALUE yields the dequoted scalar (`abc`). Wrong function = subtly wrong data.

### Vendor operators -> ->> #> #>>
From Table 9.47:
- `json/jsonb -> integer|text` → `json/jsonb` — extracts array element / object field (returns JSON).
- `json/jsonb ->> integer|text` → `text` — same, but as `text`.
- `#>` / `#>>` — extract sub-object at a `text[]` path, as JSON / as text.

**Key distinction**: `->` and `#>` return the JSON type; `->>` and `#>>` return `text`.

**Why it matters:** these are the reflexive vendor operators the skill steers away from. The `->`(JSON)
vs `->>`(text) split mirrors the JSON_QUERY/JSON_VALUE split — same trap, vendor spelling.

### Lax vs strict modes
> "lax (default) — the path engine implicitly adapts the queried data to the specified path. Any
> structural errors that cannot be fixed as described below are suppressed, producing no match.
> strict — if a structural error occurs, an error is raised."

> "An attempt to access a non-existent member of an object or element of an array is defined as a
> structural error."

**Why it matters:** the precise PG statement of lax/strict and what a "structural error" is.

### IS JSON predicate
> "expression IS [ NOT ] JSON [ { VALUE | SCALAR | ARRAY | OBJECT } ] [ { WITH | WITHOUT } UNIQUE [ KEYS ] ]
> This predicate tests whether expression can be parsed as JSON, possibly of a specified type. ... If
> WITH UNIQUE KEYS is specified, then any object ... is also tested to see if it has duplicate keys."

**Why it matters:** the validation predicate — and `WITH UNIQUE KEYS` catches duplicate-key JSON,
useful as a CHECK constraint on a text column holding JSON.

### ON ERROR / ON EMPTY defaults (per function)
- JSON_EXISTS: "The default when no `ON ERROR` clause is specified is to return the boolean value `FALSE`."
- JSON_QUERY / JSON_VALUE: "The default when `ON EMPTY` or `ON ERROR` is not specified is to return an SQL NULL value."
- JSON_TABLE top-level: "{ ERROR | EMPTY } ON ERROR ... `EMPTY` to return an empty table ... Note that
  this clause does not affect the errors that occur when evaluating columns".

**Why it matters:** the defaults differ by function (FALSE for EXISTS, NULL for VALUE/QUERY); explicit
ON ERROR/ON EMPTY is how you opt into errors instead of silent NULLs.

---

## 3. PostgreSQL — JSON_TABLE (Section 9.16.4: COLUMNS / NESTED PATH)
URL: https://www.postgresql.org/docs/current/functions-json.html

### COLUMNS clause
> "The `COLUMNS` clause defining the schema of the constructed view. In this clause, you can specify
> each column to be filled with an SQL/JSON value obtained by applying a JSON path expression against
> the row pattern."

Column kinds:
- `name FOR ORDINALITY` — "sequential row numbering starting from 1."
- `name type [FORMAT JSON] [ PATH path_expression ]` — value at path coerced to `type`.
- `name type EXISTS [ PATH path_expression ]` — boolean: whether the path yields any values.

### NESTED PATH (shredding nested arrays)
> "NESTED [ PATH ] path_expression [ AS json_path_name ] COLUMNS ( ... ) - Extracts SQL/JSON values
> from nested levels of the row pattern, generates one or more columns ... The NESTED PATH syntax is
> recursive ... It allows to unnest the hierarchy of JSON objects and arrays in a single function
> invocation rather than chaining several `JSON_TABLE` expressions."

> "Rows constructed from NESTED COLUMNS are called child rows and are joined against the row
> constructed from the columns specified in the parent COLUMNS clause to get the row in the final view."

Representative query:
```sql
SELECT jt.* FROM my_films,
 JSON_TABLE ( js, '$.favorites[*]'
   COLUMNS (
    id FOR ORDINALITY,
    kind text PATH '$.kind',
    NESTED PATH '$.films[*]' COLUMNS (
      title text FORMAT JSON PATH '$.title' OMIT QUOTES,
      director text PATH '$.director' KEEP QUOTES))) AS jt;
```
Result: one output row per nested film, parent `id`/`kind` repeated per child row.

**Why it matters:** JSON_TABLE is the standard tool to turn a JSON document/array into rows you can
JOIN, GROUP BY, and aggregate — the bridge from JSON back to relational. NESTED PATH does multi-level
unnesting in one call instead of chained extracts.

---

## 4. MySQL — The JSON Data Type
URL: https://dev.mysql.com/doc/refman/8.4/en/json.html

### Native JSON type (MySQL 8.x; 8.4 docs)
> "MySQL supports a native `JSON` (JavaScript Object Notation) data type defined by RFC 8259 that
> enables efficient access to data in JSON documents."

> "Automatic validation of JSON documents stored in `JSON` columns. Invalid documents produce an
> error. Optimized storage format. JSON documents stored in `JSON` columns are converted to an
> internal format that permits quick read access to document elements."

**Why it matters:** MySQL has a real binary JSON type with validation — the column rejects malformed
JSON. (MySQL has `JSON_VALUE` (8.0.21+) and `JSON_TABLE` (8.0.4+) too, though they aren't on this
particular page; partial standard support.)

### Constructors JSON_OBJECT / JSON_ARRAY
> "`JSON_ARRAY()` takes a (possibly empty) list of values and returns a JSON array containing those values"
> "`JSON_OBJECT()` takes a (possibly empty) list of key-value pairs and returns a JSON object containing those pairs"

**Why it matters:** the standard constructor names work in MySQL.

### -> and ->> operators (shorthand for JSON_EXTRACT)
> "You can use `column->path` ... as a synonym for `JSON_EXTRACT(column, path)`"
> `->>` is the inline path operator equivalent to `JSON_UNQUOTE(JSON_EXTRACT(...))`; "`->>` removes the
> surrounding quotes that `->` retains."

JSON_EXTRACT example:
> `SELECT JSON_EXTRACT('{"id": 14, "name": "Aztalan"}', '$.name');` → `"Aztalan"` (quoted).

**Why it matters:** MySQL's `->`/`->>` and `JSON_EXTRACT` are the vendor operators; the `->`(quoted)
vs `->>`(unquoted) split is the same JSON_QUERY/JSON_VALUE distinction in vendor clothes.

---

## 5. SQLite — The JSON1 Extension
URL: https://www.sqlite.org/json1.html

### json (text) and jsonb (binary) representations
> "SQLite stores JSON as ordinary text."
> "Beginning with version 3.45.0 (2024-01-15), SQLite allows its internal 'parse tree' representation
> of JSON to be stored on disk, as a BLOB, in a format that we call 'JSONB'." (internal use only.)

**Why it matters:** SQLite has no dedicated JSON column type — JSON lives in TEXT (or BLOB jsonb).

### -> and ->> operators (added 3.38.0)
> "Beginning with SQLite version 3.38.0 (2022-02-22), the -> and ->> operators are available for
> extracting subcomponents of JSON."
> "-> always returns a JSON representation of that subcomponent and the ->> operator always returns an
> SQL representation of that subcomponent."

Examples: `'{"a":"xyz"}' -> '$.a'` → `'"xyz"'` (quoted JSON); `->> '$.a'` → `'xyz'` (SQL text).

**Why it matters:** SQLite adopted the MySQL/PG `->`/`->>` convention; same quoted/unquoted split.

### json_extract and the MySQL incompatibility
> "There is a subtle incompatibility between the json_extract() function in SQLite and ... MySQL. The
> MySQL version of json_extract() always returns JSON. The SQLite version ... only returns JSON if
> there are two or more PATH arguments ... or if the single PATH argument references an array or object."

**Why it matters:** even the *vendor* spelling `json_extract` is not portable between SQLite and
MySQL — a direct argument for the standard functions where available, and routing the spellings to the
dialect map.

### No standard JSON_VALUE / JSON_QUERY / JSON_TABLE names
SQLite uses `json_extract()` (not JSON_VALUE/JSON_QUERY) and `json_each()` / `json_tree()`
table-valued functions (not JSON_TABLE).

> "The json_each(X) ... walk[s] only the immediate children of the top-level array or object ... The
> json_tree(X) ... recursively walk[s] through the JSON substructure".

**Why it matters:** SQLite does the JSON_TABLE job with `json_each`/`json_tree`; the standard names are
absent. Route the spellings to the dialect map.

---

## Judgment thread (for the skill's lead section)
The strongest editorial point, synthesized across sources: SQL/JSON gives you real query power over
documents, but a JSON blob is schema-less, unconstrained (beyond `IS JSON`), and un-indexable without
vendor extensions. Storing what should be columns inside a JSON document is the modern reincarnation of
the EAV / "Entity-Attribute-Value" antipattern (Karwin, *SQL Antipatterns*) — depth and design
trade-offs belong in `sql-schema-design-and-normalization`. JSON earns its place for genuinely
schema-less / sparse / third-party payloads, not as a way to dodge designing a schema.
