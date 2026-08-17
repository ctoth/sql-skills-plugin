---
name: sql-pagination-and-keyset
description: >-
  Guides correct, scalable pagination and the row-value machinery behind it. Prefer keyset/seek pagination — `WHERE (sort_key, id) greater than (:last_key, :last_id) ORDER BY sort_key, id FETCH FIRST n ROWS ONLY` — over `LIMIT n OFFSET m`, because `OFFSET` makes the server fetch-and-discard the first m rows (O(offset), so deep pages time out) and is unstable — an insert or delete between page loads shifts the window so the reader skips a row or sees one twice. Teaches the standard `OFFSET. Auto-invokes when writing or editing pagination, `LIMIT`/`OFFSET`/`FETCH` queries, infinite-scroll or cursor/keyset endpoints, multi-column `ORDER BY` with paging, row-value comparisons, or `VALUES` bulk row sets.
allowed-tools: Read, Glob, Grep
---

# SQL Pagination and Keyset

> "Offset instructs the DBMS skip the first N results of a query. However, the database must still fetch these rows from the disk and bring them in order before it can send the following ones."
> — [Markus Winand — We need tool support for keyset pagination (no-offset)](https://use-the-index-luke.com/no-offset)

> "The seek method avoids both problems because it uses the _values_ of the previous page as a delimiter. ... You will also get stable results if new rows are inserted."
> — [Markus Winand — Paging Through Results (the seek method)](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)

Pagination is where the foundation skill's first rule — *a result is a set with no order unless you write `ORDER BY`* (`sql-relational-and-null-discipline`) — stops being academic. A page is "rows 10 through 20 in some order"; if that order is not **total and stable**, every page is a different lie. PostgreSQL says it plainly: without a unique `ORDER BY` "you will get an unpredictable subset of the query's rows — you might be asking for the tenth through twentieth rows, but tenth through twentieth in what ordering?" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). This skill builds two things on that floor: the standard limiting surface, and **keyset (seek) pagination**, which replaces the popular-but-broken `LIMIT ... OFFSET` with a row-value predicate anchored on the last row you saw.

---

## 1. Standard `OFFSET ... FETCH FIRST n ROWS [ONLY | WITH TIES]` vs Non-Standard `LIMIT`

The SQL-standard way to limit a result is `FETCH FIRST`; `SQL:2008 introduced` it and PostgreSQL, Oracle 12c+, SQL Server 2012+, and DB2 all support it ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html); [Zhiyanov — LIMIT vs. FETCH](https://antonz.org/sql-fetch/)). The familiar `LIMIT` is **not in the standard at all**: "There is no `limit` clause in the SQL standard," and "`fetch first N rows only` does exactly what `limit N` does" ([Zhiyanov](https://antonz.org/sql-fetch/)). `OFFSET`, by contrast, *is* the standard keyword for skipping.

```sql
-- NON-STANDARD (PostgreSQL/MySQL/SQLite) — ubiquitous, but not portable
SELECT id, title FROM posts ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 40;

-- STANDARD (PostgreSQL/Oracle/SQL Server/DB2) — same result, portable
SELECT id, title FROM posts
 ORDER BY created_at DESC, id DESC
