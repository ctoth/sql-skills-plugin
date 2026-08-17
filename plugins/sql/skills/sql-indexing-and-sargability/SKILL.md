---
name: sql-indexing-and-sargability
description: >-
  Guides portable index design and the sargability rule — an index is a sorted B-tree, so the database can use it only when the predicate leaves the indexed column bare. Bans the recurring index-killers — wrapping an indexed column in a function or expression (`WHERE LOWER(email) = …`, `WHERE DATE(ts) = …`, `WHERE col + 0 = …`) and leading-wildcard `LIKE '%term'`, both of which force a full table scan. Auto-invokes when writing or editing `CREATE INDEX`, a slow `WHERE`/`ORDER BY`/`JOIN`/`GROUP BY`, a function or arithmetic applied to a filtered column, a `LIKE` pattern, or on "why is this query slow" / "what index do I need" / "this query does a full table scan" requests. The highest-leverage performance skill, taught vendor-neutrally.
allowed-tools: Read, Glob, Grep
---

# SQL Indexing and Sargability

> "Although there is an index on `LAST_NAME`, it is unusable—because the search is _not_ on `LAST_NAME` but on `UPPER(LAST_NAME)`. ... The `UPPER` function is just a black box."
> — [Use The Index, Luke! — Case-Insensitive Search](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search)

