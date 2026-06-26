# Common SQL Pagination & Row-Value Mistakes

Anti-patterns in LLM-generated SQL around pagination, keyset/seek cursors, and row-value comparisons,
each with wrong/right code and a primary-source citation. The skill (`sql-pagination-and-keyset`) states
the rules; this file holds the high-frequency failure modes. All RIGHT examples favor standard/portable
SQL; non-standard spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. Deep `OFFSET` for pagination (the scan-and-discard footgun)

**The problem:** The model paginates with `LIMIT n OFFSET m` and the offset grows without bound. The
database "must still fetch these rows from the disk and bring them in order before it can send the
following ones" ([Winand — no-offset](https://use-the-index-luke.com/no-offset)), so page 50,000 is O(50,000×n)
work and the endpoint eventually times out. Switch to a keyset cursor anchored on the last row's values.

```sql
-- WRONG — fetches, sorts, and discards a million rows to return twenty
SELECT id, title FROM posts
 ORDER BY created_at DESC, id DESC
 LIMIT 20 OFFSET 1000000;

-- RIGHT — keyset/seek: read only the page returned
SELECT id, title FROM posts
 WHERE (created_at, id) < (:last_created_at, :last_id)
 ORDER BY created_at DESC, id DESC
 FETCH FIRST 20 ROWS ONLY;
```

