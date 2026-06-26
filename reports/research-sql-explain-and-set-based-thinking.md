# Research: sql-explain-and-set-based-thinking

Research for skill #26 in `reports/skill-plan-sql.md`. Theme: measure don't guess (read
the plan), think in sets not rows (kill N+1/RBAR), and test at realistic scale. Every
quote below is verbatim from a primary source, with the URL, the section, and why it
matters for the skill.

Accessed: 2026-06-26.

---

## Source 1 — PostgreSQL: Using EXPLAIN

URL: https://www.postgresql.org/docs/current/using-explain.html

### 1a. The plan is a tree of nodes; leaves are scans
Section 14.1.1 (EXPLAIN Basics):
> "The structure of a query plan is a tree of _plan nodes_. Nodes at the bottom level of the tree are scan nodes: they return raw rows from a table. There are different types of scan nodes for different table access methods: sequential scans, index scans, and bitmap index scans."

> "The output of `EXPLAIN` has one line for each node in the plan tree, showing the basic node type plus the cost estimates that the planner made for the execution of that plan node."

**Why it matters:** Grounds the "EXPLAIN as a concept" section — a plan is a readable tree, leaves are the access method. This is the universal mental model regardless of engine.

### 1b. Why a sequential scan vs an index scan is chosen
> "Since this query has no `WHERE` clause, it must scan all the rows of the table, so the planner has chosen to use a simple sequential scan plan."

> "In this type of plan the table rows are fetched in index order... You'll most often see this plan type for queries that fetch just a single row."

Section 14.1.3 (Caveats):
> "An extreme example is that on a table that only occupies one disk page, you'll nearly always get a sequential scan plan whether indexes are available or not. The planner realizes that it's going to take one disk page read to process the table in any case, so there's no value in expending additional page reads to look at an index."

**Why it matters:** Scan vs index access is the first thing to read in a plan. The one-disk-page caveat is also the bridge to "test at realistic scale" — a scan is correct on a tiny table, which is exactly why dev-sized tables lie.

### 1c. The cost numbers — estimated cost and estimated rows
> "The numbers that are quoted in parentheses are (left to right): Estimated start-up cost... Estimated total cost... Estimated number of rows output by this plan node... Estimated average width of rows output by this plan node (in bytes)."

> "The costs are measured in arbitrary units determined by the planner's cost parameters."

> "It's important to understand that the cost of an upper-level node includes the cost of all its child nodes."

> "The `rows` value is a little tricky because it is not the number of rows processed or scanned by the plan node, but rather the number emitted by the node."

**Why it matters:** Cost is in arbitrary units (not ms) and is cumulative up the tree; `rows` is the *emitted* (post-filter) estimate. Needed so the skill explains what the planner's numbers actually mean before contrasting with ANALYZE actuals.

### 1d. EXPLAIN ANALYZE = actual rows + actual time
Section 14.1.2 (EXPLAIN ANALYZE):
> "It is possible to check the accuracy of the planner's estimates by using `EXPLAIN`'s `ANALYZE` option. With this option, `EXPLAIN` actually executes the query, and then displays the true row counts and true run time accumulated within each plan node, along with the same estimates that a plain `EXPLAIN` shows."

> "Note that the "actual time" values are in milliseconds of real time, whereas the `cost` estimates are expressed in arbitrary units; so they are unlikely to match up."

> "In such cases, the `loops` value reports the total number of executions of the node, and the actual time and rows values shown are averages per-execution."

**Why it matters:** Plain EXPLAIN guesses; ANALYZE runs it and shows truth. The `loops` count is the in-plan signature of a nested-loop / per-row repetition — directly relevant to N+1 thinking.

### 1e. THE TELL — estimated-vs-actual rows gap
Section 14.1.2:
> "The thing that's usually most important to look for is whether the estimated row counts are reasonably close to reality. In this example the estimates were all dead-on, but that's quite unusual in practice."

Section 14.1.1 (on where the estimate comes from):
> "it can change after each `ANALYZE` command, because the statistics produced by `ANALYZE` are taken from a randomized sample of the table."

**Why it matters:** This is the centerpiece diagnostic of the EXPLAIN section. A large gap between estimated and actual rows = the planner was misled (often stale/sampled statistics), which produces a bad plan. The stats provenance quote explains *why* the estimate drifts.

### 1f. Caveat — don't extrapolate a toy-table plan
Section 14.1.3:
> "`EXPLAIN` results should not be extrapolated to situations much different from the one you are actually testing; for example, results on a toy-sized table cannot be assumed to apply to large tables. The planner's cost estimates are not linear and so it might choose a different plan for a larger or smaller table."

**Why it matters:** Primary-source backing for "test at realistic scale" — the plan itself changes with volume, so a plan read on dev data is not the production plan.

---

## Source 2 — SQLite: The EXPLAIN QUERY PLAN Command

URL: https://www.sqlite.org/eqp.html

### 2a. What EXPLAIN QUERY PLAN does
> "The EXPLAIN QUERY PLAN SQL command is used to obtain a high-level description of the strategy or plan that SQLite uses to implement a specific SQL query. Most significantly, EXPLAIN QUERY PLAN reports on the way in which the query uses database indices."

