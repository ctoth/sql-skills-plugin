---
name: sql-constraints-and-integrity
description: >-
  Guides database-enforced data integrity — push the rules into the schema as declarative constraints rather than re-checking them in every application that writes. Covers PRIMARY KEY (= UNIQUE + NOT NULL, one per table) vs UNIQUE (many per table, and by default permits multiple NULLs — the SQL:2023 NULLS NOT DISTINCT lever fixes the duplicate-"unique"-emails trap), CHECK constraints and the pitfall that a CHECK passes when it evaluates to UNKNOWN/NULL (so pair it with NOT NULL), NOT. Auto-invokes when writing or editing CREATE TABLE/ALTER TABLE constraints, FOREIGN KEY/REFERENCES, CHECK/UNIQUE/NOT NULL/DEFAULT, or on "enforce this in the DB or the app" / "why are there orphaned rows" / "why did duplicate emails get in" decisions.
allowed-tools: Read, Glob, Grep
---

# SQL Constraints and Integrity

> "It should be noted that a check constraint is satisfied if the check expression evaluates to true or the null value."
> — [PostgreSQL — Check Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)

> "Foreign key constraints are disabled by default (for backwards compatibility), so must be enabled separately for each database connection."
> — [SQLite — Foreign Key Support](https://www.sqlite.org/foreignkeys.html)

A constraint is a promise the *database* keeps, no matter who writes the row — your ORM, a migration, a backfill script, a colleague at a `psql` prompt, or a cron job. That is the whole point of this skill: **integrity belongs in the schema, not in application code.** A rule enforced in one service's validation layer is silently absent from every other path to the table; a `CHECK`, `FOREIGN KEY`, or `UNIQUE` is enforced for all of them. This skill builds directly on `sql-relational-and-null-discipline`, which **owns** the two NULL facts that decide whether your constraints actually do what you think: *a `CHECK` passes on UNKNOWN*, and *a default `UNIQUE` permits multiple NULLs*. Those are cited and built upon below — not re-derived.

---

## 1. Integrity Belongs in the Schema, Not the App (framing)

The reflex to validate in application code — "the API checks that `quantity > 0`, so the column doesn't need a `CHECK`" — is a bet that *every* writer, forever, goes through that one code path. It never holds: the next migration, the analytics ETL, the manual hotfix, the second microservice, and the test fixture all reach the table directly. A declarative constraint is the only rule that travels with the data.

Declared constraints are also documentation the engine enforces: a reader of the schema sees that `email` is `UNIQUE`, that `order.customer_id` `REFERENCES customers`, that `status` is one of a fixed set. The discipline below is *which* constraint expresses each invariant, and how NULL quietly undermines the two most common ones.

---

## 2. `PRIMARY KEY` vs `UNIQUE`

They overlap but are not interchangeable. A primary key "requires that the values be both unique and not null," and "adding a primary key will automatically create a unique B-tree index on the column(s) ... and will force the column(s) to be marked `NOT NULL`" ([PostgreSQL — Primary Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)). So **`PRIMARY KEY` = `UNIQUE` + `NOT NULL`**, plus the table's identifying index.

The decisive differences:

- **A table can have at most one primary key**, but "there can be any number of unique constraints" ([PostgreSQL — Primary Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)). Use the PK for the row's identity; use `UNIQUE` for *other* columns that must not collide (email, slug, SKU).
- **`UNIQUE` does not imply `NOT NULL`** — and that gap is §3's trap.

```sql
-- WRONG — two ways to identify a row, and a nullable "unique" key
CREATE TABLE users (
  id        INTEGER PRIMARY KEY,
  email     VARCHAR(320) PRIMARY KEY,   -- error/ignored: only ONE primary key per table
  username  VARCHAR(50)  UNIQUE          -- allows multiple NULL usernames (§3)
);

-- RIGHT — one identity, additional uniqueness as its own NOT NULL UNIQUE
CREATE TABLE users (
  id        INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email     VARCHAR(320) NOT NULL UNIQUE,
  username  VARCHAR(50)  NOT NULL UNIQUE
);
```

(Identity/auto-increment column mechanics belong to `sql-generated-and-identity-columns`; key *choice* rationale to `sql-schema-design-and-normalization`.)

---

## 3. `UNIQUE` Permits Multiple NULLs — `NULLS NOT DISTINCT`

This is the "duplicate unique emails" trap, and its root is owned by the foundation: under three-valued logic *nothing equals NULL, not even another NULL*. The DDL consequence is explicit: "By default, two null values are not considered equal in this comparison. That means even in the presence of a unique constraint it is possible to store duplicate rows that contain a null value in at least one of the constrained columns" ([PostgreSQL — Unique Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)). A bare `UNIQUE` column therefore accepts an *unbounded* number of NULL rows.

SQL:2023 added feature F292 to make this a deliberate choice: "If nulls are considered distinct, then having more than one of them won't cause a unique constraint violation. If nulls are considered not distinct, then having more than one violates uniqueness" ([Eisentraut — SQL:2023](https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new)). The default "is implementation-defined" ([Eisentraut — SQL:2023](https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new)), so the standard default (`NULLS DISTINCT`) permits the many-NULL behavior unless you override it.

```sql
-- WRONG — assumes "at most one NULL email"; actually permits unlimited NULL rows
email VARCHAR(320) UNIQUE

-- RIGHT — if the value is mandatory, forbid the NULL outright
email VARCHAR(320) NOT NULL UNIQUE

-- RIGHT — if NULL is legitimately "not provided" but you want at most one,
-- use the SQL:2023 lever (PostgreSQL 15+; not all engines — §9)
email VARCHAR(320) UNIQUE NULLS NOT DISTINCT
```

The three-valued-logic *why* (and the contrast with `GROUP BY`/`DISTINCT`, which fold NULLs together) lives in `sql-relational-and-null-discipline`; this skill owns the constraint spelling.

---

## 4. `CHECK` Passes When It Evaluates to UNKNOWN — Pair It With `NOT NULL` (centerpiece)

The single most common constraint bug. A `CHECK` does not "accept TRUE"; it *rejects FALSE* — so a predicate that evaluates to UNKNOWN (because an operand is NULL) **passes**. PostgreSQL states it directly: "a check constraint is satisfied if the check expression evaluates to true or the null value" and "since most expressions will evaluate to the null value if any operand is null, they will not prevent null values in the constrained columns" ([PostgreSQL — Check Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)).

The three-valued-logic reason — `WHERE` keeps only TRUE while `CHECK` rejects only FALSE, so the *same* `qty > 0` predicate drops a NULL row in a `WHERE` but admits it in a `CHECK` — is owned by `sql-relational-and-null-discipline`. The schema-level fix is to close the NULL hole explicitly:

```sql
-- WRONG — a NULL qty slips straight through: NULL > 0 is UNKNOWN, and CHECK accepts UNKNOWN
CREATE TABLE line_items (
  id  INTEGER PRIMARY KEY,
  qty INTEGER CHECK (qty > 0)
);
INSERT INTO line_items (id, qty) VALUES (1, NULL);   -- succeeds (bug)

-- RIGHT — forbid the NULL so the predicate can only be TRUE or FALSE
CREATE TABLE line_items (
  id  INTEGER PRIMARY KEY,
  qty INTEGER NOT NULL CHECK (qty > 0)
);
```

`NOT NULL` is the right tool for "must be present": it "specifies that a column must not assume the null value" and is "functionally equivalent to ... `CHECK (column_name IS NOT NULL)`, but ... more efficient" ([PostgreSQL — Not-Null Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)). If a column is genuinely nullable but constrained when present, encode that intent — `CHECK (qty IS NULL OR qty > 0)` — rather than relying on the accidental UNKNOWN pass.

---

## 5. `FOREIGN KEY` — Choose the Referential Action Deliberately

A foreign key "specifies that the values in a column ... must match the values appearing in some row of another table ... this maintains the *referential integrity* between two related tables" ([PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)). The part that gets skipped — and causes either surprise errors or data loss — is the `ON DELETE` / `ON UPDATE` action. There are five, and **the default is `NO ACTION`, which does *not* cascade**:

| Action | What happens to referencing rows when the referenced row is deleted/updated | Source |
|---|---|---|
| `NO ACTION` (default) | Operation proceeds, but the FK must still hold at check time → "usually result in an error"; can be deferred within a transaction | PG §5.5.5 |
| `RESTRICT` | "Prevents deletion of a referenced row"; stricter than NO ACTION — the check **cannot** be deferred | PG §5.5.5 |
| `CASCADE` | "Row(s) referencing it should be automatically deleted as well" (on UPDATE: the new value is copied down) | PG §5.5.5 |
| `SET NULL` | Referencing column(s) "set to nulls" (column must be nullable) | PG §5.5.5 |
| `SET DEFAULT` | Referencing column(s) set to "their default values" (the default must itself satisfy the FK) | PG §5.5.5 |

All quotes from [PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html). The choice is a data-modeling decision: deleting a customer should usually `RESTRICT` (refuse while orders exist) or `SET NULL` (anonymize) — rarely a silent `CASCADE` that erases order history. Deleting a cart *should* `CASCADE` to its line items.

```sql
-- WRONG — no action chosen, and the intent is unclear; default NO ACTION will
-- error on customer delete, surprising callers who expected a clean delete
CREATE TABLE orders (
  id          INTEGER PRIMARY KEY,
  customer_id INTEGER REFERENCES customers (id)
);

-- RIGHT — state the lifecycle explicitly per relationship
CREATE TABLE orders (
  id          INTEGER PRIMARY KEY,
  customer_id INTEGER NOT NULL REFERENCES customers (id) ON DELETE RESTRICT,
  ...
);
CREATE TABLE order_items (
  id       INTEGER PRIMARY KEY,
  order_id INTEGER NOT NULL REFERENCES orders (id) ON DELETE CASCADE   -- items die with the order
);
```

Note `ON UPDATE NO ACTION` vs `ON UPDATE RESTRICT` differ subtly: NO ACTION "will allow the update to proceed and the foreign-key constraint will be checked against the state after the update," while RESTRICT "will prevent the update to run even if the state after the update would still satisfy the constraint" ([PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)).

---

## 6. `NOT NULL` and `DEFAULT`

`NOT NULL` is the cheapest, highest-value constraint — it is what makes §3 and §4 watertight. Pair it with `DEFAULT` when a column is mandatory but has a sensible fallback, so existing rows and partial inserts stay valid:

```sql
-- RIGHT — present-and-valid by construction
status     VARCHAR(20) NOT NULL DEFAULT 'pending'
            CHECK (status IN ('pending','paid','shipped','cancelled')),
created_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP
```

A `DEFAULT` does **not** add `NOT NULL` — a writer can still explicitly insert `NULL` past a default unless `NOT NULL` is also present. And resist the urge to default to a sentinel (`'9999-12-31'`, `-1`, `''`) to "avoid NULL"; that is the Fear-of-the-Unknown antipattern owned by `sql-relational-and-null-discipline`.

---

## 7. `DEFERRABLE` Constraints for Circular References

When two tables reference each other (or you bulk-load rows whose FK targets arrive later in the same transaction), an immediately-checked constraint makes the first `INSERT` impossible — the row it points at doesn't exist yet. PostgreSQL notes that "checking of foreign-key constraints can also be deferred to later in the transaction," and in that case even `NO ACTION` "would allow other commands to 'fix' the situation before the constraint is checked, for example by inserting another suitable row into the referenced table" ([PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)). The lever is `DEFERRABLE INITIALLY DEFERRED`:

```sql
-- RIGHT — break a circular FK by deferring the check to COMMIT
ALTER TABLE employees
  ADD CONSTRAINT fk_mgr FOREIGN KEY (manager_id) REFERENCES employees (id)
  DEFERRABLE INITIALLY DEFERRED;
-- now both an employee and their (later-inserted) manager can be created in one transaction
```

`RESTRICT` is the exception that "does not allow the check to be deferred" ([PostgreSQL — Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html)) — reach for `NO ACTION` (the default) when you need deferral. Transaction mechanics (and `SET CONSTRAINTS`) live in `sql-transactions-and-isolation`.

---

## 8. The SQLite Footguns — FK Enforcement Is OFF by Default

The most dangerous portability gap in this whole skill: SQLite parses your `FOREIGN KEY` declarations and then, by default, **enforces nothing**. "Foreign key constraints are disabled by default (for backwards compatibility), so must be enabled separately for each database connection" ([SQLite — Foreign Key Support](https://www.sqlite.org/foreignkeys.html)). With them off, "foreign key definitions are parsed ... but foreign key constraints are not enforced" ([SQLite — Foreign Key Support](https://www.sqlite.org/foreignkeys.html)) — so orphaned rows accumulate silently and a `DELETE` that should fail just succeeds.

```sql
-- WRONG (SQLite) — the schema looks airtight, but FK enforcement is OFF; orphans get in
-- (default: PRAGMA foreign_keys returns 0)

-- RIGHT (SQLite) — run this on EVERY connection, before any FK-dependent work
PRAGMA foreign_keys = ON;
```

Two more SQLite traps to know: `PRIMARY KEY` does **not** imply `NOT NULL` — "Unless the column is an INTEGER PRIMARY KEY ... or the column is declared NOT NULL, SQLite allows NULL values in a PRIMARY KEY column" ([SQLite — CREATE TABLE](https://www.sqlite.org/lang_createtable.html)) — so always write `NOT NULL` on key columns. And `id INTEGER PRIMARY KEY` is special: such a column "becomes an alias for the rowid" ([SQLite — CREATE TABLE](https://www.sqlite.org/lang_createtable.html)), but only when the declared type is *exactly* `INTEGER` (not `INT`/`BIGINT`).

---

## 9. Portability Snapshot

The standard defines all six constraint types and the five referential actions; the dangerous deviations are about *enforcement* and *defaults*:

| Behavior | PostgreSQL | SQLite | MySQL | Oracle |
|---|---|---|---|---|
| FK enforced by default | yes | **NO — `PRAGMA foreign_keys=ON` per connection** | yes (InnoDB) | yes |
| `PRIMARY KEY` implies `NOT NULL` | yes | **no (unless `INTEGER PK`/declared)** | yes | yes |
| Default `UNIQUE` allows multiple NULLs | yes | yes | yes | yes |
| `UNIQUE NULLS NOT DISTINCT` (SQL:2023 F292) | yes (15+) | no | no | no |
| `CHECK` actually enforced | yes | yes | yes (8.0.16+) | yes |
| `ON UPDATE`/`ON DELETE` actions supported | all 5 | all 5 (when on) | CASCADE/SET NULL/RESTRICT/NO ACTION (no SET DEFAULT) | NO ACTION/CASCADE/SET NULL only |
| `DEFERRABLE` constraints | yes | limited (FK only, via PRAGMA) | no | yes |

SQLite's FK-off-by-default and PK-allows-NULL are the two that bite hardest in cross-engine code. `UNIQUE NULLS NOT DISTINCT` availability and the exact set of supported referential actions vary — for any dialect spelling or support question, route to **`sql-standard-vs-dialect-map`**.

---

## 10. Who Suffers When Integrity Is Left to the App

A missing or misunderstood constraint doesn't error at definition time — it lets bad data in quietly, and someone downstream inherits it:

- The **on-call engineer** staring at thousands of orphaned `order_items` whose parent `orders` were deleted — because the schema *had* `FOREIGN KEY` lines but ran on SQLite with `PRAGMA foreign_keys` still `0` (§8). Every declaration was real; none was enforced. Referential integrity was a comment, not a guarantee.
- The **support team** fielding "I can't sign up" tickets that trace to three accounts already holding `email = NULL` in a column everyone believed was `UNIQUE` — the default `NULLS DISTINCT` let them all in (§3). The uniqueness the schema seemed to promise was never there for NULLs.
- The **analyst** whose revenue report is skewed by rows with `qty = NULL` that a `CHECK (qty > 0)` was supposed to block — but `CHECK` accepts UNKNOWN, so the NULLs sailed through (§4). The constraint *looked* like a guardrail and wasn't.
- The **next maintainer** who runs a routine `DELETE FROM customers WHERE id = 42` and gets a foreign-key error they didn't expect — because the FK defaulted to `NO ACTION` and no one decided whether the relationship should `RESTRICT`, `CASCADE`, or `SET NULL` (§5).

A declared, NULL-aware constraint is empathy in the schema: it fails the bad write *now*, at the one place every writer passes through, instead of leaving a silent landmine for whoever queries the table next.

---

## 11. Routing to Related Skills

- `sql-relational-and-null-discipline` — **the foundation this skill builds on**: it owns *why* a `CHECK` passes on UNKNOWN (the `WHERE`-keeps-TRUE / `CHECK`-rejects-FALSE asymmetry) and *why* a default `UNIQUE` allows multiple NULLs (nothing equals NULL). Read it for the three-valued-logic theory; this skill owns the DDL.
- `sql-schema-design-and-normalization` — choosing *which* columns are keys, natural vs surrogate keys, and normalization rationale (§2).
- `sql-generated-and-identity-columns` — `GENERATED ALWAYS AS IDENTITY`, auto-increment, and SQLite's `INTEGER PRIMARY KEY` rowid alias (§2, §8).
- `sql-data-types-and-numerics` — choosing the column type a constraint is declared on (§6).
- `sql-standard-vs-dialect-map` — FK-enforcement quirks, referential-action support, and `NULLS NOT DISTINCT` availability per engine (§9).

---

## 12. Reference Files

High-frequency constraint/integrity anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
