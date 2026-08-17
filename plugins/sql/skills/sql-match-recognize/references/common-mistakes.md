# Common `MATCH_RECOGNIZE` Mistakes
## Contents

- [1. Using MATCHRECOGNIZE on an engine that lacks it](#1-using-matchrecognize-on-an-engine-that-lacks-it)
- [2. Wrong clause order](#2-wrong-clause-order)
- [3. Forgetting ORDER BY — or assuming partition order](#3-forgetting-order-by-or-assuming-partition-order)
- [4. Confusing RUNNING and FINAL in MEASURES](#4-confusing-running-and-final-in-measures)
- [5. Greedy quantifier swallowing the next match](#5-greedy-quantifier-swallowing-the-next-match)
- [6. Wrong AFTER MATCH SKIP — missing or duplicating matches](#6-wrong-after-match-skip-missing-or-duplicating-matches)
- [7. Using an undefined variable by accident (typo anchors everything)](#7-using-an-undefined-variable-by-accident-typo-anchors-everything)


Anti-patterns in LLM-generated row-pattern-recognition SQL, each with wrong/right code and a
primary-source citation. The skill (`sql-match-recognize`) states the rules; this file holds the
high-frequency failure modes. The single biggest one is reaching for the clause on an engine that
does not have it — start there.

---

## 1. Using `MATCH_RECOGNIZE` on an engine that lacks it

**The problem:** The model emits a `MATCH_RECOGNIZE` query for PostgreSQL 16, MySQL 8, MariaDB 11, or SQLite 3.40 — all of which **do not have it**. Row pattern matching is an *optional* SQL:2016 feature (R010) that only very recently shipped in PostgreSQL 18, MySQL 9.7, MariaDB 12.3, SQLite 3.53, and SQL Server 2025 ([modern-sql.com](https://modern-sql.com/feature/match_recognize)). On an older release it is a hard syntax error, not a slow query.

```sql
-- WRONG (on deployed PostgreSQL 16 / MySQL 8 / SQLite < 3.53) — syntax error; the clause does not exist
SELECT ... FROM ticks MATCH_RECOGNIZE ( ... PATTERN (A B+ C) DEFINE ... );

-- RIGHT — confirm the engine/version first; if it lacks the feature, use the portable window fallback
--          (gaps-and-islands ROW_NUMBER-difference / running-sum-of-a-flag).
SELECT grp, MIN(ts), MAX(ts)
FROM (SELECT ts, val, val - ROW_NUMBER() OVER (ORDER BY ts) AS grp FROM ticks) t
GROUP BY grp;
```

*Source: [modern-sql.com — MATCH_RECOGNIZE](https://modern-sql.com/feature/match_recognize). Depth: skill §6; portable fallback owned by `sql-gaps-and-islands`; version matrix by `sql-standard-vs-dialect-map`.*

---

## 2. Wrong clause order

**The problem:** The sub-clauses have a **fixed order** — `PARTITION BY`, `ORDER BY`, `MEASURES`, rows-per-match, `AFTER MATCH SKIP`, `PATTERN`, `SUBSET`, `DEFINE` ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)). The model, pattern-matching on ordinary `SELECT` order, puts `DEFINE` before `PATTERN` or `PATTERN` before `MEASURES`, and the parser rejects it.

```sql
-- WRONG — DEFINE before PATTERN, MEASURES after PATTERN: order is invalid
MATCH_RECOGNIZE (
  PARTITION BY id ORDER BY ts
  DEFINE UP AS price > PREV(price)
  PATTERN (UP+)
  MEASURES LAST(UP.price) AS top )

-- RIGHT — MEASURES ... PATTERN ... DEFINE, in that order; PATTERN and DEFINE are the only mandatory parts
MATCH_RECOGNIZE (
  PARTITION BY id ORDER BY ts
  MEASURES LAST(UP.price) AS top
  PATTERN (UP+)
  DEFINE  UP AS price > PREV(price) )
```

*Source: [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html). Depth: skill §1.*

---

## 3. Forgetting `ORDER BY` — or assuming partition order

**The problem:** A pattern is meaningless without a row order, and a relational input has none on its own (foundation skill: a result is an unordered set). The model omits `ORDER BY` inside the clause and assumes insertion or key order. Engines require `ORDER BY`; even where tolerated, `PREV`/`NEXT`/`LAST` would be undefined.

```sql
-- WRONG — no ORDER BY: "previous row" is undefined, the V-shape is nonsense
MATCH_RECOGNIZE ( PARTITION BY customer_id
  PATTERN (DOWN+ UP+) DEFINE DOWN AS price < PREV(price), UP AS price > PREV(price) )

-- RIGHT — order the stream the pattern walks over
MATCH_RECOGNIZE ( PARTITION BY customer_id ORDER BY order_date
  PATTERN (DOWN+ UP+) DEFINE DOWN AS price < PREV(price), UP AS price > PREV(price) )
```

*Source: [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html). Depth: skill §1; set-ordering rule owned by `sql-relational-and-null-discipline`.*

---

## 4. Confusing RUNNING and FINAL in `MEASURES`

**The problem:** With `ALL ROWS PER MATCH`, `MEASURES` defaults to **running** semantics — each output row sees only the match-so-far. The model writes `LAST(DOWN.price)` expecting the match's eventual bottom on every row, but gets the running low instead. "The expressions of the MEASURES clause are evaluated when the match is complete," but per-row `ALL ROWS` output reads them running unless you say `FINAL` ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)).

```sql
-- WRONG (intent: "the V's bottom on every row") — gets the running low, which changes row to row
MEASURES LAST(DOWN.price) AS bottom
ALL ROWS PER MATCH

-- RIGHT — FINAL pins the completed-match value on every output row
MEASURES FINAL LAST(DOWN.price) AS bottom
ALL ROWS PER MATCH
```

(With the default `ONE ROW PER MATCH` only the completed summary emits, so the distinction is moot — the trap is specific to `ALL ROWS PER MATCH`.)

*Source: [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html). Depth: skill §4.*

---

## 5. Greedy quantifier swallowing the next match

**The problem:** Quantifiers are greedy by default — they "prefer higher number of repetitions over lower number" ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)). A greedy `B+` consumes rows that belong to the *next* occurrence, so two adjacent patterns get reported as one (or the trailing delimiter is eaten). The fix is a reluctant quantifier (`B+?`) or a tighter `AFTER MATCH SKIP`.

```sql
-- WRONG — greedy A+ extends as far as possible, merging what should be two matches
PATTERN (START A+ B)

-- RIGHT — reluctant A+? takes the fewest A rows that still let B match (note: +? = reluctant one-or-more,
--          NOT "one-or-more then optional")
PATTERN (START A+? B)
```

*Source: [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html). Depth: skill §3.*

---

## 6. Wrong `AFTER MATCH SKIP` — missing or duplicating matches

**The problem:** The model leaves the default `SKIP PAST LAST ROW` when it actually needs overlapping matches (so it misses Vs that start inside a prior V), or it sets `SKIP TO` a variable that can be the match's first row, which loops forever and errors. The default resumes "past the last row"; alternatives "skip to the next row or to a specific position" ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)).

```sql
-- WRONG — SKIP TO FIRST <start-variable> can land on the match's own first row -> infinite loop / engine error
AFTER MATCH SKIP TO FIRST START

-- RIGHT — non-overlapping (default): each match found once
AFTER MATCH SKIP PAST LAST ROW
-- RIGHT — deliberate overlap: resume one row after the match start
AFTER MATCH SKIP TO NEXT ROW
```

*Source: [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html). Depth: skill §5.*

---

## 7. Using an undefined variable by accident (typo anchors everything)

**The problem:** A pattern variable that appears in `PATTERN` but is **not** in `DEFINE` matches *any* row ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)). That is intentional for an anchor like `START`, but a typo in `DEFINE` (defining `DWN` while the pattern says `DOWN`) silently turns the intended condition into "match anything," so the pattern over-matches with no error.

```sql
-- WRONG — DEFINE names DWN, PATTERN names DOWN; DOWN is therefore undefined and matches ANY row
PATTERN (START DOWN+ UP+)
DEFINE  DWN AS price < PREV(price), UP AS price > PREV(price)

-- RIGHT — the defined name matches the pattern name exactly
PATTERN (START DOWN+ UP+)
DEFINE  DOWN AS price < PREV(price), UP AS price > PREV(price)
```

*Source: [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html). Depth: skill §1.*
