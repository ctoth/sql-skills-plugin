---
name: sql-cte-and-recursion
description: Guides common table expressions for readable query decomposition (`WITH`) and `WITH RECURSIVE` for hierarchies, transitive closure, and series generation — always with a termination or cycle guard so recursion cannot run away. Teaches the canonical anchor + recursive-member structure (only the recursive term may reference the CTE), the working-table/queue evaluation model (recursion is iteration, not a stack), the UNION-vs-UNION-ALL distinction in recursion, the standard `SEARCH` and `CYCLE` clauses (SQL:2023) versus the portable manual path-array guard, recursive series generation as the portable `generate_series` substitute, and the PostgreSQL CTE materialization fence (PG <=11 always materializes; >=12 inlines a single-reference CTE unless `MATERIALIZED`). Auto-invokes when writing or editing `WITH`/`WITH RECURSIVE`, hierarchy/tree/graph traversal, transitive-closure or generate-series logic, deeply nested subqueries that want flattening, or on "infinite recursion" / "query runs forever" / "tree query" / "recursive query" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL CTEs and Recursion

> "The general form of a recursive `WITH` query is always a _non-recursive term_, then `UNION` (or `UNION ALL`), then a _recursive term_, where only the recursive term can contain a reference to the query's own output."
> — [PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html)

> "When working with recursive queries it is important to be sure that the recursive part of the query will eventually return no tuples, or else the query will loop indefinitely."
> — [PostgreSQL — Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html)

This skill builds on the foundation skill **`sql-relational-and-null-discipline`**: a CTE is still a set of rows, recursion still produces an unordered set unless you add `ORDER BY`, and the same three-valued logic governs every `WHERE` inside a recursive term. A `WITH` clause defines "statement scoped views" so you can "improve the structure of a statement without polluting the global namespace" ([modern-sql.com — WITH](https://modern-sql.com/feature/with)) — the cure for the unreadable nested-subquery monster. `WITH RECURSIVE` then turns SQL set-based, walking trees, graphs, and integer series without a procedural loop. The single rule that separates a working recursive query from one that runs forever is the **guard** — a termination predicate or a cycle check — so most of this skill is about that.

---

## 1. Non-Recursive `WITH` — Decompose for Readability

A plain `WITH` names an intermediate result so the outer query reads top-to-bottom instead of inside-out. "SQL:1999 added the `with` clause to define 'statement scoped views'" that let you "improve the structure of a statement without polluting the global namespace" ([modern-sql.com — WITH](https://modern-sql.com/feature/with)). It is a readability and decomposition tool first — no new capability, but a 200-line nested subquery becomes a sequence of named steps.

```sql
-- WRONG — a nested-subquery monster: read it inside-out to understand it
SELECT * FROM (
  SELECT customer_id, SUM(amount) AS spend FROM (
    SELECT o.customer_id, o.amount FROM orders o
    WHERE o.created_at >= DATE '2026-01-01'
  ) recent GROUP BY customer_id
) totals WHERE totals.spend > 1000;

-- RIGHT — named steps, read top-to-bottom; each CTE is one idea
WITH recent AS (
  SELECT customer_id, amount FROM orders WHERE created_at >= DATE '2026-01-01'
), totals AS (
  SELECT customer_id, SUM(amount) AS spend FROM recent GROUP BY customer_id
)
SELECT * FROM totals WHERE spend > 1000;
```

A `WITH` clause "must be followed by `select`" — it is not a standalone command ([modern-sql.com — WITH](https://modern-sql.com/feature/with)).

---

## 2. Recursive Structure and the Working-Table Algorithm

A recursive CTE is **two queries glued by `UNION`/`UNION ALL`**: a non-recursive *anchor* (the seed rows) and a *recursive term* that references the CTE name. Per the standard, "only the recursive term can contain a reference to the query's own output" ([PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html)) — self-referencing the anchor is an error. SQLite states the same shape: the body "must be two or more individual SELECT statements separated by compound operators," of the form `initial-select UNION ALL recursive-select` ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)).

"Recursion" here is **iteration over a working table**, not a call stack. PostgreSQL's algorithm:

> "1. Evaluate the non-recursive term. ... place them in a temporary _working table_.
> 2. So long as the working table is not empty, repeat these steps: ... Evaluate the recursive term, substituting the current contents of the working table for the recursive self-reference ... place them in a temporary _intermediate table_. ... Replace the contents of the working table with the contents of the intermediate table, then empty the intermediate table." ([PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html))

SQLite frames the identical idea as a queue: "Run the initial-select and add the results to a queue. While the queue is not empty: Extract a single row ... run the recursive-select, adding all results to the queue" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)). The loop ends when an iteration produces no new rows — that is the only natural stopping condition.

```sql
-- A descendants walk: anchor = the root, recursive term = "children of what we have"
WITH RECURSIVE subordinates AS (
  SELECT id, manager_id, name FROM employees WHERE id = 1   -- anchor (seed)
  UNION ALL
  SELECT e.id, e.manager_id, e.name
  FROM employees e
  JOIN subordinates s ON e.manager_id = s.id                -- references the CTE
)
SELECT * FROM subordinates;
```

Note: a recursive term "may not use [aggregate functions] or [window functions]" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)), and all non-recursive `SELECT`s must come before the recursive ones.

