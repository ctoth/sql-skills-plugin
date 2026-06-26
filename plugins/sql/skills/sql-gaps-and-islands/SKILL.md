---
name: sql-gaps-and-islands
description: Guides the canonical "gaps and islands" pattern family — detecting unbroken runs of consecutive values (islands — active streaks, sessions, contiguous date/number ranges) and the missing stretches between them (gaps) — with the portable set-based solution instead of brittle self-joins or procedural row-by-row loops. Centerpiece is the ROW_NUMBER-difference trick — `grp = value - ROW_NUMBER() OVER (ORDER BY value)` is constant within a consecutive run, so GROUP BY grp collapses each run to its MIN/MAX boundaries (DENSE_RANK variant for runs within categories). Covers gaps via `LEAD(value) <> value + 1`, merging adjacent/overlapping ranges with a running MAX(end), and sessionization with a time-gap threshold via a running SUM of a new-session flag. Points to MATCH_RECOGNIZE where window logic gets unwieldy. Auto-invokes when writing or editing queries for consecutive sequences, streaks/runs, longest-active-streak, sessionization, collapsing or merging adjacent date/number ranges, finding missing IDs/dates, or "group consecutive rows" / "find gaps" / "count consecutive days" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Gaps and Islands

> "The task of detecting unbroken runs of sequential values (islands) and the missing values between them (gaps) in a column. ... Islands are unbroken sequences delimited by gaps [and] gaps are the missing values between islands — ranges where no rows exist."
> — [Red Gate / Simple Talk — The SQL of Gaps and Islands](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/)

> "Returns the number of the current row within its partition, counting from 1."
> — [PostgreSQL — Window Functions (`row_number`)](https://www.postgresql.org/docs/current/functions-window.html)

"Gaps and islands" is the name for a whole family of questions that all reduce to *grouping consecutive rows*: longest active streak, contiguous date ranges, user sessions, missing invoice numbers, merging overlapping reservations. The instinct is a procedural loop or a correlated self-join that walks row by row — and it is almost always wrong: slow, unportable, and a magnet for off-by-one bugs. The set-based answer is a short window query. This skill is a cookbook of those recipes. It assumes the window-function mechanics owned by **`sql-window-functions`** (frames, `ROWS` vs `RANGE`, why you can't filter a window in `WHERE`) and the set semantics of the foundation **`sql-relational-and-null-discipline`** (a result is an unordered set — every recipe here is driven by an explicit window `ORDER BY`).

---

## 1. The Problem Family — Why a Loop Is the Wrong Reflex

Every prompt below is the same shape — "group the rows that are consecutive, then summarize each group": *longest streak of active days* (islands over a date sequence); *collapse back-to-back reservations into one range* (range merging); *split a clickstream into sessions after 30 min idle* (sessionization); *which invoice numbers are missing?* (gaps).

A row-by-row procedure (cursor, application loop, recursive walk) can answer all of these, but it processes one row at a time and does not survive a port between engines. The relational answer assigns each row a **group key that is identical for every row in the same run**, then `GROUP BY` that key. The whole skill is variations on *how to compute that key*.

---

## 2. Islands via the `ROW_NUMBER` Difference — the Centerpiece

The canonical island technique: subtract a gapless row counter from the value. `ROW_NUMBER()` "returns the number of the current row within its partition, counting from 1" ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)) — so ordered by the sequence column it increases by exactly 1 each row. Within a consecutive run the *value* also increases by exactly 1 each row, so the difference `value - row_number` **stays constant**; at a gap the value jumps but the counter does not, so the difference changes and a new run begins. "Consecutive values in the same island produce the same difference, so you can `GROUP BY` that difference to find island boundaries" ([Red Gate](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/)).

```sql
-- WRONG — correlated self-join walk: find each island's start by asking, for every row,
-- whether the previous value is absent.
SELECT s.seqno AS island_start
FROM   readings s
WHERE  NOT EXISTS (SELECT 1 FROM readings p WHERE p.seqno = s.seqno - 1)
-- ...then a mirror self-join for island_end and a third correlated subquery to pair them.
-- Three passes, O(n²)-ish, fragile, and it still only handles strictly +1 integers.
```

