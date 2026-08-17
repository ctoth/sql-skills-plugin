---
name: sql-temporal-tables
description: >-
  Guides SQL:2011 temporal tables — system-versioned tables (`PERIOD FOR SYSTEM_TIME` + `WITH SYSTEM VERSIONING`) that make the engine auto-record a full history so you query past states with `FOR SYSTEM_TIME AS OF / FROM..TO / BETWEEN / ALL` instead of hand-rolling trigger-based history tables, application-time period tables (`PERIOD FOR name`, `UPDATE/DELETE ... FOR PORTION OF`, `WITHOUT OVERLAPS`) for valid-time, and the system-time vs valid-time vs bitemporal distinction. Auto-invokes when writing or editing audit-history / "as-of" / point-in-time queries, system- or valid-time period tables, slowly-changing-dimension history, temporal primary keys, or "track every change" / "what did this row look like last March" requests. Owns the standard DDL and query syntax only — versioning/storage mechanics, snapshot semantics, and retention/GC horizon belong to the sibling mvcc-time-travel-queries.
allowed-tools: Read, Glob, Grep
---

# SQL Temporal Tables (SQL:2011)

> "Temporal tables ... are a database feature that brings built-in support for providing information about data stored in the table at any point in time, rather than only the data that is correct at the current moment in time."
> — [Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)

> "Definition of system-versioned tables (elsewhere called transaction time tables), using the `PERIOD FOR SYSTEM_TIME` annotation ... System time periods are maintained automatically."
> — [Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)

SQL:2011 standardized two ways to put **time** inside a table. *System-versioned* tables let the engine keep a full history of every row automatically — the standard answer to "audit every change" and "what did this row say last March" without writing a single trigger. *Application-time period* tables let the application record *when a fact is true in the real world*. This skill builds on `sql-relational-and-null-discipline` (set semantics, NULL) and `sql-datetime-and-intervals` (timestamps, half-open intervals) and owns only the **standard DDL and query syntax**. It does **not** own how versions are stored, how snapshots are computed, or how far back you can travel — that is the hard boundary with `mvcc-time-travel-queries` (see §6).

---

## 1. Why — Audit & As-Of Without Hand-Rolled Trigger History

The recurring task: "keep a history of every change so we can reconstruct any past state and audit who-changed-what." The reflex is a shadow `_history` table plus triggers (or app code) that copy the old row on every `UPDATE`/`DELETE`. That hand-rolled machinery is exactly what SQL:2011 replaces — the standard's use cases are "Auditing all data changes and performing data forensics," "Reconstructing state of the data as of any time in the past," and "Maintaining a slowly changing dimension" ([Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

```sql
-- WRONG — hand-rolled history: a trigger that fires on every UPDATE/DELETE.
-- Drifts from the base schema, silently misses paths that bypass it (bulk loads,
-- other triggers, direct history writes), and there is no query syntax for "as of".
CREATE TABLE employee_history (LIKE employee, archived_at TIMESTAMP);

CREATE TRIGGER employee_audit BEFORE UPDATE OR DELETE ON employee
FOR EACH ROW
  INSERT INTO employee_history
  SELECT OLD.*, now();   -- one forgotten column / code path and history is wrong forever
```

```sql
-- RIGHT — system-versioned table: the engine records history automatically.
CREATE TABLE employee (
    employee_id   INT PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    salary        DECIMAL(10,2) NOT NULL,
    valid_from    TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
    valid_to      TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
) WITH SYSTEM VERSIONING;   -- MariaDB spelling; SQL Server: WITH (SYSTEM_VERSIONING = ON)
```

The DDL shape is from [MariaDB — System-Versioned Tables](https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables); the engine "store[s] the previous version of the row each time a row ... gets updated or deleted" ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

---

## 2. System-Versioned Tables & `FOR SYSTEM_TIME` (the centerpiece)

A system-versioned (= **transaction-time**) table has two `GENERATED ALWAYS AS ROW START/END` period columns and a `PERIOD FOR SYSTEM_TIME(start, end)` declaration; the modifier `WITH SYSTEM VERSIONING` turns on automatic history ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)). The engine populates the period columns from the **transaction timestamp** — on insert it sets the start and an open end of `9999-12-31`; on update/delete it closes the old row into history ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)). You never write the timestamps.

A plain `SELECT` returns only *current* rows. To see the past, add a `FOR SYSTEM_TIME` clause — "a new clause ... with five temporal-specific subclauses to query data across the current and history tables" ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)):

