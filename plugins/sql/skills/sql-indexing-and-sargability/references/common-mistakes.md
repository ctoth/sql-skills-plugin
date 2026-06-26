# Common SQL Indexing & Sargability Mistakes

Anti-patterns in LLM-generated SQL around indexes and sargable predicates, each with wrong/right
code and a primary-source citation. The skill (`sql-indexing-and-sargability`) states the rules;
this file holds the high-frequency failure modes. All RIGHT examples use portable mechanics;
engine-specific spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Wrapping an indexed column in a function

**The problem:** The model writes `WHERE LOWER(email) = …` or `WHERE UPPER(name) = …` against a plain index on the column. The index stores the raw value, not the function result, so it is "unusable—because the search is _not_ on `LAST_NAME` but on `UPPER(LAST_NAME)`" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search)). The query degrades to a full table scan.

```sql
-- WRONG — index on email is dead; full scan
SELECT * FROM users WHERE LOWER(email) = 'q@example.com';

-- RIGHT — match the index to the predicate with a function-based index
CREATE INDEX users_email_lower ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'q@example.com';
-- ...or store a normalized email_lower column and query that bare column.
```

*Source: [Use The Index, Luke! — Case-Insensitive Search](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search). Depth: this skill, §2.*

---

## 2. Applying a date/time function to a timestamp column

**The problem:** `WHERE DATE(created_at) = '2026-06-26'` (or `EXTRACT(YEAR FROM ts) = 2026`) wraps the indexed timestamp, killing the index. The fix is a sargable half-open range on the bare column.

```sql
-- WRONG — DATE(created_at) is a black box to the index
SELECT * FROM events WHERE DATE(created_at) = '2026-06-26';

-- RIGHT — half-open range leaves created_at bare and seekable
SELECT * FROM events
WHERE created_at >= DATE '2026-06-26'
  AND created_at <  DATE '2026-06-27';
```

*Source: [Use The Index, Luke! — Case-Insensitive Search](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search). Depth: this skill, §2.*

---

## 3. Arithmetic on an indexed column

**The problem:** `WHERE total + 0 = 100`, `WHERE price * 1.2 > 50`, or `WHERE id - 1 = 41` all wrap the column in an expression, so the index can't be used. Move the arithmetic to the constant side, where it's evaluated once and the column stays bare.

```sql
-- WRONG — expression on the column side defeats the index
SELECT * FROM orders WHERE total * 1.2 > 100;

-- RIGHT — rearrange so the column is bare; the constant is folded once
SELECT * FROM orders WHERE total > 100 / 1.2;
```

*Source: [Use The Index, Luke! — User-Defined Functions](https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions). Depth: this skill, §2.*

---

## 4. Leading-wildcard `LIKE`

**The problem:** `LIKE '%term'` or `LIKE '%term%'` has no known prefix, so there is nothing for the B-tree to seek to: a "`LIKE` expression that starts with a wild card ... cannot serve as an access predicate" and the database "has to scan the entire table" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning)).

```sql
-- WRONG — leading % forces a full scan
SELECT * FROM products WHERE name LIKE '%phone';

-- RIGHT — a trailing wildcard is a sargable prefix range
SELECT * FROM products WHERE name LIKE 'iphone%';
-- For real substring/contains search, use a full-text or trigram index (vendor-specific).
```

*Source: [Use The Index, Luke! — LIKE Performance](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning). Depth: this skill, §3.*

---

## 5. Composite index with the range column first

**The problem:** A range predicate on the leading column "uses up" the sort, so later equality columns can no longer narrow the seek. "Rule of thumb: index for equality first—then for ranges" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates)).

```sql
-- Query: WHERE tenant_id = 42 AND created_at >= '2026-01-01'

-- WRONG — range column leads; tenant_id can't narrow the scan
CREATE INDEX bad ON events (created_at, tenant_id);

-- RIGHT — equality column first, then the range column
CREATE INDEX good ON events (tenant_id, created_at);
```

*Source: [Use The Index, Luke! — Access/Filter Predicates](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates). Depth: this skill, §4.*

---

## 6. Expecting an index on the second column of a composite to be used alone

**The problem:** With an index on `(a, b)`, the model writes `WHERE b = …` and expects an index seek. "A two-column index does not support searching on the second column alone; that would be like searching a telephone directory by first name" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys)).

```sql
-- Index exists: (tenant_id, created_at)

-- WRONG — filtering on created_at alone cannot use this index for a seek
SELECT * FROM events WHERE created_at >= '2026-01-01';

-- RIGHT — either lead the query with tenant_id, or create an index that leads with created_at
CREATE INDEX events_created ON events (created_at);
```

