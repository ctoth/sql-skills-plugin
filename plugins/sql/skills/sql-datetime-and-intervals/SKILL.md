---
name: sql-datetime-and-intervals
description: >-
  Guides standard temporal handling in SQL — the temporal types (DATE, TIME, TIMESTAMP, and crucially TIMESTAMP WITH TIME ZONE), typed literals (DATE '...', TIMESTAMP '...', INTERVAL '...'), EXTRACT, the standard CURRENT_DATE/CURRENT_TIMESTAMP keywords, and INTERVAL arithmetic — instead of jamming dates into strings or reaching for vendor functions (NOW()/GETDATE()/SYSDATE, DATEADD/DATEDIFF, STRFTIME). Auto-invokes when writing or editing date/time columns or literals, EXTRACT, INTERVAL arithmetic, time-zone-sensitive comparisons, date truncation/bucketing, or age/duration calculations, and on "store a timestamp", "add a day/month", "why is my time off by an hour", or "yesterday's rows" requests.
allowed-tools: Read, Glob, Grep
---

# SQL Datetime and Intervals

> "In either case, the value is stored internally as UTC, and the originally stated or assumed time zone is not retained. ... All timezone-aware dates and times are stored internally in UTC."
> — [PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)

> "MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage, and back from UTC to the current time zone for retrieval. (This does not occur for other types such as `DATETIME`.)"
> — [MySQL — The DATE, DATETIME, and TIMESTAMP Types](https://dev.mysql.com/doc/refman/8.0/en/datetime.html)

A point in time and a set of wall-clock fields are *different things*, and almost every datetime bug comes from confusing them. An instant ("when did this happen") is an absolute position on the UTC timeline; a naive timestamp ("2024-03-10 02:30") is just digits with no zone, and the same digits name different instants in different places — and a different instant before and after a DST jump. This skill builds on the set semantics and three-valued logic of the foundation skill **`sql-relational-and-null-discipline`** (a NULL timestamp still propagates UNKNOWN through every comparison); here the subject is choosing the right temporal type and doing date math without strings or vendor functions.

---

## 1. TIMESTAMP vs TIMESTAMP WITH TIME ZONE — Store Instants as UTC (the centerpiece)

This is the single highest-frequency datetime bug. SQL has two timestamp types and they are not interchangeable:

- **`TIMESTAMP WITH TIME ZONE`** records an *instant*. On input, "an input string that includes an explicit time zone will be converted to UTC ... In either case, the value is stored internally as UTC, and the originally stated or assumed time zone is not retained" ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)). On output it "is always converted from UTC to the current `timezone` zone, and displayed as local time in that zone" (same source). The name is misleading: it does **not** store a zone — it stores a UTC point in time and renders in the session zone.
- **`TIMESTAMP` (without time zone)** is a *naive* wall-clock value with no instant. "PostgreSQL will silently ignore any time zone indication. That is, the resulting value is derived from the date/time fields in the input string, and is not adjusted for time zone" ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)).

Store an event in a bare `TIMESTAMP` and you have thrown away the information needed to know what UTC instant it refers to — so it shifts under DST and across server zones.

```sql
-- WRONG — a naive local timestamp for an event. Which instant is this?
-- Nobody knows: it depends on whatever zone the writer/reader assumed.
-- After a DST change the same column renders an hour off.
CREATE TABLE meetings (id bigint, starts_at TIMESTAMP);          -- no zone
INSERT INTO meetings VALUES (1, TIMESTAMP '2024-03-10 02:30:00'); -- ambiguous

-- RIGHT — record the instant. The value is normalized to a UTC point in time
-- and rendered in the reader's session zone, so it is the SAME instant everywhere.
CREATE TABLE meetings (id bigint, starts_at TIMESTAMP WITH TIME ZONE);
INSERT INTO meetings VALUES (1, TIMESTAMP WITH TIME ZONE '2024-03-10 02:30:00-05');
```

PostgreSQL's own guidance: "we recommend using date/time types that contain both date and time when using time zones. We do not recommend using the type `time with time zone`" ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)). Use `TIMESTAMP WITH TIME ZONE` for instants; reserve bare `TIMESTAMP` and `DATE`/`TIME` for genuinely zone-free facts (a birthday, a store's "opens 09:00" local rule).

---

## 2. The Standard Temporal Types

SQL defines a small set of temporal types; learn what each does and does not carry ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)):

- **`DATE`** — date only, day resolution, no time and no zone (e.g. `4713 BC .. 5874897 AD` in PostgreSQL).
- **`TIME [WITHOUT TIME ZONE]`** — time of day only, no date.
- **`TIMESTAMP [WITHOUT TIME ZONE]`** — date + time, naive (no zone), microsecond resolution.
- **`TIMESTAMP WITH TIME ZONE`** — date + time as a UTC instant (§1).
- **`INTERVAL`** — a *duration* (e.g. `-178000000 years .. 178000000 years`), not a point in time. Used for arithmetic (§5).
- `TIME WITH TIME ZONE` exists for standard compliance but is discouraged — a time without a date cannot resolve a DST offset.

