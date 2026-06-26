# Common SQL/JSON Mistakes

Anti-patterns in LLM-generated SQL around JSON, each with wrong/right code and a primary-source
citation. The skill (`sql-json`) states the rules; this file holds the high-frequency failure modes.
RIGHT examples prefer standard SQL/JSON (SQL:2016/2023); vendor-only spellings are flagged and routed
to `sql-standard-vs-dialect-map`.

---

## 1. `JSON_QUERY` (or `->`) used to extract a scalar — returns a QUOTED string

**The problem:** The model pulls a leaf string with `JSON_QUERY` (or the `->` operator), which returns
it as JSON — i.e. **quoted** (`"Ada"`) — so every comparison to plain text silently fails. "Scalar
strings returned by `JSON_VALUE` always have their quotes removed, equivalent to specifying `OMIT
QUOTES` in `JSON_QUERY`" ([PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)).

```sql
-- WRONG — '"Ada"' = 'Ada' is FALSE; matches zero rows
SELECT * FROM people WHERE JSON_QUERY(doc, '$.name') = 'Ada';

-- RIGHT — JSON_VALUE returns the dequoted scalar 'Ada'
SELECT * FROM people WHERE JSON_VALUE(doc, '$.name') = 'Ada';
```

*Source: [PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html). Depth: this skill, §3.*

---

## 2. `JSON_VALUE` used to extract an object or array

**The problem:** The model uses `JSON_VALUE` for a nested object/array. `JSON_VALUE` "extracts a
scalar JSON value—everything except object and array" ([modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016)); "getting multiple values
will be treated as an error" ([PostgreSQL](https://www.postgresql.org/docs/current/functions-json.html)) — so it errors or yields NULL on a structured value.

```sql
-- WRONG — '$.roles' is an array; JSON_VALUE wants a single scalar
SELECT JSON_VALUE(doc, '$.roles') FROM people;

-- RIGHT — JSON_QUERY for objects/arrays
SELECT JSON_QUERY(doc, '$.roles') AS roles FROM people;
```

*Source: [PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html); [modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016). Depth: this skill, §3.*

---

## 3. Fixed-index extraction instead of `JSON_TABLE`

**The problem:** To read a JSON array the model chains `$.items[0]`, `$.items[1]`, … — it sees only as
many elements as it hardcodes and can't aggregate. `JSON_TABLE` "queries JSON data and presents the
results as a relational view, which can be accessed as a regular SQL table" ([PostgreSQL — JSON_TABLE](https://www.postgresql.org/docs/current/functions-json.html)).

```sql
-- WRONG — only ever sees two items; can't SUM/JOIN/GROUP
SELECT JSON_VALUE(doc,'$.items[0].sku'), JSON_VALUE(doc,'$.items[1].sku') FROM orders;

-- RIGHT — one row per array element, ready to join and aggregate
SELECT o.id, it.sku, it.qty
FROM orders o,
     JSON_TABLE(o.doc, '$.items[*]'
       COLUMNS (sku text PATH '$.sku', qty integer PATH '$.qty')) AS it;
```

