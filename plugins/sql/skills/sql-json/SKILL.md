---
name: sql-json
description: >-
  Guides standard SQL/JSON (SQL:2016 functions, SQL:2023 type) instead of reflexive vendor operators — use the constructors (JSON_OBJECT/JSON_ARRAY/JSON_OBJECTAGG/JSON_ARRAYAGG), and the query functions JSON_VALUE (for a scalar) vs JSON_QUERY (for an object/array) vs JSON_EXISTS (for a path test) rather than `- greater than `/`- greater than greater than `/`# greater than greater than `/JSON_EXTRACT; shred documents into joinable rows with JSON_TABLE. Auto-invokes when writing or editing JSON columns/queries, `- greater than `/`- greater than greater than `/`# greater than greater than `/JSON_EXTRACT, JSON_VALUE/JSON_QUERY/JSON_TABLE, JSON path expressions, JSON aggregation, or on "extract from JSON" / "store JSON" / "query a JSON column" requests.
allowed-tools: Read, Glob, Grep
---

# SQL/JSON

> "Only use `JSON_VALUE()` if the extracted value is expected to be a single SQL/JSON scalar item ... If you expect that extracted value might be an object or an array, use the `JSON_QUERY` function instead."
> — [PostgreSQL — JSON Functions (Table 9.54)](https://www.postgresql.org/docs/current/functions-json.html)

> "The standard ... does not define a native JSON type like it does for XML. Instead, the standard uses strings to store JSON data."
> — [modern-sql.com — What's new in SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)

SQL:2016 added a real, portable JSON sublanguage — constructors, query functions, a path language, `JSON_TABLE`, and the `IS JSON` predicate — and SQL:2023 added a native `JSON` type. Most LLM-written SQL ignores all of it and reaches for the engine-specific operators (`->`, `->>`, `#>>`, `JSON_EXTRACT`) that were invented before the standard and disagree across engines. This skill teaches the standard surface and, before any of it, the judgment of *when JSON belongs in the database at all*. It builds on `sql-relational-and-null-discipline`: a JSON path that doesn't match yields NULL, and that NULL flows through three-valued logic exactly like any other.

---

## 1. First Decide: JSON or a Normalized Schema? (judgment before syntax)

Before writing a single `JSON_VALUE`, ask whether this data should be JSON at all. A JSON document is schema-less, unconstrained (the database checks nothing beyond "is this parseable JSON"), and not indexable without vendor-specific extensions. The moment you find yourself storing what are conceptually *columns* — `price`, `status`, `customer_id` — inside a JSON blob so you can "add fields without migrations," you have reinvented the **Entity-Attribute-Value (EAV) antipattern** in modern clothing ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). You lose type checking, foreign keys, `NOT NULL`, `CHECK`, cheap joins, and query planner statistics — every guarantee the relational model gives you for free.

JSON earns its place for genuinely schema-less, sparse, or externally-shaped data: third-party API payloads you store verbatim, user-defined custom fields, sparse attributes that differ per row, an audit/event envelope. It is the wrong tool for the stable, queried core of your domain.

```sql
-- WRONG — domain columns hidden in a JSON blob: unqueryable, unconstrained, un-joinable
CREATE TABLE orders (
  id     bigint PRIMARY KEY,
  data   jsonb   -- {"customer_id": 42, "total": 19.99, "status": "shipped"}
);
-- now "all shipped orders over $100" needs JSON extraction + casts on every row, no FK on customer_id

-- RIGHT — model the known shape as columns; reserve JSON for the genuinely variable part
CREATE TABLE orders (
  id          bigint PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  total       numeric(10,2) NOT NULL CHECK (total >= 0),
  status      text NOT NULL,
  metadata    jsonb   -- sparse, caller-defined extras only
);
```

The full normalization argument, EAV depth, and the "design a schema" decision belong to **`sql-schema-design-and-normalization`** — route there. This skill assumes you have already decided JSON is right and shows you how to use it well.

---

## 2. Constructing JSON — `JSON_OBJECT`, `JSON_ARRAY`, and the Aggregates

SQL:2016 defines four constructors: `JSON_OBJECT` and `JSON_ARRAY` build a value from columns; `JSON_OBJECTAGG` and `JSON_ARRAYAGG` aggregate a group into one object or array ([modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)). Prefer these over hand-built string concatenation, which never escapes quotes correctly and produces invalid JSON.

```sql
-- WRONG — string-glued "JSON" that breaks the moment a name contains a quote or the value is NULL
SELECT '{"id":' || id || ',"name":"' || name || '"}' AS doc FROM users;

-- RIGHT — constructors escape and type values correctly
SELECT JSON_OBJECT('id' VALUE id, 'name' VALUE name) AS doc FROM users;

-- RIGHT — aggregate a child collection into one array per parent
SELECT o.id,
       JSON_ARRAYAGG(li.sku ORDER BY li.line_no) AS skus
FROM orders o JOIN line_items li ON li.order_id = o.id
GROUP BY o.id;
```

`JSON_ARRAYAGG` takes an optional `ORDER BY` so the array order is deterministic — without it, array element order is as undefined as row order is without `ORDER BY` (see `sql-relational-and-null-discipline` §1). Both `JSON_OBJECT` and `JSON_ARRAY` are standard names that also work in PostgreSQL, MySQL, and (lowercased) SQLite.

---

## 3. `JSON_VALUE` vs `JSON_QUERY` vs `JSON_EXISTS` — the Wrong-Function Trap (the centerpiece)

Three query functions, three jobs, and choosing wrong is the single most common JSON bug:

- **`JSON_VALUE`** extracts "a scalar JSON value—everything except object and array" ([modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)) and returns it as an SQL scalar (default `text`). Use it for a single string/number/boolean leaf.
- **`JSON_QUERY`** "extracts a part out of [a] JSON document and returns it as a JSON string" — use it for an object or array.
- **`JSON_EXISTS`** "tests whether a specific path exists" and returns a boolean.

The trap: `JSON_VALUE` and `JSON_QUERY` *both* "succeed" on a scalar, but return **different things**. `JSON_QUERY` returns the value as JSON, so a string comes back **quoted** (`"abc"`); `JSON_VALUE` dequotes it (`abc`). PostgreSQL is explicit: "scalar strings returned by `JSON_VALUE` always have their quotes removed, equivalent to specifying `OMIT QUOTES` in `JSON_QUERY`" ([PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)). Use `JSON_QUERY` to pull a scalar and you get a quoted string that won't compare equal to your plain text, won't cast cleanly to a number, and silently corrupts every downstream comparison.

```sql
-- doc = '{"name": "Ada", "roles": ["admin","ops"], "age": 37}'

-- WRONG — JSON_QUERY on a scalar returns the QUOTED JSON string '"Ada"', not Ada
SELECT JSON_QUERY(doc, '$.name') = 'Ada' FROM people;   -- '"Ada"' = 'Ada' -> FALSE, every row

-- RIGHT — JSON_VALUE returns the bare scalar 'Ada'
SELECT * FROM people WHERE JSON_VALUE(doc, '$.name') = 'Ada';

-- WRONG — JSON_VALUE on an array/object errors or yields NULL (it demands a single scalar)
SELECT JSON_VALUE(doc, '$.roles') FROM people;          -- 'roles' is an array, not a scalar

-- RIGHT — JSON_QUERY for the structured part
SELECT JSON_QUERY(doc, '$.roles') AS roles FROM people; -- ["admin","ops"]

-- RIGHT — JSON_EXISTS for a presence test (no extraction)
SELECT * FROM people WHERE JSON_EXISTS(doc, '$.age');
```

Mirror image in vendor operators (§8): PostgreSQL/MySQL/SQLite `->` returns JSON (quoted) while `->>` returns text (dequoted) — the *exact same* quoted-vs-scalar split, which is why people reach for `->>` and still get it wrong on objects. `RETURNING` lets you type the result (`JSON_VALUE(doc, '$.age' RETURNING int)`); otherwise you get `text` and must cast.

---

## 4. `JSON_TABLE` — Shred a Document Into Joinable Rows

`JSON_TABLE` "queries JSON data and presents the results as a relational view, which can be accessed as a regular SQL table" ([PostgreSQL — JSON_TABLE](https://www.postgresql.org/docs/current/functions-json.html)). It is the bridge back to the relational world: once a JSON array is rows, you can `JOIN`, `WHERE`, `GROUP BY`, and aggregate it like any table. Reaching into a JSON array with a chain of indexed extracts (`$.items[0]`, `$.items[1]`, …) instead is the tell of someone who hasn't met `JSON_TABLE`.

The `COLUMNS` clause "defin[es] the schema of the constructed view ... each column to be filled with an SQL/JSON value obtained by applying a JSON path expression against the row pattern." `NESTED PATH` unnests deeper arrays — "the NESTED PATH syntax is recursive ... It allows to unnest the hierarchy of JSON objects and arrays in a single function invocation rather than chaining several `JSON_TABLE` expressions," with child rows "joined against the row constructed from the columns specified in the parent `COLUMNS` clause" ([PostgreSQL — JSON_TABLE](https://www.postgresql.org/docs/current/functions-json.html)).

```sql
-- WRONG — fixed-index extraction: only ever sees the first two items, can't aggregate
SELECT JSON_VALUE(doc, '$.items[0].sku'),
       JSON_VALUE(doc, '$.items[1].sku')
FROM orders;

-- RIGHT — JSON_TABLE turns the array into one row per item, ready to JOIN/GROUP/SUM
SELECT o.id, it.sku, it.qty
FROM orders o,
     JSON_TABLE(o.doc, '$.items[*]'
       COLUMNS (
         seq  FOR ORDINALITY,
         sku  text    PATH '$.sku',
         qty  integer PATH '$.qty'
       )) AS it;

-- RIGHT — NESTED PATH unnests parent + child arrays in one call
SELECT jt.*
FROM my_films,
     JSON_TABLE(js, '$.favorites[*]'
       COLUMNS (
         id   FOR ORDINALITY,
         kind text PATH '$.kind',
         NESTED PATH '$.films[*]' COLUMNS (
           title    text PATH '$.title',
           director text PATH '$.director'
         ))) AS jt;
```

`FOR ORDINALITY` adds a 1-based row counter; `... EXISTS PATH ...` columns yield a boolean per row. SQLite has no `JSON_TABLE` — it spells this `json_each()` / `json_tree()` (§8); route the spelling to the dialect map.

---

## 5. The Path Language: Lax vs Strict, and `ON ERROR` / `ON EMPTY`

The SQL/JSON path language uses `$` for the context item, `.member` for object fields, `[n]` for array elements, `*` for wildcards, and `?(@ …)` for filters ([modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)). What trips people up is **structural-error handling**, governed by two modes:

- **lax (the default)** — "the path engine implicitly adapts the queried data to the specified path. Any structural errors that cannot be fixed ... are suppressed, producing no match" ([PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)). Lax auto-unwraps arrays and wraps scalars, so a slightly-wrong path quietly returns nothing instead of erroring.
- **strict** — "if a structural error occurs, an error is raised." Use `strict` when a missing/mistyped path is a bug you want surfaced, not silently swallowed.

Independently, each function takes `ON EMPTY` and `ON ERROR` clauses to choose what happens when the path matches nothing or evaluation fails. The defaults are silent: for `JSON_QUERY`/`JSON_VALUE` "the default ... is to return an SQL NULL value"; for `JSON_EXISTS` the default is "to return the boolean value `FALSE`" ([PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)). That silent NULL then propagates through three-valued logic (`sql-relational-and-null-discipline` §2–§3) — a mismatched path looks identical to a present-but-null value unless you say otherwise.

```sql
-- QUIET — lax + default ON EMPTY: a typo'd or absent path returns NULL, no error, no signal
SELECT JSON_VALUE(doc, '$.shippng.city') FROM orders;   -- misspelled -> NULL forever

-- LOUD — be explicit about empty and error so a bad path doesn't masquerade as missing data
SELECT JSON_VALUE(doc, 'strict $.shipping.city'
         DEFAULT 'UNKNOWN' ON EMPTY
         ERROR ON ERROR) AS city
FROM orders;
```

---

## 6. `IS JSON` — Validate Before You Trust

When JSON lives in a plain `text`/`varchar` column (the SQL:2016 storage model, and still SQLite's), nothing guarantees it parses. The `IS JSON` predicate "tests whether expression can be parsed as JSON, possibly of a specified type ... If `WITH UNIQUE KEYS` is specified, then any object ... is also tested to see if it has duplicate keys" ([PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)). It comes in `IS JSON VALUE | SCALAR | ARRAY | OBJECT` flavors.

```sql
-- Guard a text column that's supposed to hold JSON objects with unique keys
ALTER TABLE events
  ADD CONSTRAINT payload_is_json
  CHECK (payload IS JSON OBJECT WITH UNIQUE KEYS);

-- Filter to rows whose text actually parses as a JSON array before shredding them
SELECT * FROM imports WHERE raw IS JSON ARRAY;
```

A native `JSON` column (§7) validates on write, making `IS JSON` redundant there — it earns its keep on the text columns that hold JSON without a typed column to enforce it.

---

## 7. The SQL:2023 Native `JSON` Type

SQL:2016 deliberately had no JSON type — "the standard uses strings to store JSON data" ([modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)). SQL:2023 added a native `JSON` type, and engines that have it validate on write and store an optimized form. MySQL's is representative: a `JSON` column gives "automatic validation of JSON documents ... Invalid documents produce an error" plus an "optimized storage format ... converted to an internal format that permits quick read access" ([MySQL — The JSON Data Type](https://dev.mysql.com/doc/refman/8.4/en/json.html)). PostgreSQL's long-standing `jsonb` is the same idea (binary, validated) and predates the standard; SQLite still stores JSON as text (with an internal `jsonb` blob form). Prefer a typed JSON/`jsonb` column over `text` when the engine offers it: you get write-time validation and faster access for free, and binary forms also support the vendor indexing (PostgreSQL GIN on `jsonb`) that a plain text column cannot — index support is a vendor matter, route to the dialect map.

---

## 8. Vendor Operators (`->`, `->>`, `#>>`, `JSON_EXTRACT`) — Prefer Standard, Route Spellings

These predate SQL:2016 and are everywhere in existing code, but they are **not portable** and repeat the §3 trap. PostgreSQL: `->` returns the JSON type, `->>` returns `text`; `#>`/`#>>` do the same at a `text[]` path ([PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)). MySQL: `col->path` is "a synonym for `JSON_EXTRACT(column, path)`" and `->>` adds `JSON_UNQUOTE` ([MySQL — JSON](https://dev.mysql.com/doc/refman/8.4/en/json.html)). SQLite added `->`/`->>` in 3.38.0 with the same "`->` returns JSON, `->>` returns an SQL representation" split ([SQLite — JSON1](https://www.sqlite.org/json1.html)).

The portability hazard is real even within the vendor spellings: SQLite warns that "the MySQL version of `json_extract()` always returns JSON [but] the SQLite version ... only returns JSON if there are two or more PATH arguments ... or if the single PATH argument references an array or object" ([SQLite — JSON1](https://www.sqlite.org/json1.html)) — the same function name, different result type, across two engines. Prefer the standard `JSON_VALUE`/`JSON_QUERY` where the engine has them; where you must use operators, the full standard-vs-vendor mapping lives in **`sql-standard-vs-dialect-map`**.

```sql
-- Vendor (PostgreSQL): ->> dequotes (text), -> stays JSON (quoted) — the §3 split, operator form
SELECT doc->>'name' AS name,    -- text:  Ada
       doc->'roles' AS roles    -- jsonb: ["admin","ops"]
FROM people;

-- Standard equivalent, portable to any engine that implements SQL/JSON
SELECT JSON_VALUE(doc, '$.name') AS name,
       JSON_QUERY(doc, '$.roles') AS roles
FROM people;
```

---

## 9. Portability Snapshot

The standard functions are arriving unevenly; the vendor operators are near-universal but divergent.

| Feature | PostgreSQL | MySQL | SQLite | Oracle |
|---|---|---|---|---|
| `JSON_VALUE` / `JSON_QUERY` / `JSON_EXISTS` | 17+ | `JSON_VALUE` 8.0.21+; no `JSON_QUERY` | no (use `json_extract`) | yes |
| `JSON_TABLE` | 17+ | 8.0.4+ | no (use `json_each`/`json_tree`) | yes |
| `JSON_OBJECT` / `JSON_ARRAY` / `*AGG` | yes | yes | `json_object`/`json_array` (no AGG until newer) | yes |
| `IS JSON` predicate | 16+ | partial | via `json_valid()` | yes |
| Native JSON / binary type | `jsonb` (predates std) | `JSON` (8.x) | text + internal `jsonb` blob | yes |
| `->` / `->>` operators | yes | yes (8.x) | 3.38.0+ | no native operator |

Three rules follow: (1) standard `JSON_VALUE`/`JSON_QUERY`/`JSON_TABLE` are the most portable *intent* and available in PostgreSQL 17+, recent MySQL, and Oracle — prefer them; (2) SQLite needs the vendor spellings (`json_extract`, `json_each`/`json_tree`) and lowercased constructors; (3) the `->`/`->>` operators exist almost everywhere but return different types and aren't even consistent across engines (`json_extract` differs MySQL↔SQLite). All dialect spellings route to **`sql-standard-vs-dialect-map`**; JSON-index support (PostgreSQL GIN on `jsonb`) is a vendor performance topic, not standard SQL.

---

## 10. Who Suffers When JSON Is Mishandled

A JSON mistake, like a NULL mistake, usually doesn't error — it returns subtly wrong data:

- The **engineer** whose `WHERE JSON_QUERY(doc,'$.status') = 'open'` matches **zero rows** because `JSON_QUERY` returned the quoted string `"open"` and `"open" = 'open'` is false (§3). No error, no rows, an afternoon lost to a missing quote-stripping they never knew they needed.
- The **analyst** whose dashboard silently drops records because a misspelled lax path (`$.shippng.city`) returns NULL on every row instead of raising — `strict` mode and `ERROR ON ERROR` would have screamed on day one (§5).
- The **next maintainer** handed a `data jsonb` column where `customer_id`, `total`, and `status` were stuffed to "avoid migrations" — now there is no foreign key, no `CHECK`, no index, and "shipped orders over $100" is a full scan with per-row extraction and casts (§1). The schema that was never designed is the bug they inherit.
- The **on-call** debugging a feed that breaks intermittently because a `text` column holds occasionally-malformed JSON that an `IS JSON` check constraint would have rejected at write time (§6).

Disciplined JSON is the same empathy as disciplined NULL: the `JSON_VALUE` you used instead of `JSON_QUERY`, the `strict`/`ON ERROR` you made explicit, and — most of all — the columns you modeled instead of burying in a blob.

---

## 11. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: a non-matching path yields NULL, which propagates through three-valued logic; array/element order is undefined without `ORDER BY` (use it in `JSON_ARRAYAGG`).
- `sql-schema-design-and-normalization` — the JSON-vs-schema decision, EAV antipattern depth, and when to model columns instead of a blob (§1).
- `sql-expressions-case-and-functions` — `COALESCE`, casting (`RETURNING`/`CAST`), and `CASE` around extracted JSON values.
- `sql-standard-vs-dialect-map` — vendor operator spellings (`->`, `->>`, `#>>`, `JSON_EXTRACT`, `json_each`/`json_tree`), the `json_extract` MySQL↔SQLite divergence, and `jsonb` GIN indexing.

---

## 12. Reference Files

High-frequency SQL/JSON anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
