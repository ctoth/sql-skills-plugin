# Research: sql-match-recognize (SQL:2016 Row Pattern Recognition)

Research for skill #29 (ADVANCED, LOW PORTABILITY). Each entry: URL, section, verbatim quote, why it matters.

---

## Source 1 — Trino blog: "Row pattern matching with MATCH_RECOGNIZE" (2021-05-19)

URL: https://trino.io/blog/2021/05/19/row_pattern_matching.html
Accessed: 2026-06-26

### 1a. Full clause structure and order

Section: syntax overview. Verbatim structure:

```
MATCH_RECOGNIZE (
  [ PARTITION BY column [, ...] ]
  [ ORDER BY column [, ...] ]
  [ MEASURES measure_definition [, ...] ]
  [ rows_per_match ]
  [ AFTER MATCH skip_to ]
  PATTERN ( row_pattern )
  [ SUBSET subset_definition [, ...] ]
  DEFINE variable_definition [, ...]
)
```

Notes: `PATTERN` and `DEFINE` are mandatory; all others optional. `PARTITION BY` / `ORDER BY` structure the input like window functions. `ONE ROW PER MATCH` (default) vs `ALL ROWS PER MATCH` control output granularity.

Why it matters: this is the canonical clause skeleton and the *order in which clauses must be written* — the centerpiece structure the skill teaches. The clause-order is fixed and non-obvious (MEASURES before PATTERN; DEFINE last).

### 1b. Worked example — V-shape / price dip-and-recovery (CENTERPIECE)

Section: worked example. Verbatim SQL:

```sql
WITH orders(customer_id, order_date, price) AS (VALUES
    ('cust_1', DATE '2020-05-11', 100),
    ('cust_1', DATE '2020-05-12', 200),
    ('cust_2', DATE '2020-05-13', 8),
    ('cust_1', DATE '2020-05-14', 100),
    ('cust_2', DATE '2020-05-15', 4),
    ('cust_1', DATE '2020-05-16', 50),
    ('cust_1', DATE '2020-05-17', 100),
    ('cust_2', DATE '2020-05-18', 6))
SELECT customer_id, start_price, bottom_price, final_price,
       start_date, final_date
    FROM orders
        MATCH_RECOGNIZE (
            PARTITION BY customer_id
            ORDER BY order_date
            MEASURES
                START.price AS start_price,
                LAST(DOWN.price) AS bottom_price,
                LAST(UP.price) AS final_price,
                START.order_date AS start_date,
                LAST(UP.order_date) AS final_date
            ONE ROW PER MATCH
            AFTER MATCH SKIP PAST LAST ROW
            PATTERN (START DOWN+ UP+)
            DEFINE
                DOWN AS price < PREV(price),
                UP AS price > PREV(price)
            );
```

Result: 2 matches. cust_1 dips 200 -> 50 -> 100; cust_2 dips 8 -> 4 -> 6.

Why it matters: this is the exact centerpiece example the spec asks for (V-shape / dip-and-recovery). It exercises every clause: PARTITION/ORDER, MEASURES with navigation (START.price, LAST(DOWN.price)), pattern variables, the `START DOWN+ UP+` regex, and the DEFINE conditions using PREV(). `START` is an undefined variable (always matches one row) used as an anchor.

### 1c. PATTERN quantifiers; greedy vs reluctant

Section: quantifiers. Verbatim:

> "Quantifiers are greedy by default. It means that they prefer higher number of repetitions over lower number. If you want it the other way, you can change a quantifier to reluctant by appending `?` immediately after it."

> "`(pattern)?` prefers a single match of the pattern, while `(pattern)??` would rather omit the pattern altogether."

Supported quantifiers: `*` (zero or more), `+` (one or more), `?` (zero or one), `{n}` (exactly n), `{n,m}` (n to m), `{n,}` (n or more), `{,n}` (zero to n).

Why it matters: greedy-vs-reluctant is a required topic (c). The `?` suffix overloads two meanings (the `?` quantifier vs the reluctant `?` suffix) — `+?` is a reluctant one-or-more. This is exactly the kind of footgun the common-mistakes file should flag.

