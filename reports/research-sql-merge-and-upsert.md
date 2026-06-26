# Research: sql-merge-and-upsert

Research log for skill #15 (`sql-merge-and-upsert`). Each entry: URL, section, verbatim quote, why it
matters. Accessed 2026-06-26.

---

## Source 1 — PostgreSQL 18 docs: MERGE

URL: https://www.postgresql.org/docs/current/sql-merge.html

### 1a. MERGE syntax and arms (Synopsis)

> ```
> MERGE INTO [ ONLY ] target_table_name [ * ] [ [ AS ] target_alias ]
>     USING data_source ON join_condition
>     when_clause [...]
> ```
> where `when_clause`:
> ```
> { WHEN MATCHED [ AND condition ] THEN { merge_update | merge_delete | DO NOTHING } |
>   WHEN NOT MATCHED BY SOURCE [ AND condition ] THEN { merge_update | merge_delete | DO NOTHING } |
>   WHEN NOT MATCHED [ BY TARGET ] [ AND condition ] THEN { merge_insert | DO NOTHING } }
> ```

**Why it matters:** This is the canonical standard MERGE shape — `USING ... ON ...` plus a list of
`WHEN MATCHED` / `WHEN NOT MATCHED` arms, each able to UPDATE, DELETE, INSERT, or DO NOTHING. PostgreSQL
also adds the `BY SOURCE` / `BY TARGET` qualifiers. This is the skeleton for the "standard MERGE syntax
& arms" section.

### 1b. Version introduced (Supported Versions)

The page is titled "PostgreSQL: Documentation: 18: MERGE" and the Supported Versions strip lists
Current (18) / 18 / 17 / 16 / **15** — i.e. MERGE first appears in **PostgreSQL 15**, confirming the
spec block's "Postgres only since v15."

**Why it matters:** Drives the portability claim — MERGE is unevenly adopted; PG only got it in v15.

### 1c. Concurrency caveat + ON CONFLICT recommendation (Notes)

> "When `MERGE` is run concurrently with other commands that modify the target table, the usual
> transaction isolation rules apply; see Section 13.2 (Transaction Isolation) for an explanation on the
> behavior at each isolation level. You may also wish to consider using `INSERT ... ON CONFLICT` as an
> alternative statement which offers the ability to run an `UPDATE` if a concurrent `INSERT` occurs.
> There are a variety of differences and restrictions between the two statement types and they are not
> interchangeable."

> "If a target row is modified more than once, a uniqueness violation or cardinality violation will
> occur; the latter behavior is required by the SQL standard."

**Why it matters:** THE concurrency footgun. MERGE is not a guaranteed no-error upsert: under concurrent
INSERTs it can still raise a unique-violation, and PostgreSQL itself points to `ON CONFLICT` as the
upsert-safe alternative. MERGE is not a substitute for a UNIQUE constraint + retry. Routes theory to
sql-transactions-and-isolation.

---

## Source 2 — SQLite docs: UPSERT

URL: https://www.sqlite.org/lang_upsert.html

### 2a. ON CONFLICT DO UPDATE / DO NOTHING (Description)

> "An UPSERT is an ordinary INSERT statement that is followed by one or more ON CONFLICT clauses, as
> shown in the syntax diagram above."

Syntax: `ON CONFLICT ( indexed-column ) [WHERE expr] DO { NOTHING | UPDATE SET column = expr [WHERE expr] }`.

**Why it matters:** The shape of the PG/SQLite upsert: `INSERT ... ON CONFLICT(col) DO UPDATE SET ...`
or `DO NOTHING`.

### 2b. The `excluded.` pseudo-table (Description, §2.1)

> "Column names in the expressions of a DO UPDATE refer to the original unchanged value of the column,
> before the attempted INSERT. To use the value that would have been inserted had the constraint not
> failed, add the special 'excluded.' table qualifier to the column name."

> "The 'excluded.' prefix causes the 'phonenumber' to refer to the value for phonenumber that would have
> been inserted had there been no conflict."

**Why it matters:** Bare column in DO UPDATE = current stored row; `excluded.col` = the proposed new
value. Confusing the two is a common bug (you keep the old value or overwrite with the wrong one).

### 2c. Conflict target REQUIRES a unique constraint (Description)

> "The UPSERT processing happens only for uniqueness constraints. A 'uniqueness constraint' is an
> explicit UNIQUE or PRIMARY KEY constraint within the CREATE TABLE statement, or a unique index."

**Why it matters:** Hard dependency: ON CONFLICT(col) can only target a column covered by a UNIQUE/PK
constraint or unique index. No constraint → no conflict to detect → the whole upsert premise fails.
Routes to sql-constraints-and-integrity.

---

## Source 3 — MySQL 8.4 docs: INSERT ... ON DUPLICATE KEY UPDATE

URL: https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html (§15.2.7.2)

### 3a. Behavior — fires on any UNIQUE/PK duplicate

> "If you specify an `ON DUPLICATE KEY UPDATE` clause and a row to be inserted would cause a duplicate
> value in a `UNIQUE` index or `PRIMARY KEY`, an `UPDATE` of the old row occurs."

