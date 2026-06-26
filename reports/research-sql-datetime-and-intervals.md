# Research: sql-datetime-and-intervals

Research backing the `sql-datetime-and-intervals` skill (skill #11 in `reports/skill-plan-sql.md`).
Each entry: source URL, section, verbatim quote, and why it matters for the skill.
Accessed 2026-06-26.

---

## Source A — PostgreSQL: Date/Time Types

URL: https://www.postgresql.org/docs/current/datatype-datetime.html

### A1. `timestamp with time zone` stores a UTC instant (THE centerpiece fact)

Section: 8.5.1.3 Time Stamps / "Time Zones".

> "For `timestamp with time zone` values, an input string that includes an explicit time zone
> will be converted to UTC (Universal Coordinated Time) using the appropriate offset for that
> time zone. If no time zone is stated in the input string, then it is assumed to be in the time
> zone indicated by the system's TimeZone parameter, and is converted to UTC using the offset for
> the timezone zone. In either case, the value is stored internally as UTC, and the originally
> stated or assumed time zone is not retained."

> "When a `timestamp with time zone` value is output, it is always converted from UTC to the
> current `timezone` zone, and displayed as local time in that zone."

> "All timezone-aware dates and times are stored internally in UTC."

**Why it matters:** This is the #1 datetime bug. `timestamp with time zone` (timestamptz) stores an
*absolute instant* (a UTC point in time). It is the correct type for "when did this event happen."
The name is misleading — it does NOT store a zone; it stores a UTC instant and renders in the session
zone. This is the centerpiece WRONG/RIGHT of the skill.

### A2. `timestamp without time zone` is a naive local timestamp (no instant)

Section: 8.5.1.3.

> "In a value that has been determined to be `timestamp without time zone`, PostgreSQL will silently
> ignore any time zone indication. That is, the resulting value is derived from the date/time fields
> in the input string, and is not adjusted for time zone."

**Why it matters:** `timestamp without time zone` is the SQL default `TIMESTAMP`. It is NOT an instant —
it is wall-clock fields with no zone. Store an event's time in one of these and you have lost the
information needed to know what UTC instant it refers to. This is the naive-local-timestamp trap that
produces the "meeting showed at the wrong hour after DST" bug.

### A3. PostgreSQL recommendation

Section: 8.5.3 Time Zones.

> "To address these difficulties, we recommend using date/time types that contain both date and time
> when using time zones. We do not recommend using the type `time with time zone` (though it is
> supported by PostgreSQL for legacy applications and for compliance with the SQL standard)."

**Why it matters:** Authority for "use timestamptz for instants; avoid `time with time zone`."

### A4. The available types, ranges, resolution

Section: Table 8.9 Date/Time Types.

- `timestamp [without time zone]` — 8 bytes, both date and time, 4713 BC .. 294276 AD, 1 microsecond.
- `timestamp with time zone` — 8 bytes, both date and time with time zone, same range, 1 microsecond.
- `date` — 4 bytes, date only, 4713 BC .. 5874897 AD, 1 day.
- `time [without time zone]` — 8 bytes, time of day only, 00:00:00 .. 24:00:00, 1 microsecond.
- `time with time zone` — 12 bytes (the legacy/discouraged one).
- `interval [fields] [(p)]` — 16 bytes, -178000000 years .. 178000000 years, 1 microsecond.

**Why it matters:** These are the standard temporal types the skill teaches: DATE, TIME, TIMESTAMP
(with/without TIME ZONE), INTERVAL. `date` has day resolution (no time), `time` has no date.

### A5. Typed literal syntax

Section: 8.5.1 Date/Time Input / 4.1.2.7 Constants of Other Types.

> "SQL requires the following syntax
>   type [ (p) ] 'value'
> where p is an optional precision specification giving the number of fractional digits in the
> seconds field."

Examples: `TIMESTAMP WITH TIME ZONE '2004-10-19 10:23:54+02'`, `DATE '2024-01-01'`,
`INTERVAL '1' YEAR`, `INTERVAL '1 day 2:03:04'`.

**Why it matters:** Typed literals (`DATE '2024-01-01'`, `TIMESTAMP '...'`, `INTERVAL '1 month'`) are
the portable, unambiguous way to write temporal constants — not strings parsed by chance, not vendor
conversion functions.

---

## Source B — PostgreSQL: Date/Time Functions and Operators

URL: https://www.postgresql.org/docs/current/functions-datetime.html

### B1. EXTRACT(field FROM source)

Section: 9.9.1 EXTRACT, date_part.

> "The extract function retrieves subfields such as year or hour from date/time values. source must
> be a value expression of type timestamp, date, time, or interval. (Timestamps and times can be with
> or without time zone.) field is an identifier or string that selects what field to extract from the
> source value. Not all fields are valid for every input data type; for example, fields smaller than a
> day cannot be extracted from a date, while fields of a day or more cannot be extracted from a time.
> The extract function returns values of type numeric."

**Why it matters:** `EXTRACT(field FROM source)` is the SQL-standard, portable way to pull
year/month/day/hour out of a temporal value — instead of vendor `DATEPART`/`strftime` or string slicing.

### B2. CURRENT_DATE / CURRENT_TIMESTAMP / LOCALTIMESTAMP are SQL-standard, transaction-start time

Section: 9.9.5 Current Date/Time.

> "PostgreSQL provides a number of functions that return values related to the current date and time.
> These SQL-standard functions all return values based on the start time of the current transaction:
>   CURRENT_DATE
>   CURRENT_TIME
>   CURRENT_TIMESTAMP
>   CURRENT_TIME(precision)
>   CURRENT_TIMESTAMP(precision)
>   LOCALTIME
>   LOCALTIMESTAMP
>   LOCALTIME(precision)
>   LOCALTIMESTAMP(precision)"

> "Since these functions return the start time of the current transaction, their values do not change
> during the transaction."

**Why it matters:** `CURRENT_DATE` / `CURRENT_TIMESTAMP` / `LOCALTIMESTAMP` are the standard spellings;
they are keywords (no parentheses), not functions like `NOW()`/`GETDATE()`/`SYSDATE`. Also the
gotcha that they are transaction-start time, not statement/wall-clock time.

### B3. now() is the PostgreSQL traditional (non-standard) equivalent

Section: 9.9.5.

> "now() is a traditional PostgreSQL equivalent to transaction_timestamp()."

> "transaction_timestamp() is equivalent to CURRENT_TIMESTAMP, but is named to clearly reflect what it
> returns."

**Why it matters:** `now()` is vendor spelling; the standard is `CURRENT_TIMESTAMP`. Route `NOW()`,
`GETDATE()`, `SYSDATE`, `SYSDATETIME()` to the dialect map.

### B4. Interval arithmetic on dates/timestamps

Section: 9.9 Table of operators; 9.9.4.

> "When adding an interval value to (or subtracting an interval value from) a timestamp or timestamp
> with time zone value, the months, days, and microseconds fields of the interval value are handled in
> turn."

Examples:
> `date '2001-09-28' + interval '1 hour'` -> `2001-09-28 01:00:00`
> `timestamp '2001-09-28 01:00' + interval '23 hours'` -> `2001-09-29 00:00:00`

**Why it matters:** `date/timestamp + INTERVAL '...'` is the standard, readable way to do date math —
not `DATEADD(day, 7, x)` or `x + 7` (integer-days math is non-portable and meaningless for months).
The "months, days, and microseconds handled in turn" detail is the seed of the month-length ambiguity
(a month is not a fixed number of days).

### B5. date_trunc

Section: 9.9.3 date_trunc.

> "The function date_trunc is conceptually similar to the trunc function for numbers."

> "source is a value expression of type timestamp, timestamp with time zone, or interval. ... field
> selects to which precision to truncate the input value. The return value is likewise of type
> timestamp, timestamp with time zone, or interval, and it has all fields that are less significant
> than the selected one set to zero (or one, for day and month)."

**Why it matters:** `date_trunc` is the common bucketing tool but it is a PostgreSQL function (it lives
in the PostgreSQL-specific function section, alongside the SQL-standard `EXTRACT`). It is NOT in the
SQL standard. SQLite uses `strftime`, SQL Server uses `DATETRUNC`/`DATEPART`. Route bucketing spellings
to the dialect map; the portable fallback is `EXTRACT` + grouping or constructing the truncated value.

---

## Source C — SQLite: Date And Time Functions

URL: https://www.sqlite.org/lang_datefunc.html

### C1. SQLite has NO native date/time type (the big portability deviation)

> "SQLite does not have a dedicated date/time datatype. Instead, date and time values can be stored as
> any of the following:" [ISO-8601 text, Julian day number as REAL, Unix timestamp as INTEGER]

The three storage formats, verbatim:
> "ISO-8601 — A text string that is one of the ISO 8601 date/time values ... Example:
> '2025-05-29 14:16:00'.
> Julian day number — The number of days including fractional days since -4713-11-24 12:00:00
> Example: 2460825.09444444.
> Unix timestamp — The number of seconds including fractional seconds since 1970-01-01 00:00:00
> Example: 1748528160."

**Why it matters:** This is the headline deviation. In SQLite there is no `DATE`/`TIMESTAMP` column type
that the engine enforces — you store text/real/integer and call functions to interpret it. Code that
relies on a real temporal type does not port to SQLite unchanged.

### C2. SQLite date/time functions

> "SQLite supports seven scalar date and time functions as follows:
>   date(...), time(...), datetime(...), julianday(...), unixepoch(...),
>   strftime(format, time-value, modifier, ...), timediff(...)"

> "The date() function returns the date as text in this format: YYYY-MM-DD."
> "The datetime() function returns the date and time formatted as YYYY-MM-DD HH:MM:SS ..."
> "The strftime() function returns the date formatted according to the format string specified as the
> first argument."

**Why it matters:** SQLite's `strftime()`/`date()`/`datetime()` are the SQLite spelling of what other
engines do with `EXTRACT`/`date_trunc`/casts. They are non-standard; route to the dialect map.

---

## Source D — MySQL: The DATE, DATETIME, and TIMESTAMP Types

URL: https://dev.mysql.com/doc/refman/8.0/en/datetime.html

### D1. DATETIME vs TIMESTAMP range

> "The supported range is '1000-01-01 00:00:00' to '9999-12-31 23:59:59'." (DATETIME)

> "TIMESTAMP has a range of '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC." (the 2038 problem)

### D2. TIMESTAMP is UTC-converted; DATETIME is not

> "MySQL converts TIMESTAMP values from the current time zone to UTC for storage, and back from UTC to
> the current time zone for retrieval. (This does not occur for other types such as DATETIME.)"

> "If you store a TIMESTAMP value, and then change the time zone and retrieve the value, the retrieved
> value is different from the value you stored."

**Why it matters:** MySQL's naming is the inverse of Postgres intuition. In MySQL, `TIMESTAMP` is the
UTC-instant type (like Postgres `timestamptz`) but it is capped at 2038; `DATETIME` is the naive
wall-clock type with the wide range. So "which type is the instant" depends on the engine — a portability
landmine. Route to the dialect map.

---

## Synthesis — what the skill must nail

1. **TIMESTAMP vs TIMESTAMP WITH TIME ZONE (centerpiece).** Store events as an instant
   (`TIMESTAMP WITH TIME ZONE` in Postgres = UTC instant). Never store a naive local timestamp for an
   event you will later compare across zones / DST. (A1, A2, A3)
2. **Typed literals & EXTRACT & INTERVAL arithmetic, not string math or vendor functions.** Use
   `DATE '2024-01-01'`, `EXTRACT(YEAR FROM x)`, `x + INTERVAL '1 month'`. (A5, B1, B4)
3. **Standard CURRENT_DATE/CURRENT_TIMESTAMP/LOCALTIMESTAMP vs NOW()/GETDATE()/SYSDATE.** (B2, B3)
4. **INTERVAL literals & month-length ambiguity.** `+ INTERVAL '1 month'` is calendar-aware (Jan 31 +
   1 month is not "+30 days"); months/days/seconds handled separately. (A4, B4)
5. **SQLite has no native temporal type** (text/real/integer + functions) — the biggest deviation;
   **MySQL DATETIME vs TIMESTAMP** epoch range + which one is the UTC instant. (C1, C2, D1, D2)
6. **Truncation/bucketing (`date_trunc`, `strftime`) is non-standard** — route spellings to the dialect
   map; the portable primitive is `EXTRACT`. (B5, C2)
