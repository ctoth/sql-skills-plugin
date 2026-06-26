---
name: sql-explain-and-set-based-thinking
description: Guides the two performance habits that matter most — think in sets, not rows, and measure the plan instead of guessing. Replaces the N+1 / RBAR ("row-by-agonizing-row") antipattern — application code looping to issue one query per row, or a correlated per-row subquery — with a single set-based query (`JOIN`/`IN`/`VALUES`/`LATERAL`). Teaches reading a query plan as a concept — the plan tree, sequential/full scan vs index access (seek), estimated cost and estimated rows — and using `EXPLAIN ANALYZE` (Postgres) / `EXPLAIN QUERY PLAN` (SQLite) to get actual rows and time, where a large gap between estimated and actual rows is the tell of a bad estimate (usually stale statistics) that drives a bad join order. Warns that a plan fine on 100 dev rows can die at 10M, so you must test at realistic scale, and that set-based does not mean one giant unreadable "Spaghetti Query." Auto-invokes when writing or editing per-row query loops in application code, a correlated per-row subquery, `EXPLAIN`/`EXPLAIN QUERY PLAN` output, or on "why is this slow" / "optimize this query" / "N+1" requests. Routes plan reading and set-thinking for the whole plugin.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL EXPLAIN and Set-Based Thinking

> "The structure of a query plan is a tree of _plan nodes_. Nodes at the bottom level of the tree are scan nodes: they return raw rows from a table."
> — [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

> "If a query is fast enough under certain testing conditions, it does not mean it will be fast enough in production. That is especially the case in development environments that have only a fraction of the data of the production system."
> — [Use The Index, Luke — Testing Scalability](https://use-the-index-luke.com/sql/testing-scalability)

SQL is a *declarative, set-oriented* language: you describe the set of rows you want and let the engine figure out how. Two reflexes follow from that and decide almost all query performance. First, **think in sets, not rows** — one statement that operates on the whole set beats a loop that touches one row at a time. Second, **measure, don't guess** — read the plan the engine actually chose (`EXPLAIN`), and check it against reality (`EXPLAIN ANALYZE`), rather than reasoning about speed in your head. This skill builds on the set semantics established in `sql-relational-and-null-discipline` (a result is a set) and is the plan-reading and set-thinking root the rest of the plugin routes back to.

---

## 1. Set-Based vs Row-by-Row — the Mental Model

A relational table is a *set* of rows (`sql-relational-and-null-discipline`), and SQL is built to transform whole sets in one operation. The opposite habit — fetching a set, then looping in the application and issuing another query per element — is **RBAR**, "row-by-agonizing-row." It works on a handful of rows and collapses on real data, because the cost is *per row* plus a network round-trip each time, instead of one engine-side set operation.

The shift is: stop asking "for each row, what do I do?" and start asking "what is the set I want, expressed as one query?" Joins, `IN`, `GROUP BY`, `VALUES`, and `LATERAL` are how you say that. The engine then has the whole problem at once and can pick scans, indexes, and join order to serve it — none of which it can do when you hand it a thousand tiny queries.

---

## 2. The N+1 / RBAR Antipattern → Set-Based Rewrite (the centerpiece)

This is the single highest-leverage performance fix in application SQL. The "N+1" name: 1 query to fetch N parent rows, then N more queries (one per parent) to fetch each child — N+1 round-trips where 1 or 2 would do. ORMs emit it silently through lazy-loaded associations.

```python
# WRONG — N+1: one query for authors, then one query PER author inside the loop
authors = db.query("SELECT id, name FROM authors")          # 1 query
for a in authors:                                            # N iterations
    books = db.query(                                        # +N queries
        "SELECT title FROM books WHERE author_id = ?", a.id)
    render(a, books)
# 1 author page -> 1 + 5000 queries. Each "fast." The page still falls over.
```

The set-based rewrite asks for the whole set once and lets the engine join:

```sql
-- RIGHT — one set-based query; the engine joins authors to books in a single pass
SELECT a.id, a.name, b.title
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
ORDER BY a.id;
```

The same antipattern shows up *inside* SQL as a correlated per-row subquery in the `SELECT` list — logically one lookup per output row:

```sql
-- WRONG — scalar subquery re-evaluated once per author row (row-by-row in disguise)
SELECT a.name,
       (SELECT COUNT(*) FROM books b WHERE b.author_id = a.id) AS book_count
FROM authors a;

-- RIGHT — one set-based aggregate over the join
SELECT a.name, COUNT(b.id) AS book_count
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
GROUP BY a.id, a.name;
```

When you need "the top-N children per parent" rather than a flat join, the set-based tool is a `LATERAL` join — owned by `sql-lateral-and-correlated-derived`, route there. Bulk-insert lists with one `VALUES` / `INSERT ... SELECT` instead of a loop of single-row inserts; collapse "does it exist in this list" loops to a single `IN` / `EXISTS` (`sql-subqueries-and-exists`).

---

## 3. EXPLAIN as a Concept — the Plan Tree

`EXPLAIN` shows the strategy the engine chose, without running the query. "The structure of a query plan is a tree of _plan nodes_. Nodes at the bottom level of the tree are scan nodes: they return raw rows from a table" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). SQLite states the same purpose plainly: it gives "a high-level description of the strategy or plan that SQLite uses to implement a specific SQL query. Most significantly, EXPLAIN QUERY PLAN reports on the way in which the query uses database indices" ([SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)).

Read the tree bottom-up: leaves are how each table is accessed, parent nodes are joins/sorts/aggregations combining them. The first question at each leaf is the access method.

```text
-- A two-table plan, read bottom-up: two scans feed a join, which feeds a sort
Sort                              (sorts the joined result)
  -> Hash Join                    (combines the two inputs on the join key)
       -> Seq Scan on orders      (leaf: how 'orders' is read)
       -> Index Scan on customers (leaf: how 'customers' is read)
```

---

## 4. Scan vs Seek — Reading the Access Method

The leaf node tells you whether the engine visits the whole table or jumps to a subset. SQLite names it in plain English: `SCAN` "is used for a full-table scan," while `SEARCH` "indicates that only a subset of the table rows are visited" ([SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)). Postgres calls the same split a **Seq Scan** (read every row) versus an **Index Scan** (fetch rows via an index).

```text
-- WRONG signal — a full scan of a large table for a selective lookup
-- SQLite:    SCAN orders
-- Postgres:  Seq Scan on orders  (Filter: customer_id = 42)

-- RIGHT signal — a seek that visits only matching rows via an index
-- SQLite:    SEARCH orders USING INDEX idx_orders_customer (customer_id=?)
-- Postgres:  Index Scan using idx_orders_customer on orders
```

A full scan is not always wrong — on a tiny table it is optimal: "on a table that only occupies one disk page, you'll nearly always get a sequential scan plan whether indexes are available or not" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). It is wrong when a *selective* predicate scans a *large* table. *Which* index to create to turn a scan into a seek is owned by `sql-indexing-and-sargability`; this skill only teaches reading the access method.