*Source: [PostgreSQL — JSON_TABLE](https://www.postgresql.org/docs/current/functions-json.html). Depth: this skill, §4; SQLite spelling (`json_each`) owned by `sql-standard-vs-dialect-map`.*

---

## 4. Relying on lax mode + default `ON EMPTY` — a typo'd path returns NULL forever

**The problem:** Lax mode (the default) "suppresses" structural errors "producing no match," and the
default `ON EMPTY` for `JSON_VALUE`/`JSON_QUERY` is "to return an SQL NULL value" ([PostgreSQL](https://www.postgresql.org/docs/current/functions-json.html)). A misspelled path looks identical to genuinely-missing data — no error, ever.

```sql
-- WRONG — '$.shippng.city' is a typo; returns NULL on every row, silently
SELECT JSON_VALUE(doc, '$.shippng.city') FROM orders;

-- RIGHT — strict mode + explicit ON ERROR surfaces the bad path
SELECT JSON_VALUE(doc, 'strict $.shipping.city' ERROR ON ERROR) AS city FROM orders;
```

*Source: [PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html); [modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016). Depth: this skill, §5.*

---

## 5. Hand-building JSON with string concatenation

**The problem:** The model glues a "JSON" string with `||`, which never escapes embedded quotes and
produces NULL/invalid output when any value is NULL. The standard constructors escape correctly.

```sql
-- WRONG — breaks on a quote in name; the whole string is NULL if name IS NULL
SELECT '{"id":' || id || ',"name":"' || name || '"}' FROM users;

-- RIGHT — JSON_OBJECT escapes and types values
SELECT JSON_OBJECT('id' VALUE id, 'name' VALUE name) FROM users;
```

*Source: [modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016). Depth: this skill, §2.*

---

## 6. `JSON_ARRAYAGG` without `ORDER BY` — nondeterministic array order

**The problem:** The model aggregates rows into a JSON array and assumes the elements come out in some
order. Aggregate input order is undefined (same as result order without `ORDER BY`); the array order
varies by plan. `JSON_ARRAYAGG` accepts an `ORDER BY`.

```sql
-- WRONG — element order of the array is undefined
SELECT order_id, JSON_ARRAYAGG(sku) FROM line_items GROUP BY order_id;

-- RIGHT — pin the order
SELECT order_id, JSON_ARRAYAGG(sku ORDER BY line_no) FROM line_items GROUP BY order_id;
```

*Source: [modern-sql.com — SQL:2016](https://modern-sql.com/blog/2017-06/whats-new-in-sql-2016). Depth: this skill, §2; order semantics owned by `sql-relational-and-null-discipline` §1.*

---

## 7. Trusting JSON in a `text` column without `IS JSON`

**The problem:** JSON stored in a plain `text`/`varchar` column (the SQL:2016 model, and SQLite's) is
unvalidated — malformed JSON sits there until a query trips over it. `IS JSON` "tests whether
expression can be parsed as JSON" and `WITH UNIQUE KEYS` also rejects duplicate keys ([PostgreSQL](https://www.postgresql.org/docs/current/functions-json.html)).

```sql
-- WRONG — nothing stops invalid JSON from being inserted into a text column
CREATE TABLE events (id bigint PRIMARY KEY, payload text);

-- RIGHT — validate on write
CREATE TABLE events (
  id bigint PRIMARY KEY,
  payload text CHECK (payload IS JSON OBJECT WITH UNIQUE KEYS)
);
-- ...or use a native JSON/jsonb column, which validates automatically
```

*Source: [PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html); [MySQL — JSON](https://dev.mysql.com/doc/refman/8.4/en/json.html). Depth: this skill, §6, §7.*

---

## 8. Storing domain columns in a JSON blob (the modern EAV antipattern)

**The problem:** To "avoid migrations," the model stuffs `customer_id`, `total`, `status` into a JSON
column — reinventing Entity-Attribute-Value ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). No foreign keys, no `CHECK`, no `NOT NULL`,
no index, no planner statistics; every query extracts and casts per row.

```sql
-- WRONG — the queried core of the domain hidden in an unconstrained blob
CREATE TABLE orders (id bigint PRIMARY KEY, data jsonb);

-- RIGHT — model the stable shape as columns; reserve JSON for genuinely variable extras
CREATE TABLE orders (
  id bigint PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  total numeric(10,2) NOT NULL CHECK (total >= 0),
  status text NOT NULL,
  metadata jsonb   -- sparse, caller-defined only
);
```

*Source: [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §1; normalization depth owned by `sql-schema-design-and-normalization`.*

---

## 9. Assuming the vendor operators are portable

**The problem:** The model reaches for `->`/`->>`/`JSON_EXTRACT` as if they were standard. They aren't
consistent across engines — SQLite warns "the MySQL version of `json_extract()` always returns JSON
[but] the SQLite version ... only returns JSON if there are two or more PATH arguments ... or if the
single PATH argument references an array or object" ([SQLite — JSON1](https://www.sqlite.org/json1.html)). Same name, different result type.

```sql
-- WRONG (portability) — vendor operator, type differs by engine
SELECT doc->>'name' FROM people;

-- RIGHT — standard SQL/JSON where the engine supports it (PG 17+, MySQL, Oracle)
SELECT JSON_VALUE(doc, '$.name') FROM people;
```

*Source: [SQLite — JSON1](https://www.sqlite.org/json1.html); [PostgreSQL — JSON Functions](https://www.postgresql.org/docs/current/functions-json.html); [MySQL — JSON](https://dev.mysql.com/doc/refman/8.4/en/json.html). Depth: this skill, §8; spellings owned by `sql-standard-vs-dialect-map`.*