---

## 3. THE CYCLE GUARD — Recursion That Cannot Run Away

A tree terminates on its own (leaves have no children). A **graph with a cycle does not** — A→B→A feeds the working table forever, and "the query will loop indefinitely" ([PostgreSQL — Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html)). You must add a guard whenever the data *can* contain a cycle (any self-referential graph, or any hierarchy not provably acyclic).

```sql
-- WRONG — no guard; if friends has any cycle (a↔b), this never terminates
WITH RECURSIVE reach AS (
  SELECT person_id, friend_id FROM friends WHERE person_id = 1
  UNION ALL
  SELECT f.person_id, f.friend_id
  FROM friends f JOIN reach r ON f.person_id = r.friend_id
)
SELECT * FROM reach;
```

**Guard form A — manual path array (portable; works in SQLite/MySQL too).** "The standard method for handling such situations is to compute an array of the already-visited values" ([PostgreSQL — Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html)). Track the path, flag when the next node is already on it, and refuse to expand a flagged row:

```sql
-- RIGHT — carry a path; stop expanding the moment a node repeats
WITH RECURSIVE reach(person_id, friend_id, path, is_cycle) AS (
  SELECT person_id, friend_id, ARRAY[person_id], false
  FROM friends WHERE person_id = 1
  UNION ALL
  SELECT f.person_id, f.friend_id,
         r.path || f.friend_id,
         f.friend_id = ANY(r.path)          -- have we been here before?
  FROM friends f JOIN reach r ON f.person_id = r.friend_id
  WHERE NOT r.is_cycle                       -- the guard: don't expand a cyclic row
)
SELECT * FROM reach WHERE NOT is_cycle;
```

**Guard form B — the standard `CYCLE` clause (PostgreSQL 14+, SQL:2023).** "There is built-in syntax to simplify cycle detection." The `CYCLE` clause "specifies first the list of columns to track for cycle detection, then a column name that will show whether a cycle has been detected, and finally the name of another column that will track the path" ([PostgreSQL — Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html)):

```sql
-- RIGHT — same effect, declarative; PG 14+ only
WITH RECURSIVE reach(person_id, friend_id) AS (
  SELECT person_id, friend_id FROM friends WHERE person_id = 1
  UNION ALL
  SELECT f.person_id, f.friend_id
  FROM friends f JOIN reach r ON f.person_id = r.friend_id
) CYCLE friend_id SET is_cycle USING path
SELECT * FROM reach WHERE NOT is_cycle;
```

The `CYCLE` mark was modernized in SQL:2023: "When recursive queries were added to SQL, there was no `boolean` type, so the old standard required you to use a character string" — T133 now allows a boolean mark ([Eisentraut — SQL:2023](https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new)). Where neither `CYCLE` nor arrays exist, a `WHERE depth < N` ceiling is a crude-but-portable backstop.

---

## 4. `UNION` vs `UNION ALL` in Recursion

