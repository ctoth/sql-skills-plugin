# Common SQL Datetime & Interval Mistakes
## Contents

- [1. Storing an event as a naive TIMESTAMP instead of an instant](#1-storing-an-event-as-a-naive-timestamp-instead-of-an-instant)
- [2. Bare-string date literals with ambiguous format](#2-bare-string-date-literals-with-ambiguous-format)
- [3. String date math for "yesterday" / ranges](#3-string-date-math-for-yesterday-ranges)
- [4. Integer arithmetic for date offsets](#4-integer-arithmetic-for-date-offsets)
- [5. Treating a month as a fixed number of days](#5-treating-a-month-as-a-fixed-number-of-days)
- [6. Vendor NOW()/GETDATE()/SYSDATE instead of CURRENTTIMESTAMP](#6-vendor-nowgetdatesysdate-instead-of-currenttimestamp)
- [7. Assuming CURRENTTIMESTAMP is per-statement wall-clock time](#7-assuming-currenttimestamp-is-per-statement-wall-clock-time)
- [8. Extracting fields by slicing the string form](#8-extracting-fields-by-slicing-the-string-form)
- [9. Assuming datetrunc/strftime is portable standard SQL](#9-assuming-datetruncstrftime-is-portable-standard-sql)
- [10. Porting TIMESTAMP between PostgreSQL and MySQL assuming it means the same thing](#10-porting-timestamp-between-postgresql-and-mysql-assuming-it-means-the-same-thing)
- [11. Relying on a native date type in SQLite](#11-relying-on-a-native-date-type-in-sqlite)
- [12. Using TIME WITH TIME ZONE for an instant](#12-using-time-with-time-zone-for-an-instant)


Anti-patterns in LLM-generated SQL around temporal types, time zones, and date math, each with wrong/right
code and a primary-source citation. The policy is owned by `sql-datetime-and-intervals`; this file holds the
high-frequency failure modes. All RIGHT examples are standard/portable SQL where possible; non-standard
spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Storing an event as a naive `TIMESTAMP` instead of an instant

**The problem:** The model picks bare `TIMESTAMP` (no zone) for an event time. It records wall-clock fields with no zone, so the same value names a different instant in every zone and shifts an hour across DST. PostgreSQL "will silently ignore any time zone indication ... the resulting value ... is not adjusted for time zone" ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)); the instant type stores UTC: "the value is stored internally as UTC" (same source).

```sql
-- WRONG — naive local timestamp; which instant? renders an hour off after DST
CREATE TABLE meetings (id bigint, starts_at TIMESTAMP);

-- RIGHT — a UTC instant, rendered in the reader's session zone
CREATE TABLE meetings (id bigint, starts_at TIMESTAMP WITH TIME ZONE);
```

*Source: [PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html). Depth: this skill, §1.*

---

## 2. Bare-string date literals with ambiguous format

**The problem:** The model writes a date as a plain string and lets the engine guess the format. `'01/02/2024'` is Jan 2 or Feb 1 depending on locale/engine. The standard typed-literal form `type 'value'` pins both type and interpretation ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)).

```sql
-- WRONG — implicit, locale-dependent parsing
SELECT * FROM events WHERE day = '01/02/2024';

-- RIGHT — typed ISO-8601 literal
SELECT * FROM events WHERE day = DATE '2024-01-02';
```

*Source: [PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html). Depth: this skill, §3.*

---

## 3. String date math for "yesterday" / ranges

**The problem:** The model slices the text form of a timestamp to filter by day, which ignores the real type and time-of-day, and breaks across zones and midnight boundaries. Compare instants instead.

```sql
-- WRONG — text surgery; ignores time-of-day and zone, off-by-one at boundaries
SELECT * FROM logs WHERE LEFT(CAST(created_at AS text), 10) = '2024-03-09';

-- RIGHT — half-open instant range with typed literals
SELECT * FROM logs
WHERE created_at >= TIMESTAMP WITH TIME ZONE '2024-03-09 00:00:00+00'
  AND created_at <  TIMESTAMP WITH TIME ZONE '2024-03-10 00:00:00+00';
```

*Source: [PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html). Depth: this skill, §5, §6.*

---

## 4. Integer arithmetic for date offsets

**The problem:** The model adds a bare integer to a date (`+ 7`) for "a week later." It is non-portable (engines disagree on whether/how integers add to dates) and meaningless for months. Add an `INTERVAL`: "the months, days, and microseconds fields of the interval value are handled in turn" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)).

```sql
-- WRONG — integer + date is non-portable
SELECT due_date + 7 FROM invoices;

-- RIGHT — explicit interval
SELECT due_date + INTERVAL '7 days' FROM invoices;
```

*Source: [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html). Depth: this skill, §5.*

---

## 5. Treating a month as a fixed number of days

**The problem:** The model approximates "one month" as `+ 30 days` (or `+ INTERVAL '30 days'`). Calendar months vary in length and leap years exist; `INTERVAL '1 month'` is calendar-aware, `30 days` is not. Jan 31 + 1 month is a February date, not Jan 31 + 30 days.

```sql
-- WRONG — "+30 days" is not "+1 month"
SELECT start_date + INTERVAL '30 days' AS next_cycle FROM subscriptions;

-- RIGHT — calendar-aware month arithmetic
SELECT start_date + INTERVAL '1 month' AS next_cycle FROM subscriptions;
```

*Source: [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html). Depth: this skill, §5.*

---

## 6. Vendor `NOW()`/`GETDATE()`/`SYSDATE` instead of `CURRENT_TIMESTAMP`

**The problem:** The model reaches for a vendor function for "now." `now()` is "a traditional PostgreSQL equivalent"; `GETDATE()`/`SYSDATE`/`SYSDATETIME()` are other engines'. The standard spelling is the keyword `CURRENT_TIMESTAMP` ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)).

```sql
-- WRONG — vendor-specific
SELECT * FROM sessions WHERE started_at > NOW() - INTERVAL '1 hour';

-- RIGHT — standard keyword (no parentheses)
SELECT * FROM sessions WHERE started_at > CURRENT_TIMESTAMP - INTERVAL '1 hour';
```

*Source: [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html). Depth: this skill, §4; spellings owned by `sql-standard-vs-dialect-map`.*

---

## 7. Assuming `CURRENT_TIMESTAMP` is per-statement wall-clock time

**The problem:** The model uses `CURRENT_TIMESTAMP` twice in a transaction expecting two different readings. These "return values based on the start time of the current transaction," and "their values do not change during the transaction" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)). For an actual elapsed-time reading, the engine offers `clock_timestamp()`/`statement_timestamp()` (non-standard).

```sql
-- WRONG (assumption) — both reads return the SAME instant inside one transaction
-- so this "elapsed" is always 0
SELECT CURRENT_TIMESTAMP - CURRENT_TIMESTAMP;

-- RIGHT — understand it is transaction-start time; use clock_timestamp() (PG) for wall-clock
SELECT clock_timestamp();   -- non-standard; route the spelling to the dialect map
```

*Source: [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html). Depth: this skill, §4.*

---

## 8. Extracting fields by slicing the string form

**The problem:** The model pulls the year/hour by `SUBSTRING` on the text form, assuming a fixed display format. Use `EXTRACT`, which "retrieves subfields such as year or hour from date/time values" of "type timestamp, date, time, or interval" ([PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)).

```sql
-- WRONG — assumes a text layout; fragile and non-portable
SELECT SUBSTRING(CAST(created_at AS text), 12, 2) AS hr FROM orders;

-- RIGHT — standard field extraction
SELECT EXTRACT(HOUR FROM created_at) AS hr FROM orders;
```

*Source: [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html). Depth: this skill, §4.*

---

## 9. Assuming `date_trunc`/`strftime` is portable standard SQL

**The problem:** The model writes `date_trunc('day', ...)` (or `strftime`) and treats it as portable. `date_trunc` lives in PostgreSQL's engine-specific function set, not the standard; SQLite uses `strftime`, SQL Server `DATETRUNC`. The portable primitive is `EXTRACT` plus grouping.

```sql
-- WRONG (portability) — date_trunc is PostgreSQL-specific
SELECT date_trunc('day', created_at) AS day, COUNT(*) FROM orders GROUP BY 1;

-- RIGHT — portable bucketing by extracted fields
SELECT EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at),
       EXTRACT(DAY FROM created_at), COUNT(*)
FROM orders GROUP BY 1, 2, 3;
```

*Source: [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html); [SQLite — Date And Time Functions](https://www.sqlite.org/lang_datefunc.html). Depth: this skill, §6; spellings owned by `sql-standard-vs-dialect-map`.*

---

## 10. Porting `TIMESTAMP` between PostgreSQL and MySQL assuming it means the same thing

**The problem:** The model migrates a schema and keeps `TIMESTAMP`, assuming identical semantics. In MySQL `TIMESTAMP` is UTC-converted (the instant type) but capped at 2038, while `DATETIME` is the naive wide-range type — the reverse of PostgreSQL, where `timestamp` is naive and `timestamp with time zone` is the instant. "MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage ... (This does not occur for other types such as `DATETIME`.)" ([MySQL — DATE, DATETIME, TIMESTAMP](https://dev.mysql.com/doc/refman/8.0/en/datetime.html)).

```sql
-- WRONG (assumption) — same word, opposite meaning across engines; silent corruption
-- PostgreSQL: TIMESTAMP = naive (no conversion)
-- MySQL:      TIMESTAMP = UTC-converted instant, but dies after 2038-01-19

-- RIGHT — pick by intent, per engine:
--   instant  -> PostgreSQL TIMESTAMP WITH TIME ZONE ; MySQL TIMESTAMP (mind 2038) or store UTC in DATETIME
--   wall-clock-> PostgreSQL TIMESTAMP               ; MySQL DATETIME
```

*Source: [MySQL — The DATE, DATETIME, and TIMESTAMP Types](https://dev.mysql.com/doc/refman/8.0/en/datetime.html). Depth: this skill, §7; routed to `sql-standard-vs-dialect-map`.*

---

## 11. Relying on a native date type in SQLite

**The problem:** The model declares a `DATE`/`TIMESTAMP` column in SQLite and assumes the engine enforces and stores it as a real temporal type. SQLite "does not have a dedicated date/time datatype"; values are stored as ISO-8601 text, a Julian day REAL, or a Unix-epoch INTEGER ([SQLite — Date And Time Functions](https://www.sqlite.org/lang_datefunc.html)), interpreted by `date()`/`datetime()`/`strftime()`.

```sql
-- WRONG (assumption) — SQLite does not enforce this as a date; it's affinity only
CREATE TABLE events (id INTEGER, at TIMESTAMP);

-- RIGHT — store a canonical ISO-8601 UTC string (or epoch INTEGER) and use the functions
CREATE TABLE events (id INTEGER, at TEXT);  -- '2024-03-10T07:30:00Z'
SELECT strftime('%Y-%m-%d', at) AS day FROM events;   -- SQLite spelling
```

*Source: [SQLite — Date And Time Functions](https://www.sqlite.org/lang_datefunc.html). Depth: this skill, §7; routed to `sql-standard-vs-dialect-map`.*

---

## 12. Using `TIME WITH TIME ZONE` for an instant

**The problem:** The model stores a time-of-day with a zone offset to "make it timezone-aware." A time without a date cannot resolve a DST offset, so the zone is meaningless. PostgreSQL: "We do not recommend using the type `time with time zone` ... we recommend using date/time types that contain both date and time when using time zones" ([PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)).

```sql
-- WRONG — a zoned time-of-day cannot represent an instant; offset is unresolvable
CREATE TABLE shifts (id bigint, begins_at TIME WITH TIME ZONE);

-- RIGHT — use a full timestamp for an instant; bare TIME for a zone-free daily rule
CREATE TABLE shifts (id bigint, begins_at TIMESTAMP WITH TIME ZONE);
CREATE TABLE store_hours (dow int, opens_at TIME);   -- "opens 09:00 local", no instant
```

*Source: [PostgreSQL — Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html). Depth: this skill, §1, §2.*
