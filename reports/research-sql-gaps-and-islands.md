# Research: sql-gaps-and-islands

Research backing the `sql-gaps-and-islands` skill (skill #20 in `reports/skill-plan-sql.md`).
Each entry: URL, section, verbatim quote, why it matters. Accessed 2026-06-26.

This is a **pattern cookbook** skill. It builds on `sql-window-functions` (window mechanics/frames)
and the foundation `sql-relational-and-null-discipline` (set semantics, undefined order without
`ORDER BY`). The canonical techniques are well-known; the job is to capture them verbatim from
reputable sources and route mechanics/escalation to the sibling skills.

---

## Source 1: PostgreSQL — Window Functions (the building blocks)

URL: https://www.postgresql.org/docs/current/functions-window.html

**ROW_NUMBER** — section "General-Purpose Window Functions":
> "Returns the number of the current row within its partition, counting from 1."

Why it matters: `ROW_NUMBER()` counting from 1 over `ORDER BY value` is the engine of the
islands "row_number difference" trick — subtract it from the value and the difference is constant
within a consecutive run. The "counting from 1, no gaps" property is exactly what makes the
subtraction cancel out within a run.

**RANK / DENSE_RANK**:
> "Returns the rank of the current row, with gaps; that is, the `row_number` of the first row in its peer group." (rank)
> "Returns the rank of the current row, without gaps; this function effectively counts peer groups." (dense_rank)

Why it matters: the **dense_rank** variant of the islands trick is needed when the consecutive
key is grouped within categories — you partition and use `DENSE_RANK()` so the per-category counter
has no gaps even when the global sequence does. `dense_rank` "counts peer groups" — the gapless
counter property is what the islands subtraction relies on.

**LAG**:
> "Returns _value_ evaluated at the row that is _offset_ rows before the current row within the partition; if there is no such row, instead returns _default_ ... If omitted, _offset_ defaults to 1 and _default_ to `NULL`."

**LEAD**:
> "Returns _value_ evaluated at the row that is _offset_ rows after the current row within the partition; if there is no such row, instead returns _default_ ... If omitted, _offset_ defaults to 1 and _default_ to `NULL`."

Why it matters: `LAG`/`LEAD` are the gap-detection and "new run starts here" tools. A gap exists
where `LEAD(value) OVER (ORDER BY value) <> value + 1`. A new island/session starts where the
distance from the previous row exceeds the threshold — computed with `LAG`. The third-argument
`default` (e.g. `LAG(ts, 1, ts)`) is the clean way to make the first row of a partition flag as a
new session without a NULL leaking through.

Routing note: full window mechanics (frames, RANGE vs ROWS, why you can't filter a window in WHERE)
are owned by `sql-window-functions`. This skill only uses them.

---

## Source 2: Red Gate / Simple Talk — The SQL of Gaps and Islands in Sequences

URL: https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/

**Definition** — intro:
> "The task of detecting unbroken runs of sequential values (islands) and the missing values between them (gaps) in a column."
> "Islands are unbroken sequences delimited by gaps" while "Gaps are the missing values between islands – ranges where no rows exist."

Why it matters: this is the canonical framing of the problem family. The skill's opening section
restates it verbatim.

**Islands via the row_number-difference trick** — the centerpiece:
```sql
SELECT ID, StartSeqNo=MIN(SeqNo), EndSeqNo=MAX(SeqNo)
FROM (
    SELECT ID, SeqNo
        ,rn=SeqNo-ROW_NUMBER() OVER (PARTITION BY ID ORDER BY SeqNo)
    FROM dbo.GapsIslands
) a
GROUP BY ID, rn;
```
> "Consecutive values in the same island produce the same difference, so you can GROUP BY that difference to find island boundaries."

Why it matters: THE canonical island technique and the skill's centerpiece. The grouping factor
`grp = value - ROW_NUMBER() OVER (ORDER BY value)` is **constant within a consecutive run** because
both value and row_number increase by 1 in lockstep; at a gap the value jumps but row_number does
not, so the difference changes and a new group begins. Then `GROUP BY grp` with
`MIN(value)`/`MAX(value)` collapses each run to its boundaries. The set-based RIGHT replaces the
procedural/self-join WRONG.

**Gaps via comparing consecutive rows** (the article uses a row_number self-join; LEAD is the
modern portable equivalent):
```sql
WITH C AS (
    SELECT ID, SeqNo, ROW_NUMBER() OVER(PARTITION BY ID ORDER BY SeqNo) AS rownum
    FROM dbo.GapsIslands
)
SELECT Cur.ID, StartSeqNo=Cur.SeqNo + 1, EndSeqNo=Nxt.SeqNo - 1
FROM C AS Cur
JOIN C AS Nxt ON Cur.ID = Nxt.ID AND Nxt.rownum = Cur.rownum + 1
WHERE Nxt.SeqNo - Cur.SeqNo > 1;
```
> "This self-join identifies where consecutive row numbers have non-consecutive sequence values, marking gap boundaries."

Why it matters: shows the gap-finding logic — a gap exists where the next value is more than 1
beyond the current. The modern, portable rewrite uses `LEAD(SeqNo) OVER (ORDER BY SeqNo)` directly
(no self-join): a gap is where `LEAD(seqno) - seqno > 1`, and the gap runs from `seqno + 1` to
`LEAD(seqno) - 1`. This is the WRONG (self-join) / RIGHT (LEAD) contrast for the gaps section.

---

## Source 3: use-the-index-luke.com — Sorting and Grouping (index support for the ordering)

URL: https://use-the-index-luke.com/sql/sorting-grouping

> "An index provides an ordered representation of the indexed data."
> "An index stores the data in a presorted fashion. The index is, in fact, sorted just like when using the index definition in an `order by` clause."
> "We can use indexes to avoid the sort operation to satisfy an `order by` clause."
> "An indexed `order by` execution not only saves the sorting effort, however; it is also able to return the first results without processing all input data. The `order by` is thus executed in a _pipelined_ manner."

Why it matters: every gaps-and-islands recipe is driven by a window `ORDER BY` over the sequence/time
column. Sorting is "a very resource intensive operation"; an index on the partition+order columns lets
the engine read rows in order and skip the sort, executing the window in a pipelined manner. This is
the portability/performance footnote — and the route to `sql-indexing-and-sargability` for index
design depth.

---

## Source 4: modern-sql.com — MATCH_RECOGNIZE (when to escalate)

URL: https://modern-sql.com/feature/match_recognize

> "uses simple regular expressions across multiple rows to filter them"

Useful for (listed): "Finding series of consecutive events"; "Pattern matching: trend reversal,
periodic events"; "Top-N per Group".

> Row pattern matching was "introduced by SQL:2016" as an optional feature.

Database support (as of 2026-06): BigQuery, Db2 (LUW) 12.1.4, DuckDB 1.5.0, H2, MariaDB 12.3.2,
MySQL 9.7.0, Oracle DB 23, PostgreSQL 18, SQL Server 2025, SQLite 3.53.0.

Why it matters: `MATCH_RECOGNIZE` is the cleaner standard tool for complex consecutive-event
patterns (multi-condition runs, "A then B+ then C") that the row_number trick handles awkwardly.
But it's an optional SQL:2016 feature only recently landing across engines (PostgreSQL 18,
MySQL 9.7, SQL Server 2025), so it is NOT broadly portable yet. The skill routes the regex-over-rows
detail and engine-support matrix to `sql-match-recognize`, and tells the reader to escalate when the
window trick gets unwieldy — but to prefer pure window functions for portability today.

---

## Source 5: randyzwitch.com — Sessionizing Log Data Using SQL (sessionization technique)

URL: https://randyzwitch.com/sessionizing-log-data-sql/
(corroborated by DZone "Event Analytics: How to Define User Sessions With SQL",
https://dzone.com/articles/event-analytics-how-to-define-user-sessions-with-s)

The canonical sessionization pattern, widely used in production clickstream analytics (Spotify,
Airbnb-style): a session is a run of events from one user where no gap between consecutive events
exceeds a timeout (30 minutes is the web-analytics standard). Two-step window pattern:

1. **Flag boundaries with LAG**: compare each event to the previous event's timestamp; emit `1` when
   the gap exceeds the threshold (a new session starts) or the row is the user's first event, else `0`.
2. **Running sum of the flag**: `SUM(is_new_session) OVER (PARTITION BY user ORDER BY ts ROWS BETWEEN
   UNBOUNDED PRECEDING AND CURRENT ROW)` accumulates the flags into a stable session id — it stays
   flat inside a session and increments by 1 at every boundary.

Quote (paraphrased from the writeups, technique is standard):
> "Every time a 1 shows up, the session_id increments by 1; a value of 0 leaves the running sum
> unchanged, so consecutive events keep the same session_id."

Why it matters: this is the sessionization recipe — and the same running-sum-of-a-boundary-flag
technique generalizes islands to value/time thresholds (where the plain `value - row_number` trick
only works for strictly-+1 integer sequences). The explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND
CURRENT ROW` frame matters here: the default `RANGE` frame would lump tied timestamps into the same
running-sum value (a `sql-window-functions` §5 trap), so the frame must be `ROWS`. Route the frame
mechanics to `sql-window-functions`.

---

## Synthesis: the canonical techniques the skill must nail

1. **Islands via row_number difference (CENTERPIECE)**: `grp = value - ROW_NUMBER() OVER (ORDER BY
   value)` is constant within a consecutive integer run; `GROUP BY grp` + `MIN/MAX` gives island
   boundaries. The `DENSE_RANK` variant groups runs within categories. WRONG = procedural
   loop / self-join; RIGHT = set-based window. (Source 1, 2)
2. **Gaps via LEAD/LAG**: a gap exists where `LEAD(value) OVER (ORDER BY value) <> value + 1`; the
   gap spans `value + 1` .. `LEAD(value) - 1`. WRONG = row_number self-join; RIGHT = LEAD. (Source 1, 2)
3. **Range collapsing (merge adjacent/overlapping ranges)**: order intervals, carry a running
   `MAX(end)` of all prior rows; a new merged group starts where the current `start` exceeds the
   running max-end-so-far (i.e., does not overlap/touch). Running-sum-of-flag again. (Source 5 technique)
4. **Sessionization with a time-gap threshold**: LAG to flag gaps > N minutes (or first event),
   running `SUM` of the flag over a `ROWS` frame = session id. (Source 5)
5. **Escalate to MATCH_RECOGNIZE**: for complex multi-condition row patterns; optional SQL:2016,
   only recently portable — route to `sql-match-recognize`, prefer windows for portability. (Source 4)

Portability: all of recipes 1-4 use only `ROW_NUMBER`/`DENSE_RANK`/`LAG`/`LEAD`/`SUM OVER` — standard
SQL:2003 window functions supported by PostgreSQL 8.4+, SQLite 3.25+, MySQL 8.0+. So the whole
cookbook is broadly portable. The only non-portable escalation is `MATCH_RECOGNIZE` (route out).
Index support for the driving `ORDER BY` routes to `sql-indexing-and-sargability` (Source 3).
