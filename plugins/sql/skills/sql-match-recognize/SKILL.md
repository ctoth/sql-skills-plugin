---
name: sql-match-recognize
description: Guides `MATCH_RECOGNIZE` (SQL:2016 row pattern recognition) — regex-style pattern matching across ordered rows for time-series problems (V-shapes/dip-and-recovery, trend reversals, threshold breaches, complex sessionization) — instead of convoluted self-joins or nested LAG/CASE window gymnastics. Covers the fixed clause order (PARTITION BY / ORDER BY / MEASURES / ONE ROW|ALL ROWS PER MATCH / AFTER MATCH SKIP / PATTERN / DEFINE), PATTERN regex quantifiers and greedy-vs-reluctant matching, pattern-variable conditions, MEASURES with RUNNING vs FINAL semantics, and the AFTER MATCH SKIP modes that control overlapping matches. LOW PORTABILITY — established only on Oracle, Trino, Snowflake, Flink, Vertica, and DB2; absent from deployed PostgreSQL (landed only in 18), MySQL 8, MariaDB (<12.3), and SQLite (<3.53). Auto-invokes when writing or editing time-series pattern detection, row-sequence matching, trend-reversal/dip detection, complex multi-state sessionization, or "find this shape/sequence in rows" requests. CAVEAT — confirm engine support before recommending; when the target engine lacks it, route to `sql-gaps-and-islands` for the portable window-function fallback.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL MATCH_RECOGNIZE — Row Pattern Recognition

> "From the SQL viewpoint, you can think of row pattern matching as extended window functions. ... Before the introduction of MATCH_RECOGNIZE, you had to feed your data to external tools to reason about trends and patterns. Now, you can achieve it directly in your query."
> — [Trino — Row pattern matching](https://trino.io/blog/2021/05/19/row_pattern_matching.html)

> "[MATCH_RECOGNIZE] uses simple regular expressions across multiple rows to filter them. ... useful to implement [f]inding series of consecutive events [and p]attern matching: trend reversal, periodic events."
> — [modern-sql.com — MATCH_RECOGNIZE](https://modern-sql.com/feature/match_recognize)

`MATCH_RECOGNIZE` runs a **regular expression over an ordered stream of rows**. You partition and sort the rows (exactly like a window), define named row conditions (`DEFINE`), describe the shape you want as a regex over those names (`PATTERN`), and project results from each match (`MEASURES`). Problems that otherwise demand a tangle of correlated self-joins or nested `LAG`/`CASE` — "a price dip then recovery," "login then 2+ failures then success," "three rising bars then a reversal" — collapse to a few declarative lines. This skill assumes the set semantics and three-valued logic of the foundation **`sql-relational-and-null-discipline`** (the match runs over an explicit `ORDER BY`; nothing is ordered without it) and the window mechanics of **`sql-window-functions`**. It is the escalation target named by **`sql-gaps-and-islands`** §6 when window tricks get unwieldy.

**Read this first — the portability reality (full detail in §6):** row pattern matching is an **optional** SQL:2016 feature. It is established on Oracle, Trino, Snowflake, Flink, Vertica, and DB2, but it is **absent from the PostgreSQL, MySQL, MariaDB, and SQLite versions most people actually run today**. If your engine does not have it, do not reach for it — use the window-function recipes in **`sql-gaps-and-islands`** instead.

---

## 1. What `MATCH_RECOGNIZE` Is — and the Fixed Clause Order

The clause sits in the `FROM`, wrapping a table. The sub-clauses must appear **in this exact order**, and only `PATTERN` and `DEFINE` are mandatory ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)):

```sql
SELECT ...
FROM some_table
  MATCH_RECOGNIZE (
    [ PARTITION BY col [, ...] ]      -- split into independent streams (like a window partition)
    [ ORDER BY    col [, ...] ]       -- REQUIRED ordering within each partition
    [ MEASURES    expr AS name [, ...] ]  -- columns computed from a match
    [ ONE ROW PER MATCH | ALL ROWS PER MATCH ]  -- output granularity (default ONE ROW)
    [ AFTER MATCH SKIP ... ]          -- where to resume after a match
    PATTERN ( row_pattern )           -- the regex over pattern variables  (MANDATORY)
    [ SUBSET ... ]                    -- union of variables, for MEASURES navigation
    DEFINE  var AS condition [, ...]  -- the boolean test each variable's row must pass (MANDATORY)
  );
```