```sql
-- RIGHT — one pass. The difference is the group key; GROUP BY it and take MIN/MAX.
SELECT user_id,
       MIN(seqno) AS island_start,
       MAX(seqno) AS island_end,
       COUNT(*)   AS run_length
FROM (
  SELECT user_id, seqno,
         seqno - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY seqno) AS grp
  FROM   readings
) t
GROUP BY user_id, grp
ORDER BY user_id, island_start;
```

`grp` is meaningless on its own — it is only a *stable label* for "rows in the same run." The `PARTITION BY user_id` keeps each user's runs separate. **Longest active streak** falls straight out: `ORDER BY run_length DESC`.

**Dates, not integers:** if the sequence is one row per *day*, subtract a row counter's worth of days (an interval, not a bare integer):

```sql
-- One row per active day; islands = runs of consecutive calendar days.
SELECT user_id, MIN(activity_date) AS streak_start, MAX(activity_date) AS streak_end, COUNT(*) AS days
FROM (
  SELECT user_id, activity_date,
         activity_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date)
                          * INTERVAL '1 day') AS grp
  FROM   daily_activity
) t
GROUP BY user_id, grp;
```

**Categories within the sequence — the `DENSE_RANK` variant:** when you want runs of consecutive rows that share the *same status* (e.g. consecutive days a server stayed `up`), one `ROW_NUMBER` is not enough. Subtract a **per-category** gapless counter — `DENSE_RANK`, which "counts peer groups" without gaps ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)) — from the global one. The two counters drift by a constant only while the status is unbroken:

```sql
-- Runs of consecutive same-status readings (the "Ben-Gan two-rank" islands variant).
SELECT host, status, MIN(seqno) AS run_start, MAX(seqno) AS run_end
FROM (
  SELECT host, status, seqno,
         ROW_NUMBER() OVER (PARTITION BY host            ORDER BY seqno)
       - ROW_NUMBER() OVER (PARTITION BY host, status     ORDER BY seqno) AS grp
  FROM   host_status
) t
GROUP BY host, status, grp;
```

---

## 3. Gaps via `LEAD` / `LAG`

