# Research: sql-indexing-and-sargability

Skill #25 (plan line 356-364). Highest-leverage performance skill. Taught vendor-neutrally,
primarily from Markus Winand (use-the-index-luke.com / SQL Performance Explained) plus the
SQLite query-planner walkthrough as a concrete, readable B-tree story. Accessed 2026-06-26.

Format conventions matched against reference skill `sql-relational-and-null-discipline`
(SKILL.md + references/sources.yaml + references/common-mistakes.md).

---

## Source 1 — Winand, Anatomy of an Index (and its sub-pages)

URL: https://use-the-index-luke.com/sql/anatomy
URL: https://use-the-index-luke.com/sql/anatomy/the-leaf-nodes
URL: https://use-the-index-luke.com/sql/anatomy/the-tree

Section: "The database combines two data structures to meet the challenge."
Verbatim: "The database combines two data structures to meet the challenge: a doubly linked list
and a search tree."

Section: The Leaf Nodes.
Verbatim: "Databases use doubly linked lists to connect the so-called index _leaf nodes_."
Verbatim: "It enables the database to read the index forwards or backwards as needed."
Verbatim: "Each index entry consists of the indexed columns (the key, column 2) and refers to the
corresponding table row (via `ROWID` or `RID`)."

Section: The B-Tree.
Verbatim: "the logarithmic growth of the tree depth. That means that the tree depth grows very
slowly compared to the number of leaf nodes."
Verbatim: "The tree depth is therefore log4(number-of-index-entries)." / "Real world indexes
with millions of records have a tree depth of four or five."
Verbatim: "The B-tree enables the database to find a leaf node quickly." Described as "the
_first power of indexing_. It works almost instantly—even on a huge data set."

WHY IT MATTERS: This is the mechanical foundation for the whole skill. The leaf-node doubly
linked list (kept in sorted order, independent of physical row storage) is *why* an index
supports range scans and ordered reads, not just equality; the balanced tree on top is *why*
the equality lookup is logarithmic. Every sargability rule downstream is a consequence of "the
index is a sorted structure you binary-search into and then walk." Use this to motivate
=, range, and ORDER BY support from one diagram.

---

## Source 2 — SQLite, The SQLite Query Optimizer Overview (concrete B-tree walkthrough)

URL: https://www.sqlite.org/queryplanner.html

Section: Full table scan.
Verbatim: "To satisfy this query, SQLite reads every row out of the table, checks to see if the
'fruit' column has the value of 'Peach' and if so, outputs the 'price' column from that row...
This algorithm is called a _full table scan_ since the entire content of the table must be read
and examined in order to find the one row of interest. With a table of only 7 rows, a full table
scan is acceptable, but if the table contained 7 million rows, a full table scan might read
megabytes of content in order to find a single 8-byte number."

Section: Binary search on an index.
Verbatim: "If the table contains N elements, the time required to look up the desired row is
proportional to logN rather than being proportional to N as in a full table scan. If the table
contains 10 million elements, that means the query will be on the order of N/logN or about 1
million times faster."

Section: Multi-column index.
Verbatim: "A multi-column index follows the same pattern as a single-column index; the indexed
columns are added in front of the rowid... The left-most column is the primary key used for
ordering the rows in the index. The second column is used to break ties in the left-most column.
If there were a third column, it would be used to break ties for the first two columns. And so
forth for all columns in the index."

Section: Covering index.
Verbatim: "This new index contains all the columns of the original FruitsForSale table that are
used by the query - both the search terms and the output. We call this a 'covering index'.
Because all of the information needed is in the covering index, SQLite never needs to consult the
original table in order to find the price... by adding extra 'output' columns onto the end of an
index, one can avoid having to reference the original table and thereby cut the number of binary
searches for a query in half."

WHY IT MATTERS: Gives a vendor-neutral, intuitive number ("about 1 million times faster") that
makes the O(N) -> O(log N) stakes concrete, and a plain-language definition of how a multi-column
index orders by leftmost-then-tiebreak. This is the readable counterpart to Winand's more formal
treatment and grounds the "why a missing index hurts" argument.

---

## Source 3 — Winand, The WHERE Clause / Functions (SARGABILITY centerpiece)

URL: https://use-the-index-luke.com/sql/where-clause
URL: https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search
URL: https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions

Section: Case-insensitive search — function wrapping defeats the index.
Verbatim: "Although there is an index on `LAST_NAME`, it is unusable—because the search is _not_ on
`LAST_NAME` but on `UPPER(LAST_NAME)`."
Verbatim: "The `UPPER` function is just a black box. The parameters to the function are not
relevant because there is no general relationship between the function's parameters and the
result."

Section: The fix — function-based index.
Verbatim: "To support that query, we need an index that covers the actual search term. That means
we do not need an index on `LAST_NAME` but on `UPPER(LAST_NAME)`: `CREATE INDEX emp_up_name ON
employees (UPPER(last_name))`"
Verbatim: "The database can use a function-based index if the _exact_ expression of the index
definition appears in an SQL statement."