Map it to a regex you already know:

- **`DEFINE`** is the alphabet: each pattern variable is a boolean predicate a row must satisfy to "be" that symbol (`DOWN AS price < PREV(price)`). A variable used in `PATTERN` but never defined matches *any* row (a useful anchor).
- **`PATTERN`** is the regular expression over those symbols: `START DOWN+ UP+` means "one anchor row, then one-or-more DOWN rows, then one-or-more UP rows."
- **`MEASURES`** projects the result: navigate within the match with `FIRST`/`LAST`/`PREV`/`NEXT` and `var.column` (`LAST(DOWN.price)` = the price at the last DOWN row).
- **`PARTITION BY`/`ORDER BY`** structure the input "similarly to window functions" ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)).

`PREV(price)` reads the prior row's value — pattern variables are evaluated **as the match is being built** (running semantics, §4).

---

## 2. The Worked Example — Detect a V-Shape (Dip and Recovery)

This is the centerpiece. Goal: per customer, find a run where the price **falls for one or more steps, then rises for one or more steps** — a V. First, the way it gets written without `MATCH_RECOGNIZE`.

### BEFORE — window/self-join gymnastics

A genuine "1+ down steps then 1+ up steps, of unbounded length" cannot be expressed by a fixed `LAG(price, k)` chain — the dip length is variable. The portable approximations are ugly: either gaps-and-islands grouping over a per-row `down`/`up` direction flag with a running-sum group key and extra logic to require a *down-run immediately followed by an up-run*, or a correlated self-join that, for every candidate bottom row, scans backward for a monotone fall and forward for a monotone rise:

```sql
-- BEFORE (sketch) — find each "bottom" b, then prove a monotone fall into it and a monotone rise out.
-- Each EXISTS/NOT EXISTS hides a correlated scan; pinning the exact start/end of the V needs more.
-- Real-world versions of this run 60-100 lines and are a magnet for off-by-one and "what about ties" bugs.
SELECT b.customer_id, b.order_date AS bottom_date, b.price AS bottom_price
FROM   orders b
WHERE  EXISTS (SELECT 1 FROM orders d           -- a strictly-lower predecessor exists
               WHERE d.customer_id = b.customer_id AND d.order_date < b.order_date
                 AND d.price > b.price
                 AND NOT EXISTS (SELECT 1 FROM orders m  -- ...with nothing rising in between
                                 WHERE m.customer_id = b.customer_id
                                   AND m.order_date BETWEEN d.order_date AND b.order_date
                                   AND m.price > /* prior row's price */ 0))   -- gets hairy fast
  AND  EXISTS (SELECT 1 FROM orders u WHERE u.customer_id = b.customer_id
               AND u.order_date > b.order_date AND u.price > b.price);
-- ...and you still have not produced start_price, final_price, or the V's start/end dates cleanly.
```

### AFTER — `MATCH_RECOGNIZE`

The whole thing is the regex `START DOWN+ UP+` over two row predicates ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)):

```sql
SELECT customer_id, start_price, bottom_price, final_price, start_date, final_date
FROM orders
  MATCH_RECOGNIZE (
    PARTITION BY customer_id
    ORDER BY order_date
    MEASURES
      START.price          AS start_price,
      LAST(DOWN.price)     AS bottom_price,   -- price at the deepest matched DOWN row
      LAST(UP.price)       AS final_price,
      START.order_date     AS start_date,
      LAST(UP.order_date)  AS final_date
    ONE ROW PER MATCH
    AFTER MATCH SKIP PAST LAST ROW
    PATTERN (START DOWN+ UP+)
    DEFINE
      DOWN AS price < PREV(price),           -- this row is lower than the one before it
      UP   AS price > PREV(price)            -- this row is higher than the one before it
  );
```

For data where cust_1 goes `100, 200, 100, 50, 100` and cust_2 goes `8, 4, 6`, this returns one row per detected V: cust_1's `200 -> 50 -> 100` and cust_2's `8 -> 4 -> 6` ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)). `START` is undefined, so it matches the anchor row; `DOWN+` consumes the fall; `UP+` consumes the recovery. Six lines, one pass, no self-join, and the `MEASURES` hand you the start/bottom/final prices directly. *This* is the payoff in the pull-quote.

---

## 3. `PATTERN` Quantifiers — Greedy vs Reluctant

`PATTERN` is a regex over pattern variables, with the quantifiers you expect ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)):

