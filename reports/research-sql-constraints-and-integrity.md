# Research — sql-constraints-and-integrity

Date accessed: 2026-06-26. Every entry below is URL + section + verbatim quote + why it matters for the skill.

---

## Source 1 — PostgreSQL: Constraints (ddl-constraints.html)

URL: https://www.postgresql.org/docs/current/ddl-constraints.html

### §5.5.1 Check Constraints
- Verbatim: "It should be noted that a check constraint is satisfied if the check expression evaluates to true or the null value."
- Verbatim: "Since most expressions will evaluate to the null value if any operand is null, they will not prevent null values in the constrained columns."
- Why it matters: This is the CENTERPIECE pitfall. The foundation (`sql-relational-and-null-discipline`) OWNS the three-valued-logic reason (CHECK rejects only FALSE, accepts TRUE and UNKNOWN). This page is the primary-source confirmation in DDL terms — a `CHECK (qty > 0)` does not stop a NULL `qty`. Pair with `NOT NULL`.

### §5.5.2 Not-Null Constraints
- Verbatim: "A not-null constraint simply specifies that a column must not assume the null value."
- Verbatim: "A not-null constraint is functionally equivalent to creating a check constraint `CHECK (column_name IS NOT NULL)`, but in PostgreSQL creating an explicit not-null constraint is more efficient."
- Why it matters: Justifies the "pair NOT NULL with CHECK" fix and explains why NOT NULL is its own declaration rather than a CHECK.

### §5.5.3 Unique Constraints
- Verbatim: "Unique constraints ensure that the data contained in a column, or a group of columns, is unique among all the rows in the table."
- Verbatim: "By default, two null values are not considered equal in this comparison. That means even in the presence of a unique constraint it is possible to store duplicate rows that contain a null value in at least one of the constrained columns."
- Verbatim: "This behavior can be changed by adding the clause `NULLS NOT DISTINCT`".
- Verbatim: "The default behavior can be specified explicitly using `NULLS DISTINCT`. The default null treatment in unique constraints is implementation-defined according to the SQL standard, and other implementations have a different behavior."
- Why it matters: The "duplicate unique emails because the column was nullable" footgun. Foundation owns the theory; here we own the constraint syntax and the `NULLS NOT DISTINCT` lever.

### §5.5.4 Primary Keys
- Verbatim: "A primary key constraint indicates that a column, or group of columns, can be used as a unique identifier for rows in the table. This requires that the values be both unique and not null."
- Verbatim: "Adding a primary key will automatically create a unique B-tree index on the column or group of columns listed in the primary key, and will force the column(s) to be marked `NOT NULL`."
- Verbatim: "A table can have at most one primary key. (There can be any number of unique constraints, which combined with not-null constraints are functionally almost the same thing, but only one can be identified as the primary key.)"
- Why it matters: Direct support for "PRIMARY KEY = UNIQUE + NOT NULL, one per table" vs "UNIQUE: many per table, allows NULLs."

### §5.5.5 Foreign Keys + referential actions
- Verbatim (definition): "A foreign key constraint specifies that the values in a column (or a group of columns) must match the values appearing in some row of another table. We say this maintains the _referential integrity_ between two related tables."
- Verbatim (default): "The default `ON DELETE` action is `ON DELETE NO ACTION`; this does not need to be specified. This means that the deletion in the referenced table is allowed to proceed. But the foreign-key constraint is still required to be satisfied, so this operation will usually result in an error."
- Verbatim (NO ACTION deferral / DEFERRABLE link): "But checking of foreign-key constraints can also be deferred to later in the transaction... In that case, the `NO ACTION` setting would allow other commands to 'fix' the situation before the constraint is checked, for example by inserting another suitable row into the referenced table or by deleting the now-dangling rows from the referencing table."
- Verbatim (RESTRICT): "`RESTRICT` is a stricter setting than `NO ACTION`. It prevents deletion of a referenced row. `RESTRICT` does not allow the check to be deferred until later in the transaction."
- Verbatim (CASCADE): "`CASCADE` specifies that when a referenced row is deleted, row(s) referencing it should be automatically deleted as well."
- Verbatim (SET NULL / SET DEFAULT): "There are two other options: `SET NULL` and `SET DEFAULT`. These cause the referencing column(s) in the referencing row(s) to be set to nulls or their default values, respectively, when the referenced row is deleted. Note that these do not excuse you from observing any constraints. For example, if an action specifies `SET DEFAULT` but the default value would not satisfy the foreign key constraint, the operation will fail."
- Verbatim (ON UPDATE): "Analogous to `ON DELETE` there is also `ON UPDATE` which is invoked when a referenced column is changed (updated). The possible actions are the same... There is also a noticeable difference between `ON UPDATE NO ACTION` (the default) and `ON UPDATE RESTRICT`. The former will allow the update to proceed and the foreign-key constraint will be checked against the state after the update. The latter will prevent the update to run even if the state after the update would still satisfy the constraint."
- Why it matters: This is the 5-action table (NO ACTION default, RESTRICT, CASCADE, SET NULL, SET DEFAULT) and the deliberate-choice argument. The default being NO ACTION (which errors rather than cascades) is the surprising fact. Also gives the DEFERRABLE hook for circular references — checking deferred to end of transaction.

### Note on DEFERRABLE
The §5.5 page references deferral of checks ("checking of foreign-key constraints can also be deferred to later in the transaction") but the explicit `DEFERRABLE INITIALLY DEFERRED` syntax lives on the CREATE TABLE / SET CONSTRAINTS reference pages. The skill cites the deferral language above for the circular-reference use case.

