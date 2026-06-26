# Research — sql-data-types-and-numerics

Primary-source research for the `sql-data-types-and-numerics` skill. Each finding records the
URL, the section it came from, a verbatim quote, and why it matters for the skill. Accessed
2026-06-26.

---

## Source A — PostgreSQL: Numeric Types
URL: https://www.postgresql.org/docs/current/datatype-numeric.html

### A1. NUMERIC/DECIMAL is exact; recommended for money (CENTERPIECE)
Section: 8.1.2 Arbitrary Precision Numbers
> "The type `numeric` can store numbers with a very large number of digits. It is especially
> recommended for storing monetary amounts and other quantities where exactness is required.
> Calculations with `numeric` values yield exact results where possible, e.g., addition,
> subtraction, multiplication."

Why it matters: This is the spine of the centerpiece section. The standard's own exact type
explicitly names money as its use case and guarantees exact results for the operations that
financial math relies on. Pairs `numeric`/`decimal` as the correct money type.

### A2. real/double precision are inexact and store approximations (CENTERPIECE)
Section: 8.1.3 Floating-Point Types
> "The data types `real` and `double precision` are inexact, variable-precision numeric types."

> "Inexact means that some values cannot be converted exactly to the internal format and are
> stored as approximations, so that storing and retrieving a value might show slight
> discrepancies."

Why it matters: Direct, citable proof that binary floating point cannot store decimal fractions
like 0.1 exactly. This is the WRONG side of the centerpiece — values silently drift, and the
drift compounds when summed.

### A3. Explicit recommendation against float for money
Section: 8.1.3 Floating-Point Types
> "If you require exact storage and calculations (such as for monetary amounts), use the
> `numeric` type instead."

Why it matters: The docs themselves issue the rule the skill enforces: money is exact storage →
NUMERIC, never float. Lets the skill cite the rule, not just assert it.

### A4. Integer ranges
Section: Table 8.2 (Integer Types) / 8.1.1
- `smallint`: 2 bytes, range "-32768 to +32767"
- `integer`: 4 bytes, range "-2147483648 to +2147483647"
- `bigint`: 8 bytes, range "-9223372036854775808 to +9223372036854775807"

Why it matters: Exact range boundaries for the integer-sizing section. The +2,147,483,647 ceiling
is the famous "2.1 billion" overflow that kills an INTEGER surrogate key at scale; BIGINT is the
fix.

### A5. Overflow on precision/scale
Section: 8.1.2
> "if the number of digits to the left of the decimal point exceeds the declared precision minus
> the declared scale, an error is raised."

> "For example, a column declared as `NUMERIC(3, 1)` will round values to 1 decimal place and can
> store values between -99.9 and 99.9, inclusive."

Why it matters: Grounds the precision/scale + overflow section. NUMERIC(p,s) is not free-form: the
integer part is bounded by p−s, and exceeding it errors (it does not silently wrap). Scale rounds.

### A6. Serial/sequence gaps
Section: 8.1.4 Serial Types
> "Because `smallserial`, `serial` and `bigserial` are implemented using sequences, there may be
> "holes" or gaps in the sequence of values which appears in the column, even if no rows are ever
> deleted. A value allocated from the sequence is still "used up" even if a row containing that
> value is never successfully inserted into the table column."

Why it matters: Brief note that auto-increment columns are not gap-free; full identity/sequence
treatment is routed to `sql-generated-and-identity-columns`. Also: `serial` is a Postgres
shorthand, not standard SQL.

---

## Source B — PostgreSQL: Data Types (categories overview)
URL: https://www.postgresql.org/docs/current/datatype.html

### B1. Character types
Section: Table 8.1
- `character varying [ (n) ]` / `varchar [ (n) ]` — "variable-length character string"
- `character [ (n) ]` / `char [ (n) ]` — "fixed-length character string"
- `text` — "variable-length character string"

