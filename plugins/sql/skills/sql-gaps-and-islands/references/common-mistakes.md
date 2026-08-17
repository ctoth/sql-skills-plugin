# Common Gaps-and-Islands Mistakes
## Contents

- [1. A procedural loop / cursor instead of a set-based group key](#1-a-procedural-loop-cursor-instead-of-a-set-based-group-key)
- [2. Using RANK instead of ROWNUMBER in the island difference](#2-using-rank-instead-of-rownumber-in-the-island-difference)
- [3. Applying the integer rownumber trick directly to dates or timestamps](#3-applying-the-integer-rownumber-trick-directly-to-dates-or-timestamps)
- [4. Single ROWNUMBER when islands must respect a category (status runs)](#4-single-rownumber-when-islands-must-respect-a-category-status-runs)
- [5. Trying to filter a window result in WHERE (gaps via LEAD)](#5-trying-to-filter-a-window-result-in-where-gaps-via-lead)
- [6. Sessionization / range-merge running sum on the default RANGE frame](#6-sessionization-range-merge-running-sum-on-the-default-range-frame)
- [7. Forgetting the first-row / NULL case in LAG-based session flagging](#7-forgetting-the-first-row-null-case-in-lag-based-session-flagging)
- [8. Merging ranges by comparing only to the immediately previous row's end](#8-merging-ranges-by-comparing-only-to-the-immediately-previous-rows-end)
- [9. Reaching for MATCHRECOGNIZE where it isn't supported (or hand-rolling where it is)](#9-reaching-for-matchrecognize-where-it-isnt-supported-or-hand-rolling-where-it-is)


Anti-patterns in LLM-generated SQL for consecutive-runs (islands), gaps-between-runs, range merging,
and sessionization, each with wrong/right code and a primary-source citation. The skill
(`sql-gaps-and-islands`) teaches the recipes; this file holds the high-frequency failure modes. All
RIGHT examples are standard/portable window-function SQL (PostgreSQL 8.4+, SQLite 3.25+, MySQL 8.0+);
non-standard spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. A procedural loop / cursor instead of a set-based group key

**The problem:** Asked for "longest consecutive streak" or "group consecutive rows," the model emits a
cursor, a recursive walk, or application-side row-by-row logic. It is slow, unportable, and bug-prone at
every run boundary. The relational answer assigns each row a group key that is identical across a run,
then `GROUP BY` it — "consecutive values in the same island produce the same difference, so you can
`GROUP BY` that difference" ([Red Gate](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/)).

```sql
-- WRONG — cursor / loop walking one row at a time to detect run boundaries (sketch)
-- DECLARE cur CURSOR FOR SELECT seqno FROM readings ORDER BY seqno; ... fetch, compare prev, ...

-- RIGHT — one pass: value minus a gapless row counter is the group key
SELECT user_id, MIN(seqno) AS island_start, MAX(seqno) AS island_end
FROM (
  SELECT user_id, seqno,
         seqno - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY seqno) AS grp
  FROM   readings
) t
GROUP BY user_id, grp;
```

*Source: [Red Gate — Gaps and Islands](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/); [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §2.*

---

## 2. Using `RANK` instead of `ROW_NUMBER` in the island difference

**The problem:** The model writes `value - RANK() OVER (ORDER BY value)`. `RANK` is "with gaps"
([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)) — on
ties it skips numbers, so the difference is no longer constant within a run and islands split or merge
incorrectly. The island trick needs a **gapless, strictly +1** counter: `ROW_NUMBER` (or `DENSE_RANK`
for the per-category variant).

```sql
-- WRONG — RANK skips on ties; the difference is not constant within a run
seqno - RANK() OVER (ORDER BY seqno) AS grp

-- RIGHT — ROW_NUMBER counts 1,2,3,... with no gaps
seqno - ROW_NUMBER() OVER (ORDER BY seqno) AS grp
```

*Source: [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §2; ranking-function tie behavior owned by `sql-window-functions` §2.*

---

## 3. Applying the integer `row_number` trick directly to dates or timestamps

**The problem:** `activity_date - ROW_NUMBER() OVER (...)` is a date-minus-integer type error (or silently
wrong) in most engines, and timestamps are never strictly +1 apart so the trick does not apply at all.
For one-row-per-day data, subtract a *row counter's worth of days*; for irregular timestamps, use the
threshold/running-sum pattern (mistake #6).

```sql
-- WRONG — date minus a bare integer
activity_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date) AS grp

-- RIGHT — subtract an interval scaled by the row number (one row per calendar day)
activity_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date)
                 * INTERVAL '1 day') AS grp
```

*Source: [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §2; date/interval spellings owned by `sql-standard-vs-dialect-map`.*

---

## 4. Single `ROW_NUMBER` when islands must respect a category (status runs)

**The problem:** To find runs of consecutive rows that share a status (e.g. consecutive `up` readings),
the model uses one `ROW_NUMBER` ordered by sequence and gets garbage — it groups by raw consecutiveness,
ignoring status changes. The fix is the two-rank variant: subtract a **per-category** `DENSE_RANK` (which
"counts peer groups" without gaps, [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)) from the global `ROW_NUMBER`.

```sql
-- WRONG — ignores status; merges an "up" run and a following "down" run
seqno - ROW_NUMBER() OVER (PARTITION BY host ORDER BY seqno) AS grp

-- RIGHT — two counters; their difference is constant only while status is unbroken
  ROW_NUMBER() OVER (PARTITION BY host         ORDER BY seqno)
- ROW_NUMBER() OVER (PARTITION BY host, status ORDER BY seqno) AS grp
```

*Source: [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §2.*

---

## 5. Trying to filter a window result in `WHERE` (gaps via `LEAD`)

**The problem:** The model writes `WHERE LEAD(seqno) OVER (...) <> seqno + 1` and gets a syntax error,
because a window function cannot appear in `WHERE` — it "logically execute[s] after" `WHERE`
([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)).
`QUALIFY` would fix it but is not portable (no PostgreSQL/MySQL/SQLite). Wrap the window in a subquery
and filter outside.

```sql
-- WRONG — window function illegal in WHERE
SELECT seqno + 1 AS gap_start FROM invoices
WHERE LEAD(seqno) OVER (ORDER BY seqno) <> seqno + 1;

-- RIGHT — portable: compute in a subquery, filter in the outer query
SELECT seqno + 1 AS gap_start, nxt - 1 AS gap_end
FROM (
  SELECT seqno, LEAD(seqno) OVER (ORDER BY seqno) AS nxt FROM invoices
) t
WHERE nxt IS NOT NULL AND nxt <> seqno + 1;
```

*Source: [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html). Depth: this skill, §3; the wrap pattern + `QUALIFY` portability owned by `sql-window-functions` §6.*

---

## 6. Sessionization / range-merge running sum on the default `RANGE` frame

**The problem:** Building a session id with `SUM(is_new_session) OVER (PARTITION BY user ORDER BY event_at)`
and omitting the frame. The default frame is `RANGE ... CURRENT ROW`, which includes the whole peer group
of tied timestamps — so events sharing a timestamp all get the same running-sum value and the session
boundary is mislaid. Specify `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

```sql
-- WRONG — default RANGE frame lumps tied timestamps; session_id corrupted on ties
SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_at) AS session_id

-- RIGHT — ROWS frame counts physical rows
SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_at
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_id
```

*Source: [randyzwitch — Sessionizing Log Data Using SQL](https://randyzwitch.com/sessionizing-log-data-sql/). Depth: this skill, §5; the RANGE-vs-ROWS default-frame trap owned by `sql-window-functions` §5.*

---

## 7. Forgetting the first-row / NULL case in `LAG`-based session flagging

**The problem:** Flagging a new session only with `event_at - LAG(event_at) OVER (...) > INTERVAL '30 minutes'`.
For the user's first event, `LAG` returns its default `NULL`, so `NULL > interval` is UNKNOWN, the flag is
not set, and the first session never gets id 1 (the running sum starts at 0 or NULL). Handle the first row
explicitly — or seed `LAG`'s third-argument default.

```sql
-- WRONG — first event's flag is UNKNOWN (LAG is NULL); session ids are off
CASE WHEN event_at - LAG(event_at) OVER (PARTITION BY user_id ORDER BY event_at)
          > INTERVAL '30 minutes' THEN 1 ELSE 0 END

-- RIGHT — the first event explicitly opens a session
CASE
  WHEN LAG(event_at) OVER (PARTITION BY user_id ORDER BY event_at) IS NULL THEN 1
  WHEN event_at - LAG(event_at) OVER (PARTITION BY user_id ORDER BY event_at)
       > INTERVAL '30 minutes' THEN 1
  ELSE 0
END
```

*Source: [PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html) (LAG default-on-missing); three-valued logic in `sql-relational-and-null-discipline`. Depth: this skill, §5.*

---

## 8. Merging ranges by comparing only to the immediately previous row's end

**The problem:** To collapse overlapping intervals, the model checks `starts_at > LAG(ends_at)`. That
compares only to the *previous* row's end, missing the case where an earlier, longer interval already
covers the current one (the previous row was short but one before it was long). Carry the **running
maximum end of all prior rows**, not just `LAG`.

```sql
-- WRONG — only looks one row back; an earlier long interval is missed
CASE WHEN starts_at > LAG(ends_at) OVER (PARTITION BY room_id ORDER BY starts_at)
     THEN 1 ELSE 0 END AS new_group

-- RIGHT — compare to the max end of EVERY prior row
CASE WHEN starts_at > MAX(ends_at) OVER (PARTITION BY room_id ORDER BY starts_at
                                         ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)
     THEN 1 ELSE 0 END AS new_group
```

*Source: [Red Gate — Gaps and Islands](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/) (islands generalize to intervals). Depth: this skill, §4.*

---

## 9. Reaching for `MATCH_RECOGNIZE` where it isn't supported (or hand-rolling where it is)

**The problem:** Two opposite errors. (a) The model writes `MATCH_RECOGNIZE` for a simple consecutive-run
query targeting PostgreSQL 16 or MySQL 8 — it does not exist there; row pattern matching is an optional
SQL:2016 feature only recently shipped (PostgreSQL 18, MySQL 9.7, SQL Server 2025) ([modern-sql.com](https://modern-sql.com/feature/match_recognize)).
(b) Conversely, for a genuinely complex multi-state pattern on an engine that *has* it, the model builds
an unreadable tower of nested CASE/running-sum logic instead of the regex-over-rows tool built for it.

```sql
-- WRONG (a) — MATCH_RECOGNIZE on an engine that lacks it: a hard error, not a slow query
-- WRONG (b) — five nested CASE expressions emulating "A then B+ then C" where MATCH_RECOGNIZE is clean

-- RIGHT — portable today: plain window recipe (§2–§5) for simple consecutiveness
-- RIGHT — escalate to MATCH_RECOGNIZE for rich patterns ON A SUPPORTING ENGINE (route to sql-match-recognize)
```

*Source: [modern-sql.com — MATCH_RECOGNIZE](https://modern-sql.com/feature/match_recognize). Depth: this skill, §6; engine support owned by `sql-match-recognize`.*
