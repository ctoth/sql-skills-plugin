# Common SQL CTE & Recursion Mistakes
## Contents

- [1. Recursive query with no cycle guard on cyclic data](#1-recursive-query-with-no-cycle-guard-on-cyclic-data)
- [2. Believing UNION makes recursion cycle-safe once a depth/path column exists](#2-believing-union-makes-recursion-cycle-safe-once-a-depthpath-column-exists)
- [3. Referencing the CTE from the anchor (non-recursive) term](#3-referencing-the-cte-from-the-anchor-non-recursive-term)
- [4. Series generation with no termination predicate](#4-series-generation-with-no-termination-predicate)
- [5. Aggregate or window function inside the recursive term](#5-aggregate-or-window-function-inside-the-recursive-term)
- [6. Assuming recursion (or any CTE) returns rows in a defined order](#6-assuming-recursion-or-any-cte-returns-rows-in-a-defined-order)
- [7. Treating a CTE as a guaranteed optimization fence](#7-treating-a-cte-as-a-guaranteed-optimization-fence)
- [8. A deeply nested subquery where a WITH would be readable](#8-a-deeply-nested-subquery-where-a-with-would-be-readable)


Anti-patterns in LLM-generated SQL around `WITH` and `WITH RECURSIVE`, each with wrong/right code
and a primary-source citation. The skill (`sql-cte-and-recursion`) states the rules; this file holds
the high-frequency failure modes. All RIGHT examples are standard/portable SQL; non-standard
spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Recursive query with no cycle guard on cyclic data

**The problem:** The model walks a self-referential graph (friends, follows, manager chains, parts-explosion) with `UNION ALL` and no path check. On any cycle the recursive term never returns empty and "the query will loop indefinitely" ([PostgreSQL — Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html)). It does not error — it hangs and consumes memory/CPU until something times out.

```sql
-- WRONG — a↔b cycle in friends loops forever
WITH RECURSIVE reach AS (
  SELECT person_id, friend_id FROM friends WHERE person_id = 1
  UNION ALL
  SELECT f.person_id, f.friend_id
  FROM friends f JOIN reach r ON f.person_id = r.friend_id
)
SELECT * FROM reach;

-- RIGHT — carry a path, refuse to expand a node already visited
WITH RECURSIVE reach(person_id, friend_id, path, is_cycle) AS (
  SELECT person_id, friend_id, ARRAY[person_id], false
  FROM friends WHERE person_id = 1
  UNION ALL
  SELECT f.person_id, f.friend_id, r.path || f.friend_id,
         f.friend_id = ANY(r.path)
  FROM friends f JOIN reach r ON f.person_id = r.friend_id
  WHERE NOT r.is_cycle
)
SELECT * FROM reach WHERE NOT is_cycle;
```

*Source: [PostgreSQL — Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html) ("The standard method ... is to compute an array of the already-visited values"). Depth: this skill, §3.*

---

## 2. Believing `UNION` makes recursion cycle-safe once a `depth`/`path` column exists

**The problem:** The model adds `UNION` (not `UNION ALL`) for safety, then adds a `depth` counter. `UNION` only discards rows that "duplicate any previous result row" ([PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html)); a monotonically increasing `depth` makes every row distinct, so dedup never fires and a cyclic graph still loops.

```sql
-- WRONG — depth makes every row unique, so UNION never dedups -> still infinite
WITH RECURSIVE reach(node, depth) AS (
  SELECT 1, 0
  UNION
  SELECT e.dst, r.depth + 1 FROM edges e JOIN reach r ON e.src = r.node
)
SELECT * FROM reach;

-- RIGHT — UNION ALL + an explicit cycle guard (path array)
WITH RECURSIVE reach(node, depth, path, is_cycle) AS (
  SELECT 1, 0, ARRAY[1], false
  UNION ALL
  SELECT e.dst, r.depth + 1, r.path || e.dst, e.dst = ANY(r.path)
  FROM edges e JOIN reach r ON e.src = r.node
  WHERE NOT r.is_cycle
)
SELECT * FROM reach WHERE NOT is_cycle;
```

*Source: [PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html); [SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html) ("UNION is used instead of UNION ALL to prevent ... an infinite loop"). Depth: this skill, §4.*

---

## 3. Referencing the CTE from the anchor (non-recursive) term

**The problem:** The model self-references in the seed query. "Only the recursive term can contain a reference to the query's own output" ([PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html)); SQLite requires "all non-recursive SELECT statements must occur before any recursive SELECT statements" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)).

```sql
-- WRONG — anchor references the CTE itself; no real seed
WITH RECURSIVE t AS (
  SELECT id, parent_id FROM t            -- self-reference in the anchor
  UNION ALL
  SELECT c.id, c.parent_id FROM nodes c JOIN t ON c.parent_id = t.id
)
SELECT * FROM t;

-- RIGHT — anchor selects real seed rows from the base table
WITH RECURSIVE t AS (
  SELECT id, parent_id FROM nodes WHERE parent_id IS NULL   -- seed = roots
  UNION ALL
  SELECT c.id, c.parent_id FROM nodes c JOIN t ON c.parent_id = t.id
)
SELECT * FROM t;
```

*Source: [PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html). Depth: this skill, §2.*

---

## 4. Series generation with no termination predicate

**The problem:** The model writes a recursive counter but forgets the `WHERE n < N` ceiling, so the count climbs without bound. The termination predicate *is* the guard for a series.

```sql
-- WRONG — nothing stops it; runs until it exhausts resources
WITH RECURSIVE seq(n) AS (
  VALUES (1)
  UNION ALL
  SELECT n + 1 FROM seq
)
SELECT n FROM seq;

-- RIGHT — explicit ceiling terminates the recursion
WITH RECURSIVE seq(n) AS (
  VALUES (1)
  UNION ALL
  SELECT n + 1 FROM seq WHERE n < 100
)
SELECT n FROM seq;
```

*Source: [PostgreSQL — Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html) (sum integers 1..100 example); [SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html). Depth: this skill, §5.*

---

## 5. Aggregate or window function inside the recursive term

**The problem:** The model puts `SUM(...)`, `COUNT(...)`, or a window function in the recursive `SELECT`. "Recursive SELECT statements may not use [aggregate functions] or [window functions]" ([SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)); aggregate the *result* of the CTE in the outer query instead.

```sql
-- WRONG — aggregate inside the recursive term (rejected / ill-defined)
WITH RECURSIVE t(id, total) AS (
  SELECT id, amount FROM nodes WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, SUM(c.amount) FROM nodes c JOIN t ON c.parent_id = t.id
)
SELECT * FROM t;

-- RIGHT — walk in the CTE, aggregate in the outer query
WITH RECURSIVE t(id, parent_id, amount) AS (
  SELECT id, parent_id, amount FROM nodes WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.amount FROM nodes c JOIN t ON c.parent_id = t.id
)
SELECT SUM(amount) AS total FROM t;
```

*Source: [SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html). Depth: this skill, §2.*

---

## 6. Assuming recursion (or any CTE) returns rows in a defined order

**The problem:** The model relies on parent-before-child or level order from a recursive walk without an `ORDER BY`. A recursive result is still an unordered set; the working-table iteration order is not the output order. PostgreSQL exposes order via `SEARCH ... SET ordercol`; SQLite via `ORDER BY` on the recursive-select, but the *output* still needs an outer `ORDER BY` ([PostgreSQL — Search Order](https://www.postgresql.org/docs/current/queries-with.html); [SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html)).

```sql
-- WRONG — assumes breadth-first / parent-first output; undefined
WITH RECURSIVE t(id, link, data) AS (
  SELECT id, link, data FROM tree WHERE link IS NULL
  UNION ALL
  SELECT c.id, c.link, c.data FROM tree c JOIN t ON c.link = t.id
)
SELECT * FROM t;

-- RIGHT (PG 14+) — emit an ordering column and ORDER BY it
WITH RECURSIVE t(id, link, data) AS (
  SELECT id, link, data FROM tree WHERE link IS NULL
  UNION ALL
  SELECT c.id, c.link, c.data FROM tree c JOIN t ON c.link = t.id
) SEARCH DEPTH FIRST BY id SET ordercol
SELECT * FROM t ORDER BY ordercol;
```

*Source: [PostgreSQL — Search Order](https://www.postgresql.org/docs/current/queries-with.html); [SQLite — Recursive CTEs](https://www.sqlite.org/lang_with.html). Depth: this skill, §6; set semantics owned by `sql-relational-and-null-discipline`.*

---

## 7. Treating a CTE as a guaranteed optimization fence

**The problem:** The model assumes a `WITH` is always materialized once (true in PostgreSQL <=11), so it relies on the fence for performance or correctness. In PG 12+ a single-reference, side-effect-free CTE "can be folded into the parent query" unless you write `MATERIALIZED` ([PostgreSQL — CTE Materialization](https://www.postgresql.org/docs/current/queries-with.html)) — a plan can change silently on upgrade.

```sql
-- WRONG (fragile assumption) — relies on the CTE being computed once and fenced
WITH w AS (SELECT * FROM big_table WHERE expensive_predicate(x))
SELECT * FROM w WHERE key = 123;   -- PG 12+ may inline and push key=123 inward

-- RIGHT — say what you mean when the fence matters
WITH w AS MATERIALIZED (SELECT * FROM big_table WHERE expensive_predicate(x))
SELECT * FROM w WHERE key = 123;
```

*Source: [PostgreSQL — CTE Materialization](https://www.postgresql.org/docs/current/queries-with.html). Depth: this skill, §7; plan-reading owned by `sql-explain-and-set-based-thinking`.*

---

## 8. A deeply nested subquery where a `WITH` would be readable

**The problem:** The model stacks derived tables inside-out instead of naming intermediate steps, producing a query nobody can safely edit. `WITH` defines "statement scoped views" to "improve the structure of a statement" ([modern-sql.com — WITH](https://modern-sql.com/feature/with)) — same result, legible.

```sql
-- WRONG — read it inside-out to understand it
SELECT * FROM (
  SELECT customer_id, SUM(amount) AS spend
  FROM (SELECT customer_id, amount FROM orders WHERE created_at >= DATE '2026-01-01') r
  GROUP BY customer_id
) t WHERE t.spend > 1000;

-- RIGHT — named steps, top-to-bottom
WITH recent AS (
  SELECT customer_id, amount FROM orders WHERE created_at >= DATE '2026-01-01'
), totals AS (
  SELECT customer_id, SUM(amount) AS spend FROM recent GROUP BY customer_id
)
SELECT * FROM totals WHERE spend > 1000;
```

*Source: [modern-sql.com — WITH](https://modern-sql.com/feature/with). Depth: this skill, §1.*
