# Research: sql-cte-and-recursion

Research for the standard-SQL skill `sql-cte-and-recursion`. Each entry: URL, section, verbatim
quote, and why it matters for the skill. Accessed 2026-06-26.

---

## Source A — PostgreSQL: WITH Queries (CTEs)
URL: https://www.postgresql.org/docs/current/queries-with.html

### A1. Recursive structure: anchor + UNION ALL + recursive term
**Section 7.8.2 Recursive Queries**
> "The general form of a recursive `WITH` query is always a _non-recursive term_, then `UNION` (or `UNION ALL`), then a _recursive term_, where only the recursive term can contain a reference to the query's own output."

Why it matters: This is the canonical skeleton the skill teaches. The constraint "only the recursive term can contain a reference to the query's own output" is the rule LLMs violate when they self-reference in the anchor.

### A2. The working-table evaluation algorithm
**Section 7.8.2 Recursive Queries (Recursive Query Evaluation)**
> "1. Evaluate the non-recursive term. For `UNION` (but not `UNION ALL`), discard duplicate rows. Include all remaining rows in the result of the recursive query, and also place them in a temporary _working table_.
> 2. So long as the working table is not empty, repeat these steps:
>     1. Evaluate the recursive term, substituting the current contents of the working table for the recursive self-reference. For `UNION` (but not `UNION ALL`), discard duplicate rows and rows that duplicate any previous result row. Include all remaining rows in the result of the recursive query, and also place them in a temporary _intermediate table_.
>     2. Replace the contents of the working table with the contents of the intermediate table, then empty the intermediate table."

Why it matters: The mechanical model the skill must convey — "recursive" is iteration over a working table, not stack recursion. It also pins exactly where `UNION` dedup happens (both phases), which is the cycle-stopping mechanism.

### A3. Infinite recursion warning
**Section 7.8.2.2 Cycle Detection**
> "When working with recursive queries it is important to be sure that the recursive part of the query will eventually return no tuples, or else the query will loop indefinitely."

Why it matters: The runaway-recursion danger, stated by the primary source. Anchors the "THE CYCLE GUARD" section's WRONG case.

### A4. Manual path-array cycle guard
**Section 7.8.2.2 Cycle Detection**
> "However, often a cycle does not involve output rows that are completely duplicate: it may be necessary to check just one or a few fields to see if the same point has been reached before. The standard method for handling such situations is to compute an array of the already-visited values."

PG example (path array, manual guard):
```sql
WITH RECURSIVE search_graph(id, link, data, depth, is_cycle, path) AS (
    SELECT g.id, g.link, g.data, 0, false, ARRAY[g.id]
    FROM graph g
  UNION ALL
    SELECT g.id, g.link, g.data, sg.depth + 1,
      g.id = ANY(path), path || g.id
    FROM graph g, search_graph sg
    WHERE g.id = sg.link AND NOT is_cycle
)
SELECT * FROM search_graph;
```

Why it matters: The portable cycle guard that works even where `CYCLE` is unsupported (SQLite/MySQL). The `g.id = ANY(path)` flag + `WHERE ... AND NOT is_cycle` is the RIGHT pattern.

### A5. CYCLE clause (SQL standard)
**Section 7.8.2.2 Cycle Detection**
> "There is built-in syntax to simplify cycle detection."
> "The `CYCLE` clause specifies first the list of columns to track for cycle detection, then a column name that will show whether a cycle has been detected, and finally the name of another column that will track the path. The cycle and path columns will implicitly be added to the output rows of the CTE."

```sql
... ) CYCLE id SET is_cycle USING path
SELECT * FROM search_graph;
```

Why it matters: The standard `CYCLE col SET flag USING pathcol` clause — the readable RIGHT answer where supported (PG 14+).

### A6. SEARCH clause (BREADTH/DEPTH FIRST)
**Section 7.8.2.1 Search Order**
> "The `SEARCH` clause specifies whether depth- or breadth first search is wanted, the list of columns to track for sorting, and a column name that will contain the result data that can be used for sorting. That column will implicitly be added to the output rows of the CTE."

```sql
... ) SEARCH DEPTH FIRST BY id SET ordercol
SELECT * FROM search_tree ORDER BY ordercol;
```

Why it matters: SEARCH controls traversal order portably-ish (PG 14+). Routes availability to dialect-map.