---

## Source 2 — SQLite: Foreign Key Support (foreignkeys.html)

URL: https://www.sqlite.org/foreignkeys.html

### §2 Enabling Foreign Key Support
- Verbatim: "Foreign key constraints are disabled by default (for backwards compatibility), so must be enabled separately for each database connection."
- Verbatim: "Assuming the library is compiled with foreign key constraints enabled, it must still be enabled by the application at runtime, using the PRAGMA foreign_keys command."
- Example: `PRAGMA foreign_keys = ON;`
- Verbatim (parsed-but-not-enforced scenario): "foreign key definitions are parsed and may be queried using PRAGMA foreign_key_list, but foreign key constraints are not enforced."
- Status check: `PRAGMA foreign_keys;` returns `0` (off) until set `ON` → `1`.
- Why it matters: THE big SQLite footgun. A schema with picture-perfect `FOREIGN KEY ... REFERENCES` declarations enforces NOTHING by default — orphaned rows and silent corruption. Must be enabled per connection. This drives the "Who suffers" orphaned-rows scenario and the portability block.

---

## Source 3 — SQLite: CREATE TABLE (lang_createtable.html)

URL: https://www.sqlite.org/lang_createtable.html

### Column-constraint syntax
- Column constraints include: `PRIMARY KEY [ASC|DESC] conflict-clause [AUTOINCREMENT]`, `NOT NULL conflict-clause`, `UNIQUE conflict-clause`, `CHECK ( expr )`, `DEFAULT (...)`, `COLLATE`, foreign-key-clause, `GENERATED ALWAYS AS ( expr ) [VIRTUAL|STORED]`.

### INTEGER PRIMARY KEY / rowid alias
- Verbatim: "With one exception noted below, if a rowid table has a primary key that consists of a single column and the declared type of that column is 'INTEGER' in any mixture of upper and lower case, then the column becomes an alias for the rowid."
- Verbatim: "A PRIMARY KEY column only becomes an integer primary key if the declared type name is exactly 'INTEGER'. Other integer type names like 'INT' or 'BIGINT'... causes the primary key column to behave as an ordinary table column with integer affinity and a unique index, not as an alias for the rowid."
- Why it matters: The portable-looking `id INTEGER PRIMARY KEY` has special rowid-alias meaning in SQLite. Worth a portability note; deep identity-column treatment routes to `sql-generated-and-identity-columns`.

### NULL handling in PK / UNIQUE
- Verbatim (PK): "According to the SQL standard, PRIMARY KEY should always imply NOT NULL. Unfortunately, due to a bug in some early versions, this is not the case in SQLite. Unless the column is an INTEGER PRIMARY KEY or the table is a WITHOUT ROWID table or a STRICT table or the column is declared NOT NULL, SQLite allows NULL values in a PRIMARY KEY column."
- Verbatim (UNIQUE): "For the purposes of UNIQUE constraints, NULL values are considered distinct from all other values, including other NULLs."
- Why it matters: A second SQLite footgun — `PRIMARY KEY` does NOT imply `NOT NULL` in SQLite (a NULL can slip into a non-INTEGER PK column). Confirms UNIQUE allows multiple NULLs in SQLite too (matches foundation). Reinforces "declare NOT NULL explicitly."

---

## Source 4 — Eisentraut: SQL:2023 is finished (F292)

URL: https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new

### F292 — UNIQUE null treatment
- Verbatim: "This feature deals with how null values are handled in unique constraints."
- Verbatim: "If nulls are considered distinct, then having more than one of them won't cause a unique constraint violation. If nulls are considered not distinct, then having more than one violates uniqueness."
- Options: `UNIQUE NULLS DISTINCT` (allows multiple nulls) / `UNIQUE NULLS NOT DISTINCT` (treats multiple nulls as a violation).
- Verbatim: "The default for this option is implementation-defined, so existing implementations can keep their current behavior."
- When: standardized in SQL:2023 (feature F292).
- Why it matters: Dates the `NULLS [NOT] DISTINCT` syntax (SQL:2023 F292, implemented in PostgreSQL 15+). Availability is uneven → route to `sql-standard-vs-dialect-map`. Foundation introduced the lever; this is the standards provenance.

---

## Synthesis for the skill

1. Integrity belongs in the schema, not the app — declarative constraints are enforced for every writer (psql, ORM, migration, ad-hoc); app-layer checks are not. (framing)
2. PRIMARY KEY = UNIQUE + NOT NULL, exactly one per table, auto-indexed; UNIQUE = any number, allows NULLs by default. (PG §5.5.4, §5.5.3)
3. UNIQUE allows MULTIPLE NULLs by default (NULLS DISTINCT) — the "duplicate unique emails" trap; SQL:2023 `NULLS NOT DISTINCT` permits at most one. Theory owned by foundation. (PG §5.5.3, F292)
4. CHECK passes on UNKNOWN/NULL — CENTERPIECE; pair with NOT NULL. Theory (3VL) owned by foundation; DDL confirmation here. (PG §5.5.1, §5.5.2)
5. FOREIGN KEY referential actions — choose deliberately; default is NO ACTION (errors, does not cascade). 5-action table. (PG §5.5.5)
6. DEFERRABLE INITIALLY DEFERRED for circular references — checks deferred to end of transaction. (PG §5.5.5 deferral language)
7. SQLite `PRAGMA foreign_keys = ON` — FK enforcement OFF by default, per connection. Plus SQLite PK does not imply NOT NULL. (SQLite foreignkeys §2, createtable)