OFFSET 40 ROWS FETCH FIRST 20 ROWS ONLY;
```

`FIRST` and `NEXT` are synonyms, as are `ROW` and `ROWS` ([Zhiyanov](https://antonz.org/sql-fetch/)) — `FETCH NEXT 20 ROWS ONLY` reads more naturally after an `OFFSET`. `WITH TIES` "is used to return any additional rows that tie for the last place in the result set according to the `ORDER BY` clause; `ORDER BY` is mandatory in this case" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)) — so a top-3 query with `WITH TIES` returns four rows if the 3rd and 4th share the ordering key. `ONLY` cuts hard at n.

> **Note:** every example here carries a *total* `ORDER BY` (a unique tiebreaker like `id`). That is not decoration — see §3. MySQL and SQLite "do not support `fetch` at all" ([Zhiyanov](https://antonz.org/sql-fetch/)), so portable code there is stuck with `LIMIT`; see §8.

---

## 2. Why `OFFSET` Degrades and Is Unstable — the Centerpiece

`OFFSET m` has two independent defects, and both get worse as `m` grows.

**It is O(offset) scan-and-discard.** "the database must still fetch these rows from the disk and bring them in order before it can send the following ones" ([Winand — no-offset](https://use-the-index-luke.com/no-offset)). The server does not jump to row 1,000,000; it produces the first 1,000,000 rows, orders them, throws them away, and *then* starts your page. "big offsets impose a lot of work on the database" ([Winand](https://use-the-index-luke.com/no-offset)). PostgreSQL corroborates that the skipped rows are real work — "rows skipped over by `OFFSET` will get locked" ([PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)).

**It is unstable under concurrent writes.** The offset is just "a number of rows to be dropped — no more context" ([Winand](https://use-the-index-luke.com/no-offset)), so it has no anchor in the data. Insert or delete a row before the offset point between two page loads and the whole window shifts: "you'll get duplicates in case there were new rows inserted between fetching two pages" ([Winand](https://use-the-index-luke.com/no-offset)) — and the symmetric case skips a row. "The idea to use the number of rows seen to skip over them later is simply wrong" ([Winand](https://use-the-index-luke.com/no-offset)).

```sql
-- WRONG — deep OFFSET: O(1,000,000) scan-and-discard, and the page
-- shifts (skips/dupes) if any row is inserted/deleted before row 1,000,000
SELECT id, title FROM posts
 ORDER BY created_at DESC, id DESC
 LIMIT 20 OFFSET 1000000;

-- RIGHT — keyset/seek: anchor on the LAST row of the previous page.
-- No offset to scan; stable even as rows are inserted/deleted earlier.
SELECT id, title FROM posts
 WHERE (created_at, id) < (:last_created_at, :last_id)   -- row-value predicate, §4
 ORDER BY created_at DESC, id DESC
 FETCH FIRST 20 ROWS ONLY;
```

Keyset is the fix for *both* defects at once: it reads only the page-worth of rows it returns, and because it remembers *where* you were (the values) rather than *how many* you skipped, concurrent edits earlier in the list cannot make you re-see or miss a row.

---

## 3. The Keyset/Seek Pattern Needs a Total, Stable `ORDER BY`

The seek method "uses the _values_ of the previous page as a delimiter. Instead of a row number, you use the last value of the previous page to specify the lower bound" ([Winand — seek](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)). The recipe, straight from Winand:

```sql
-- First page: no predicate, just the ordered top-n
SELECT sale_date, sale_id, amount
  FROM sales
 ORDER BY sale_date DESC, sale_id DESC
 FETCH FIRST 10 ROWS ONLY;

-- Next page: carry the last row's (sale_date, sale_id) forward as the cursor
SELECT sale_date, sale_id, amount
  FROM sales
 WHERE (sale_date, sale_id) < (:last_date, :last_id)
 ORDER BY sale_date DESC, sale_id DESC
 FETCH FIRST 10 ROWS ONLY;
```

The `ORDER BY` must be **total** — tie-broken on a unique column (here `sale_id`). If you order only by `sale_date` and two rows share a date, the cursor `sale_date < :last_date` either skips the rest of that date or re-reads it: the boundary is ambiguous exactly because the order is not unique. This is the same rule the foundation skill states for *any* deterministic result (`sql-relational-and-null-discipline`); pagination just makes the cost of ignoring it a visible, recurring bug. The unique tiebreaker is also what makes the row-value comparison in §4 unambiguous, and what lets a composite index (§7) resolve the cursor as a range scan.

The cursor (the last row's key values) is what an API returns as an opaque "next page" token — keyset pagination *is* cursor pagination.

---

## 4. Row Value Comparison `(a, b) > (x, y)` — Lexicographic, with a NULL Trap

The heart of keyset is the **row value constructor comparison**. `(a, b) > (x, y)` does not compare tuples by some opaque rule — it compares them the way a dictionary orders words: "the row elements are compared left-to-right, stopping as soon as an unequal or null pair of elements is found" ([PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html)). So `(created_at, id) < (:d, :i)` means *earlier date, or same date and smaller id* — precisely the seek boundary.

It is the clean, correct equivalent of the verbose expansion everyone writes by hand and half of them get wrong:

```sql
-- VERBOSE / ERROR-PRONE — easy to flip an operator or drop the equality leg
WHERE created_at < :d
   OR (created_at = :d AND id < :i)

