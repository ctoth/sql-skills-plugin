# Research: sql-temporal-tables (SQL:2011)

Research backing the `sql-temporal-tables` skill. Each entry: URL, section, verbatim
quote, and why it matters for the skill. Accessed 2026-06-26.

Boundary note (HARD): this skill owns the **standard DDL/query syntax** of SQL:2011
temporal tables. Versioning/storage mechanics, snapshot semantics, retention/GC horizon,
and MVCC time-travel internals belong to the sibling `mvcc-time-travel-queries`
(mvcc-skills-plugin). Confirmed that sibling skill exists at
`C:\Users\Q\code\mvcc-skills-plugin\plugins\mvcc\skills\mvcc-time-travel-queries\SKILL.md`.

---

## Source 1 — Wikipedia: SQL:2011

URL: https://en.wikipedia.org/wiki/SQL:2011

### System-versioned tables (= transaction-time)
> "Definition of system-versioned tables (elsewhere called transaction time tables),
> using the PERIOD FOR SYSTEM_TIME annotation"
> "WITH SYSTEM VERSIONING modifier. System time periods are maintained automatically."

Why it matters: This is the centerpiece. The standard added a declarative way to make the
engine auto-record history — `PERIOD FOR SYSTEM_TIME` + `WITH SYSTEM VERSIONING` — so you
do NOT hand-roll trigger-based history tables. "Maintained automatically" is the entire
selling point of the WRONG/RIGHT framing.

### Querying system-versioned tables
> Syntax includes: "AS OF SYSTEM TIME" and "VERSIONS BETWEEN SYSTEM TIME ... AND ..." clauses

Why it matters: Confirms FOR SYSTEM_TIME query clauses are standardized (engines spell the
AS OF/BETWEEN variants slightly differently — see MariaDB/SQL Server below).

### Application-time period tables (= valid-time)
> "Definition of application time period tables (elsewhere called valid time tables),
> using the PERIOD FOR annotation"
> "Update and deletion of application time rows with automatic time period splitting"
> "Temporal primary keys incorporating application time periods with optional
> non-overlapping constraints via the WITHOUT OVERLAPS clause"

Why it matters: The second temporal flavor. `PERIOD FOR <name>` (not SYSTEM_TIME) models
*application/valid* time — when a fact is true in the real world, controlled by the
application, not the system clock. "Automatic time period splitting" is the FOR PORTION OF
behavior. WITHOUT OVERLAPS is the temporal-PK constraint.

### Temporal predicates
> "Application time tables are queried using regular query syntax or using new temporal
> predicates for time periods including CONTAINS, OVERLAPS, EQUALS, PRECEDES, SUCCEEDS,
> IMMEDIATELY PRECEDES and IMMEDIATELY SUCCEEDS"

Why it matters: Period-comparison vocabulary for valid-time queries.

### Bitemporal
> "Application time and system versioning can be used together to provide bitemporal tables"

Why it matters: Defines the third mode — bitemporal = both axes at once. Crisp basis for
the system-time vs valid-time vs bitemporal section.

### Vendor support (portability — LOW)
> Strong compliance: IBM DB2 v10, Microsoft SQL Server 2016, MariaDB 10.3+, SAP HANA 2.0
> Partial/Alternative syntax: Oracle (Flashback Queries), PostgreSQL (requires extension)
> Supported: CockroachDB (AS OF SYSTEM TIME since v1.0.7)

Why it matters: Direct evidence for the prominent low-portability block — DB2/SQL
Server/MariaDB/SAP HANA have the standard feature; Oracle approximates via Flashback;
PostgreSQL needs an extension; MySQL/SQLite are absent.

---

## Source 2 — modern-sql.com: SQL standard versions

URL: https://modern-sql.com/standard  (note: /standard/2011 returns HTTP 404)

> "ISO released a free technical report on 'SQL Support for Time-Related Information'
> (covers temporal tables)."

Why it matters: Confirms SQL:2011 is the standard edition that introduced temporal/
time-related table support, and that ISO published a dedicated technical report on it.
(The page does not enumerate engine support; portability evidence comes from Source 1 and
the vendor docs in Sources 3-4.)

---

## Source 3 — MariaDB: System-Versioned Tables

URL: https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables

### Minimal DDL
> ```sql
> CREATE TABLE t (x INT) WITH SYSTEM VERSIONING;
> ```

### Explicit period columns
> ```sql
> CREATE TABLE t(
>    x INT,
>    start_timestamp TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
>    end_timestamp   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
>    PERIOD FOR SYSTEM_TIME(start_timestamp, end_timestamp)
> ) WITH SYSTEM VERSIONING;
> ```

Why it matters: The concrete standard-shaped DDL — `GENERATED ALWAYS AS ROW START/END`
period columns + `PERIOD FOR SYSTEM_TIME(...)` + `WITH SYSTEM VERSIONING`. Engine fills the
period columns automatically. This is the RIGHT example.

### Query clauses
> AS OF: `SELECT * FROM t FOR SYSTEM_TIME AS OF TIMESTAMP'2016-10-09 08:07:06';`
> BETWEEN (inclusive both ends): `... FOR SYSTEM_TIME BETWEEN (NOW() - INTERVAL 1 YEAR) AND NOW();`
> FROM..TO (start inclusive, end exclusive): `... FOR SYSTEM_TIME FROM '2016-01-01 00:00:00' TO '2017-01-01 00:00:00';`
> ALL: `SELECT * FROM t FOR SYSTEM_TIME ALL;`

Why it matters: All four FOR SYSTEM_TIME variants with their boundary semantics. ALL =
current + history. Note MariaDB ROW_START/ROW_END pseudo-columns and `DELETE HISTORY FROM t`
for pruning.