Section: User-defined functions / function-based indexing generality.
Verbatim: "Function-based indexing is a very generic approach. Besides functions like `UPPER` you
can also index expressions like `A + B`..."
Verbatim: "Only functions that always return the same result for the same parameters—functions
that are deterministic—can be indexed."
Verbatim: "PostgreSQL and the Oracle database require functions to be _declared_ to be
deterministic when used in an index so you have to use the keyword `DETERMINISTIC` (Oracle) or
`IMMUTABLE` (PostgreSQL)."
Verbatim: "Db2 (LUW) cannot use user-defined functions in indexes (not even if they are
deterministic)."

WHY IT MATTERS: THE CENTERPIECE. The index is on the bare column; the moment the query searches
on `f(column)` instead of `column`, the optimizer sees a black box it cannot match to the sorted
index, so it falls back to a full scan. This is the single most common LLM-generated performance
bug: `WHERE LOWER(email) = ...`, `WHERE DATE(ts) = ...`, `WHERE col + 0 = ...`. The fixes are:
(a) leave the column bare and adapt the query (e.g. a range on `ts` instead of `DATE(ts)`),
(b) store a normalized form, or (c) build a function-based / expression index whose definition
*exactly* matches the predicate. "Sargable" (Search ARGument ABLE) is the standard name for the
class of predicate the index can use; Winand teaches it as access-predicate vs filter-predicate
but the principle is identical.

---

## Source 4 — Winand, Indexing LIKE filters (leading wildcard)

URL: https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning

Verbatim: "Avoid `LIKE` expressions with leading wildcards (e.g., `'%TERM'`)."
Verbatim: "Only the part before the first wild card serves as an access predicate."
Verbatim: "The remaining characters do not narrow the scanned index range—non-matching entries
are just left out of the result."
Verbatim: "A `LIKE` expression that starts with a wild card... cannot serve as an access
predicate."
Verbatim: "The database has to scan the entire table if there are no other conditions that
provide access predicates."

WHY IT MATTERS: A B-tree is sorted left-to-right like a dictionary. `LIKE 'abc%'` is a range
(everything from `abc` up to `abd`) and uses the index; `LIKE '%abc'` has no known prefix, so
there is nothing to seek to — it degenerates to a full scan. Same root cause as the function
rule: the index can only help when the *leading edge* of the value is pinned. Full-text search
needs a different index type (route to vendor plugins).

---

## Source 5 — Winand, Concatenated (composite) index column order

URL: https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys
URL: https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates

Section: Column order matters.
Verbatim: "Note that the column order of a concatenated index has great impact on its usability so
it must be chosen carefully."
Verbatim: "A database can use a concatenated index when searching with the leading (leftmost)
columns."
Verbatim: "The database does not use the index because it cannot use single columns from a
concatenated index arbitrarily."
Verbatim: "A two-column index does not support searching on the second column alone; that would be
like searching a telephone directory by first name."
Verbatim: "The most important consideration when defining a concatenated index is how to choose
the column order so it can be used as often as possible."