### A7. Series generation
**Section 7.8.2 Recursive Queries**
> "A very simple example is this query to sum the integers from 1 through 100:"
```sql
WITH RECURSIVE t(n) AS (
    VALUES (1)
  UNION ALL
    SELECT n+1 FROM t WHERE n < 100
)
SELECT sum(n) FROM t;
```

Why it matters: The portable `generate_series` substitute. The `WHERE n < 100` is the explicit termination predicate (the "guard" for a series, not a cycle).

### A8. Materialization fence
**Section 7.8.3 Common Table Expression Materialization**
> "However, if a `WITH` query is non-recursive and side-effect-free (that is, it is a `SELECT` containing no volatile functions) then it can be folded into the parent query, allowing joint optimization of the two query levels. By default, this happens if the parent query references the `WITH` query just once, but not if it references the `WITH` query more than once. You can override that decision by specifying `MATERIALIZED` to force separate calculation of the `WITH` query, or by specifying `NOT MATERIALIZED` to force it to be merged into the parent query."

Why it matters: PG 12+ inlines a single-reference CTE unless you write `MATERIALIZED`; PG <=11 always materialized (an optimization barrier). The fence note. Deep plan detail routes to sql-explain-and-set-based-thinking.

---

## Source B — SQLite: The WITH Clause
URL: https://www.sqlite.org/lang_with.html

### B1. Recursive CTE definition + compound-select structure
**Section 3. Recursive Common Table Expressions**
> "A recursive common table expression can be used to write a query that walks a tree or graph."
> "The '[select-stmt]' must be a [compound select]. That is to say, the CTE body must be two or more individual SELECT statements separated by compound operators like UNION, UNION ALL, INTERSECT, or EXCEPT."
> Canonical form: "cte-table-name AS ( initial-select UNION ALL recursive-select )"

Why it matters: Second independent statement of the same skeleton (initial/recursive split), with SQLite's own vocabulary (initial-select / recursive-select).

### B2. SQLite's explicit queue algorithm
**Section 3. Recursive Common Table Expressions**
> "The basic algorithm for computing the content of the recursive table is as follows:
> 1. Run the initial-select and add the results to a queue.
> 2. While the queue is not empty:
>    1. Extract a single row from the queue.
>    2. Insert that single row into the recursive table
>    3. Pretend that the single row just extracted is the only row in the recursive table and run the recursive-select, adding all results to the queue."

Why it matters: A concrete, queue-based mental model complementing PG's working-table phrasing. Makes the iteration visible.

### B3. Structural limits
**Section 3. Recursive Common Table Expressions**
> "A SELECT statement is recursive if its FROM clause contains exactly once reference to the CTE table..."
> "All non-recursive SELECT statements must occur before any recursive SELECT statements."
> "Recursive SELECT statements may not use [aggregate functions] or [window functions]."

Why it matters: Constraints LLMs trip over (aggregate/window in recursive term; anchor must precede recursive term).

### B4. ORDER BY controls traversal order; LIMIT bounds it
**Section 3.4 Controlling Depth-First Versus Breadth-First Search Using ORDER BY**
> "An ORDER BY clause on the recursive-select can be used to control whether the search of a tree is depth-first or breadth-first."
> "When the ORDER BY clause is omitted from the recursive-select, the queue behaves as a FIFO, which results in a breadth-first search."
> "The LIMIT clause, if present, determines the maximum number of rows that will ever be added to the recursive table in step 2b. Once the limit is reached, the recursion stops."

Why it matters: SQLite lacks SEARCH/CYCLE keywords; ORDER BY + LIMIT inside the recursive CTE is its portable substitute for traversal control and a hard safety bound.

### B5. UNION stops cycles
**Section 3.3 Queries Against A Graph**
> "UNION is used instead of UNION ALL to prevent the recursion from entering an infinite loop if the graph contains cycles."
> "If a UNION operator connects the initial-select with the recursive-select, then only add rows to the queue if no identical row has been previously added to the queue."

Why it matters: The UNION-vs-UNION-ALL distinction stated as a cycle-safety lever — but note it only dedups whole rows (so a depth/path column defeats it; see synthesis).

### B6. Integer sequence generation
**Section 3.1 Recursive Query Examples**
```sql
WITH RECURSIVE cnt(x) AS (
  VALUES(1) UNION ALL SELECT x+1 FROM cnt WHERE x<1000000
) SELECT x FROM cnt;
```
Also a LIMIT-bounded variant (`SELECT 1 UNION ALL SELECT x+1 FROM cnt LIMIT 1000000`).