A gap is the mirror image: a place where the next value is more than one step away. `LEAD` "returns _value_ evaluated at the row that is _offset_ rows after the current row within the partition" ([PostgreSQL — window functions](https://www.postgresql.org/docs/current/functions-window.html)). So a gap exists wherever `LEAD(value) OVER (ORDER BY value) <> value + 1`, and the missing stretch runs from `value + 1` up to `LEAD(value) - 1`.

```sql
-- WRONG — self-join on row_number to pair each row with its successor (Red Gate's pre-window form):
-- correct but verbose, and the join is exactly what LEAD exists to eliminate.
WITH c AS (SELECT seqno, ROW_NUMBER() OVER (ORDER BY seqno) rn FROM invoices)
SELECT cur.seqno + 1 AS gap_start, nxt.seqno - 1 AS gap_end
FROM c cur JOIN c nxt ON nxt.rn = cur.rn + 1
WHERE nxt.seqno - cur.seqno > 1;
```

You cannot put `LEAD(...) <> seqno + 1` in `WHERE` (a window can't be filtered there — `sql-window-functions` §6), and the `QUALIFY` shortcut that would is **not portable** (Snowflake/BigQuery/DuckDB only). Portably, compute the window in a subquery and filter outside:

```sql
-- RIGHT — LEAD reads the next row directly (no self-join); wrap it, then filter.
SELECT gap_start, gap_end
FROM (
  SELECT seqno + 1                            AS gap_start,
         LEAD(seqno) OVER (ORDER BY seqno) - 1 AS gap_end,
         LEAD(seqno) OVER (ORDER BY seqno)     AS nxt
  FROM   invoices
) t
WHERE nxt IS NOT NULL          -- LEAD past the last row is NULL; that is the open end, not a gap
  AND nxt <> seqno + 1
ORDER BY gap_start;
```

The `nxt IS NOT NULL` guard documents intent: `LEAD` past the final row returns its default (`NULL`), and `NULL <> seqno + 1` is UNKNOWN — `WHERE` drops it anyway, but the explicit guard says the trailing edge is deliberately not a gap (foundation skill, three-valued logic).

---

## 4. Collapsing Adjacent / Overlapping Ranges

Given rows that are already *intervals* (`[start, end]`) — reservations, validity periods, on-call shifts — "merge the ones that touch or overlap into single ranges." This is islands over intervals. Order by `start`, carry the **running maximum end of all prior rows**, and start a new merged group wherever the current `start` is past everything seen so far (no overlap, no adjacency):

```sql
-- Merge overlapping/adjacent [starts_at, ends_at] reservations per room into maximal ranges.
WITH flagged AS (
  SELECT room_id, starts_at, ends_at,
         MAX(ends_at) OVER (PARTITION BY room_id ORDER BY starts_at  -- max end of all earlier rows
                            ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prior_max_end
  FROM   reservations
),
grouped AS (
  SELECT room_id, starts_at, ends_at,
         -- new group where this interval does NOT overlap/abut the prior coverage
         SUM(CASE WHEN prior_max_end IS NULL OR starts_at > prior_max_end
                  THEN 1 ELSE 0 END)
             OVER (PARTITION BY room_id ORDER BY starts_at
                   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS grp
  FROM   flagged
)
SELECT room_id, MIN(starts_at) AS merged_start, MAX(ends_at) AS merged_end
FROM   grouped
GROUP BY room_id, grp
ORDER BY room_id, merged_start;
```

The running `SUM` of a 0/1 "new group starts here" flag is the key idea that recurs in §5: it stays flat inside a group and increments by 1 at each boundary, producing a stable group id. Use `> prior_max_end` to merge only true overlaps; use `>=` (or `> prior_max_end + INTERVAL '1 day'`-style adjacency) when back-to-back ranges should also merge — decide deliberately. The `ROWS` frame is required so tied `starts_at` values don't collapse the running aggregate (`sql-window-functions` §5).

---

## 5. Sessionization with a Time-Gap Threshold

Splitting an event stream into sessions ("new session after 30 minutes idle") is islands where the "consecutive" test is a *threshold* rather than strict +1. The plain `value - row_number` trick does not apply — the gaps are irregular — so use the running-sum-of-a-flag pattern. Step 1: with `LAG`, flag each event that opens a new session (idle gap over the threshold, or the user's first event). Step 2: a running `SUM` of that flag is the session id.

```sql
WITH flagged AS (
  SELECT user_id, event_at,
         CASE
           WHEN event_at - LAG(event_at) OVER (PARTITION BY user_id ORDER BY event_at)
                > INTERVAL '30 minutes'
             THEN 1
           WHEN LAG(event_at) OVER (PARTITION BY user_id ORDER BY event_at) IS NULL
             THEN 1                              -- first event for this user opens a session
           ELSE 0
         END AS is_new_session
  FROM   events
)
SELECT user_id, event_at,
       SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_at
                                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_id
FROM   flagged;
```

"Every time a `1` shows up, the `session_id` increments; a value of `0` leaves the running sum unchanged, so consecutive events keep the same `session_id`" ([randyzwitch — Sessionizing Log Data Using SQL](https://randyzwitch.com/sessionizing-log-data-sql/)). Group by `(user_id, session_id)` to get per-session start/end/duration/event-count. The `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame is mandatory: the default `RANGE` frame would lump events sharing a timestamp into one running-sum value and corrupt the ids (`sql-window-functions` §5). `LAG`'s default-on-missing returns `NULL` for the first row, which is why the first-event branch is explicit.

---

## 6. When to Escalate to `MATCH_RECOGNIZE`

The window recipes above handle "consecutive" and "gap over a threshold" cleanly. When the *pattern* gets richer — "a dip of 3+ falling prices followed by a rise," "login then 2+ failures then success," multi-state runs — the row_number/flag tricks turn into a tangle of nested CASE expressions. That is the signal to reach for row pattern recognition. `MATCH_RECOGNIZE` "uses simple regular expressions across multiple rows to filter them," and is purpose-built for "finding series of consecutive events" and "pattern matching: trend reversal, periodic events" ([modern-sql.com — MATCH_RECOGNIZE](https://modern-sql.com/feature/match_recognize)).

The catch is portability: row pattern matching is an **optional SQL:2016 feature** that only recently shipped across engines (PostgreSQL 18, MySQL 9.7, SQL Server 2025, Oracle 23, DuckDB, MariaDB 12.3) ([modern-sql.com](https://modern-sql.com/feature/match_recognize)). For code that must run on today's PostgreSQL 16/17, SQLite, or MySQL 8, stay with the window recipes here. The full syntax, semantics, and engine-support matrix are owned by **`sql-match-recognize`** — route there when you escalate.

---

## 7. Portability Snapshot

Recipes §2–§5 use only `ROW_NUMBER`, `DENSE_RANK`, `LAG`, `LEAD`, and `SUM() OVER` — standard SQL:2003 window functions, so the whole cookbook is **broadly portable**.

| Building block | PostgreSQL | SQLite | MySQL | Portable? |
|---|---|---|---|---|
| `ROW_NUMBER`/`DENSE_RANK`/`LAG`/`LEAD`/`SUM OVER` (§2–§5) | 8.4+ | 3.25+ | 8.0+ | yes |
| Explicit `ROWS` frame for running sums (§4, §5) | yes | yes | yes | yes |
| `QUALIFY` to filter a window inline (§3) | no | no | no | **no — wrap in a subquery** |
| `MATCH_RECOGNIZE` (§6) | 18+ | 3.53+ | 9.7+ | **no — recent/optional; route out** |

Two rules from the sibling skills apply throughout: a window result cannot be filtered in `WHERE`/`HAVING` — wrap it in a subquery/CTE instead of the non-portable `QUALIFY` (`sql-window-functions` §6); and running sums need an explicit `ROWS` frame or tied sort keys corrupt the result (`sql-window-functions` §5).

**Performance:** every recipe is driven by a window `ORDER BY` over the sequence/time column. "An index provides an ordered representation of the indexed data" and "we can use indexes to avoid the sort operation" so the `order by` runs "in a _pipelined_ manner" ([use-the-index-luke.com — Sorting and Grouping](https://use-the-index-luke.com/sql/sorting-grouping)). An index on `(partition_col, order_col)` lets the engine feed the window in order and skip a costly sort — index design routes to **`sql-indexing-and-sargability`**.

---

## 8. Who Suffers When Gaps-and-Islands Is Done by Hand

These queries are correctness-critical and the hand-rolled versions fail quietly:

- The **analyst** who hand-wrote a 100-line stored procedure with a cursor to compute "longest consecutive active streak per user" — one row at a time, an hour to run, and a fencepost bug at every island boundary. The `ROW_NUMBER`-difference query in §2 is six lines, one pass, and has no loop to get wrong.
- The **billing engineer** chasing a revenue discrepancy that turns out to be miscounted *consecutive* subscription days: a self-join that double-counted the boundary day inflated every active-streak by one, so proration was wrong on thousands of accounts. No error was ever thrown — just a number that was quietly off. The set-based island grouping counts each run exactly once.
- The **data engineer** whose sessionization "worked in testing" but assigned two events the same `session_id` across a 4-hour gap, because the default `RANGE` frame lumped same-timestamp events (§5) — a metrics bug that overstated session length in every dashboard downstream.
- The **next maintainer** handed a procedural loop they can't port to the new warehouse, when the whole thing was a `GROUP BY` on a difference.

The window query you wrote instead of the loop is a gift to whoever audits the revenue number under deadline.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics, undefined order without `ORDER BY`, and the three-valued logic behind the `LEAD ... IS NOT NULL` gap guard (§3).
- `sql-window-functions` — the mechanics every recipe stands on: `ROW_NUMBER`/`DENSE_RANK`/`LAG`/`LEAD`, the `ROWS`-vs-`RANGE` frame trap (§4, §5), and the wrap-in-a-CTE pattern for filtering a window (§3).
- `sql-match-recognize` — `MATCH_RECOGNIZE` regex-over-rows for complex multi-state patterns and its engine-support matrix, when the window tricks get unwieldy (§6).
- `sql-indexing-and-sargability` — indexing the `(partition, order)` columns so the driving window `ORDER BY` is pipelined instead of sorted (§7).
- `sql-standard-vs-dialect-map` — date/interval arithmetic spellings (§2), `QUALIFY` availability (§3), and window version gaps.

---

## 10. Reference Files

High-frequency gaps-and-islands anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