Section: Equality first, then range (THE #1 composite mistake).
Verbatim: "Rule of thumb: index for equality first—then for ranges."
Verbatim: "The _access predicates_ are the start and stop conditions for an index lookup. They
define the scanned index range. _Index filter predicates_ are applied during the leaf node
traversal only. They do not narrow the scanned index range."
Verbatim: "The difference is that the equals operator limits the first index column to a single
value. Within the range for this value... the index is sorted according to the second column."
Verbatim (effect of a range too early): "The ordering of `SUBSIDIARY_ID` is therefore useless
during tree traversal... The filter on `DATE_OF_BIRTH` is therefore the only condition that
limits the scanned index range."

WHY IT MATTERS: The leftmost-prefix rule + equality-before-range is the #1 composite-index
mistake. Because the index is sorted leftmost-column-first, a range predicate on an early column
"uses up" the sort: every column after the first range can only be a filter, not an access
predicate, so it stops narrowing the seek. The correct order puts all equality columns first
(in any order among themselves), then the single column used for the range or the ORDER BY.

---

## Source 6 — Winand, Clustering / Index-Only Scan (covering index)

URL: https://use-the-index-luke.com/sql/clustering
URL: https://use-the-index-luke.com/sql/clustering/index-only-scan-covering-index

Verbatim: "The index-only scan is one of the most powerful tuning methods of all."
Verbatim: "To cover an entire query, an index must contain _all_ columns from the SQL statement."
Verbatim: "The index has a copy of the `EUR_VALUE` column so the database can use the value stored
in the index. Accessing the table is not required because the index has all of the information to
satisfy the query."
Verbatim: "The performance advantage of an index-only scan depends on the number of accessed rows
and the index clustering factor."
Verbatim (caution): "Extending the `where` clause can cause 'illogical' performance behavior.
Check the execution plan before extending queries."

WHY IT MATTERS: A covering index lets the query be answered from the index alone — no second
lookup into the table heap per row. For a query that touches many rows this can be the difference
between fast and unusable. The cost is a wider index (more write/storage). Engines spell the
"extra payload columns" differently — Postgres/SQL Server `INCLUDE`, others just append columns —
route spellings to the dialect map.

---

## Source 7 — Winand, Sorting and Grouping (one index serving WHERE + ORDER BY + GROUP BY)

URL: https://use-the-index-luke.com/sql/sorting-grouping

Verbatim: "An index provides an ordered representation of the indexed data... stores the data in a
presorted fashion."
Verbatim: "The index is, in fact, sorted just like when using the index definition in an `order
by` clause."
Verbatim: "An indexed `order by` execution not only saves the sorting effort, however; it is also
able to return the first results without processing all input data."
Verbatim: "Pipelined `order by` is the third power of indexing."
Verbatim: "This chapter explains how to use an index for a pipelined `order by` execution... The
chapter concludes by applying these techniques to `group by` clauses as well."

WHY IT MATTERS: Because leaf nodes are already in sorted order, one index can do triple duty:
filter the WHERE, return rows in ORDER BY order without a sort step (a "pipelined" order by that
streams the first rows immediately), and feed GROUP BY a presorted stream. This is the payoff for
getting composite column order right (equality columns -> the ORDER BY column). It directly
underpins keyset pagination (route to sql-pagination-and-keyset).

---

## Source 8 — Winand, The Insert Statement (DML write cost of indexes; index shotgun)

URL: https://use-the-index-luke.com/sql/dml/insert

Verbatim: "The number of indexes on a table is the most dominant factor for `insert`
performance. The more indexes a table has, the slower the execution becomes."
Verbatim: "If there are indexes on the table, the database must make sure the new entry is also
found via these indexes. For this reason it has to add the new entry to each and every index on
that table."
Verbatim: "The index maintenance is, after all, the most expensive part of the `insert`
operation."
Verbatim: "The number of indexes is therefore a multiplier for the cost of an `insert`
statement."
Verbatim: "adding a single index is enough to increase the execute time by a factor of a hundred.
Each additional index slows the execution down further."
Verbatim: "Use indexes deliberately and sparingly, and avoid redundant indexes whenever possible.
This is also beneficial for `delete` and `update` statements."

WHY IT MATTERS: The counterweight to "just add an index." Every index must be maintained on every
INSERT/UPDATE/DELETE, so the "index shotgun" antipattern (one single-column index per column,
many of them redundant because a composite already covers the leading column) silently destroys
write throughput while contributing nothing reads can't already get. The discipline: index
deliberately, prefer one well-ordered composite over many single-column indexes, and remember a
read win can be a write loss.

---

## Synthesis / skill outline (for the draft)

Centerpiece = SARGABILITY (function-wrapped vs bare column). Section order:
1. B-tree anatomy — doubly linked leaf list + balanced tree => supports =, range, ordered read.
   (Sources 1, 2)
2. SARGABILITY centerpiece — bare column wins; WRONG `LOWER()/DATE()/col+0/'%x'` vs RIGHT bare
   column / range rewrite / function-based index. (Sources 3, 4)
3. Composite column order — leftmost prefix + equality-first-then-range. (Source 5)
4. Covering index / index-only scan. (Sources 2, 6)
5. One index for ORDER BY + GROUP BY. (Source 7)
6. Index shotgun + DML write cost; what to index (selective predicates). (Source 8)
7. Portability block: B-tree + sargability universal; CREATE INDEX ubiquitous though not core
   SQL standard; engine-specific index types (GIN/GiST/hash/bitmap/INCLUDE) -> vendor plugins;
   route to sql-standard-vs-dialect-map.
8. Who suffers: prod outage from full scan (LOWER(email)= killed index); slow dashboard from
   missing composite; write-throughput collapse from index shotgun.
9. Routing: foundation (sql-relational-and-null-discipline), explain-and-set-based-thinking,
   pagination-and-keyset, joins, standard-vs-dialect-map.

Portability facts to assert:
- B-tree, sargability, leftmost-prefix, equality-first-then-range, covering index, index write
  cost: UNIVERSAL across PostgreSQL/MySQL/SQLite/Oracle/SQL Server.
- `CREATE INDEX` is near-universal but NOT part of the core SQL standard (the standard does not
  define index DDL); spelling is consistent enough to treat as portable. (Assert as widely
  supported, route exact options to dialect map.)
- Function-based / expression indexes: PostgreSQL & Oracle yes (need IMMUTABLE/DETERMINISTIC),
  SQLite yes (expression indexes), MySQL 8.0+ functional indexes, SQL Server via computed column.
- Covering "payload" columns: Postgres 11+/SQL Server `INCLUDE`; others append to key.
- NULLS FIRST/LAST in index ordering and partial indexes are engine-divergent -> dialect map.