### History pruning
> `DELETE HISTORY FROM t BEFORE SYSTEM_TIME '2016-10-09 08:07:06';`

Why it matters: Shows the engine owns history storage; pruning/retention sizing is the MVCC
sibling's territory (boundary).

---

## Source 4 — Microsoft Learn: Temporal Tables (SQL Server)

URL: https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables

### Definition / problem solved
> "Temporal tables ... are a database feature that brings built-in support for providing
> information about data stored in the table at any point in time, rather than only the
> data that is correct at the current moment in time."
> "A system-versioned temporal table is a type of user table designed to keep a full
> history of data changes, allowing easy point-in-time analysis ... because the system
> manages the period of validity for each row."

### Use cases ("Why temporal?")
> "Auditing all data changes and performing data forensics when necessary"
> "Reconstructing state of the data as of any time in the past"
> "Maintaining a slowly changing dimension for decision support applications"
> "Recovering from accidental data changes and application errors"

Why it matters: Direct support for the WHY framing and the "Who suffers" section — audit,
forensics, as-of, slowly-changing-dimension.

### Mechanics: paired table + period columns
> "System-versioning for a table is implemented as a pair of tables: a current table, and a
> history table ... two extra datetime2 columns are used to define the period of validity
> for each row."
> "The system uses the history table to automatically store the previous version of the row
> each time a row in the temporal table gets updated or deleted."

### DDL
> ```sql
> CREATE TABLE dbo.Employee (
>     [EmployeeID] INT NOT NULL PRIMARY KEY CLUSTERED,
>     ... ,
>     [ValidFrom] DATETIME2 GENERATED ALWAYS AS ROW START,
>     [ValidTo]   DATETIME2 GENERATED ALWAYS AS ROW END,
>     PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
> ) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.EmployeeHistory));
> ```

Why it matters: SQL Server's spelling — `SYSTEM_VERSIONING = ON` and an explicit
`HISTORY_TABLE`, vs MariaDB's `WITH SYSTEM VERSIONING`. Same standard concept, divergent
syntax = portability story.

### FOR SYSTEM_TIME subclauses + boundary semantics (verbatim table)
> "AS OF date_time" → `ValidFrom <= date_time AND ValidTo > date_time` — "values that were
> current at the specified point in time in the past."
> "FROM start TO end" → `ValidFrom < end AND ValidTo > start` — start excluded on lower
> boundary, end excluded on upper boundary.
> "BETWEEN start AND end" → `ValidFrom <= end AND ValidTo > start` — "same as FROM..TO except
> ... includes rows that became active on the upper boundary."
> "CONTAINED IN (start, end)" → `ValidFrom >= start AND ValidTo <= end` — rows opened AND
> closed within the range.
> "ALL" → "the union of rows that belong to the current and the history table."

Why it matters: Exact qualifying-row predicates for every clause — the precise boundary
behavior (half-open intervals) that trips people up. CONTAINED IN is SQL-Server-specific
(not in MariaDB's set), a portability data point.

### Insert/update/delete behavior
> "Inserts: the system sets ValidFrom to the begin time of the current transaction (UTC) ...
> and assigns ValidTo the maximum value of 9999-12-31. This marks the row as open."
> "Updates: the system stores the previous value of the row in the history table and sets
> ValidTo to the begin time of the current transaction ..."
> "the times recorded ... are based on the begin time of the transaction itself."

Why it matters: Confirms system-time is transaction/clock-driven and automatic — the
contrast with application/valid-time (app-controlled). The transaction-time anchoring is
where this skill defers to mvcc-time-travel-queries for the underlying version mechanics.

### Hidden period columns
> "You can choose to hide the period columns, such that queries that don't explicitly
> reference them don't return these columns ... INSERT and BULK INSERT statements continue
> as if these new period columns weren't present (and the column values are auto-populated)."

Why it matters: Practical note — `HIDDEN` keeps `SELECT *` and inserts clean.

---

## Synthesis — the five claims the skill must nail

1. **System-versioned tables auto-record history.** `PERIOD FOR SYSTEM_TIME(start,end)` +
   `WITH SYSTEM VERSIONING` (MariaDB) / `SYSTEM_VERSIONING = ON` (SQL Server). Query the
   past with `FOR SYSTEM_TIME AS OF / FROM..TO / BETWEEN / ALL`. Standard replacement for
   hand-rolled trigger history. [Src 1, 3, 4]

2. **Application-time period tables = valid-time.** `PERIOD FOR <name>` (any name, not
   SYSTEM_TIME); `UPDATE/DELETE ... FOR PORTION OF` splits periods automatically; `WITHOUT
   OVERLAPS` temporal PK. App controls the timestamps. [Src 1]

3. **Three modes.** system-time (transaction time, system-controlled) vs valid-time
   (application time, app-controlled) vs bitemporal (both axes together). [Src 1, 4]

4. **LOW PORTABILITY.** DB2 / SQL Server 2016+ / MariaDB 10.3+ / SAP HANA / Oracle (12c, +
   Flashback) support it; PostgreSQL needs an extension; MySQL and SQLite have no native
   support → manual modeling. Syntax diverges even among supporters (`WITH SYSTEM
   VERSIONING` vs `SYSTEM_VERSIONING = ON`; CONTAINED IN only on SQL Server). [Src 1, 4]
   Confirm engine support before recommending.

5. **HARD MVCC boundary.** This skill = standard DDL/query syntax. Versioning/storage
   mechanics, snapshot semantics, retention/GC horizon ("how far back can I travel"), and
   MVCC time-travel internals → `mvcc-time-travel-queries`. Engine support matrix →
   `sql-standard-vs-dialect-map`. Hand-rolled history without the standard feature →
   `sql-schema-design-and-normalization`.