A `DATE` has no time component and a `TIME` has no date; don't store an instant in either.

---

## 3. Typed Literals — Not Strings Parsed by Chance

Write temporal constants with the standard typed-literal syntax `type 'value'`, not bare strings you hope the engine parses the way you meant. The standard form is "`type [ (p) ] 'value'` where p is an optional precision specification giving the number of fractional digits in the seconds field" ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)).

```sql
-- WRONG — a bare string; parsing/format is implicit and locale/engine-dependent
SELECT * FROM events WHERE day = '01/02/2024';   -- Jan 2 or Feb 1? who decides?

-- RIGHT — typed, unambiguous literals (ISO 8601 values)
SELECT * FROM events WHERE day = DATE '2024-01-02';
SELECT TIMESTAMP WITH TIME ZONE '2004-10-19 10:23:54+02';
SELECT INTERVAL '1' YEAR, INTERVAL '1 day 2:03:04';
```

A typed literal pins both the type and the interpretation; a bare string defers both to whatever the dialect's input parser happens to do.

---

## 4. EXTRACT and CURRENT_* — Standard Spellings vs Vendor Functions

**Pulling fields out:** `EXTRACT(field FROM source)` "retrieves subfields such as year or hour from date/time values," returning numeric, where source is "a value expression of type timestamp, date, time, or interval" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)). It is the portable alternative to vendor `DATEPART`/`strftime` or string slicing.

```sql
-- WRONG — string slicing assumes a fixed text format and ignores the real type
SELECT SUBSTRING(CAST(created_at AS text), 1, 4) AS yr FROM orders;

-- RIGHT — standard field extraction
SELECT EXTRACT(YEAR FROM created_at) AS yr,
       EXTRACT(HOUR FROM created_at) AS hr
FROM orders;
```

**Current date/time:** the SQL-standard spellings are *keywords*, not functions — `CURRENT_DATE`, `CURRENT_TIMESTAMP`, `LOCALTIMESTAMP` (no parentheses). "These SQL-standard functions all return values based on the start time of the current transaction" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)). The common vendor spellings are non-standard: PostgreSQL's own `now()` is "a traditional PostgreSQL equivalent to `transaction_timestamp()`" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)), and `GETDATE()`/`SYSDATE`/`SYSDATETIME()` are other engines'.

```sql
-- WRONG — vendor spelling; not portable
SELECT * FROM sessions WHERE started_at > NOW() - INTERVAL '1 day';   -- now() is PG-specific

-- RIGHT — standard keyword
SELECT * FROM sessions WHERE started_at > CURRENT_TIMESTAMP - INTERVAL '1 day';
```

Gotcha: these return the *transaction* start time — "their values do not change during the transaction" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)), not a fresh wall-clock reading per statement. Route `NOW()`/`GETDATE()`/`SYSDATE`/`DATEPART` spellings to **`sql-standard-vs-dialect-map`**.

---

## 5. INTERVAL Arithmetic — and the Month-Length Ambiguity

Do date math by adding an `INTERVAL` to a date/timestamp, not by adding integers (which mean nothing for months) or by string surgery. "When adding an interval value to (or subtracting an interval value from) a timestamp or timestamp with time zone value, the months, days, and microseconds fields of the interval value are handled in turn" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)).

```sql
-- WRONG — integer "+ 7" is non-portable and meaningless for months
SELECT due_date + 7 FROM invoices;          -- works in some engines, breaks in others

-- RIGHT — explicit, readable, standard interval arithmetic
SELECT due_date + INTERVAL '7 days'  FROM invoices;
SELECT ts       + INTERVAL '1 month' FROM events;
```

That "months, days, and microseconds handled in turn" rule is the seed of a real trap: **a month is not a fixed number of days.** `+ INTERVAL '1 month'` is calendar-aware — adding one month to Jan 31 lands on a Feb date, not "Jan 31 + 30 days." Mixing units, leap years, and DST mean you cannot reduce month arithmetic to day counting. When you need exact day counts, use a day interval; when you mean calendar months, use a month interval — they are different operations.

---

## 6. Truncation and Bucketing Are Mostly Non-Standard

Rounding a timestamp down to the start of its hour/day/month ("bucketing") is everyday work but the common tools are *not* in the SQL standard. PostgreSQL's `date_trunc` is "conceptually similar to the trunc function for numbers" and sets "all fields that are less significant than the selected one ... to zero" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)) — but it lives in the engine-specific function section, not the standard `EXTRACT`. SQLite spells the same idea with `strftime`; SQL Server uses `DATETRUNC`/`DATEPART`.