-- CLEAN — same lexicographic meaning, one predicate, index-friendly
WHERE (created_at, id) < (:d, :i)
```

Prefer the row-value form wherever the engine supports it. **The NULL trap:** row comparison inherits three-valued logic. "If either of this pair of elements is null, the result of the row comparison is unknown (null)" ([PostgreSQL — Row-Wise Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html)) — and `WHERE` drops UNKNOWN rows (foundation §2). Comparison stops at the first *decisive* pair, so `ROW(1,2,NULL) < ROW(1,3,0)` "yields true, not null, because the third pair of elements are not considered" ([PostgreSQL](https://www.postgresql.org/docs/current/functions-comparisons.html)) — but if your sort key itself is nullable, a NULL in the *deciding* position makes the comparison UNKNOWN and the row vanishes from paging. Keyset cursors therefore want **non-nullable** sort keys (or a `COALESCE`d / `NULLS`-pinned expression); NULL comparison theory is owned by `sql-relational-and-null-discipline`.

---

## 5. Multi-Row `IN ((..), (..))` — Row Values in a Predicate

The same row-value syntax works in `IN`: a list of row constructors matches on the *combination* of columns, not column-by-column.

```sql
-- Match specific (tenant, sku) pairs — NOT the cross product
SELECT * FROM inventory
 WHERE (tenant_id, sku) IN ((1, 'A-100'), (1, 'B-200'), (7, 'A-100'));

-- WRONG shape — this matches any of tenant {1,7} crossed with any sku, far too broad
-- WHERE tenant_id IN (1, 7) AND sku IN ('A-100', 'B-200')
```

This is the natural way to fetch or filter a set of composite keys in one round trip. (The NULL caveats of `IN` / `NOT IN` are owned by the foundation skill; `NOT IN` over row values with any NULL is the same footgun.)

---

## 6. `VALUES` Row-Lists — a Literal Table for Set-Based Bulk Work

A multi-row `VALUES` is not just `INSERT` syntax: "VALUES computes a row value or set of row values" and "can also be used where a sub-`SELECT` might be written, for example in a `FROM` clause" — indeed "`VALUES` is syntactically allowed anywhere that `SELECT` is" ([PostgreSQL — VALUES](https://www.postgresql.org/docs/current/sql-values.html)). Treat it as an inline constant table and stay set-based instead of looping in application code.

```sql
-- Bulk insert: one statement, not N round trips
INSERT INTO tags (post_id, label) VALUES
  (1, 'sql'), (1, 'paging'), (2, 'index');

-- As a derived table: JOIN against an ad-hoc set of rows
SELECT v.sku, v.want_qty, i.on_hand
  FROM (VALUES ('A-100', 5), ('B-200', 12)) AS v(sku, want_qty)
  JOIN inventory i ON i.sku = v.sku;