**Why it matters:** Different engine, same concept — a readable, high-level plan, focused on index use. Anchors the portability block (concepts universal, spelling per-engine).

### 2b. SCAN vs SEARCH
> "SCAN" is used for a full-table scan, including cases where SQLite iterates through all records in a table in an order defined by an index. "SEARCH" indicates that only a subset of the table rows are visited."

**Why it matters:** SQLite's plain-English vocabulary makes the scan-vs-seek distinction obvious: SCAN = visit everything, SEARCH = visit a subset. Perfect contrast pair for the scan-vs-index-access section.

### 2c. USING INDEX / USING COVERING INDEX
> "If the query were able to use an index, then the SCAN/SEARCH record would include the name of the index and, for a SEARCH record, an indication of how the subset of rows visited is identified."

Examples from the doc:
```
`--SEARCH t1 USING INDEX i1 (a=?)
`--SEARCH t1 USING COVERING INDEX i2 (a=?)
```

**Why it matters:** Shows the plan naming the index it used (or didn't). A bare `SCAN t1` on a big table in the plan is the readable signal that no index applied.

### 2d. Plan is a readable tree
> "A query plan is represented as a tree... The command-line shell will usually intercept this table and renders it as an ASCII-art graph for more convenient viewing."

**Why it matters:** Reinforces "plan is a tree" cross-engine, and that SQLite's output is deliberately human-readable plain language.

---

## Source 3 — Use The Index, Luke: Testing Scalability

URL: https://use-the-index-luke.com/sql/testing-scalability  (detail at /data-volume)

### 3a. Fast in test ≠ fast in production
> "If a query is fast enough under certain testing conditions, it does not mean it will be fast enough in production. That is especially the case in development environments that have only a fraction of the data of the production system."

**Why it matters:** The thesis sentence for "test at realistic scale." Directly the "fast in dev" failure in the Who-suffers section.

### 3b. The two queries diverge wildly as volume grows
> "On the right hand side of the chart, when the data volume is a hundred times as high, the faster query needs more than twice as long as it originally did while the response time of the slower query increased by a factor of 20 to more than one second."

> "The chart shows a growing response time for both indexes."

**Why it matters:** Quantifies the divergence — a query 100x the data can blow up 20x while the good one barely moves. Two queries that look identical on 100 rows are not the same query at 10M. Backs the "measure on realistic volume" + scan-vs-index point.

---

## Source 4 — MySQL: EXPLAIN Output Format (portability block only)

URL: https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

> "The `EXPLAIN` statement provides information about how MySQL executes statements."

> (on the `rows` column, InnoDB) "this number is an estimate, and may not always be exact."

**Why it matters:** Third engine for the portability table — MySQL spells it `EXPLAIN` / `EXPLAIN FORMAT=JSON` / `EXPLAIN ANALYZE`, and like Postgres its `rows` is an estimate. Confirms concepts are universal, output format is per-engine.

---

## Source 5 — Bill Karwin, *SQL Antipatterns* (the balance + RBAR)

URL: https://pragprog.com/titles/bksqla/sql-antipatterns/

Used for two named antipatterns (book, cited by name — already used by the foundation
skill in this plugin):
- **"Spaghetti Query"** — cramming an entire problem into one giant unreadable query.
  The counterweight to set-based zeal: set-based does NOT mean one monstrous query.
  Split into intermediate steps (temp tables / CTEs) when a single query becomes
  unreadable or the optimizer mis-plans it.
- The N+1 / **RBAR** ("row-by-agonizing-row") shape — application code issuing one query
  per row in a loop instead of one set-based query — is the canonical performance
  antipattern this skill centers on.

**Why it matters:** Provides the "don't overcorrect" balance section and names the RBAR
antipattern with an authoritative citation, matching the foundation skill's use of Karwin.

---

## Synthesis — what the skill must nail

1. **MEASURE, don't guess.** Read the plan (EXPLAIN); use ANALYZE/QUERY PLAN for actual rows/time. (1a–1d, 2a)
2. **Scan vs seek/index access** — SEQ SCAN / `SCAN t` = visit everything; INDEX SCAN / `SEARCH ... USING INDEX` = visit a subset. (1b, 2b, 2c)
3. **Cardinality/selectivity & the estimated-vs-actual-rows gap** is the tell of a bad plan, usually stale/sampled stats. (1c, 1e)
4. **Join order** follows from cardinality — the planner orders joins by estimated row counts; a bad estimate → bad join order. (1c, 1e)
5. **N+1 / RBAR is the centerpiece**: app loop issuing one query per row → rewrite as ONE set-based query (JOIN / IN / VALUES / LATERAL). The `loops` count in ANALYZE is its in-plan signature. (1d, Source 5)
6. **Spaghetti-Query balance**: set-based ≠ one giant query; split when unreadable. (Source 5)
7. **Test at realistic scale**: the plan and the response time both change with volume; toy tables lie. (1f, 3a, 3b)
8. **Portability**: EXPLAIN is non-standard, per-engine output. Postgres `EXPLAIN [ANALYZE]`, SQLite `EXPLAIN QUERY PLAN`, MySQL `EXPLAIN [FORMAT=JSON|ANALYZE]`. Concepts universal; format → vendor plugins / dialect-map. (2a, Source 4)
</content>
</invoke>