*Source: [Winand — no-offset](https://use-the-index-luke.com/no-offset). Depth: this skill, §2.*

---

## 2. `OFFSET` paging an actively-written list (skips and duplicates)

**The problem:** Even at small offsets, `LIMIT/OFFSET` is unstable. The offset is just "a number of rows
to be dropped — no more context" ([Winand](https://use-the-index-luke.com/no-offset)). Insert a row before
the offset point between page loads and "you'll get duplicates in case there were new rows inserted between
fetching two pages" ([Winand](https://use-the-index-luke.com/no-offset)); a delete makes the reader skip a
row. Infinite feeds show the same item twice or lose one.

```sql
-- WRONG — page 2 dupes/skips if rows are inserted/deleted before the offset
SELECT * FROM feed ORDER BY posted_at DESC, id DESC LIMIT 20 OFFSET 20;

-- RIGHT — cursor on values is stable across concurrent inserts/deletes
SELECT * FROM feed
 WHERE (posted_at, id) < (:last_posted_at, :last_id)
 ORDER BY posted_at DESC, id DESC
 FETCH FIRST 20 ROWS ONLY;
```

*Source: [Winand — no-offset](https://use-the-index-luke.com/no-offset); [Winand — seek](https://use-the-index-luke.com/sql/partial-results/fetch-next-page). Depth: this skill, §2–§3.*

---

## 3. Keyset (or any paging) on a non-unique `ORDER BY`

**The problem:** The model orders by a non-unique column (a timestamp, a name) with no tiebreaker. The
boundary between pages is ambiguous: rows sharing the key are split arbitrarily, so the cursor re-reads or
skips them. PostgreSQL warns you need "an `ORDER BY` clause that constrains the result rows into a unique
order. Otherwise you will get an unpredictable subset" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

```sql
-- WRONG — created_at is not unique; the cursor < :last_created_at
-- either re-reads or skips the rows that share :last_created_at
SELECT * FROM posts
 WHERE created_at < :last_created_at
 ORDER BY created_at DESC
 FETCH FIRST 20 ROWS ONLY;

-- RIGHT — total order: tie-break on a unique column, and compare as a row value
SELECT * FROM posts
 WHERE (created_at, id) < (:last_created_at, :last_id)
 ORDER BY created_at DESC, id DESC
 FETCH FIRST 20 ROWS ONLY;
```

*Source: [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §3; undefined-order rule owned by `sql-relational-and-null-discipline`.*

---

## 4. Hand-rolled `OR` expansion of the keyset predicate (operator flipped)

**The problem:** The model writes the cursor as `(a < ? OR (a = ? AND b < ?))` and gets a leg wrong — flips
an operator, drops the equality conjunct, or mismatches the `ORDER BY` direction. The row-value comparison
`(a, b) < (?, ?)` is the same lexicographic meaning in one predicate and cannot have a half-wrong leg: "the
row elements are compared left-to-right, stopping as soon as an unequal or null pair of elements is found"
([PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html)).

```sql
-- WRONG — verbose and easy to break; here the second leg uses > instead of <
WHERE created_at < :d OR (created_at = :d AND id > :i)

-- RIGHT — one row-value predicate, lexicographic, matches ORDER BY created_at, id
WHERE (created_at, id) < (:d, :i)
```

*Source: [PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html); [Winand — seek](https://use-the-index-luke.com/sql/partial-results/fetch-next-page). Depth: this skill, §4. (On engines lacking row values — SQLite, SQL Server — the manual expansion is required; route to `sql-standard-vs-dialect-map`.)*

---

## 5. Keyset cursor on a nullable sort key (rows vanish)

**The problem:** The model builds a keyset cursor on a column that can be NULL. Row comparison inherits
three-valued logic: "If either of this pair of elements is null, the result of the row comparison is unknown
(null)" ([PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html)),
and `WHERE` drops UNKNOWN — so rows with a NULL in the deciding key silently disappear from paging.

```sql
-- WRONG — ended_at is nullable; rows with ended_at IS NULL never page
SELECT * FROM jobs
 WHERE (ended_at, id) < (:last_ended_at, :last_id)
 ORDER BY ended_at DESC, id DESC
 FETCH FIRST 20 ROWS ONLY;

-- RIGHT — page on a non-nullable key (or a COALESCE'd/NULLS-pinned expression)
SELECT * FROM jobs
 WHERE (created_at, id) < (:last_created_at, :last_id)
 ORDER BY created_at DESC, id DESC
 FETCH FIRST 20 ROWS ONLY;
```

*Source: [PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html). Depth: this skill, §4; NULL/3VL theory owned by `sql-relational-and-null-discipline`.*

---

## 6. Column-wise `IN` where a multi-row row-value `IN` was meant

**The problem:** The model wants to match specific `(tenant, sku)` pairs but writes
`tenant_id IN (...) AND sku IN (...)`, which matches the full cross product — far more rows than intended.
A row-value `IN` matches the *combinations*.

```sql
-- WRONG — matches any tenant in {1,7} crossed with any sku in {'A-100','B-200'}
SELECT * FROM inventory WHERE tenant_id IN (1, 7) AND sku IN ('A-100', 'B-200');

-- RIGHT — match the exact pairs
SELECT * FROM inventory
 WHERE (tenant_id, sku) IN ((1, 'A-100'), (7, 'B-200'));
```

*Source: [PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html). Depth: this skill, §5; `NOT IN` + NULL footgun owned by `sql-relational-and-null-discipline`.*

---

## 7. Row-by-row loop where a `VALUES` row-list is set-based

**The problem:** The model inserts or matches a list of rows by looping N single-row statements in
application code. A multi-row `VALUES` "computes a row value or set of row values" and "is syntactically
allowed anywhere that `SELECT` is" ([PostgreSQL — VALUES](https://www.postgresql.org/docs/current/sql-values.html))
— one statement, one round trip, set-based.

```sql
-- WRONG (shape) — N separate round trips from app code
-- INSERT INTO tags (post_id, label) VALUES (1, 'sql');
-- INSERT INTO tags (post_id, label) VALUES (1, 'paging');  ... ×N

-- RIGHT — one multi-row VALUES
INSERT INTO tags (post_id, label) VALUES (1, 'sql'), (1, 'paging'), (2, 'index');

-- RIGHT — VALUES as a derived table to JOIN an ad-hoc set
SELECT v.sku, v.want_qty, i.on_hand
  FROM (VALUES ('A-100', 5), ('B-200', 12)) AS v(sku, want_qty)
  JOIN inventory i ON i.sku = v.sku;
```

*Source: [PostgreSQL — VALUES](https://www.postgresql.org/docs/current/sql-values.html). Depth: this skill, §6.*

---

## 8. Assuming `FETCH FIRST` / `WITH TIES` works everywhere

**The problem:** The model writes `FETCH FIRST n ROWS ONLY` for MySQL or SQLite. "MySQL and SQLite do not
support `fetch` at all" ([Zhiyanov — LIMIT vs. FETCH](https://antonz.org/sql-fetch/)); they have only the
non-standard `LIMIT`. Conversely `LIMIT` is not standard ("There is no `limit` clause in the SQL standard"),
so it is not portable to Oracle/SQL Server/DB2. `WITH TIES` is also unsupported on MySQL/SQLite.

```sql
-- WRONG — fails on MySQL/SQLite, which have no FETCH FIRST
SELECT * FROM posts ORDER BY created_at DESC, id DESC FETCH FIRST 20 ROWS ONLY;  -- on MySQL/SQLite

-- RIGHT (portable across MySQL/SQLite/PostgreSQL) — non-standard but ubiquitous
SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 20;

-- RIGHT (standard; PostgreSQL/Oracle/SQL Server/DB2)
SELECT * FROM posts ORDER BY created_at DESC, id DESC FETCH FIRST 20 ROWS ONLY;
```

*Source: [Zhiyanov — LIMIT vs. FETCH](https://antonz.org/sql-fetch/); [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §1, §8; full dialect table owned by `sql-standard-vs-dialect-map`.*