```

"When more than one row is specified, all the rows must have the same number of elements" ([PostgreSQL — VALUES](https://www.postgresql.org/docs/current/sql-values.html)). This is the set-based answer to "I have a list of pairs in code and need to match them in the database."

---

## 7. Keyset Is Only O(page) *With the Right Index*

Keyset's performance win is not magic — it depends on an index whose column order matches the `ORDER BY`. Winand: build `CREATE INDEX sl_dtid ON sales (sale_date, sale_id)` and "The database can use the `SALE_DATE < ?` condition for index access. That means that the database can truly skip the rows from the previous pages" ([Winand — seek](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)). Without that composite index the engine falls back to a scan and you have merely moved the cost around. The full treatment of composite-index column order, sargability, and why the index direction must match the sort lives in `sql-indexing-and-sargability` — route there for the O(1)-per-page guarantee.

---

## 8. Portability Snapshot

| Feature | PostgreSQL | Oracle | SQL Server | MySQL | SQLite |
|---|---|---|---|---|---|
| `FETCH FIRST n ROWS ONLY` (standard) | yes | 12c+ | 2012+ | **no** | **no** |
| `LIMIT` (non-standard) | yes | no | no | yes | yes |
| `OFFSET` keyword | yes | yes | yes | yes (in `LIMIT`) | yes |
| `WITH TIES` | yes | yes | yes | no | no |
| Row-value comparison `(a,b) < (x,y)` | yes | yes | **no** | 8.0+ | **no** |

`FETCH FIRST` is the standard but "MySQL and SQLite do not support `fetch` at all" ([Zhiyanov](https://antonz.org/sql-fetch/)), so cross-engine code often must use the non-standard `LIMIT` — ubiquitous, not portable. Row-value comparison is standard but "only a few databases support it" ([Winand — seek](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)): SQLite and SQL Server lack it, and older MySQL did too. Where it is missing, expand by hand — Winand's approximation:

```sql
-- Manual equivalent of  (sale_date, sale_id) < (?, ?)  for engines lacking row values
WHERE sale_date <= ?
  AND NOT (sale_date = ? AND sale_id >= ?)
```

This expansion is verbose and easy to get wrong (it is exactly the trap §4 avoids), so confine it to engines that force it. The full `LIMIT`/`FETCH` dialect table and the SQLite/old-MySQL row-comparison gaps are owned by `sql-standard-vs-dialect-map`.

---

## 9. Who Suffers When Pagination Is Wrong

Pagination bugs are silent at write time and surface as someone's bad day:

- The **reader scrolling an infinite feed** who sees the same post twice — or never sees one at all — because the app paged with `LIMIT/OFFSET` and a new post was inserted at the top between scrolls, shifting every offset by one (§2). No error; the data just lied.
- The **API that times out on page 50,000.** `OFFSET 1000000` made the database fetch, sort, and discard a million rows before returning twenty (§2); the endpoint that was snappy on page 1 falls over deep in a large table, and the on-call engineer finds a request that "worked in testing" timing out in production.
- The **maintainer** who inherited the hand-rolled `(a < ? OR (a = ? AND b < ?))` cursor, found one leg's operator flipped, and spent an afternoon proving the page boundary was off by exactly the rows sharing a sort key — a bug the row-value comparison `(a, b) < (?, ?)` never has (§4).
- The **analyst** whose "stable" report changes row order between runs because the paging query ordered only on a non-unique column and the plan broke ties differently each time (§3).

Correct pagination is a kindness to all of them: a keyset cursor on a total order, anchored on values not counts.

---

## 10. Routing to Related Skills

- `sql-relational-and-null-discipline` — the policy root: undefined order without `ORDER BY` (§3), and three-valued logic behind the row-comparison NULL trap (§4).
- `sql-indexing-and-sargability` — the composite index that makes keyset O(page) instead of O(scan) (§7).
- `sql-window-functions` — `ROW_NUMBER()`/`RANK()` ranked paging and "top-n per group," the alternative when offset-style numbering is genuinely required.
- `sql-select-and-query-processing` — where `ORDER BY` / `OFFSET` / `FETCH` sit in logical query processing.
- `sql-set-operations` — applying `FETCH FIRST` to a `UNION`/`INTERSECT` result.
- `sql-standard-vs-dialect-map` — `LIMIT` vs `FETCH` per engine and the SQLite/SQL Server/old-MySQL row-value-comparison gaps (§8).

---

## 11. Reference Files

High-frequency pagination and row-value anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