```sql
-- NON-STANDARD (PostgreSQL) — fine if you target only PG; route the spelling otherwise
SELECT date_trunc('day', created_at) AS day, COUNT(*) FROM orders GROUP BY 1;

-- MORE PORTABLE — group by extracted fields (EXTRACT is standard)
SELECT EXTRACT(YEAR FROM created_at) AS y,
       EXTRACT(MONTH FROM created_at) AS m,
       EXTRACT(DAY FROM created_at) AS d, COUNT(*)
FROM orders GROUP BY 1, 2, 3;
```

`date_trunc`/`strftime`/`DATETRUNC` spellings belong in **`sql-standard-vs-dialect-map`**; the portable primitive is `EXTRACT` plus grouping. General `CAST` and format templating live in **`sql-expressions-case-and-functions`**.

---

## 7. Portability Snapshot

The standard defines `DATE`/`TIME`/`TIMESTAMP [WITH TIME ZONE]`/`INTERVAL`, typed literals, `EXTRACT`, and `CURRENT_*`. Engines diverge sharply below that:

| Concern | PostgreSQL | SQLite | MySQL |
|---|---|---|---|
| Native temporal types | rich (`date`, `time`, `timestamp[tz]`, `interval`) | **none** — store TEXT/REAL/INTEGER | `DATE`, `TIME`, `DATETIME`, `TIMESTAMP` |
| Which type is a UTC instant | `timestamp with time zone` | n/a (you encode it) | **`TIMESTAMP`** (UTC-converted, capped ~2038) |
| Naive wall-clock type | `timestamp` (no tz) | n/a | **`DATETIME`** (wide range, no conversion) |
| Stored `INTERVAL` type | yes | no | no (interval only in expressions) |
| Truncation/bucketing | `date_trunc` | `strftime` | `DATE_FORMAT` / functions |

Two landmines stand out:

- **SQLite has no dedicated date/time datatype.** Values "can be stored as" ISO-8601 text, a Julian day number as REAL, or a Unix timestamp as INTEGER ([SQLite — Date And Time Functions](https://www.sqlite.org/lang_datefunc.html)), and you interpret them with `date()`/`datetime()`/`strftime()`. Code relying on an engine-enforced temporal type does not port to SQLite unchanged.
- **MySQL inverts the PostgreSQL intuition.** "MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage ... (This does not occur for other types such as `DATETIME`.)" ([MySQL — DATE, DATETIME, TIMESTAMP](https://dev.mysql.com/doc/refman/8.0/en/datetime.html)) — so MySQL `TIMESTAMP` is the instant type (but ranges only "'1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC", the 2038 problem), while `DATETIME` is the naive wide-range type. Which type is "the instant" depends on the engine.

`DATEADD`/`DATEDIFF`/`STRFTIME` and every per-engine spelling route to **`sql-standard-vs-dialect-map`**.

---

## 8. Who Suffers When Datetime Is Mishandled

A temporal bug rarely errors at write time — it surfaces as a wrong number or a wrong hour, and someone downstream pays:

- The **person who showed up an hour early** (or late) to a meeting, because the event was stored in a naive `TIMESTAMP` and re-rendered after a DST transition shifted the wall-clock fields against the real instant (§1). The database "worked"; the calendar lied.
- The **analyst chasing an off-by-one in "yesterday's rows,"** because the query did string date math (`LEFT(created_at, 10) = '2024-03-09'`) instead of comparing instants — and a row at `23:30` in one zone is "today" in another, or a midnight boundary fell inside a DST jump (§5, §6).
- The **maintainer migrating PostgreSQL to MySQL** who assumed `TIMESTAMP` meant the same thing in both, and silently turned UTC instants into unconverted wall-clock values (or hit the 2038 ceiling) (§7).
- The **next developer on SQLite** whose `INTERVAL '1 month'` and `date_trunc` simply do not exist, discovering at runtime that the column was never a real date (§7).

Using `TIMESTAMP WITH TIME ZONE` for instants, `INTERVAL` for math, and `EXTRACT` for fields is empathy in code: it hands the next person a value that means the same thing in every zone and every engine.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics and three-valued logic (a NULL timestamp propagates UNKNOWN through comparisons and `BETWEEN`).
- `sql-expressions-case-and-functions` — general `CAST`, `COALESCE`, and format templating around temporal values.
- `sql-data-types-and-numerics` — numeric precision/scale for the values `EXTRACT` returns and for epoch integers.
- `sql-temporal-tables` — system-time / valid-time history (period predicates, `AS OF`), distinct from the scalar types here.
- `sql-standard-vs-dialect-map` — per-engine spellings: `NOW()`/`GETDATE()`/`SYSDATE`, `DATEADD`/`DATEDIFF`, `date_trunc`/`strftime`/`DATETRUNC`, and the MySQL `TIMESTAMP`/`DATETIME` divergence.

---

## 10. Reference Files

High-frequency datetime/interval anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