Why it matters: Confirms series generation is portable and identical in shape to PG's.

---

## Source C — modern-sql.com: WITH
URL: https://modern-sql.com/feature/with

### C1. WITH purpose: decomposition without global namespace pollution
**Section Introduction**
> "SQL:1999 added the `with` clause to define 'statement scoped views'."
> "This makes it possible to improve the structure of a statement without polluting the global namespace."

**Section Syntax**
> "`With` is not a stand alone command like `create view` is: it must be followed by `select`."

Why it matters: Vendor-neutral justification for non-recursive WITH as a readability/decomposition tool (the alternative to a 200-line nested-subquery monster). Note: this page covers only non-recursive WITH; it explicitly states the recursive form "is covered in another article," so recursive portability claims here lean on PG/SQLite/Eisentraut.

---

## Source D — Eisentraut: SQL:2023 is finished, here is what's new
URL: https://peter.eisentraut.org/blog/2023/04/04/sql-2023-is-finished-here-is-whats-new

### D1. Cycle clause enhancement (T133)
**Section Enhanced cycle mark values (T133)**
> "The `CYCLE` clause is a lesser-known feature of recursive queries."
> "When recursive queries were added to SQL, there was no `boolean` type, so the old standard required you to use a character string..." (SQL:2023 T133 lets the cycle mark be a boolean.)

Why it matters: Confirms CYCLE is a standardized clause and that SQL:2023 modernized its mark values (boolean instead of character literals). Dates the feature evolution.

### D2. Property Graph Queries (SQL/PGQ) — new Part 16
**Section Property Graph Queries**
> "A whole new part 16 was added to the SQL standard, titled 'Property Graph Queries (SQL/PGQ)'."
> "This allows data in tables to be queried as if it were a graph database."

Why it matters: Context — the standard's heavier graph-traversal answer lives outside recursive CTEs; out of scope for this skill but worth a one-line nod.

### D3. General framing
**Section (intro/closing)**
> "The two major enhancements in SQL:2023 are functionality to integrate with non-relational data management."
> "The core, relational SQL language is pretty complete, but as can be seen, there is room for usability enhancements and gentle modernization."

Why it matters: Frames CYCLE/SEARCH as "gentle modernization" of an otherwise-complete core — supports the skill's portability stance (the manual path-array still works everywhere).

---

## Synthesis for the skill

- **Skeleton:** anchor (non-recursive) `UNION [ALL]` recursive term; only the recursive term references the CTE name (A1, B1).
- **Mental model:** iteration over a working table / queue until it empties, not stack recursion (A2, B2).
- **Runaway danger:** without a guard the recursive term never returns empty and the query loops indefinitely (A3).
- **Cycle guard, two forms:** (1) manual path array `g.id = ANY(path)` + `WHERE NOT is_cycle` — portable (A4); (2) standard `CYCLE col SET flag USING pathcol` — PG 14+, SQL:2023 (A5, D1).
- **UNION vs UNION ALL:** UNION dedups whole rows and can stop simple cycles (B5), but it only helps when rows are identical; once you add a `depth`/`path` column every row is distinct, so UNION no longer protects you — you need an explicit guard (A4, A2 dedup note). UNION ALL is the default for trees (no cycles) and is faster.
- **Series generation:** recursive counter with explicit `WHERE n < N` termination — the portable `generate_series` substitute (A7, B6).
- **Traversal order:** PG `SEARCH DEPTH/BREADTH FIRST` (A6); SQLite via `ORDER BY` in the recursive-select, FIFO=breadth-first default, plus `LIMIT` as a hard bound (B4).
- **Materialization fence:** PG <=11 always materializes a CTE (optimization barrier); PG 12+ inlines single-reference non-recursive CTEs unless `MATERIALIZED` (A8). Deep plan detail → sql-explain-and-set-based-thinking.
- **Non-recursive WITH:** statement-scoped views for decomposition/readability (C1).
- **Portability:** structure portable across PG / SQLite / MySQL 8+ / MariaDB; SEARCH/CYCLE keywords are PG 14+ only (SQLite/MySQL need the manual path array). Route spellings → sql-standard-vs-dialect-map.