Example: `INSERT INTO t1 (a,b,c) VALUES (1,2,3) ON DUPLICATE KEY UPDATE c=c+1;`

**Why it matters:** MySQL's spelling of upsert. Like ON CONFLICT it also depends on a UNIQUE/PK index —
but unlike ON CONFLICT there is no explicit conflict target; it fires on ANY unique key.

### 3b. VALUES() deprecated; row alias `AS new`

> "Note: The use of `VALUES()` to refer to the new row and columns is deprecated, and subject to removal
> in a future version of MySQL."

> "Using the row alias `new`, the statement shown previously using `VALUES()` to access the new column
> values can be written in the form shown here:"
> ```sql
> INSERT INTO t1 (a,b,c) VALUES (1,2,3),(4,5,6) AS new
>   ON DUPLICATE KEY UPDATE c = new.a+new.b;
> ```

**Why it matters:** The MySQL analog of `excluded.` is `VALUES(col)` (deprecated) → `new.col` via row
alias (MySQL 8.0.19+). Important for writing current MySQL upserts.

### 3c. Multiple unique indexes → ambiguous / wrong-key footgun

> "If column `b` is also unique, the `INSERT` is equivalent to this `UPDATE` statement instead:
> `UPDATE t1 SET c=c+1 WHERE a=1 OR b=2 LIMIT 1;` ... If `a=1 OR b=2` matches several rows, only one row
> is updated. **In general, you should try to avoid using an `ON DUPLICATE KEY UPDATE` clause on tables
> with multiple unique indexes.**"

**Why it matters:** The data-loss footgun for "Who suffers." With more than one unique index, ON
DUPLICATE KEY can fire on the *wrong* key and silently UPDATE a row you did not mean to touch — only one
row, non-deterministically chosen. Unlike ON CONFLICT, you cannot name which constraint to target.

---

## Source 4 — Wikipedia: Merge (SQL)

URL: https://en.wikipedia.org/wiki/Merge_(SQL)

### 4a. History / standardization

> "It was officially introduced in the SQL:2003 standard, and expanded in the SQL:2008 standard."

Arms: `WHEN MATCHED THEN UPDATE SET ...` / `WHEN NOT MATCHED THEN INSERT (...) VALUES (...)`.

**Why it matters:** MERGE is genuinely standard (SQL:2003/2008) — the dialect alternatives are the
non-standard part.

### 4b. Engine support — the three spellings

> "PostgreSQL, Oracle Database, IBM Db2, Teradata, EXASOL, Firebird, CUBRID, H2, HSQLDB, MS SQL, MonetDB,
> Vectorwise and Apache Derby support the standard syntax."

> "PostgreSQL (v9.5+) and SQLite (v3.24+)" use the upsert term via INSERT ON CONFLICT.

> "MySQL supports the use of `INSERT ... ON DUPLICATE KEY UPDATE` syntax which can be used to achieve a
> similar effect."

**Why it matters:** The full dialect map: standard MERGE (PG15+/SQL Server/Oracle/DB2/...),
INSERT...ON CONFLICT (PostgreSQL 9.5+/SQLite 3.24+), ON DUPLICATE KEY UPDATE (MySQL/MariaDB). Routes the
table to sql-standard-vs-dialect-map.

### 4c. The term "upsert"

> "Some database implementations adopted the term upsert (a portmanteau of update and insert) to a
> database statement, or combination of statements, that inserts a record to a table in a database if
> the record does not exist or, if the record already exists, updates the existing record."

**Why it matters:** Defines the term and the goal — one atomic insert-or-update.

---

## Synthesis — the five load-bearing claims for the skill

1. **The check-then-act race (centerpiece).** App-side `SELECT` then `INSERT`-or-`UPDATE` is a
   read-modify-write across a gap; two concurrent sessions both see "absent" and both INSERT
   (duplicate-key error or duplicate rows) or both UPDATE (lost update). The fix is a single atomic
   upsert statement that the engine serializes on the unique index. Theory routes to
   sql-transactions-and-isolation (MVCC, isolation levels).

2. **Standard MERGE arms.** `MERGE INTO t USING src ON cond WHEN MATCHED THEN UPDATE/DELETE WHEN NOT
   MATCHED THEN INSERT` — SQL:2003/2008 (Source 4a, Source 1a).

3. **Three real-world spellings** (Source 4b): MERGE (PG15+/SQL Server/Oracle/DB2), INSERT...ON CONFLICT
   (PostgreSQL/SQLite), ON DUPLICATE KEY UPDATE (MySQL/MariaDB) → full table to dialect-map.

4. **Upsert REQUIRES a UNIQUE/PK constraint** to define the "conflict" (Source 2c, Source 3a) →
   depends on sql-constraints-and-integrity.

5. **MERGE concurrency footgun** (Source 1c): MERGE can still raise a unique-violation under concurrency
   and is NOT a substitute for a unique constraint + retry/ON CONFLICT.
