# Common SQL Temporal-Table Mistakes

Anti-patterns in LLM-generated SQL around SQL:2011 temporal tables, each with wrong/right
code and a primary-source citation. The skill (`sql-temporal-tables`) states the rules; this
file holds the high-frequency failure modes. Temporal tables are a LOW-PORTABILITY feature —
every RIGHT example notes which engines accept it and routes spellings to
`sql-standard-vs-dialect-map`.

---

## 1. Hand-rolling trigger history instead of system-versioning

**The problem:** Asked to "keep a history of every change," the model writes a `_history`
shadow table plus an `UPDATE`/`DELETE` trigger. It drifts from the base schema, silently
misses any path that bypasses it, and offers no "as of" query syntax. SQL:2011's use cases
are exactly "Auditing all data changes" and "Reconstructing state of the data as of any time
in the past" ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

```sql
-- WRONG — fragile trigger history; one missed code path corrupts the audit trail
CREATE TABLE employee_history (LIKE employee, archived_at TIMESTAMP);
CREATE TRIGGER employee_audit BEFORE UPDATE OR DELETE ON employee
FOR EACH ROW INSERT INTO employee_history SELECT OLD.*, now();

-- RIGHT — the engine records history automatically (MariaDB spelling)
CREATE TABLE employee (
    employee_id INT PRIMARY KEY,
    salary      DECIMAL(10,2) NOT NULL,
    valid_from  TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
    valid_to    TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
) WITH SYSTEM VERSIONING;
```

*Source: [Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables); [MariaDB — System-Versioned Tables](https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables). Depth: this skill, §1-2. Engine without support → `sql-schema-design-and-normalization`.*

---

## 2. Querying the history table by hand instead of `FOR SYSTEM_TIME AS OF`