`UNION ALL` keeps every row and is the default for trees (no cycles, faster). `UNION` deduplicates: in PostgreSQL's algorithm, "For `UNION` (but not `UNION ALL`), discard duplicate rows and rows that duplicate any previous result row" ([PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html)). SQLite uses this as a cycle lever: "`UNION` is used instead of `UNION ALL` to prevent the recursion from entering an infinite loop if the graph contains cycles" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)).

But `UNION` only saves you when **whole rows repeat**. The moment you add a `depth` counter or a `path` column — which you usually want — every row becomes distinct, dedup never fires, and `UNION` silently stops protecting you:

```sql
-- TRAP — UNION looks safe, but adding `depth` makes every row unique,
-- so dedup never triggers and a cyclic graph still loops forever
WITH RECURSIVE reach(node, depth) AS (
  SELECT 1, 0
  UNION                      -- dedups whole rows...
  SELECT e.dst, r.depth + 1  -- ...but depth makes each row distinct -> no dedup
  FROM edges e JOIN reach r ON e.src = r.node
)
SELECT * FROM reach;
```

Rule of thumb: use `UNION ALL` + an explicit cycle guard (§3) for graphs. Treat `UNION` as a convenience for the narrow case of bare, repeat-free rows — not a substitute for a guard.

---

## 5. Series Generation — the Portable `generate_series`

A recursive counter is the portable substitute for `generate_series` (PostgreSQL-only) or a numbers table. The termination predicate is the guard. PostgreSQL: "A very simple example is this query to sum the integers from 1 through 100" ([PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html)); SQLite ships the identical shape ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)).

```sql
-- RIGHT — portable integer series 1..100; WHERE n < 100 is the termination guard
WITH RECURSIVE seq(n) AS (
  VALUES (1)
  UNION ALL
  SELECT n + 1 FROM seq WHERE n < 100
)
SELECT n FROM seq;
```

```sql
-- WRONG — no termination predicate; counts up with no ceiling, runs forever
WITH RECURSIVE seq(n) AS (
  VALUES (1)
  UNION ALL
  SELECT n + 1 FROM seq          -- nothing stops it
)
SELECT n FROM seq;
```

Use it to fill date ranges, expand quantities into rows, or join against a dense axis. SQLite also bounds recursion with `LIMIT`, which "determines the maximum number of rows that will ever be added to the recursive table ... Once the limit is reached, the recursion stops" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)).

---

## 6. Traversal Order — `SEARCH`, or `ORDER BY` in the Recursive Term

A recursive result is still an unordered set; the iteration order is not the output order. PostgreSQL 14+ has the standard `SEARCH` clause: it "specifies whether depth- or breadth first search is wanted, the list of columns to track for sorting, and a column name that will contain the result data that can be used for sorting" ([PostgreSQL — Search Order](https://www.postgresql.org/docs/current/queries-with.html)).

```sql
-- PG 14+ — emit an ordering column, then ORDER BY it
WITH RECURSIVE t(id, link, data) AS (
  SELECT id, link, data FROM tree
  UNION ALL
  SELECT c.id, c.link, c.data FROM tree c JOIN t ON c.link = t.id
) SEARCH DEPTH FIRST BY id SET ordercol
SELECT * FROM t ORDER BY ordercol;
```

SQLite has no `SEARCH` keyword; instead an "`ORDER BY` clause on the recursive-select can be used to control whether the search of a tree is depth-first or breadth-first," and "when the `ORDER BY` clause is omitted ... the queue behaves as a FIFO, which results in a breadth-first search" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)). Either way, finish with an outer `ORDER BY` for a defined output order (foundation skill, set semantics).

---

## 7. The Materialization Fence (PostgreSQL)

