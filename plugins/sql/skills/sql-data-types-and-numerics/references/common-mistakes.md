# Common SQL Data-Type and Numeric Mistakes
## Contents

- [1. Storing money in FLOAT/DOUBLE PRECISION](#1-storing-money-in-floatdouble-precision)
- [2. NUMERIC(p, s) whose integer part is too small](#2-numericp-s-whose-integer-part-is-too-small)
- [3. INTEGER surrogate key that overflows at 2.1 billion](#3-integer-surrogate-key-that-overflows-at-21-billion)
- [4. The reflexive VARCHAR(255)](#4-the-reflexive-varchar255)
- [5. CHAR(n) for variable-length text (surprise blank-padding)](#5-charn-for-variable-length-text-surprise-blank-padding)
- [6. Faking a boolean as integer or char](#6-faking-a-boolean-as-integer-or-char)
- [7. Trusting SQLite to enforce a declared type](#7-trusting-sqlite-to-enforce-a-declared-type)
- [8. Expecting exact decimal money in SQLite](#8-expecting-exact-decimal-money-in-sqlite)


Anti-patterns in LLM-generated SQL around type selection, each with wrong/right code and a
primary-source citation. The skill (`sql-data-types-and-numerics`) states the rules; this file holds
the high-frequency failure modes. All RIGHT examples are standard/portable SQL; non-standard
spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Storing money in `FLOAT`/`DOUBLE PRECISION`

**The problem:** The model picks a floating-point type for a price or balance. Binary floating point
cannot represent decimal fractions like `0.10` exactly — it "is inexact ... some values cannot be
converted exactly to the internal format and are stored as approximations" ([PostgreSQL —
Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)) — and the error compounds
across sums, leaving a ledger a cent off with no error ever raised.

```sql
-- WRONG — approximate type for exact money; sums drift
amount DOUBLE PRECISION

-- RIGHT — exact decimal; "recommended for storing monetary amounts"
amount NUMERIC(19, 2) NOT NULL    -- DECIMAL(19,2) is identical
```

*Source: [PostgreSQL — Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html). Depth: this skill, §1.*

---

## 2. `NUMERIC(p, s)` whose integer part is too small

**The problem:** The model sizes a decimal by the fractional digits it remembers and forgets that
the integer part is bounded by `p − s`. Storing a value over that bound is a hard error: "if the
number of digits to the left of the decimal point exceeds the declared precision minus the declared
scale, an error is raised" ([PostgreSQL —
Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)).

```sql
-- WRONG — NUMERIC(5,2) caps the integer part at 3 digits (max 999.99); $1,000 errors
price NUMERIC(5, 2)

-- RIGHT — precision large enough that (p - s) covers the real maximum
price NUMERIC(9, 2)               -- up to 9,999,999.99
```

*Source: [PostgreSQL — Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html). Depth: this skill, §2.*

---

## 3. `INTEGER` surrogate key that overflows at 2.1 billion

**The problem:** The model defaults a high-volume key or counter to 4-byte `INTEGER`, range
"−2147483648 to +2147483647" ([PostgreSQL —
Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html)). A growing table reaches
+2,147,483,647 and the next `INSERT` fails in production.

```sql
-- WRONG — 4-byte key; dies near 2.1 billion rows
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY

-- RIGHT — 8-byte key for any unbounded counter
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

*Source: [PostgreSQL — Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html). Depth: this skill, §3; identity mechanics owned by `sql-generated-and-identity-columns`.*

---

## 4. The reflexive `VARCHAR(255)`

**The problem:** The model puts `VARCHAR(255)` on every text column. 255 is an old MySQL row-format
artifact, not a business rule; the cap buys no performance ("There is no performance difference among
these three types ... In most situations `text` or `character varying` should be used instead")
and an over-length value is *rejected*, not truncated ("An attempt to store a longer string ... will
result in an error") ([PostgreSQL —
Character](https://www.postgresql.org/docs/current/datatype-character.html)).

```sql
-- WRONG — arbitrary cap that errors on a legitimate long value
full_name VARCHAR(255)

-- RIGHT — unbounded, or a length that reflects a REAL domain limit
full_name VARCHAR              -- (or TEXT)
email     VARCHAR(320)         -- 320 = actual RFC 5321 maximum
```

*Source: [PostgreSQL — Character Types](https://www.postgresql.org/docs/current/datatype-character.html). Depth: this skill, §4.*

---

## 5. `CHAR(n)` for variable-length text (surprise blank-padding)

**The problem:** The model uses `CHAR(n)` for a name or description. `CHARACTER(n)` is fixed-width
and blank-pads: "Values of type `character` are physically padded with spaces to the specified width
`n`, and are stored and displayed that way" ([PostgreSQL —
Character](https://www.postgresql.org/docs/current/datatype-character.html)). The trailing spaces
surprise comparisons and round-trips.

```sql
-- WRONG — pads with trailing spaces; 'Q' is stored as 'Q   ...'
city CHAR(100)

-- RIGHT — variable-length for variable data
city VARCHAR(100)             -- (or TEXT); reserve CHAR(n) for fixed codes like CHAR(2)
```

*Source: [PostgreSQL — Character Types](https://www.postgresql.org/docs/current/datatype-character.html). Depth: this skill, §4.*

---

## 6. Faking a boolean as integer or char

**The problem:** The model encodes a flag as `0`/`1` or `'Y'`/`'N'`, admitting invalid values and
losing three-valued NULL semantics. Standard SQL has a real `boolean` — "logical Boolean
(true/false)" ([PostgreSQL — Data Types](https://www.postgresql.org/docs/current/datatype.html)).

```sql
-- WRONG — nothing stops 2 or 'X'
is_active SMALLINT NOT NULL DEFAULT 0

-- RIGHT — the real type
is_active BOOLEAN NOT NULL DEFAULT FALSE
```

*Source: [PostgreSQL — Data Types](https://www.postgresql.org/docs/current/datatype.html). Depth: this skill, §5; per-engine spellings (BIT, TINYINT(1)) owned by `sql-standard-vs-dialect-map`.*

---

## 7. Trusting SQLite to enforce a declared type

**The problem:** The model writes a standard DDL for SQLite and assumes the declared types constrain
the data. They do not: "the datatype of a value is associated with the value itself, not with its
container," and a column's affinity "is recommended, not required. Any column can still store any
type of data" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)).

```sql
-- WRONG (assumption) — in SQLite these declarations enforce NOTHING
CREATE TABLE t (n INTEGER, amount NUMERIC);
INSERT INTO t VALUES ('hello', 'oops');     -- accepted, no error

-- RIGHT — enforce the domain with a CHECK, since the type won't
CREATE TABLE t (
    n      INTEGER CHECK (typeof(n) = 'integer'),
    amount NUMERIC CHECK (typeof(amount) IN ('integer','real'))
);
```

*Source: [SQLite — Datatypes In SQLite](https://www.sqlite.org/datatype3.html). Depth: this skill, §7; deviation map owned by `sql-standard-vs-dialect-map`.*

---

## 8. Expecting exact decimal money in SQLite

**The problem:** The model relies on a `NUMERIC`/`DECIMAL` column for exact money in SQLite, but
SQLite has no exact-decimal storage class — NUMERIC affinity converts a value "to INTEGER or REAL (in
order of preference)" ([SQLite — Datatypes](https://www.sqlite.org/datatype3.html)), so money can
land in binary float, reintroducing the §1 rounding error.

```sql
-- WRONG — SQLite may store this as REAL (binary float); cents can drift
amount NUMERIC

-- RIGHT — store exact minor units as an integer (cents), format on display
amount_cents INTEGER NOT NULL    -- 1099 == $10.99
```

*Source: [SQLite — Datatypes In SQLite](https://www.sqlite.org/datatype3.html). Depth: this skill, §1 and §7.*