---

## 5. Cost and Rows — What the Planner's Numbers Mean

Each node carries estimates. The parenthesized numbers are "Estimated start-up cost... Estimated total cost... Estimated number of rows output by this plan node... Estimated average width of rows" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). Two things trip people up: cost is in "arbitrary units" (not milliseconds), and "the cost of an upper-level node includes the cost of all its child nodes" — so the top node's total is the whole-query estimate. The `rows` figure "is not the number of rows processed or scanned by the plan node, but rather the number emitted by the node" — i.e. after `WHERE` filtering.

That estimated row count drives everything downstream — chiefly **join order**. The planner orders joins by how many rows it thinks each step produces (start from the smallest set, keep intermediate results small). **Selectivity** (what fraction a predicate keeps — `status = 'rare'` is highly selective, `status IS NOT NULL` usually is not) and **cardinality** (how many distinct values / rows) are how it guesses those counts. A highly selective predicate emits few rows, so the planner can seek and join that small set first; a non-selective one emits most of the table, favoring a scan. Get the count wrong and the join order — and the whole plan — goes wrong, because every choice above that leaf was made for the wrong number of rows.

---

## 6. EXPLAIN ANALYZE and the Estimated-vs-Actual Gap (the bad-plan tell)

Plain `EXPLAIN` only guesses. To see the truth, run it: with `ANALYZE`, "`EXPLAIN` actually executes the query, and then displays the true row counts and true run time accumulated within each plan node, along with the same estimates that a plain `EXPLAIN` shows" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). Note actual time is in milliseconds while cost is in arbitrary units, "so they are unlikely to match up."

The most important thing to read is the gap between estimate and reality: "The thing that's usually most important to look for is whether the estimated row counts are reasonably close to reality... that's quite unusual in practice" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)).

```text
-- THE TELL — estimate off by ~1000x: the planner was misled
Seq Scan on orders  (cost=0.00..18334.00 rows=12 width=44)
                    (actual time=0.014..240.3 rows=11873 loops=1)
--                                ^estimate 12      ^reality 11873
```

A large estimated-vs-actual divergence means the planner optimized for the wrong row count, so it likely picked the wrong access method and join order. The usual cause is **stale statistics** — the estimate is sampled and "can change after each `ANALYZE` command, because the statistics produced by `ANALYZE` are taken from a randomized sample of the table" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). Refresh stats (`ANALYZE` the table) before blaming the query. Also watch `loops=N` on an inner node: that node ran N times — the in-plan fingerprint of a per-row repetition (§2).

---

## 7. Test at Realistic Scale — Toy Data Lies