```sql
-- The table AS IT WAS at one instant (point-in-time reconstruction)
SELECT * FROM employee FOR SYSTEM_TIME AS OF TIMESTAMP '2026-03-01 00:00:00';

-- Every version overlapping a window (half-open: FROM inclusive, TO exclusive)
SELECT * FROM employee
  FOR SYSTEM_TIME FROM '2026-01-01' TO '2026-04-01';

-- Like FROM..TO but the upper bound is INCLUSIVE
SELECT * FROM employee
  FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-04-01';

-- Current + all history
SELECT * FROM employee FOR SYSTEM_TIME ALL;
```

The boundary semantics are exact and easy to get wrong. `AS OF t` returns rows where `valid_from <= t AND valid_to > t`; `FROM s TO e` returns `valid_from < e AND valid_to > s` (both endpoints exclusive at their own edge); `BETWEEN s AND e` is the same but **includes** rows that became active exactly at `e` (`valid_from <= e AND valid_to > s`) ([Microsoft Learn — qualifying-rows table](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)). These are **half-open** `[start, end)` periods — depth on interval boundary math lives in `sql-datetime-and-intervals`.

```sql
-- WRONG — querying the history table by hand and guessing the overlap predicate
SELECT * FROM employee_history WHERE archived_at <= '2026-03-01';  -- off-by-boundary, misses current row

-- RIGHT — let the clause compute the overlap across current + history
SELECT * FROM employee FOR SYSTEM_TIME AS OF TIMESTAMP '2026-03-01 00:00:00';
```

---

## 3. Application-Time Period Tables & `FOR PORTION OF` (valid-time)

System time answers "when did the *database* believe this." **Application time** (= valid-time) answers "when is this fact *true in the real world*" — a contract valid Jan–Jun, a price effective next Monday — and the **application** supplies those dates. You declare a named period (any name, **not** `SYSTEM_TIME`) over two app-managed columns: "Definition of application time period tables (elsewhere called valid time tables), using the `PERIOD FOR` annotation" ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)).

```sql
CREATE TABLE price (
    sku        VARCHAR(20),
    amount     DECIMAL(10,2) NOT NULL,
    valid_from DATE NOT NULL,
    valid_to   DATE NOT NULL,
    PERIOD FOR business_time (valid_from, valid_to),
    -- temporal PK: one price per sku per instant, periods may not overlap
    PRIMARY KEY (sku, business_time WITHOUT OVERLAPS)
);
```

`WITHOUT OVERLAPS` is the standard's "non-overlapping constraints" on a temporal primary key ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)). The payoff clause is `FOR PORTION OF`: an `UPDATE`/`DELETE ... FOR PORTION OF <period> FROM s TO e` changes the row **only within that sub-interval** and the engine **splits** the period automatically — "Update and deletion of application time rows with automatic time period splitting" ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)).

```sql
-- WRONG — clobbers the whole row's validity; loses the before/after segments
UPDATE price SET amount = 9.99 WHERE sku = 'A1';

-- RIGHT — change only Mar–Apr; the engine splits the period into 3 rows
UPDATE price FOR PORTION OF business_time FROM '2026-03-01' TO '2026-04-01'
  SET amount = 9.99 WHERE sku = 'A1';
```

Valid-time periods are queried with ordinary predicates or the SQL:2011 period predicates "`CONTAINS`, `OVERLAPS`, `EQUALS`, `PRECEDES`, `SUCCEEDS`, `IMMEDIATELY PRECEDES` and `IMMEDIATELY SUCCEEDS`" ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)).

---

## 4. System-Time vs Valid-Time vs Bitemporal

Two independent time axes, three table designs. Picking the wrong axis is the deepest temporal modeling error:

| Mode | Period | Who sets the timestamps | Answers |
|---|---|---|---|
| **System-versioned** (transaction time) | `PERIOD FOR SYSTEM_TIME` + `WITH SYSTEM VERSIONING` | the **engine**, from the transaction clock | "what did the database *record*, and when" — audit, forensics, as-of |
| **Application-time** (valid time) | `PERIOD FOR <name>` (app columns) | the **application** | "when is/was this fact *true in the world*" — effective dates, history of reality |
| **Bitemporal** | both, on one table | engine *and* application | "what did we *believe on date X* about what was *true on date Y*" |

"Application time and system versioning can be used together to provide bitemporal tables" ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)). Use system time for *audit/correction tracking*; use application/valid time for *real-world effectivity*; use bitemporal only when you genuinely must reconstruct *past beliefs about the past* (regulatory restatements, retroactive corrections). Most needs are one axis, not both.

---

## 5. Portability — LOW (read this before recommending)

**This is a low-portability feature. Confirm the target engine supports it — and which spelling — before you recommend it.** Three engines that *do* support system-versioning still disagree on the DDL, and two major open-source engines have no native support at all.