**The problem:** The model reconstructs a past state by `SELECT`ing the history table with a
hand-written timestamp predicate. It usually gets the overlap wrong and forgets that the
*current* row (still open) also overlaps the instant. `FOR SYSTEM_TIME AS OF` unions current
+ history and computes `valid_from <= t AND valid_to > t` correctly
([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

```sql
-- WRONG — misses the still-current row; boundary predicate is a guess
SELECT * FROM employee_history WHERE archived_at <= '2026-03-01';

-- RIGHT — the engine computes the overlap across current + history
SELECT * FROM employee FOR SYSTEM_TIME AS OF TIMESTAMP '2026-03-01 00:00:00';
```

*Source: [Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables). Depth: this skill, §2.*

---

## 3. Confusing `FROM..TO` (end-exclusive) with `BETWEEN` (end-inclusive)

**The problem:** The model treats the two window clauses as synonyms. They differ only at
the upper boundary: `FROM s TO e` is `valid_from < e AND valid_to > s` (excludes rows that
became active exactly at `e`); `BETWEEN s AND e` is `valid_from <= e AND valid_to > s`
(includes them) ([Microsoft Learn — qualifying-rows table](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

```sql
-- These are NOT the same — BETWEEN includes versions that started exactly at the upper bound
SELECT * FROM employee FOR SYSTEM_TIME FROM '2026-01-01' TO '2026-04-01';   -- end-exclusive
SELECT * FROM employee FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-04-01'; -- end-inclusive
```

*Source: [Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables). Depth: this skill, §2; interval math → `sql-datetime-and-intervals`.*

---

## 4. Writing the period columns by hand

**The problem:** The model `INSERT`s or `UPDATE`s `valid_from`/`valid_to` directly. Those
columns are `GENERATED ALWAYS AS ROW START/END`; the engine fills them from the transaction
clock and rejects manual writes — on insert it sets start = transaction time and end =
`9999-12-31` automatically ([Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

```sql
-- WRONG — period columns are system-generated; this errors or is ignored
INSERT INTO employee (employee_id, salary, valid_from, valid_to)
VALUES (1, 50000, '2026-01-01', '9999-12-31');

-- RIGHT — insert the data columns only; the system populates the period
INSERT INTO employee (employee_id, salary) VALUES (1, 50000);
```

*Source: [Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables). Depth: this skill, §2.*

---

## 5. Using system-time when the need is real-world effective dates (valid-time)

**The problem:** Asked to model effective/expiry dates the *application* controls (a price
effective next Monday, a contract valid Jan–Jun), the model reaches for system-versioning.
But system-time timestamps come from the transaction clock and cannot be backdated — that is
*transaction* time, not *valid* time. Use an application-time `PERIOD FOR <name>` over
app-managed columns ([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)).

```sql
-- WRONG — system-versioning records WHEN THE DB CHANGED, not when the price is effective
... PERIOD FOR SYSTEM_TIME (valid_from, valid_to) WITH SYSTEM VERSIONING;

-- RIGHT — application time: the app sets the dates; periods may not overlap
CREATE TABLE price (
    sku VARCHAR(20), amount DECIMAL(10,2) NOT NULL,
    valid_from DATE NOT NULL, valid_to DATE NOT NULL,
    PERIOD FOR business_time (valid_from, valid_to),
    PRIMARY KEY (sku, business_time WITHOUT OVERLAPS)
);
```

*Source: [Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011). Depth: this skill, §3-4.*

---

## 6. Overwriting a whole valid-time row instead of `FOR PORTION OF`

**The problem:** To change a price for one sub-interval, the model issues a plain `UPDATE`,
clobbering the row's entire validity and losing the before/after segments. SQL:2011 provides
`UPDATE/DELETE ... FOR PORTION OF` with "automatic time period splitting"
([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)).

```sql
-- WRONG — destroys the period structure; the Jan-Feb and Apr+ history is lost
UPDATE price SET amount = 9.99 WHERE sku = 'A1';

-- RIGHT — change only the sub-interval; the engine splits the period
UPDATE price FOR PORTION OF business_time FROM '2026-03-01' TO '2026-04-01'
  SET amount = 9.99 WHERE sku = 'A1';
```

*Source: [Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011). Depth: this skill, §3.*

---

## 7. Recommending temporal syntax on an engine that lacks it

**The problem:** The model emits `WITH SYSTEM VERSIONING` / `FOR SYSTEM_TIME` for a
PostgreSQL, MySQL, or SQLite target — none of which have native temporal tables — producing
a production syntax error. Native support is MariaDB 10.3+, SQL Server 2016+, DB2, SAP HANA;
PostgreSQL "requires extension" and MySQL/SQLite need manual modeling
([Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)).

```sql
-- WRONG on PostgreSQL/MySQL/SQLite — not valid syntax there
CREATE TABLE t (...) WITH SYSTEM VERSIONING;

-- RIGHT (unsupported engine) — hand-model a history table; see schema-design skill
CREATE TABLE t_history (id INT, ..., valid_from TIMESTAMP, valid_to TIMESTAMP);
-- ...maintained by app logic / triggers, queried with explicit overlap predicates
```

*Source: [Wikipedia — SQL:2011](https://en.wikipedia.org/wiki/SQL:2011). Depth: this skill, §5; matrix → `sql-standard-vs-dialect-map`; hand-modeling → `sql-schema-design-and-normalization`.*

---

## 8. Assuming one DDL spelling is portable across supporters

**The problem:** The model copies MariaDB's `WITH SYSTEM VERSIONING` to SQL Server (which
wants `WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = ...))`), or uses SQL Server's
`CONTAINED IN` clause on MariaDB (which lacks it). Even engines that support the standard
diverge in syntax ([MariaDB](https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables); [Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)).

```sql
-- MariaDB
CREATE TABLE t (...) WITH SYSTEM VERSIONING;
-- SQL Server (explicit history table; CONTAINED IN clause available)
CREATE TABLE t (...) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.t_History));
SELECT * FROM t FOR SYSTEM_TIME CONTAINED IN ('2026-01-01', '2026-04-01');  -- SQL Server only
```

*Source: [MariaDB — System-Versioned Tables](https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables); [Microsoft Learn — Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables). Depth: this skill, §5; spellings → `sql-standard-vs-dialect-map`.*

---

## 9. Conflating temporal-table syntax with MVCC retention / "how far back"

**The problem:** The model treats `FOR SYSTEM_TIME AS OF <old timestamp>` as if history is
infinite, or tries to answer "how far back can I query" / "why did my as-of query error"
inside temporal-table syntax. The query *clause* is owned here; the *retention horizon*
(how long versions are kept before GC) and snapshot mechanics belong to
`mvcc-time-travel-queries`. An `AS OF` past the retention window errors or returns nothing.

```sql
-- The clause is correct syntax...
SELECT * FROM employee FOR SYSTEM_TIME AS OF TIMESTAMP '2010-01-01 00:00:00';
-- ...but whether 2010 data still EXISTS is a retention/GC question, not a syntax one.
```

*Source: boundary with `mvcc-time-travel-queries`. Depth: this skill, §6. Retention/GC, storage, snapshots → `mvcc-time-travel-queries`.*