Why it matters: Establishes the standard spellings. `CHARACTER VARYING` is the standard name;
`varchar` is the alias. `text` is a non-standard but widely available unbounded variant.

### B2. Boolean
Section: Table 8.1
- `boolean` / `bool` — "logical Boolean (true/false)"

Why it matters: Standard SQL has a real BOOLEAN type. The portability story (SQL Server BIT, old
MySQL TINYINT(1)) is the deviation the skill routes to the dialect map.

### B3. Binary data
Section: Table 8.1
- `bytea` — "binary data ("byte array")"

Why it matters: Postgres spells binary data `bytea`; the standard spelling is `BLOB` / `BINARY
LARGE OBJECT` (and `CLOB` for character LOBs). Names the portability split for the BLOB/CLOB
section.

---

## Source C — PostgreSQL: Character Types (detail)
URL: https://www.postgresql.org/docs/current/datatype-character.html

### C1. varchar(n) overflow = error
Section: 8.3
> "Both of these types can store strings up to `n` characters (not bytes) in length. An attempt to
> store a longer string into a column of these types will result in an error, unless the excess
> characters are all spaces, in which case the string will be truncated to the maximum length."

Why it matters: A too-small VARCHAR(n) limit does not silently truncate on INSERT — it errors and
rejects the row. That is the "truncated name" failure mode in Who-suffers: an arbitrary limit that
rejects legitimate long values.

### C2. char(n) is blank-padded
Section: 8.3
> "Values of type `character` are physically padded with spaces to the specified width `n`, and
> are stored and displayed that way."

> "If the string to be stored is shorter than the declared length, values of type `character` will
> be space-padded; values of type `character varying` will simply store the shorter string."

Why it matters: CHAR(n) pads with trailing spaces, which surprises comparisons and round-trips.
Justifies the "default to VARCHAR/TEXT, reach for CHAR only for genuinely fixed-width codes" rule.

### C3. No performance advantage to a length limit
Section: 8.3 (Tip)
> "There is no performance difference among these three types, apart from increased storage space
> when using the blank-padded type, and a few extra CPU cycles to check the length when storing
> into a length-constrained column. While `character(n)` has performance advantages in some other
> database systems, there is no such advantage in PostgreSQL; in fact `character(n)` is usually the
> slowest of the three because of its additional storage costs."

> "In most situations `text` or `character varying` should be used instead."

Why it matters: Demolishes the VARCHAR(255) cargo cult. The limit buys no speed; it only risks
rejecting valid data. The number 255 is an artifact of old MySQL row-format encoding, not a
meaningful business constraint. Pick a limit only when the domain has a real maximum.

---

## Source D — SQLite: Datatypes In SQLite (THE big deviation)
URL: https://www.sqlite.org/datatype3.html

### D1. Dynamic typing — value carries the type, not the column
Section: 1. Datatypes In SQLite
> "SQLite uses a more general dynamic type system. In SQLite, the datatype of a value is associated
> with the value itself, not with its container."

Why it matters: The headline deviation. In standard SQL the column's declared type constrains every
value; in SQLite it does not. A declared type is a hint, not a contract.

### D2. Affinity is recommended, not required
Section: 3. Type Affinity
> "The type affinity of a column is the recommended type for data stored in that column. The
> important idea here is that the type is recommended, not required. Any column can still store any
> type of data."

Why it matters: Spells out the consequence: a column declared `INTEGER` or `NUMERIC` will still
accept and store a string or a float. The type system you wrote does not enforce itself. This is
the single sentence the skill quotes to anchor the deviation.

### D3. NUMERIC affinity definition
Section: 3.1 Determination Of Column Affinity
> "A column with NUMERIC affinity may contain values using all five storage classes. When text data
> is inserted into a NUMERIC column, the storage class of the text is converted to INTEGER or REAL
> (in order of preference) if the text is a well-formed integer or real literal, respectively."