| Quantifier | Meaning |
|---|---|
| `*` | zero or more |
| `+` | one or more |
| `?` | zero or one |
| `{n}` | exactly n |
| `{n,m}` | n to m |
| `{n,}` | n or more |
| `{,n}` | zero to n |

Concatenation (`A B`), alternation (`A | B`), grouping (`( A B )`), and the empty pattern (`()`) all work as in ordinary regex.

**Quantifiers are greedy by default**: "they prefer higher number of repetitions over lower number. If you want it the other way, you can change a quantifier to reluctant by appending `?` immediately after it" ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)). So `DOWN+` (greedy) extends the dip as far as it can; `DOWN+?` (reluctant) stops at the minimum. The same `?` appended to the optional quantifier flips its preference: "`(pattern)?` prefers a single match of the pattern, while `(pattern)??` would rather omit the pattern altogether" ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)).

```sql
-- Greedy: B+ swallows every consecutive B it can. The match is as LONG as possible.
PATTERN (A B+ C)
-- Reluctant: B+? takes the FEWEST B rows that still let C match. The match is as SHORT as possible.
PATTERN (A B+? C)
```

The trap: the trailing `?` is overloaded. `+?` is "one-or-more, reluctant," **not** "one-or-more, then optional." Reach for reluctant when a greedy quantifier would swallow rows that belong to the *next* match (see common-mistakes §5).

---

## 4. `MEASURES` and RUNNING vs FINAL

Two evaluation times exist, and conflating them is the subtlest bug here ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)):

- **`DEFINE` conditions are evaluated *while the match is in progress*** — "running" semantics. They have to be: a variable's predicate decides whether to extend the match, so it can only see rows matched *so far*. `LAST(DOWN.price)` inside `DEFINE` means "the most recent DOWN row up to the current one," not the eventual last one.
- **`MEASURES` expressions are evaluated *when the match is complete*** — by default these read "final" values for the whole match.

When you ask for one summary row (`ONE ROW PER MATCH`, §5), the distinction is mostly invisible — you get the completed match's values. It becomes load-bearing with `ALL ROWS PER MATCH`, where each matched row emits and you can choose per measure:

```sql
MEASURES
  RUNNING LAST(DOWN.price) AS running_low,   -- the lowest-so-far at THIS output row
  FINAL   LAST(DOWN.price) AS final_low      -- the match's eventual bottom, on every output row
ALL ROWS PER MATCH
```

`RUNNING` is the default in both clauses; `FINAL` applies only in `MEASURES` ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)). Rule of thumb: in `DEFINE` you are *always* running (you cannot see the future); in `MEASURES` with `ALL ROWS PER MATCH`, say `FINAL` explicitly when you want the completed-match value on every row, or you will silently get the running one.

---

## 5. `ONE ROW` vs `ALL ROWS PER MATCH`, and `AFTER MATCH SKIP`

**Output granularity.** `ONE ROW PER MATCH` (the default) emits a **single summary row per match** — use it for "give me each detected V with its start/bottom/end." `ALL ROWS PER MATCH` emits **one row per matched input row**, carrying the running measures — use it when you need to see, and label, every row inside the match (e.g. tag each row with its pattern variable and the running low).

```sql
-- ONE ROW PER MATCH  -> 1 row per V         (summary: start_price, bottom_price, final_price)
-- ALL ROWS PER MATCH -> N rows per V         (every order row in the V, plus running measures)
```