### 1d. RUNNING vs FINAL semantics

Section: running/final. Verbatim:

> "The expressions in the DEFINE clause are evaluated when the pattern matching is in progress." (running semantics)

> "The expressions of the MEASURES clause are evaluated when the match is complete." (final semantics)

With `ALL ROWS PER MATCH` you can explicitly write `RUNNING LAST(DOWN.price)` or `FINAL LAST(DOWN.price)`. RUNNING is the default in both clauses; FINAL only applies to MEASURES. DEFINE conditions are always evaluated in RUNNING semantics (you cannot see rows that have not yet matched).

Why it matters: required topic (d). The key teachable: DEFINE is necessarily RUNNING (you're mid-match); MEASURES defaults to RUNNING per-row in ALL ROWS PER MATCH but you usually want FINAL there. With ONE ROW PER MATCH the distinction is moot because only the final summary row emits.

### 1e. AFTER MATCH SKIP options

Section: after match skip. Verbatim:

> "The default option is `AFTER MATCH SKIP PAST LAST ROW`, but you can also skip to the next row or to a specific position in the match based on the matched pattern variables."

Options: `SKIP PAST LAST ROW` (default; resume after final matched row -> non-overlapping matches), `SKIP TO NEXT ROW`, and skip to a variable position (`SKIP TO [FIRST|LAST] variable`).

Why it matters: required topic (e). Controls overlapping vs non-overlapping matches — the difference between "find every V" and "find every overlapping V." Skip-to-variable can cause infinite loops if it skips to the match start; engines error on that.

### 1f. Why it beats self-joins / window functions

Section: motivation. Verbatim:

> "From the SQL viewpoint, you can think of row pattern matching as extended window functions."

> "Before the introduction of MATCH_RECOGNIZE, you had to feed your data to external tools to reason about trends and patterns. Now, you can achieve it directly in your query."

Why it matters: required topic (f) — the value proposition. Pattern detection that previously meant exporting to external tools or chaining window functions + self-joins is now a single declarative clause.

---

## Source 2 — modern-sql.com: "MATCH_RECOGNIZE"

URL: https://modern-sql.com/feature/match_recognize
Accessed: 2026-06-26

### 2a. Standard status / feature IDs

Section: standard status. Row pattern matching introduced in **SQL:2016** as an **optional** feature, three components:

- **R010** (basis): the `match_recognize` clause with aggregates min, max, sum, count, avg.
- **R020**: patterns in the `over` clause for frame definition.
- **R030**: all aggregate functions for row patterns (e.g. stddev_pop).

Why it matters: required fact (e) — SQL:2016, optional feature R010. "Optional" is exactly why portability is low: conforming engines are not required to implement it.

### 2b. Engine support matrix (verbatim table)

| Database | Version | Support |
|---|---|---|
| BigQuery | 2026-06-01 | from clause |
| Db2 (LUW) | 12.1.4 | from clause |
| DuckDB | 1.5.0 | from clause |
| H2 | 2.4.240 | from clause |
| MariaDB | 12.3.2 | from clause |
| MySQL | 9.7.0 | from clause |
| Oracle DB | 23.26.2 | from clause |
| PostgreSQL | 18 | from clause |
| SQL Server | 2025 | from clause |
| SQLite | 3.53.0 | from clause |

Why it matters: this is the engine-support reality. Note: modern-sql tracks the *bleeding edge*. The spec's framing (Oracle/Trino/Snowflake/Flink/Vertica/DB2 = the established implementations; Postgres/SQLite/MySQL/MariaDB historically lacked it) reflects the *deployed* reality — support only very recently landed in PG 18 / MySQL 9.7 / MariaDB 12.3 / SQLite 3.53 / SQL Server 2025, so for any currently-deployed PG 16/17, MySQL 8, SQLite < 3.53, the feature is absent. Trino, Snowflake, Flink, Vertica (not in this table because it tracks specific products) are the long-standing analytics implementations. The skill must present BOTH: the established set (Oracle, Trino, Snowflake, Flink, Vertica, DB2) and the caveat that "your deployed Postgres/MySQL almost certainly does not have it."

### 2c. Use cases / what it does

Verbatim — useful to implement:

1. "Finding series of consecutive events"
2. "Pattern matching: trend reversal, periodic events, …"
3. "Top-N per Group"

And: it uses "simple regular expressions across multiple rows to filter them."

Why it matters: frames the use cases and the one-line mental model (regex over ordered rows).

---

## Source 3 — Datenbank-Spektrum paper (PAYWALLED)

URL: https://link.springer.com/article/10.1007/s13222-022-00404-3
DOI: 10.1007/s13222-022-00404-3
Accessed: 2026-06-26

Status: PAYWALLED. The Springer link 303-redirects to an IdP authorization wall (`idp.springer.com/authorize`); the abstract was not retrievable via WebFetch, and doi.org likewise redirects into the same wall. No verbatim abstract captured.

Bibliographic context (from the citation / journal): Datenbank-Spektrum (the German DB systems journal), 2022, an overview of row pattern recognition / complex event recognition in SQL:2016. The spec describes it as covering "semantics, greedy/reluctant." Because the text was inaccessible, the skill cites it only as the academic-overview backstop for the semantics it does NOT take verbatim from it; all concrete semantic claims (greedy default, reluctant `?`, running vs final) are sourced to the Trino blog (Source 1), which states them explicitly. The paper is listed in sources.yaml with a `note:` recording the paywall so no claim is falsely attributed to it.

Why it matters: honesty about provenance. Greedy/reluctant and running/final are load-bearing claims; they are cited to Trino (1c, 1d) which states them in plain language, not to the inaccessible paper.

---

## Synthesis for the skill

- **Mental model:** MATCH_RECOGNIZE = a regular expression over a partitioned, ordered stream of rows. PATTERN is the regex; pattern variables (DEFINE) are the "character classes" (boolean row predicates); MEASURES projects results from the matched span.
- **Clause order (fixed):** PARTITION BY -> ORDER BY -> MEASURES -> [ONE ROW|ALL ROWS] PER MATCH -> AFTER MATCH SKIP -> PATTERN -> [SUBSET] -> DEFINE. PATTERN + DEFINE mandatory.
- **Centerpiece:** the Trino V-shape `PATTERN (START DOWN+ UP+)` with `DOWN AS price < PREV(price)`, `UP AS price > PREV(price)`. BEFORE = window/self-join gymnastics (nested LAG/CASE or correlated self-joins to detect a multi-row dip); AFTER = the 6-line declarative clause.
- **Greedy vs reluctant (c):** quantifiers greedy by default; append `?` to make reluctant (`DOWN+?`). Greedy `DOWN+` extends the dip as far as possible; reluctant stops early.
- **Running vs final (d):** DEFINE always RUNNING; MEASURES defaults RUNNING but `FINAL` available (matters mainly with ALL ROWS PER MATCH); ONE ROW PER MATCH emits one summary row so FINAL is the natural reading.
- **ONE ROW vs ALL ROWS + SKIP (e):** ONE ROW PER MATCH = one summary row per match; ALL ROWS PER MATCH = one output row per matched input row (with per-row running measures). AFTER MATCH SKIP PAST LAST ROW (default, non-overlapping) vs SKIP TO NEXT ROW / SKIP TO variable (overlapping).
- **Portability (LOW):** SQL:2016 optional R010. Established: Oracle, Trino, Snowflake, Flink, Vertica, DB2. Only very recently in PG 18 / MySQL 9.7 / MariaDB 12.3 / SQLite 3.53 / SQL Server 2025 — so deployed PG 16/17, MySQL 8, MariaDB < 12.3, SQLite < 3.53 do NOT have it. Portable fallback = sql-gaps-and-islands (window functions). Route engine matrix to sql-standard-vs-dialect-map.
- **Who suffers:** the time-series engineer who hand-wrote 80 lines of fragile self-joins for a pattern MATCH_RECOGNIZE expresses in 6; the analyst whose engine lacks it and who needs the gaps-and-islands fallback.
