---
name: sql-data-types-and-numerics
description: >-
  Guides standard-SQL type selection so the storage matches the data's meaning — exact `NUMERIC`/`DECIMAL(p,s)` for money and counts (never `FLOAT`/`REAL`/`DOUBLE`, whose binary floating point cannot store 0.1 exactly and whose rounding error compounds when summed), `SMALLINT`/`INTEGER`/`BIGINT` chosen by the value range (the `INTEGER` ceiling is +2,147,483,647), `CHARACTER VARYING` sized to a real domain limit instead of a cargo-culted `VARCHAR(255)`, fixed-width `CHARACTER(n)` only for. Auto-invokes when writing or editing column type declarations, `CREATE TABLE`/`ALTER TABLE` type choices, `CAST` targets, money/currency/price fields, high-precision or large-range numbers, identifier/key column widths, or string-length and boolean columns. Routes dialect spellings (BIT, TINYINT(1), bytea) to sql-standard-vs-dialect-map.
allowed-tools: Read, Glob, Grep
---

# SQL Data Types and Numerics

> "It is especially recommended for storing monetary amounts and other quantities where exactness is required. Calculations with `numeric` values yield exact results where possible, e.g., addition, subtraction, multiplication."
> — [PostgreSQL — Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html)

> "Inexact means that some values cannot be converted exactly to the internal format and are stored as approximations, so that storing and retrieving a value might show slight discrepancies."
> — [PostgreSQL — Numeric Types (floating-point)](https://www.postgresql.org/docs/current/datatype-numeric.html)

The column type is a promise about what a value *means*: how much it can hold, whether it is exact, and what comparisons make sense. Picking a type for convenience instead of meaning is how a price ends up a cent short, a surrogate key dies at 2.1 billion, or a legitimate name is rejected by an arbitrary length cap. This skill assumes the set-and-NULL semantics owned by `sql-relational-and-null-discipline` (a wrong type still propagates NULL and still fails a `CHECK` the same way) and focuses on the one decision a `CREATE TABLE` forces: which type, and how wide.

---

## 1. Exact vs Approximate Numerics — Money Is `NUMERIC`, Never Float (the centerpiece)

There are two numeric families and they are not interchangeable. **Exact** types (`NUMERIC`, alias `DECIMAL`) store a value digit-for-digit: "Calculations with `numeric` values yield exact results where possible, e.g., addition, subtraction, multiplication," and the type "is especially recommended for storing monetary amounts and other quantities where exactness is required" ([PostgreSQL — Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)). **Approximate** types (`REAL`, `DOUBLE PRECISION`, the spelling `FLOAT`) are binary floating point: they "are inexact, variable-precision numeric types," meaning "some values cannot be converted exactly to the internal format and are stored as approximations, so that storing and retrieving a value might show slight discrepancies" ([PostgreSQL — Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)). The docs state the rule outright: "If you require exact storage and calculations (such as for monetary amounts), use the `numeric` type instead."

The reason is base-2. A binary float cannot represent the decimal fraction `0.1` exactly — only the nearest representable approximation — and every arithmetic step carries that error forward. It compounds:

```sql
-- WRONG — money in binary floating point; each value is an approximation
CREATE TABLE invoice_line (
    description  TEXT,
    amount       DOUBLE PRECISION   -- 0.10 is NOT stored as exactly 0.10
);

-- RIGHT — exact decimal: 2 fractional digits, up to 17 total digits
CREATE TABLE invoice_line (
    description  TEXT,
    amount       NUMERIC(19, 2) NOT NULL   -- standard DECIMAL(19,2) is identical
);
```

Worked rounding error — sum ten dimes and compare to a dollar:

```sql
-- WRONG — floating point: the sum drifts off the exact value
SELECT 0.1::double precision * 10 = 1.0::double precision;   -- may be FALSE
SELECT SUM(amount) FROM invoice_line;  -- 10 rows of 0.10 can total 0.9999999999999999

-- RIGHT — exact decimal: the sum is precisely 1.00, and the equality holds
SELECT 0.1::numeric * 10 = 1.0::numeric;                     -- TRUE
SELECT SUM(amount) FROM invoice_line;  -- 10 rows of 0.10 total exactly 1.00
```

Across a million rows those sub-cent drifts accumulate into a real discrepancy that no `ROUND` at the end can reliably undo, because the error is already baked into stored values. Use `NUMERIC`/`DECIMAL(p,s)` for currency, tax rates, balances, exact counts, and any quantity a human will reconcile. Reserve `DOUBLE PRECISION` for genuinely approximate scientific or statistical values (measurements, ratios, coordinates) where a tiny relative error is acceptable and exact equality is never tested.

---

## 2. Precision and Scale — and Overflow

`NUMERIC(p, s)` declares **precision** `p` (total significant digits) and **scale** `s` (digits after the decimal point). The integer part is therefore bounded by `p − s` digits, and exceeding it is a hard error, not a silent wrap: "if the number of digits to the left of the decimal point exceeds the declared precision minus the declared scale, an error is raised" ([PostgreSQL — Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)). Scale, by contrast, rounds: "a column declared as `NUMERIC(3, 1)` will round values to 1 decimal place and can store values between -99.9 and 99.9, inclusive" ([PostgreSQL — Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)).

```sql
-- WRONG — NUMERIC(5,2) allows only 3 integer digits (max 999.99); 1000.00 errors out
price NUMERIC(5, 2)        -- a $1,000 item cannot be stored

-- RIGHT — size the integer part to the real maximum (here up to 9,999,999.99)
price NUMERIC(9, 2)
```

Choose `p` and `s` from the domain: pick `s` for the smallest unit you must represent exactly (2 for dollars-and-cents, 4 for many FX rates) and `p` large enough that `p − s` covers the biggest integer part you will ever store. Bare `NUMERIC` with no `(p, s)` stores any precision but loses the schema-level guard against an out-of-range value.

---

## 3. Integer Sizing — Pick by Range, Mind the Ceiling

Integers are fixed-width; the type is a promise about range, and overflow errors rather than wrapping. The standard three ([PostgreSQL — Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)):

| Type | Bytes | Range |
|---|---|---|
| `SMALLINT` | 2 | −32,768 to +32,767 |
| `INTEGER` | 4 | −2,147,483,648 to +2,147,483,647 |
| `BIGINT` | 8 | −9,223,372,036,854,775,808 to +9,223,372,036,854,775,807 |

The trap is the `INTEGER` ceiling of **+2,147,483,647** — about 2.1 billion. It is a comfortable-looking number that a growing table of orders, events, or log rows will quietly reach, at which point the next `INSERT` fails.

```sql
-- WRONG — a high-volume surrogate key in a 4-byte INTEGER; dies near 2.1 billion
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY

-- RIGHT — 8-byte BIGINT for any key/counter that can grow without a hard cap
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

Default to `BIGINT` for surrogate keys and unbounded counters; `INTEGER` is fine for bounded foreign attributes; `SMALLINT` suits small enumerations or ages. (Auto-increment columns are not gap-free — "there may be 'holes' or gaps in the sequence of values ... A value allocated from the sequence is still 'used up' even if a row ... is never successfully inserted" ([PostgreSQL — Serial Types](https://www.postgresql.org/docs/current/datatype-numeric.html)); identity/sequence mechanics are owned by `sql-generated-and-identity-columns`.)

---

## 4. `CHARACTER VARYING` and the `VARCHAR(255)` Cargo Cult

The standard variable-length string type is `CHARACTER VARYING(n)` (alias `VARCHAR(n)`); fixed-length is `CHARACTER(n)` (`CHAR(n)`) ([PostgreSQL — Data Types](https://www.postgresql.org/docs/current/datatype.html)). The reflexive `VARCHAR(255)` on every text column is a cargo cult: 255 is an artifact of an old MySQL row-format encoding boundary, not a business rule, and the limit buys nothing. "There is no performance difference among these three types, apart from increased storage space when using the blank-padded type ... In most situations `text` or `character varying` should be used instead" ([PostgreSQL — Character Types](https://www.postgresql.org/docs/current/datatype-character.html)).

Worse, an arbitrary limit actively rejects valid data — it does not silently truncate on insert: "An attempt to store a longer string into a column of these types will result in an error, unless the excess characters are all spaces" ([PostgreSQL — Character Types](https://www.postgresql.org/docs/current/datatype-character.html)).

```sql
-- WRONG — an arbitrary cap that errors on a legitimate long name or email
full_name  VARCHAR(255)
email      VARCHAR(255)

-- RIGHT — unbounded text, or a length that reflects a REAL domain limit
full_name  VARCHAR        -- (or TEXT) no business maximum on a name
email      VARCHAR(320)   -- 320 = the actual RFC 5321 maximum, a real constraint
country    CHAR(2)        -- genuinely fixed-width: ISO 3166-1 alpha-2 code
```

Reach for `CHARACTER(n)` only for genuinely fixed-width codes, and know it blank-pads: "Values of type `character` are physically padded with spaces to the specified width `n`, and are stored and displayed that way" ([PostgreSQL — Character Types](https://www.postgresql.org/docs/current/datatype-character.html)). That trailing padding surprises comparisons and round-trips, which is why variable-length is the default.

---

## 5. `BOOLEAN` — Real Type, Uneven Support

Standard SQL has a first-class `BOOLEAN` type holding "logical Boolean (true/false)" ([PostgreSQL — Data Types](https://www.postgresql.org/docs/current/datatype.html)). Use it; do not encode truth as `0`/`1` integers or `'Y'`/`'N'` strings, which invite invalid third values and lose three-valued NULL semantics (a `BOOLEAN` can also be NULL — see `sql-relational-and-null-discipline`).

```sql
-- WRONG — a flag faked as an integer or char; nothing stops the value 2 or 'X'
is_active SMALLINT NOT NULL DEFAULT 0
is_active CHAR(1)  NOT NULL DEFAULT 'N'

-- RIGHT — the real type
is_active BOOLEAN NOT NULL DEFAULT FALSE
```

Support is uneven, which is the portability story: SQL Server has no `BOOLEAN` (use `BIT`), older MySQL maps `BOOLEAN` to `TINYINT(1)`, and SQLite "does not have a separate Boolean storage class. Instead, Boolean values are stored as integers 0 (false) and 1 (true)" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)). Write `BOOLEAN` in portable DDL and route the per-engine spellings to `sql-standard-vs-dialect-map`.

---

## 6. Large Objects — `BLOB` and `CLOB`

For data too large for ordinary columns, standard SQL provides `BLOB` (Binary Large OBject) and `CLOB` (Character Large OBject). The spellings diverge sharply: PostgreSQL has no `BLOB` keyword — it uses `bytea` for "binary data ('byte array')" ([PostgreSQL — Data Types](https://www.postgresql.org/docs/current/datatype.html)) and `text` in place of `CLOB`; SQLite uses a `BLOB` storage class; MySQL and Oracle support `BLOB`/`CLOB` directly. Because the keywords are not portable, treat the choice as semantic (binary vs character payload) and route the concrete type name to `sql-standard-vs-dialect-map`. Storing large blobs inline versus by reference is a schema-design question owned by `sql-schema-design-and-normalization`.

---

## 7. SQLite Dynamic Typing — Declared Types Are Only Advisory (the big deviation)

Every type rule above assumes the engine *enforces* the declared type. SQLite does not. "SQLite uses a more general dynamic type system. In SQLite, the datatype of a value is associated with the value itself, not with its container" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)). A column's declared type sets only an *affinity* — a preference — and "the type is recommended, not required. Any column can still store any type of data" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)).

```sql
-- In SQLite, the declared types below DO NOT constrain values:
CREATE TABLE t (n INTEGER, amount NUMERIC, flag BOOLEAN);
INSERT INTO t VALUES ('hello', 3.14159, 'maybe');   -- accepted; no error
```

Two consequences for type choices that work elsewhere:
- Affinity is decided by substring matching on the declared name — `"INT"` → INTEGER, `"CHAR"/"CLOB"/"TEXT"` → TEXT, `"BLOB"` or no type → BLOB, `"REAL"/"FLOA"/"DOUB"` → REAL, "Otherwise ... NUMERIC" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)). So a standard DDL parses and "works," but enforces nothing.
- There is **no exact-decimal storage class**. A `NUMERIC`/`DECIMAL` column may fall back to REAL (binary float): on insert "the storage class of the text is converted to INTEGER or REAL (in order of preference)" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)). So the money rule from §1 cannot be enforced by the column type in SQLite — store money as integer minor units (cents) or as TEXT, and there is no separate `BOOLEAN` or `DATE` class either. Route the deviation details to `sql-standard-vs-dialect-map`.

---

## 8. Portability Snapshot

The standard families — exact `NUMERIC`/`DECIMAL`, approximate `REAL`/`DOUBLE PRECISION`, the integer trio, `CHARACTER VARYING`/`CHARACTER`, `BOOLEAN`, `BLOB`/`CLOB` — are widely shared in spirit; the spellings and enforcement diverge:

| Concept | PostgreSQL | SQLite | MySQL | SQL Server |
|---|---|---|---|---|
| Exact decimal | `numeric`/`decimal` | NUMERIC affinity (may store as REAL) | `decimal` | `decimal`/`numeric` |
| Boolean | `boolean` | integer 0/1 (no BOOLEAN class) | `tinyint(1)` | `bit` |
| Binary LOB | `bytea` | `BLOB` | `blob` | `varbinary(max)` |
| Declared type enforced? | yes | **no — advisory affinity** | yes | yes |

The two deviations most likely to bite: **SQLite does not enforce declared types** (§7), so a schema that "compiles" can silently hold mistyped data; and **BOOLEAN/money have no universal spelling**, so `BOOLEAN`, `BIT`, `TINYINT(1)` and the decimal/float fallbacks must be reconciled per engine. For every concrete spelling, route to `sql-standard-vs-dialect-map`.

---

## 9. Who Suffers When the Type Is Wrong

A type mistake rarely errors at `CREATE TABLE` — it surfaces later as a wrong number or a rejected row, and someone downstream pays:

- The **accountant** reconciling a ledger that is off by a penny, because amounts were stored as `DOUBLE PRECISION` and ten `0.10` rows summed to `0.9999999999999999` (§1). No error was ever raised; the books just do not balance, and the discrepancy grows with volume.
- The **user** whose legitimately long name or email is *rejected* — not truncated — by a `VARCHAR(255)` that someone copied from a tutorial, never matching any real limit (§4). Support files a bug; the fix is a migration that should never have been needed.
- The **on-call engineer** paged at 2.1 billion when an `INTEGER` surrogate key overflows and every `INSERT` starts failing in production (§3) — a `BIGINT` from day one would have cost nothing.
- The **SQLite maintainer** who trusted a `NUMERIC` column to hold exact money and a `BOOLEAN` to hold true/false, and finds floats and the string `'maybe'` in there, because the declared types were only advice (§7).

Choosing the type that matches the data's meaning is empathy in the schema: the `NUMERIC` instead of `DOUBLE`, the `BIGINT` instead of `INTEGER`, and the honest length limit are all gifts to whoever maintains this under load.

---

## 10. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics and three-valued logic that every typed column still obeys (NULL propagation, `CHECK` passing on UNKNOWN).
- `sql-datetime-and-intervals` — `DATE`/`TIME`/`TIMESTAMP`/`INTERVAL` types and time-zone semantics (deliberately out of scope here).
- `sql-generated-and-identity-columns` — `GENERATED ... AS IDENTITY`, sequences, and the serial gaps noted in §3.
- `sql-constraints-and-integrity` — `CHECK`/`NOT NULL`/`DEFAULT` that complement a type to bound a column's domain.
- `sql-standard-vs-dialect-map` — BOOLEAN/BIT/TINYINT(1), money/decimal-vs-float fallbacks, `bytea`/`BLOB`/`CLOB`, and SQLite's affinity deviation.

---

## 11. Reference Files

High-frequency data-type and numeric anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