| Engine | System-versioned? | Spelling / notes |
|---|---|---|
| **MariaDB** 10.3+ | ✓ | `WITH SYSTEM VERSIONING`; `ROW_START`/`ROW_END` pseudo-cols; `DELETE HISTORY` to prune ([MariaDB](https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables)) |
| **SQL Server** 2016+ | ✓ | `WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = ...))`; adds `CONTAINED IN`; `HIDDEN` period cols ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)) |
| **IBM DB2** v10, **SAP HANA** 2.0 | ✓ | "Strong compliance" with the standard ([Wikipedia](https://en.wikipedia.org/wiki/SQL:2011)) |
| **Oracle** | ~ | Approximates via **Flashback Query** (`AS OF TIMESTAMP/SCN`), not the standard DDL ([Wikipedia](https://en.wikipedia.org/wiki/SQL:2011)) |
| **PostgreSQL** | ✗ | No native system-versioning; "requires extension" (e.g. `temporal_tables`) or manual modeling ([Wikipedia](https://en.wikipedia.org/wiki/SQL:2011)) |
| **MySQL** | ✗ | No temporal tables — manual history table + triggers, or app-level versioning |
| **SQLite** | ✗ | No temporal tables — manual modeling only |

Application-time / `FOR PORTION OF` / `WITHOUT OVERLAPS` support is even thinner and lags system-versioning even on engines that have it. The full support-and-spelling matrix is owned by **`sql-standard-vs-dialect-map`** — route there for "does engine X have this, and how does it spell it." When the target engine is on the ✗ row, the correct answer is a hand-modeled history table (route to `sql-schema-design-and-normalization`), *not* temporal-table syntax that will not parse.

---

## 6. The MVCC Boundary (hard)

A live MVCC engine already keeps old row versions to serve concurrent snapshots — and that machinery is what *enables* time-travel. This skill and `mvcc-time-travel-queries` (in the sibling mvcc-skills-plugin) touch the same `FOR SYSTEM_TIME` keyword from opposite sides:

- **This skill owns the *language*:** the standard SQL:2011 DDL (`PERIOD FOR`, `WITH SYSTEM VERSIONING`, `GENERATED ALWAYS AS ROW START/END`, `WITHOUT OVERLAPS`) and the query clauses (`FOR SYSTEM_TIME AS OF / FROM..TO / BETWEEN / ALL`, `FOR PORTION OF`) — what to *write*.
- **`mvcc-time-travel-queries` owns the *mechanics*:** how versions are physically stored and chained, how a snapshot is computed, and the **retention/GC horizon** — the rule that you can only travel back as far as you still retain versions (too short → "snapshot too old" / empty reads; too long → storage bloat). Pruning (`DELETE HISTORY`, retention policies) and "how far back can I go" are its territory.

Rule of thumb: *what clause do I write* → here. *Why did my as-of query error / how long is history kept / how is it stored* → `mvcc-time-travel-queries`.

---

## 7. Who Suffers When This Is Done Wrong

- The **auditor / compliance officer** handed a `_history` table built from triggers that silently missed a code path — a bulk `UPDATE` that bypassed the trigger, or a second trigger that fired first. The audit trail has holes, and nobody knew until the regulator asked. System-versioning records *every* modification by construction ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).
- The **analyst** asked "what did this customer's record look like last March" against a table with no history at all — an unanswerable question, because the old values were overwritten in place and are simply gone. Without versioning there is nothing to query.
- The **engineer** who hand-wrote the overlap predicate against a history table and got the half-open boundary wrong (`<=` where it should be `<`), double-counting rows at period edges — the exact arithmetic `FOR SYSTEM_TIME` computes correctly (§2).
- The **developer** who shipped `WITH SYSTEM VERSIONING` to a PostgreSQL or MySQL target and got a syntax error in production, because the feature is MariaDB/SQL Server/DB2-only (§5).
- The **data steward** who used system-time to model real-world effective dates, then discovered the timestamps are transaction-clock values they cannot backdate — they needed *application* time, not system time (§4).

---

## 8. Routing to Related Skills

- `sql-relational-and-null-discipline` — the policy root: set semantics, NULL, three-valued logic that every query here assumes.
- `sql-datetime-and-intervals` — timestamp types and the half-open `[start, end)` interval math underlying every period and `FOR SYSTEM_TIME` boundary (§2).
- `sql-schema-design-and-normalization` — when the engine lacks temporal tables (§5), the hand-modeled history-table / slowly-changing-dimension design.
- `sql-standard-vs-dialect-map` — the full support-and-spelling matrix: which engine has system/application time and how it spells the DDL (§5).
- `mvcc-time-travel-queries` (mvcc-skills-plugin) — **hard boundary**: version storage, snapshot semantics, and the retention/GC horizon behind as-of reads (§6).

---

## 9. Reference Files

High-frequency temporal-table anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
