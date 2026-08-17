# Common SQL Constraint & Integrity Mistakes
## Contents

- [1. Trusting application code to enforce an invariant the schema doesn't](#1-trusting-application-code-to-enforce-an-invariant-the-schema-doesnt)
- [2. Expecting CHECK to reject a NULL](#2-expecting-check-to-reject-a-null)
- [3. Expecting a default UNIQUE column to allow at most one NULL](#3-expecting-a-default-unique-column-to-allow-at-most-one-null)
- [4. Defining a FOREIGN KEY without choosing a referential action](#4-defining-a-foreign-key-without-choosing-a-referential-action)
- [5. Relying on foreign keys in SQLite without PRAGMA foreignkeys = ON](#5-relying-on-foreign-keys-in-sqlite-without-pragma-foreignkeys-on)
- [6. Assuming PRIMARY KEY implies NOT NULL in SQLite](#6-assuming-primary-key-implies-not-null-in-sqlite)
- [7. Declaring two PRIMARY KEYs instead of PRIMARY KEY + UNIQUE](#7-declaring-two-primary-keys-instead-of-primary-key-unique)
- [8. A circular / self foreign key with no way to insert the first row](#8-a-circular-self-foreign-key-with-no-way-to-insert-the-first-row)
- [9. Using DEFAULT as if it also forbade NULL](#9-using-default-as-if-it-also-forbade-null)
- [10. A sentinel DEFAULT to "avoid NULL"](#10-a-sentinel-default-to-avoid-null)


Anti-patterns in LLM-generated DDL around constraints and referential integrity, each with wrong/right
code and a primary-source citation. The skill (`sql-constraints-and-integrity`) states the rules; this
file holds the high-frequency failure modes. The NULL *theory* behind several of these is owned by the
foundation, `sql-relational-and-null-discipline` — cited where it applies. All RIGHT examples are
standard/portable SQL unless flagged; dialect spellings route to `sql-standard-vs-dialect-map`.

---

## 1. Trusting application code to enforce an invariant the schema doesn't

**The problem:** The model validates `quantity > 0` (or uniqueness, or a foreign key) only in the service layer, assuming every writer goes through it. The next migration, ETL job, or `psql` session writes around it and corrupts the table. A constraint travels with the data; a code check does not.

```sql
-- WRONG — "the API guarantees it", so the column is unconstrained
qty INTEGER

-- RIGHT — the database enforces it for every writer
qty INTEGER NOT NULL CHECK (qty > 0)
```

*Source: [PostgreSQL — Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html). Depth: this skill, §1.*

---

## 2. Expecting `CHECK` to reject a NULL

**The problem:** The model writes `CHECK (qty > 0)` and assumes a NULL `qty` is blocked. But "a check constraint is satisfied if the check expression evaluates to true or the null value" — and "since most expressions will evaluate to the null value if any operand is null, they will not prevent null values" ([PostgreSQL — Check Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)). `NULL > 0` is UNKNOWN, which `CHECK` accepts.

```sql
-- WRONG — a NULL qty passes; CHECK only rejects a definite FALSE
qty INTEGER CHECK (qty > 0)

-- RIGHT — close the NULL hole so the predicate is only TRUE or FALSE
qty INTEGER NOT NULL CHECK (qty > 0)

-- RIGHT — if NULL is legitimately allowed, say so explicitly
qty INTEGER CHECK (qty IS NULL OR qty > 0)
```

*Source: [PostgreSQL — Check Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html). Depth: this skill, §4; three-valued-logic theory owned by `sql-relational-and-null-discipline`.*

---

## 3. Expecting a default `UNIQUE` column to allow at most one NULL

**The problem:** The model declares `email VARCHAR UNIQUE` and assumes uniqueness covers NULL. It doesn't: "By default, two null values are not considered equal ... it is possible to store duplicate rows that contain a null value" ([PostgreSQL — Unique Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)). Unlimited NULL rows get in.

```sql
-- WRONG (assumption) — still permits MANY rows with email = NULL
email VARCHAR(320) UNIQUE

-- RIGHT — require presence...
email VARCHAR(320) NOT NULL UNIQUE
-- ...or use the SQL:2023 lever to cap NULLs at one (PostgreSQL 15+; not all engines)
email VARCHAR(320) UNIQUE NULLS NOT DISTINCT
```

*Source: [PostgreSQL — Unique Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html); [Eisentraut — SQL:2023 F292](https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new). Depth: this skill, §3; NULL theory owned by `sql-relational-and-null-discipline`.*

---

## 4. Defining a `FOREIGN KEY` without choosing a referential action

**The problem:** The model writes `REFERENCES customers (id)` and stops, inheriting the default `ON DELETE NO ACTION`, which "will usually result in an error" on a parent delete ([PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)). Either callers hit surprise FK errors, or (worse, expecting a cascade) someone adds `ON DELETE CASCADE` reflexively and silently deletes history.

```sql
-- WRONG — no deliberate lifecycle; default NO ACTION errors on parent delete
customer_id INTEGER REFERENCES customers (id)

-- RIGHT — decide per relationship what happens to children
customer_id INTEGER NOT NULL REFERENCES customers (id) ON DELETE RESTRICT   -- refuse while orders exist
-- or: ON DELETE SET NULL   (anonymize; column must be nullable)
-- or: ON DELETE CASCADE    (only when children truly cannot outlive the parent)
```

*Source: [PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html). Depth: this skill, §5.*

---

## 5. Relying on foreign keys in SQLite without `PRAGMA foreign_keys = ON`

**The problem:** The model writes a perfect set of `FOREIGN KEY` declarations and runs them on SQLite, where FK enforcement is off by default — "foreign key definitions are parsed ... but foreign key constraints are not enforced" ([SQLite — Foreign Key Support](https://www.sqlite.org/foreignkeys.html)). Orphaned rows accumulate; invalid deletes succeed. The pragma is **per connection**, so it must run every time.

```sql
-- WRONG (SQLite) — FK lines present, but enforcement is OFF; PRAGMA foreign_keys returns 0
-- ... bad INSERTs and orphaning DELETEs all succeed silently

-- RIGHT (SQLite) — enable on every connection before FK-dependent work
PRAGMA foreign_keys = ON;
```

*Source: [SQLite — Foreign Key Support](https://www.sqlite.org/foreignkeys.html). Depth: this skill, §8; dialect quirks route to `sql-standard-vs-dialect-map`.*

---

## 6. Assuming `PRIMARY KEY` implies `NOT NULL` in SQLite

**The problem:** The model relies on the standard rule that a primary key is never null. In SQLite this is broken: "Unless the column is an INTEGER PRIMARY KEY ... or the column is declared NOT NULL, SQLite allows NULL values in a PRIMARY KEY column" ([SQLite — CREATE TABLE](https://www.sqlite.org/lang_createtable.html)). A NULL can land in a `TEXT PRIMARY KEY`.

```sql
-- WRONG (SQLite) — code TEXT PRIMARY KEY can still hold NULL
CREATE TABLE country (code TEXT PRIMARY KEY, name TEXT NOT NULL);

-- RIGHT — declare NOT NULL explicitly on non-INTEGER key columns
CREATE TABLE country (code TEXT PRIMARY KEY NOT NULL, name TEXT NOT NULL);
```

*Source: [SQLite — CREATE TABLE](https://www.sqlite.org/lang_createtable.html). Depth: this skill, §8.*

---

## 7. Declaring two `PRIMARY KEY`s instead of `PRIMARY KEY` + `UNIQUE`

**The problem:** The model marks both `id` and `email` as `PRIMARY KEY`, but "a table can have at most one primary key" ([PostgreSQL — Primary Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)). A second uniqueness requirement is a `UNIQUE` constraint, not a second PK.

```sql
-- WRONG — only one primary key is allowed per table
id INTEGER PRIMARY KEY,
email VARCHAR(320) PRIMARY KEY

-- RIGHT — one identity, additional uniqueness as NOT NULL UNIQUE
id INTEGER PRIMARY KEY,
email VARCHAR(320) NOT NULL UNIQUE
```

*Source: [PostgreSQL — Primary Keys](https://www.postgresql.org/docs/current/ddl-constraints.html). Depth: this skill, §2.*

---

## 8. A circular / self foreign key with no way to insert the first row

**The problem:** Two tables reference each other (or a self-reference like `manager_id`), and an immediately-checked FK makes the first `INSERT` impossible because its target doesn't exist yet. The model tries to reorder inserts forever. The fix is to defer the check to `COMMIT`.

```sql
-- WRONG — immediate check: can't insert any employee before their manager exists
manager_id INTEGER REFERENCES employees (id)

-- RIGHT — defer the check so both rows can be created in one transaction
manager_id INTEGER REFERENCES employees (id) DEFERRABLE INITIALLY DEFERRED
```

*Source: [PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html). Depth: this skill, §7; transaction mechanics route to `sql-transactions-and-isolation`.*

---

## 9. Using `DEFAULT` as if it also forbade NULL

**The problem:** The model writes `status VARCHAR DEFAULT 'pending'` and assumes the column can never be NULL. A `DEFAULT` only supplies a value when none is given; an explicit `INSERT ... (status) VALUES (NULL)` still stores NULL. Presence requires `NOT NULL`.

```sql
-- WRONG — explicit NULL bypasses the default
status VARCHAR(20) DEFAULT 'pending'

-- RIGHT — DEFAULT for convenience, NOT NULL for the guarantee
status VARCHAR(20) NOT NULL DEFAULT 'pending'
       CHECK (status IN ('pending','paid','shipped','cancelled'))
```

*Source: [PostgreSQL — Not-Null Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html). Depth: this skill, §6.*

---

## 10. A sentinel `DEFAULT` to "avoid NULL"

**The problem:** To dodge NULL, the model defaults a missing value to `-1`, `''`, or `'9999-12-31'`. The sentinel poisons aggregates and collides with real data — the Fear-of-the-Unknown antipattern. NULL is the correct marker for "absent."

```sql
-- WRONG — sentinel default that every query must special-case
end_date DATE NOT NULL DEFAULT '9999-12-31'

-- RIGHT — let absence be NULL
end_date DATE   -- nullable; test with IS NULL
```

*Source: theory owned by [`sql-relational-and-null-discipline`](https://www.postgresql.org/docs/current/ddl-constraints.html) (Karwin, *SQL Antipatterns* — "Fear of the Unknown"). Depth: this skill, §6.*