Why it matters: Shows that `DECIMAL`/`NUMERIC` in SQLite does not give you exact decimal storage —
it can fall back to REAL (binary float). So the "money = NUMERIC" rule must be qualified for SQLite,
which has no true exact-decimal storage class.

### D4. INTEGER affinity behaves like NUMERIC
Section: 3.1
> "A column that uses INTEGER affinity behaves the same as a column with NUMERIC affinity. The
> difference between INTEGER and NUMERIC affinity is only evident in a CAST expression: The
> expression "CAST(4.0 AS INT)" returns an integer 4, whereas "CAST(4.0 AS NUMERIC)" leaves the
> value as a floating-point 4.0."

Why it matters: Reinforces that SQLite collapses much of the numeric type system.

### D5. No separate BOOLEAN
Section: 2.1 / Boolean Datatype
> "SQLite does not have a separate Boolean storage class. Instead, Boolean values are stored as
> integers 0 (false) and 1 (true)."

Why it matters: SQLite's BOOLEAN is a façade over INTEGER 0/1, paralleling SQL Server BIT and old
MySQL TINYINT(1). Strengthens the BOOLEAN-portability section.

### D6. No separate date/time (and by extension, decimal) storage class
Section: 2.2
> "SQLite does not have a storage class set aside for storing dates and/or times."

Why it matters: Confirms SQLite lacks dedicated DATE and exact-DECIMAL storage classes; routed to
`sql-datetime-and-intervals` for dates, noted here for decimals.

### D7. Affinity-determination rules (the substring quirk)
Section: 3.1
> "If the declared type contains the string "INT" then it is assigned INTEGER affinity."
> "If the declared type of the column contains any of the strings "CHAR", "CLOB", or "TEXT" then
> that column has TEXT affinity."
> "If the declared type for a column contains the string "BLOB" or if no type is specified then the
> column has affinity BLOB."
> "If the declared type for a column contains any of the strings "REAL", "FLOA", or "DOUB" then the
> column has REAL affinity."
> "Otherwise, the affinity is NUMERIC."

Why it matters: Affinity is decided by substring matching on the declared type name — so `VARCHAR`
→ TEXT, `DOUBLE` → REAL, and an unknown name like `DECIMAL` → NUMERIC. Explains why copying a
standard DDL into SQLite "works" but enforces nothing.

---

## Synthesis — section map for the skill

1. **Exact vs approximate numerics (CENTERPIECE)** — A1, A2, A3, D3. Money/exact counts =
   NUMERIC/DECIMAL(p,s); never FLOAT/DOUBLE. Worked rounding-error example: 0.1 not representable,
   sums drift by a cent.
2. **Precision/scale & overflow** — A5. NUMERIC(p,s): p−s bounds the integer part; overflow errors;
   scale rounds.
3. **Integer sizing** — A4, A6. SMALLINT/INTEGER/BIGINT by range; the 2.1B INTEGER ceiling; serial
   gaps (route to identity skill).
4. **CHARACTER VARYING & the VARCHAR(255) cargo cult** — B1, C1, C2, C3. No perf benefit; arbitrary
   limits reject valid data; CHAR pads; 255 is a MySQL row-format artifact.
5. **BOOLEAN portability** — B2, D5. Standard BOOLEAN; SQL Server BIT; old MySQL TINYINT(1); SQLite
   INTEGER 0/1.
6. **BLOB/CLOB** — B3. Standard BLOB/CLOB vs Postgres bytea/text.
7. **SQLite dynamic typing / affinity deviation** — D1, D2, D4, D7. Declared types advisory; any
   value in any column; substring affinity rules.
8. **Portability block** — route BOOLEAN/BIT, money types, SQLite affinity to
   `sql-standard-vs-dialect-map`.
9. **Who suffers** — accounting discrepancy ($0.01 off from float sums); rejected/truncated name
   from arbitrary VARCHAR limit; INTEGER id overflow at 2.1 billion.