**Where matching resumes — `AFTER MATCH SKIP`.** After a match completes, the engine restarts the search somewhere. "The default option is `AFTER MATCH SKIP PAST LAST ROW`, but you can also skip to the next row or to a specific position in the match based on the matched pattern variables" ([Trino](https://trino.io/blog/2021/05/19/row_pattern_matching.html)).

| Skip mode | Effect |
|---|---|
| `SKIP PAST LAST ROW` (default) | resume after the match's last row -> **non-overlapping** matches |
| `SKIP TO NEXT ROW` | resume at the row after the match's *start* -> matches may **overlap** |
| `SKIP TO [FIRST\|LAST] var` | resume at a matched variable's row -> controlled overlap |

Choose deliberately: the default finds each V once; `SKIP TO NEXT ROW` finds every overlapping V. **Warning:** a `SKIP TO` that lands on the match's own first row is an error in conforming engines — it would loop forever — so do not skip to the anchor variable.

---

## 6. The Low-Portability Reality — and the Portable Fallback

This is the part that decides whether you can use the clause at all. Row pattern matching is an **optional** SQL:2016 feature (feature **R010**) ([modern-sql.com](https://modern-sql.com/feature/match_recognize)) — "optional" means a conforming engine is *not required* to implement it, and most did not for years.

> **PORTABILITY — confirm engine support before recommending `MATCH_RECOGNIZE`.**
>
> | Engine | `MATCH_RECOGNIZE`? |
> |---|---|
> | Oracle (12c R1+), Trino, Snowflake, Flink SQL, Vertica, DB2 (LUW 12.1.4+) | **yes — established** |
> | PostgreSQL | **only 18+** (absent from deployed 16/17) |
> | MySQL | **only 9.7+** (absent from 8.x) |
> | MariaDB | **only 12.3+** |
> | SQLite | **only 3.53+** |
> | SQL Server | **only 2025+** |
>
> Versions per [modern-sql.com — MATCH_RECOGNIZE](https://modern-sql.com/feature/match_recognize). The takeaway: the analytics engines (Oracle, Trino, Snowflake, Flink, Vertica, DB2) have had it for years; the popular OLTP engines got it only in their newest, mostly-not-yet-deployed releases. **Assume your Postgres/MySQL/MariaDB/SQLite does not have it unless you have checked the version.**

**When the engine lacks it, use the portable window-function fallback in [`sql-gaps-and-islands`](../sql-gaps-and-islands/SKILL.md).** "Consecutive runs" and "gap over a threshold" map cleanly onto the `ROW_NUMBER`-difference and running-sum-of-a-flag recipes there, which use only standard SQL:2003 window functions and run on PostgreSQL 8.4+, MySQL 8, SQLite 3.25+, everywhere. Those recipes get unwieldy precisely for the rich multi-state patterns `MATCH_RECOGNIZE` was built for (its own §6 routes *here* for that escalation) — which is the whole tension: the clean tool is not portable, and the portable tool is not always clean. **The route matrix and per-engine version detail are owned by [`sql-standard-vs-dialect-map`](../sql-standard-vs-dialect-map/SKILL.md).**

---

## 7. Who Suffers When This Is Done by Hand (or Reached For Blindly)

- The **time-series engineer** who wrote 80 lines of fragile correlated self-joins to detect a price dip-and-recovery — three passes, an off-by-one at every boundary row, and *still* no clean way to emit the start/bottom/final prices — when the engine running the warehouse (Trino/Snowflake/Oracle) supports the 6-line `PATTERN (START DOWN+ UP+)` in §2. The hand-rolled version is the bug they are now debugging under a deadline.
- The **analyst** who copied a slick `MATCH_RECOGNIZE` query from a blog into their **PostgreSQL 16** instance and got a syntax error, because the feature landed only in PG 18 (§6). They needed to be routed to the gaps-and-islands window fallback *before* writing a line — confirming engine support is step zero, not a debugging afterthought.
- The **next maintainer** handed a greedy `DOWN+` pattern that silently swallowed the first row of the next dip, so two adjacent Vs were reported as one. No error — just a metric that was quietly wrong. A reluctant `DOWN+?` or the right `AFTER MATCH SKIP` was the fix (§3, §5).

The check you ran ("does this engine have `MATCH_RECOGNIZE`?") and the reluctant quantifier you reached for are gifts to whoever audits the result under load.

---

## 8. Routing to Related Skills

- **`sql-relational-and-null-discipline`** — the foundation: a result is an unordered set, so the match runs over an explicit `ORDER BY`; three-valued logic governs the `DEFINE` predicates (a `NULL` comparison is UNKNOWN, so the row does not match that variable).
- **`sql-gaps-and-islands`** — the **portable fallback** (§6): `ROW_NUMBER`-difference and running-sum-of-a-flag window recipes for consecutive runs and threshold sessionization when your engine lacks `MATCH_RECOGNIZE`. It routes back here when the pattern outgrows those tricks.
- **`sql-window-functions`** — the window mechanics `MATCH_RECOGNIZE` extends: `PARTITION BY`/`ORDER BY`, `LAG`/`LEAD` (the portable cousins of `PREV`/`NEXT`), and frames.
- **`sql-standard-vs-dialect-map`** — the authoritative per-engine support matrix and version thresholds (§6), plus dialect quirks in pattern/measure spelling.

---

## 9. Reference Files

High-frequency `MATCH_RECOGNIZE` anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