> "If the table contains N elements, the time required to look up the desired row is proportional to logN rather than being proportional to N as in a full table scan. If the table contains 10 million elements, that means the query will be on the order of N/logN or about 1 million times faster."
> — [SQLite — The Query Optimizer Overview](https://www.sqlite.org/queryplanner.html)

An index is a **sorted copy of one or more columns** that the database can binary-search instead of reading the whole table. That single fact decides everything below: an index helps a query only when the predicate is shaped so the database can seek into the sorted structure — when the column appears **bare**, not buried inside a function, an expression, or behind a leading wildcard. The standard name for "shaped so the index can be used" is **sargable** (Search ARGument ABLE). This is the highest-leverage performance skill in the plugin; it assumes the set semantics and three-valued logic of `sql-relational-and-null-discipline` and routes plan-reading to `sql-explain-and-set-based-thinking`.

---

## 1. B-Tree Anatomy — Why an Index Supports `=`, Ranges, and Ordering

A database index "combines two data structures ... a doubly linked list and a search tree" ([Use The Index, Luke! — Anatomy](https://use-the-index-luke.com/sql/anatomy)). The **leaf nodes** hold the indexed columns plus a pointer back to the row (`ROWID`/`RID`) and are joined into a doubly linked list — "databases use doubly linked lists to connect the so-called index _leaf nodes_," which "enables the database to read the index forwards or backwards as needed" ([Use The Index, Luke! — The Leaf Nodes](https://use-the-index-luke.com/sql/anatomy/the-leaf-nodes)). On top sits a **balanced tree** whose depth shows "logarithmic growth ... the tree depth grows very slowly compared to the number of leaf nodes," so "real world indexes with millions of records have a tree depth of four or five" ([Use The Index, Luke! — The Tree](https://use-the-index-luke.com/sql/anatomy/the-tree)).

Three query shapes fall straight out of "sorted list you binary-search into":

- **Equality (`=`)** — walk the tree to the one matching value. "Almost instantly—even on a huge data set" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/anatomy/the-tree)).
- **Range (`<`, `>`, `BETWEEN`, `LIKE 'abc%'`)** — seek to the start of the range, then walk the leaf list.
- **Ordering (`ORDER BY`, `GROUP BY`)** — the leaves are *already* in sorted order, so reading them in order is free (§5).

Without an index the only option is a **full table scan**: "the entire content of the table must be read and examined in order to find the one row of interest ... if the table contained 7 million rows, a full table scan might read megabytes of content in order to find a single 8-byte number" ([SQLite — Query Optimizer](https://www.sqlite.org/queryplanner.html)). The whole game is keeping your predicates in a shape the tree can seek.

One vocabulary distinction makes the rest of this skill precise. An **access predicate** is "the start and stop conditions for an index lookup ... [it defines] the scanned index range"; a **filter predicate** is "applied during the leaf node traversal only [and does] not narrow the scanned index range" ([Use The Index, Luke! — Access/Filter Predicates](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates)). A sargable predicate becomes an access predicate (a fast seek); a non-sargable one is, at best, a filter applied to every row the scan already read. The goal is to maximize what the database can do as an access predicate.

---

## 2. Sargability — Bare Column vs Function-Wrapped (the centerpiece)

The index stores `column`. If the query filters on `f(column)`, the optimizer sees a black box, not the indexed value: "The parameters to the function are not relevant because there is no general relationship between the function's parameters and the result" ([Use The Index, Luke! — Case-Insensitive Search](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search)). The index on the bare column is **unusable** and the query falls back to a full scan. This is the single most common performance bug in generated SQL.

```sql
-- WRONG — every one of these wraps the indexed column, so the index is dead:
SELECT * FROM users   WHERE LOWER(email) = 'q@example.com';   -- function on column
SELECT * FROM events  WHERE DATE(created_at) = '2026-06-26';  -- function on column
SELECT * FROM orders  WHERE total + 0 = 100;                  -- arithmetic on column
SELECT * FROM users   WHERE email LIKE '%@example.com';       -- leading wildcard (§3)
```

Three sargable fixes — pick by situation:

```sql
-- RIGHT (a) — leave the column BARE; turn a function-on-a-date into a range
SELECT * FROM events
WHERE created_at >= DATE '2026-06-26'
  AND created_at <  DATE '2026-06-27';   -- index on created_at seeks the range

-- RIGHT (b) — store the normalized form and index/query THAT bare column
--   (e.g. an email_lower column kept lowercase on write), then:
SELECT * FROM users WHERE email_lower = 'q@example.com';

-- RIGHT (c) — build a function-based / expression index whose definition
--   EXACTLY matches the predicate
CREATE INDEX users_email_lower ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'q@example.com';   -- now sargable
```

"To support that query, we need an index that covers the actual search term ... `CREATE INDEX emp_up_name ON employees (UPPER(last_name))`," and "the database can use a function-based index if the _exact_ expression of the index definition appears in an SQL statement" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search)). Only deterministic expressions can be indexed, and engines differ on declaring it (PostgreSQL `IMMUTABLE`, Oracle `DETERMINISTIC`, MySQL 8.0+ functional indexes, SQL Server via a computed column) — "Db2 (LUW) cannot use user-defined functions in indexes" at all ([Use The Index, Luke! — User-Defined Functions](https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions)). Route spellings to `sql-standard-vs-dialect-map`.

The same trap hides in **implicit type conversion**: comparing an indexed column to a value of a different type can make the database wrap the *column* in a cast — `WHERE varchar_col = 42` may become `WHERE TO_NUMBER(varchar_col) = 42`, which is a function on the column and defeats the index exactly like `LOWER(...)` does. Compare like types (`WHERE varchar_col = '42'`), or fix the column's type, so the column stays bare.

---

## 3. Leading-Wildcard `LIKE` Can't Use a B-Tree

A B-tree is sorted left-to-right like a dictionary, so `LIKE 'abc%'` is a range (everything from `abc` up to the next prefix) and uses the index, but `LIKE '%abc'` has no known starting point. "Only the part before the first wild card serves as an access predicate," and a `LIKE` expression that "starts with a wild card ... cannot serve as an access predicate" ([Use The Index, Luke! — LIKE](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning)). With no prefix to seek to, "the database has to scan the entire table."

```sql
-- WRONG — leading % means there is nothing to seek to: full scan
SELECT * FROM products WHERE name LIKE '%phone';

-- RIGHT — a trailing wildcard is a sargable range on the prefix
SELECT * FROM products WHERE name LIKE 'iphone%';
```

Substring / "contains" / full-text search is a different problem that needs a different index type (trigram, inverted, full-text) — those are engine-specific; route to vendor plugins and `sql-standard-vs-dialect-map`.

---

## 4. Composite-Index Column Order — Equality First, Then the Range/Sort Column

A multi-column index is sorted by its leftmost column first, then the next as a tiebreak: "the left-most column is the primary key used for ordering ... the second column is used to break ties in the left-most column" ([SQLite — Query Optimizer](https://www.sqlite.org/queryplanner.html)). Two rules follow, and they are the #1 composite-index mistakes:

**Leftmost-prefix:** "a database can use a concatenated index when searching with the leading (leftmost) columns," and "a two-column index does not support searching on the second column alone; that would be like searching a telephone directory by first name" ([Use The Index, Luke! — Concatenated Keys](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys)).

**Equality first, then range:** "Rule of thumb: index for equality first—then for ranges" ([Use The Index, Luke! — Access/Filter Predicates](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates)). A range on an early column "uses up" the sort — every column after the first range can only filter during leaf traversal, not narrow the seek: "index filter predicates ... do not narrow the scanned index range."

```sql
-- Query: WHERE tenant_id = 42 AND created_at >= '2026-01-01' ORDER BY created_at

-- WRONG — range column first; tenant_id after it can't narrow the seek
CREATE INDEX bad ON events (created_at, tenant_id);

-- RIGHT — equality column first, then the range/sort column
CREATE INDEX good ON events (tenant_id, created_at);
--   seeks straight to tenant 42, then walks created_at in order (serves the ORDER BY free, §5)
```

Put **all** equality columns first (their order among themselves barely matters for this query), then the single column used for the range or the `ORDER BY`.

---

## 5. Covering Index / Index-Only Scan, and One Index for `ORDER BY` + `GROUP BY`

**Covering index (index-only scan).** If the index already contains every column the query reads, the database never touches the table: "to cover an entire query, an index must contain _all_ columns from the SQL statement," and then "accessing the table is not required because the index has all of the information to satisfy the query" ([Use The Index, Luke! — Index-Only Scan](https://use-the-index-luke.com/sql/clustering/index-only-scan-covering-index)). "By adding extra 'output' columns onto the end of an index, one can avoid having to reference the original table and thereby cut the number of binary searches for a query in half" ([SQLite](https://www.sqlite.org/queryplanner.html)). It "is one of the most powerful tuning methods of all" — but the win "depends on the number of accessed rows," so check the plan before widening every index.

```sql
-- Query: SELECT total FROM orders WHERE tenant_id = 42;
CREATE INDEX orders_cover ON orders (tenant_id, total);   -- total is in the index -> no table fetch
```

**Ordering and grouping for free.** Because the leaves are presorted, one index can filter the `WHERE` *and* feed the `ORDER BY`/`GROUP BY` with no sort step: an index "stores the data in a presorted fashion ... just like when using the index definition in an `order by` clause," and an indexed order by "is also able to return the first results without processing all input data" — Winand calls this pipelined order by "the third power of indexing" ([Use The Index, Luke! — Sorting and Grouping](https://use-the-index-luke.com/sql/sorting-grouping)). The same `(tenant_id, created_at)` index from §4 filters on `tenant_id` and returns rows already ordered by `created_at`. This is exactly what makes keyset pagination cheap — route to `sql-pagination-and-keyset`.

---

## 6. The Index Shotgun — Every Index Has a Write Cost

Indexes are not free: each one must be updated on every write. "The number of indexes on a table is the most dominant factor for `insert` performance. The more indexes a table has, the slower the execution becomes," because the database "has to add the new entry to each and every index on that table" — "the number of indexes is therefore a multiplier for the cost of an `insert` statement," and the same drag applies to `UPDATE` and `DELETE` ([Use The Index, Luke! — The Insert Statement](https://use-the-index-luke.com/sql/dml/insert)).

The **index shotgun** antipattern is creating one single-column index per column "just in case." It bloats write latency and storage, and most of those indexes are redundant — a composite `(a, b, c)` already serves queries on `a` and on `a, b` via the leftmost-prefix rule (§4), so separate single-column indexes on `a` and `b` add cost without adding reach.

```sql
-- WRONG — index shotgun: redundant single-column indexes, every write pays for all of them
CREATE INDEX i1 ON orders (tenant_id);
CREATE INDEX i2 ON orders (created_at);
CREATE INDEX i3 ON orders (status);
CREATE INDEX i4 ON orders (tenant_id, created_at);   -- i1 is now redundant with this

-- RIGHT — index deliberately for the queries you actually run
CREATE INDEX orders_tenant_created ON orders (tenant_id, created_at);  -- covers tenant_id alone too
```

**What to index:** selective predicates — columns whose equality/range narrows the result to a small fraction of rows (high-cardinality foreign keys, the columns in your hot `WHERE`/`JOIN`/`ORDER BY`). Indexing a low-selectivity column (a 3-value `status`) rarely beats a scan and still costs on every write. "Use indexes deliberately and sparingly, and avoid redundant indexes whenever possible" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/dml/insert)). Prove selectivity by reading the plan — `sql-explain-and-set-based-thinking`.

---

## 7. Portability

The *mechanics* in this skill are universal — B-trees, sargability, the leftmost-prefix and equality-first-then-range rules, covering indexes, and the per-write maintenance cost behave the same across PostgreSQL, MySQL, SQLite, Oracle, and SQL Server, because they all index with B-trees by default.

| Concept | Portable? | Notes |
|---|---|---|
| B-tree `=`/range/ordered access; sargability | universal | the core of this skill |
| Leftmost-prefix; equality-first-then-range | universal | composite column-order logic |
| `CREATE INDEX name ON t (cols)` | near-universal | **not** in the core SQL standard, but spelled the same everywhere |
| Function/expression index | yes, spelled differently | PG `IMMUTABLE`, Oracle `DETERMINISTIC`, MySQL 8.0+ functional, SQL Server computed column |
| Covering "payload" columns | yes, spelled differently | PostgreSQL 11+/SQL Server `INCLUDE (...)`; others append to the key |
| Engine-specific index types | **not portable** | GIN/GiST/BRIN (PG), hash, bitmap (Oracle), full-text/trigram — vendor plugins |
| Partial/filtered index, `NULLS FIRST/LAST` in index | engine-divergent | route to `sql-standard-vs-dialect-map` |

The SQL standard defines query and constraint syntax but **not** index DDL — `CREATE INDEX` is a near-universal de-facto extension. Treat the mechanics as portable; route every engine-specific spelling and index type to `sql-standard-vs-dialect-map` and vendor plugins.

---

## 8. Who Suffers When Indexing Is Wrong

A bad index decision rarely errors — it just makes the database do far more work than it should, and someone pays at the worst moment:

- The **on-call engineer** during a 2 a.m. outage, tracing a CPU-pegged database to one endpoint whose `WHERE LOWER(email) = ?` made the perfectly good index on `email` unusable (§2), so every login does a full table scan. The query was "valid" and fast in dev with 100 rows; at 10 million rows it took the site down.
- The **analyst** whose dashboard times out because the `tenant_id = ? ... ORDER BY created_at` query has no composite index, or has the columns in the wrong order (§4), forcing a scan-plus-sort over the whole table on every refresh.
- The **platform team** watching write throughput collapse after someone "added indexes to make it fast" — an index shotgun (§6) that multiplied the cost of every `INSERT` and `UPDATE`, turning a write-heavy table into a bottleneck while adding nothing the reads couldn't already get.

A sargable predicate, a correctly ordered composite index, and the *restraint* not to over-index are all gifts to whoever runs this under production load.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics and three-valued logic this skill assumes.
- `sql-explain-and-set-based-thinking` — read the plan to *prove* an index is used (scan vs seek), and the N+1/RBAR rewrite that no index can fix.
- `sql-pagination-and-keyset` — keyset pagination is the direct payoff of an index that serves both the `WHERE` and the `ORDER BY` (§5).
- `sql-joins` — join columns are prime index candidates; join-key indexing decides nested-loop vs hash/merge cost.
- `sql-standard-vs-dialect-map` — `CREATE INDEX` options, `INCLUDE`, function-index declarations, partial indexes, and engine-specific index types.

---

## 10. Reference Files

High-frequency indexing and sargability anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