A non-recursive `WITH` can be an optimization barrier or transparent, depending on version. PostgreSQL: a side-effect-free `WITH` query "can be folded into the parent query ... By default, this happens if the parent query references the `WITH` query just once, but not if it references the `WITH` query more than once. You can override that decision by specifying `MATERIALIZED` to force separate calculation ... or by specifying `NOT MATERIALIZED` to force it to be merged" ([PostgreSQL — CTE Materialization](https://www.postgresql.org/docs/current/queries-with.html)).

The version split matters: **PostgreSQL <=11 always materialized every CTE** (a hard fence — outer predicates could not push down: sometimes useful, often a performance trap). **PG 12+ inlines a single-reference CTE** unless you write `MATERIALIZED`, so a query tuned for the old fence can change plans on upgrade. Use `MATERIALIZED` to keep the fence, `NOT MATERIALIZED` to force inlining.

```sql
-- Force the old always-materialize behavior (compute big_table once, fence it)
WITH w AS MATERIALIZED (SELECT * FROM big_table)
SELECT * FROM w WHERE key = 123;
```

Deciding *whether* a fence helps is a query-plan question — read the plan, do not guess. Deep `EXPLAIN`/planning detail belongs to **`sql-explain-and-set-based-thinking`**.

---

## 8. Portability Snapshot

The recursive `WITH RECURSIVE` skeleton (anchor + `UNION [ALL]` + recursive term) is portable; the *cycle-handling keywords* are not.

| Feature | PostgreSQL | SQLite | MySQL 8+ / MariaDB |
|---|---|---|---|
| `WITH` / `WITH RECURSIVE` | yes | yes | yes |
| Manual path-array cycle guard | yes (`ARRAY`/`= ANY`) | emulate (no array type) | emulate (JSON/string path) |
| `SEARCH` / `CYCLE` clauses | **14+** | no keyword | no keyword |
| `ORDER BY` in recursive term for traversal | (use `SEARCH`) | yes | limited |
| `LIMIT` to bound recursion | yes | yes | yes |
| Materialization fence control (`[NOT] MATERIALIZED`) | 12+ | n/a | n/a |
| `generate_series(...)` | yes (use instead of a counter) | no (use the counter) | no (use the counter) |

Where `SEARCH`/`CYCLE` are absent, fall back to the manual path-array guard (§3) — it expresses the same intent everywhere. SQLite lacks an array type, so emulate the path with a delimited string or JSON. For exact dialect spellings, route to **`sql-standard-vs-dialect-map`**.

---

## 9. Who Suffers When a CTE Goes Wrong

A recursive query without a guard does not fail at write time — it ships, then detonates on the first cyclic row:

- The **on-call engineer** paged at 2 a.m. because a transitive-closure query "ran forever" and pinned a CPU — the org chart got a circular `manager_id` (someone reports to their own report), there was no cycle guard, and the working table grew without bound until the connection timed out or memory blew. The `WHERE NOT is_cycle` they didn't write is the incident they're now mitigating (§3).
- The **analyst** whose graph reachability query quietly returned duplicates and inflated a count, because `UNION ALL` re-walked the same nodes through different paths and nobody deduplicated or guarded (§4).
- The **next maintainer** handed a 200-line query built from subqueries nested six deep, who has to mentally unwind it inside-out to change one filter — a `WITH` decomposition would have made it a readable list of named steps (§1).
- The **DBA** debugging a plan that changed after a PostgreSQL 11→12 upgrade: a CTE that used to be fenced now inlines, the predicate pushes down, and a once-fast query regressed (or improved) without a line of SQL changing (§7).

A guard is empathy in code: the path array, the `WHERE n < 100`, and the explicit `ORDER BY` are gifts to whoever runs this against real, messy, cyclic data.

---

## 10. Routing to Related Skills

- **`sql-relational-and-null-discipline`** — the foundation: CTE results are unordered sets, three-valued logic governs the recursive `WHERE`, NULLs in join keys still drop rows.
- **`sql-window-functions`** — window/ranking functions used *around* a CTE (they are banned *inside* a recursive term, §2); top-N and running totals.
- **`sql-schema-design-and-normalization`** — choosing the tree *model* (adjacency list vs nested set vs closure table) that the recursive walk runs over.
- **`sql-explain-and-set-based-thinking`** — reading the plan to decide if a materialization fence helps (§7) and recognizing when a recursive set beats a procedural loop.
- **`sql-standard-vs-dialect-map`** — `SEARCH`/`CYCLE` availability, array-vs-string path emulation, and `generate_series` vs the recursive counter.

---

## 11. Reference Files

High-frequency CTE and recursion anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
