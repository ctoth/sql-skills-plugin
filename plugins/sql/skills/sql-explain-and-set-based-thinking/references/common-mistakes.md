# Common EXPLAIN & Set-Based-Thinking Mistakes

Anti-patterns in LLM-generated SQL (and the application code around it) about performance —
looping where a set operation belongs, and guessing where the plan should be read. The skill
(`sql-explain-and-set-based-thinking`) states the model; this file holds the high-frequency
failure modes. All RIGHT examples are standard/portable SQL where possible; engine-specific
plan output is flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. The N+1 query loop (the classic RBAR)

**The problem:** The model fetches a list, then loops in application code issuing one query per element. It works in tests and issues thousands of round-trips in production. Each query is fast; the aggregate is not. "If a query is fast enough under certain testing conditions, it does not mean it will be fast enough in production" ([Use The Index, Luke — Testing Scalability](https://use-the-index-luke.com/sql/testing-scalability)).

```python
# WRONG — 1 + N queries
users = db.query("SELECT id FROM users WHERE active")
for u in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", u.id)
```

```sql
-- RIGHT — one set-based query; fetch the whole set in a single pass
SELECT u.id, o.*
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.active;
```

*Source: [Use The Index, Luke — Testing Scalability](https://use-the-index-luke.com/sql/testing-scalability); [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §2.*

---

## 2. Correlated scalar subquery as a per-row lookup

**The problem:** A subquery in the `SELECT` list is evaluated once per output row — row-by-row in disguise. It reads fine but scales like a loop.

```sql
-- WRONG — re-runs the count once per author row
SELECT a.name,
       (SELECT COUNT(*) FROM books b WHERE b.author_id = a.id) AS books
FROM authors a;

-- RIGHT — one aggregate over the join
SELECT a.name, COUNT(b.id) AS books
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
GROUP BY a.id, a.name;
```

*Source: [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) (`loops` reports per-row re-execution). Depth: this skill, §2.*

---

## 3. A loop of single-row INSERTs instead of a set insert

**The problem:** The model inserts a batch by looping one `INSERT` per item — N round-trips and N transactions' worth of overhead.

```sql
-- WRONG — one statement per row
INSERT INTO tags (post_id, name) VALUES (1, 'sql');
INSERT INTO tags (post_id, name) VALUES (1, 'perf');
-- ...repeated N times

-- RIGHT — one multi-row VALUES (or INSERT ... SELECT from a set)
INSERT INTO tags (post_id, name) VALUES
  (1, 'sql'), (1, 'perf'), (1, 'index');
```

*Source: [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §2.*

---

## 4. Guessing at performance instead of reading the plan

**The problem:** The model claims a query is "fast" or "uses the index" without running `EXPLAIN`. The plan is the only authority on what the engine actually does — "EXPLAIN QUERY PLAN reports on the way in which the query uses database indices" ([SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)).

```sql
-- WRONG (workflow) — assert speed with no evidence

-- RIGHT — ask the engine what it will do
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;          -- Postgres
EXPLAIN QUERY PLAN SELECT * FROM orders WHERE customer_id = 42; -- SQLite
```

*Source: [SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html); [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html). Depth: this skill, §3.*

---

## 5. Ignoring a full scan on a large, selectively-filtered table

**The problem:** The plan shows a full scan (`SCAN orders` / `Seq Scan on orders`) for a selective predicate on a big table, and the model treats it as fine. A scan visits every row; a search visits only a subset — "SCAN" is "a full-table scan"; "SEARCH" means "only a subset of the table rows are visited" ([SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)).

```text
-- WRONG signal — full scan for a one-customer lookup over millions of rows
SCAN orders
Seq Scan on orders  (Filter: customer_id = 42)

-- RIGHT signal — a seek via an index visits only matching rows
SEARCH orders USING INDEX idx_orders_customer (customer_id=?)
Index Scan using idx_orders_customer on orders
```

(A full scan is correct on a tiny table — see this skill §4. *Which* index to add → `sql-indexing-and-sargability`.)

*Source: [SQLite — EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html); [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html). Depth: this skill, §4.*

---

## 6. Reading plain EXPLAIN as if it were measured truth

**The problem:** The model treats plain `EXPLAIN`'s `cost`/`rows` as actual numbers. They are estimates in arbitrary units. Only `ANALYZE` runs the query and "displays the true row counts and true run time accumulated within each plan node" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)); actual time is in ms while cost is arbitrary, so "they are unlikely to match up."

```sql
-- WRONG (interpretation) — "cost=18334" is not milliseconds and not measured
EXPLAIN SELECT ...;

-- RIGHT — run it and read actual rows + actual time (careful: it executes)
EXPLAIN ANALYZE SELECT ...;
```

*Source: [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html). Depth: this skill, §5, §6.*

---

## 7. Missing the estimated-vs-actual rows gap (the bad-plan tell)

**The problem:** The model reads an `ANALYZE` plan but ignores the divergence between estimated and actual rows — the single most diagnostic signal. "The thing that's usually most important to look for is whether the estimated row counts are reasonably close to reality" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). A big gap means the planner was misled, usually by stale statistics.

```text
-- THE TELL — estimate 12, reality 2,000,000: the join order is built on a lie
Nested Loop  (rows=12) (actual rows=2000000 loops=1)
  -> Seq Scan on big  (rows=12) (actual rows=2000000)

-- FIX — refresh the sampled statistics, then re-read the plan
ANALYZE big;
```

*Source: [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) (stats are "taken from a randomized sample of the table"). Depth: this skill, §6.*

---

## 8. Testing only on tiny development data

**The problem:** The model validates a query on a handful of seeded rows and declares it production-ready. Plans are not linear in table size — "results on a toy-sized table cannot be assumed to apply to large tables... it might choose a different plan for a larger or smaller table" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)).

```sql
-- WRONG (workflow) — 100 seeded rows; seq scan looks instant, ships it

-- RIGHT — measure on production-representative volume, re-read the plan there
-- (and expect the access method / join order to change as the table grows)
EXPLAIN ANALYZE SELECT ...;   -- against a realistically-sized dataset
```

*Source: [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html); [Use The Index, Luke — Testing Scalability](https://use-the-index-luke.com/sql/testing-scalability). Depth: this skill, §7.*

---

## 9. Overcorrecting into one giant Spaghetti Query

**The problem:** Having learned "set-based, not loops," the model crams the entire problem into one sprawling statement with many joins and nested aggregates — unreadable, and easy for the optimizer to mis-plan or accidentally cross-join. This is Karwin's "Spaghetti Query" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG — one 150-line query nobody (including the planner) can reason about

-- RIGHT — a few named, individually-plannable steps
WITH active_users AS (
  SELECT id FROM users WHERE active
),
recent_orders AS (
  SELECT user_id, SUM(total) AS spend
  FROM orders WHERE created_at > now() - interval '30 days'
  GROUP BY user_id
)
SELECT u.id, COALESCE(r.spend, 0) AS spend
FROM active_users u
LEFT JOIN recent_orders r ON r.user_id = u.id;
```

*Source: [Karwin, *SQL Antipatterns* — "Spaghetti Query"](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §8; CTE syntax owned by `sql-cte-and-recursion`.*

---

## 10. Mistaking `loops=N` in the plan for noise

**The problem:** The model sees `loops=5000` on an inner node and overlooks it. That count is the in-plan fingerprint of a per-row repetition — "the `loops` value reports the total number of executions of the node, and the actual time and rows values shown are averages per-execution" ([PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)). A high `loops` on an inner index scan is the N+1 pattern surfacing inside a single statement.

```text
-- TELL — inner node ran 5000 times; per-row work hidden in a nested loop
Nested Loop
  -> Seq Scan on authors  (actual rows=5000 loops=1)
  -> Index Scan on books   (actual rows=1 loops=5000)   <-- 5000 executions

-- Reframe as a single set-based join/aggregate (this skill §2) if selectivity allows.
```

*Source: [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html). Depth: this skill, §2, §6.*
</content>