A plan read on dev data is not the production plan. "`EXPLAIN` results should not be extrapolated to situations much different from the one you are actually testing; for example, results on a toy-sized table cannot be assumed to apply to large tables. The planner's cost estimates are not linear and so it might choose a different plan for a larger or smaller table" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). The engine that picks a fine seq scan on 100 rows picks a very different plan at 10M.

The response time diverges, too: as data grows "a hundred times as high, the faster query needs more than twice as long as it originally did while the response time of the slower query increased by a factor of 20 to more than one second" ([Use The Index, Luke — Testing Scalability](https://use-the-index-luke.com/sql/testing-scalability)). Two queries that look equally fast on 100 rows are not the same query at scale. Measure on production-representative volume — or at least understand how the plan changes as the table grows.

---

## 8. Set-Based ≠ One Giant Query (the Spaghetti-Query balance)

Thinking in sets is not a mandate to cram the whole problem into one monstrous statement. The opposite overcorrection is Karwin's **"Spaghetti Query"** antipattern ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)): a single unreadable query that fans out joins/aggregates until both humans and the optimizer lose the plot, often producing accidental cross-products or a plan the planner mis-estimates.

The balance: replace *row-by-row loops* with set operations (§2), but split a genuinely complex problem into a few readable, well-named steps — CTEs (`sql-cte-and-recursion`) or intermediate/temp tables — each of which the optimizer can plan well. Fewer round-trips, not fewer-but-incomprehensible queries. If a single query needs a paragraph of comments to follow, decompose it.

---

## 9. Portability — Concepts Universal, Plan Format Per-Engine

`EXPLAIN` is **not** standard SQL, and its output format and options differ by engine. What is universal is the *conceptual* plan: an access method (scan vs seek), estimated rows/cost, join order, and actual-vs-estimated comparison.

| Concept | PostgreSQL | SQLite | MySQL |
|---|---|---|---|
| Show the plan (estimates) | `EXPLAIN` | `EXPLAIN QUERY PLAN` | `EXPLAIN` / `EXPLAIN FORMAT=JSON` |
| Run + actual rows/time | `EXPLAIN ANALYZE` | (run query; check timing) | `EXPLAIN ANALYZE` (8.0.18+) |
| Full scan / seek vocabulary | `Seq Scan` / `Index Scan` | `SCAN` / `SEARCH ... USING INDEX` | `type: ALL` / `ref`,`range`,`eq_ref` |
| `rows` column | estimate (emitted) | (n/a in EQP) | estimate; for InnoDB "may not always be exact" |

SQLite's plan is deliberately readable plain language ([SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)); MySQL's `EXPLAIN` "provides information about how MySQL executes statements" and `FORMAT=JSON` gives a structured tree ([MySQL — EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)). For exact per-engine plan format, node types, buffers/InnoDB internals → vendor plugins; for the spelling map → `sql-standard-vs-dialect-map`.

---

## 10. Who Suffers When You Guess Instead of Measure

A bad plan never errors — it returns the right answer too slowly, and someone downstream pays:

- The **API team** whose "render one page" endpoint fell over in production because the ORM silently issued 5000 lazy-load queries (§2). Each query was 1 ms and "fine"; 5000 round-trips under load was not. No SQL looked wrong — there was no slow query, just thousands of fast ones.
- The **engineer** whose query was instant in dev and timed out on real data, because the seq scan that was optimal on 100 rows scanned 40 million in production (§4, §7). The plan they never ran on realistic volume was the plan that mattered.
- The **on-call DBA** staring at a plan with `rows=12` estimated and `rows=2,000,000` actual (§6) — a stale-statistics estimate that picked a nested loop where a hash join belonged. One `ANALYZE` to refresh stats fixed what looked like a query bug.
- The **next maintainer** handed a 200-line Spaghetti Query (§8) nobody — including the optimizer — can reason about, when three named CTEs would have been clear and fast.

Reading the plan and thinking in sets is empathy in code: the one `JOIN` you wrote instead of the loop, and the `EXPLAIN ANALYZE` you ran instead of guessing, are gifts to whoever owns this at 3 a.m.

---

## 11. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: a result is a *set*, which is why set-based beats row-by-row (§1).
- `sql-indexing-and-sargability` — *which* index turns a scan into a seek, and sargable predicates that let the planner use it (§4).
- `sql-lateral-and-correlated-derived` — the set-based "top-N per group" rewrite via `LATERAL` (§2).
- `sql-window-functions` — set-based running totals/ranking that replace per-row self-join loops (§2).
- `sql-set-operations` — `UNION`/`INTERSECT`/`EXCEPT` as set-at-a-time alternatives to row loops.
- `sql-subqueries-and-exists` — `IN`/`EXISTS` semi-joins that collapse per-row existence loops (§2).
- `sql-standard-vs-dialect-map` — per-engine `EXPLAIN` spellings and plan vocabulary (§9).

---

## 12. Reference Files

High-frequency plan-reading and set-vs-row anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
</content>
