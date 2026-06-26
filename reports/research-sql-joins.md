# Research: sql-joins

Research backing the `sql-joins` skill. Each entry: URL, section, verbatim quote, why it matters.
Accessed 2026-06-26.

---

## Source A — PostgreSQL: Table Expressions / Joined Tables

URL: https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN

### A1. CROSS JOIN definition
Section: "Joined Tables — CROSS JOIN".
> "For every possible combination of rows from _T1_ and _T2_ (i.e., a Cartesian product), the joined table will contain a row consisting of all columns in _T1_ followed by all columns in _T2_. If the tables have N and M rows respectively, the joined table will have N \* M rows."

Why it matters: defines the cross product and the N×M blow-up — the exact thing an accidental cross product (missing/typo'd predicate) produces. Citable fact for the "accidental cross product" section.

### A2. INNER JOIN definition
> "For each row R1 of T1, the joined table has a row for each row in T2 that satisfies the join condition with R1."

Why it matters: baseline semantics; INNER keeps only matched pairs.

### A3. LEFT OUTER JOIN definition
> "First, an inner join is performed. Then, for each row in T1 that does not satisfy the join condition with any row in T2, a joined row is added with null values in columns of T2. Thus, the joined table always has at least one row for each row in T1."

Why it matters: this is the precise mechanism — null-extension of the right side — that the WHERE-filter trap destroys. "Always at least one row for each row in T1" is the promise a WHERE filter silently breaks.

### A4. RIGHT OUTER JOIN definition
> "First, an inner join is performed. Then, for each row in T2 that does not satisfy the join condition with any row in T1, a joined row is added with null values in columns of T1. This is the converse of a left join: the result table will always have a row for each row in T2."

### A5. FULL OUTER JOIN definition
> "First, an inner join is performed. Then, for each row in T1 that does not satisfy the join condition with any row in T2, a joined row is added with null values in columns of T2. Also, for each row of T2 that does not satisfy the join condition with any row in T1, a joined row with null values in the columns of T1 is added."

Why it matters: FULL JOIN keeps unmatched rows from both sides. Note: MySQL lacks FULL JOIN → route emulation (LEFT UNION RIGHT) to dialect-map.

### A6. ON clause
> "The `ON` clause is the most general kind of join condition: it takes a Boolean value expression of the same kind as is used in a `WHERE` clause. A pair of rows from _T1_ and _T2_ match if the `ON` expression evaluates to true."

Why it matters: ON is the general, recommended condition; a Boolean expression like WHERE — so any predicate can go there, including filters on the outer table (the fix for the trap).

### A7. USING clause
> "The `USING` clause is a shorthand that allows you to take advantage of the specific situation where both sides of the join use the same name for the joining column(s). It takes a comma-separated list of the shared column names and forms a join condition that includes an equality comparison for each one. For example, joining _T1_ and _T2_ with `USING (a, b)` produces the join condition `ON _T1_.a = _T2_.a AND _T1_.b = _T2_.b`."
> "Furthermore, the output of `JOIN USING` suppresses redundant columns: there is no need to print both of the matched columns, since they must have equal values. While `JOIN ON` produces all columns from _T1_ followed by all columns from _T2_, `JOIN USING` produces one output column for each of the listed column pairs (in the listed order), followed by any remaining columns from _T1_, followed by any remaining columns from _T2_."

Why it matters: USING = equality on explicitly listed shared names + column de-duplication. The contrast with NATURAL is that USING names the columns explicitly (safe); NATURAL does not.

### A8. NATURAL — the landmine
> "Finally, `NATURAL` is a shorthand form of `USING`: it forms a `USING` list consisting of all column names that appear in both input tables. As with `USING`, these columns appear only once in the output table. If there are no common column names, `NATURAL JOIN` behaves like `CROSS JOIN`."

Why it matters: THE centerpiece evidence for the NATURAL schema-change landmine. NATURAL joins on *all* same-named columns — so adding a column with a name that happens to collide (e.g. `created_at`, `status`, `id`) silently changes the join key set; and the fallback when there are no common columns is a CROSS JOIN (N×M explosion). Both are silent, schema-driven behavior changes.

### A9. ON vs WHERE for outer joins — THE CENTERPIECE FACT
> "This is because a restriction placed in the `ON` clause is processed _before_ the join, while a restriction placed in the `WHERE` clause is processed _after_ the join. That does not matter with inner joins, but it matters a lot with outer joins."

Why it matters: this is the exact mechanism of the outer-join filter trap. A predicate on the right (nullable) table in WHERE runs *after* null-extension; the null-extended rows have NULL in that column, the predicate evaluates to UNKNOWN/FALSE, and WHERE drops them — silently demoting LEFT JOIN to INNER JOIN. The same predicate in ON runs *before*, so unmatched left rows still survive null-extended. (Ties to foundation: WHERE keeps only TRUE; NULL → UNKNOWN → dropped.)

---

## Source B — SQLite: SELECT / join-operator

URL: https://www.sqlite.org/lang_select.html

### B1. Supported join operators (syntax diagram)
Section: join-operator syntax diagram.
Confirmed operators: `NATURAL`, `LEFT`, `RIGHT`, `FULL`, `INNER`, `CROSS`, `OUTER`, `JOIN`.
join-constraint: `USING ( column-name, ... )` or `ON expr`.

Why it matters: SQLite supports all five join types (RIGHT and FULL OUTER added in SQLite 3.39.0, 2022) plus NATURAL and USING/ON — confirms portability of the standard forms. (NOTE: the descriptive prose for NATURAL semantics did not survive the fetch-to-markdown truncation; the authoritative definitional wording is taken from PostgreSQL A8, and SQLite's NATURAL behaves the same way — equivalent to USING on all common columns. Flag in skill as PostgreSQL-sourced definition.)

---

## Source C — Markus Winand, "SQL JOIN" (Use The Index, Luke)

URL: https://use-the-index-luke.com/sql/join  and  .../sql/join/nested-loops-join

### C1. What a join does
> "The join operation transforms data from a normalized model into a denormalized form that suits a specific processing purpose."

Why it matters: framing — a join recombines normalized tables; correctness (which rows, how many) is the join's job, and getting the predicate wrong recombines them wrongly (fan-out, cross product).

### C2. Joins are seek-sensitive; indexing is the fix
> "Joining is particularly sensitive to disk seek latencies because it combines scattered data fragments. Proper indexing is again the best solution to reduce response times."

Why it matters: motivates routing join *performance* to sql-indexing-and-sargability — this skill owns correctness, not plans.

### C3. Nested loops join — one row at a time
> "It works like using two nested queries: the outer or driving query to fetch the results from one table and a second query _for each row_ from the driving query to fetch the corresponding data from the other table."
> "The nested loops join delivers good performance if the driving query returns a small result set."

Why it matters: the conceptual "for each row, look up the other side" model — useful one-liner for explaining what a join physically is, before routing depth to indexing skill.

### C4. Indexing depends on the algorithm
> "The correct index however depends on which of the three common join algorithms is used for the query." (nested loops / hash join / sort-merge join)

Why it matters: names the three physical join algorithms conceptually (nested loops, hash, merge) and explicitly defers index choice — keep this skill's mention to a sentence and route to indexing.

---

## Synthesis — the six facts to nail (with citations)

1. **Five join types** — INNER (A2), LEFT (A3), RIGHT (A4), FULL (A5), CROSS (A1). All standard; FULL absent in MySQL → dialect-map.
2. **Outer-join filter trap (centerpiece)** — A predicate on the outer/nullable table in WHERE runs *after* the join (A9), so it nukes the null-extended rows and demotes LEFT→INNER. Fix: put the filter in ON (A6, A9). Legit case: a filter on the *preserved* (left) table, or a deliberate "find unmatched" `WHERE r.id IS NULL`, belongs in WHERE.
3. **NATURAL is a schema-change landmine** — joins on ALL same-named columns (A8); add a colliding column and the key set silently changes; no common columns → silent CROSS JOIN. Recommend explicit ON.
4. **USING vs ON** — USING = equality on explicitly listed shared names + dedups the column (A7); ON = general Boolean (A6). Prefer explicit ON (or USING with named columns) over NATURAL.
5. **Accidental cross product** — a missing or typo'd join predicate yields the Cartesian product, N×M rows (A1). CROSS JOIN should be explicit when intended.
6. **One-to-many fan-out** — joining a one-to-many without aggregation multiplies the "one" side's rows by the number of "many" matches, inflating any downstream SUM/COUNT. (Derived from INNER/LEFT semantics A2/A3: "a row for each row in T2 that satisfies" — multiple matches → multiple output rows.)
7. **Join-column nullability + 3VL** — `ON a.k = b.k` never matches when either key is NULL (`NULL = NULL` is UNKNOWN; foundation skill). Null-safe match needs `IS NOT DISTINCT FROM`. Cross-links foundation.