*Source: [Use The Index, Luke! — Concatenated Keys](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys). Depth: this skill, §4.*

---

## 7. The index shotgun — one single-column index per column

**The problem:** To "make it fast," the model adds an index to every column. Each index is maintained on every write: "the number of indexes is therefore a multiplier for the cost of an `insert` statement" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/dml/insert)), and most are redundant with a composite via the leftmost-prefix rule.

```sql
-- WRONG — redundant indexes; every INSERT/UPDATE pays for all of them
CREATE INDEX i1 ON orders (tenant_id);
CREATE INDEX i2 ON orders (tenant_id, created_at);   -- i1 is redundant with this

-- RIGHT — one well-ordered composite serves tenant_id alone AND (tenant_id, created_at)
CREATE INDEX orders_tenant_created ON orders (tenant_id, created_at);
```

*Source: [Use The Index, Luke! — The Insert Statement](https://use-the-index-luke.com/sql/dml/insert). Depth: this skill, §6.*

---

## 8. Indexing a low-selectivity column

**The problem:** The model indexes a column with few distinct values (a 2-3 value `status`, a boolean flag). The index can't narrow the result enough to beat a scan, yet it still costs on every write. Index selective predicates — high-cardinality columns in hot `WHERE`/`JOIN`/`ORDER BY` clauses — and prove it with the plan.

```sql
-- WRONG — status has 3 values; this index rarely beats a scan but always costs writes
CREATE INDEX orders_status ON orders (status);

-- RIGHT — index the selective column; combine with status only if a query needs it
CREATE INDEX orders_tenant ON orders (tenant_id);
-- (A partial index WHERE status = 'open' is an engine-specific option -> dialect map.)
```

*Source: [Use The Index, Luke! — The Insert Statement](https://use-the-index-luke.com/sql/dml/insert). Depth: this skill, §6; plan-reading owned by `sql-explain-and-set-based-thinking`.*

---

## 9. Missing covering columns — an avoidable table fetch per row

**The problem:** A hot read query seeks the index, then fetches each matching row from the table just to read one extra column. Adding that column to the index makes it covering: "accessing the table is not required because the index has all of the information to satisfy the query" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/clustering/index-only-scan-covering-index)).

```sql
-- Query: SELECT total FROM orders WHERE tenant_id = 42;

-- WRONG (suboptimal) — index seeks, then fetches each row for `total`
CREATE INDEX orders_tenant ON orders (tenant_id);

-- RIGHT — covering index answers the query from the index alone (index-only scan)
CREATE INDEX orders_cover ON orders (tenant_id, total);
-- PostgreSQL 11+/SQL Server: ... (tenant_id) INCLUDE (total) -> dialect map.
```

*Source: [Use The Index, Luke! — Index-Only Scan](https://use-the-index-luke.com/sql/clustering/index-only-scan-covering-index). Depth: this skill, §5.*

---

## 10. A separate sort when an index could have ordered the rows

**The problem:** A query filters and orders on related columns, but the index order doesn't match, forcing an explicit sort over the whole result. An index in the right order returns rows presorted — a "pipelined `order by`" that "is also able to return the first results without processing all input data" ([Use The Index, Luke!](https://use-the-index-luke.com/sql/sorting-grouping)).

```sql
-- Query: WHERE tenant_id = 42 ORDER BY created_at DESC

-- WRONG — index on tenant_id only; the ORDER BY needs a separate sort step
CREATE INDEX orders_tenant ON orders (tenant_id);

-- RIGHT — the composite filters AND returns rows already ordered by created_at
CREATE INDEX orders_tenant_created ON orders (tenant_id, created_at);
```

*Source: [Use The Index, Luke! — Sorting and Grouping](https://use-the-index-luke.com/sql/sorting-grouping). Depth: this skill, §5; keyset use owned by `sql-pagination-and-keyset`.*

---

## 11. Implicit type conversion that wraps the indexed column

**The problem:** Comparing an indexed column to a literal of a different type can make the database cast the *column* to match the literal — e.g. `WHERE phone = 12345` against a `VARCHAR` column becomes `TO_NUMBER(phone) = 12345`, a function on the column that defeats the index just like `LOWER(...)`. Compare like types instead.

```sql
-- WRONG — int literal forces an implicit cast of the varchar column; index unusable
SELECT * FROM users WHERE phone = 12345;

-- RIGHT — compare a string to the string column; the column stays bare
SELECT * FROM users WHERE phone = '12345';
```

*Source: [Use The Index, Luke! — User-Defined Functions](https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions). Depth: this skill, §2.*
