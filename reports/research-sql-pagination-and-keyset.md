# Research: sql-pagination-and-keyset

Research backing the skill `sql-pagination-and-keyset` (absorbs row-value-constructors / row-comparisons).
Each entry: URL, section, verbatim quote, why it matters.

Accessed: 2026-06-26.

> NOTE ON A DEAD SOURCE: the spec named `https://modern-sql.com/feature/fetch-first` as a primary
> source. As of 2026-06-26 that URL returns HTTP 404 (the modern-sql feature page has moved/been
> removed). The claims it was cited for — LIMIT is non-standard, OFFSET/FETCH FIRST is the standard
> keyword, WITH TIES semantics — are covered verbatim below by the PostgreSQL SELECT reference and
> Anton Zhiyanov's "LIMIT vs. FETCH in SQL", both reachable and authoritative. Those are the sources
> cited in the skill instead.

---

## 1. use-the-index-luke.com/no-offset (Winand — OFFSET is unstable and scans/discards)

URL: https://use-the-index-luke.com/no-offset

### Performance — OFFSET scans and discards (O(offset))
> "Offset instructs the DBMS skip the first N results of a query. However, the database must still
> fetch these rows from the disk and bring them in order before it can send the following ones."

> "big offsets impose a lot of work on the database"

**Why it matters:** This is the verbatim basis for the centerpiece claim that `OFFSET m` is O(m) scan-and-discard
— the server reads and orders the skipped rows, then throws them away. Justifies the "API timed out paginating
deep into a huge table with OFFSET 1000000" persona.

### Instability — shifting rows cause skips/duplicates
> "think about what happens if a new row is inserted between fetching two pages?"

> "you'll get duplicates in case there were new rows inserted between fetching two pages."

> "There are other anomalies possible too, this is just the most common one."

> "The idea to use the number of rows seen to skip over them later is simply wrong."

### Root cause
> "The root problem all these methods have in common is that they just provide a number of rows to be
> dropped—no more context."

**Why it matters:** Verbatim basis for "OFFSET is unstable — rows shift between page loads → skips/duplicates."
A row count carries no anchor into the data, so any insert/delete before the offset point shifts the window.
Justifies the infinite-feed "saw a row twice / missed a row" persona.

---

## 2. use-the-index-luke.com/sql/partial-results/fetch-next-page (Winand — the seek method / keyset)

URL: https://use-the-index-luke.com/sql/partial-results/fetch-next-page
(the `/sql/partial-results` chapter index links here; the index page itself has no SQL detail)

### Core concept — uses values, not a row count
> "The seek method avoids both problems because it uses the _values_ of the previous page as a delimiter."

> "Instead of a row number, you use the last value of the previous page to specify the lower bound."

### The canonical SQL (row values + matching index)
> ```sql
> CREATE INDEX sl_dtid ON sales (sale_date, sale_id)
>
> SELECT *
>   FROM sales
>  WHERE (sale_date, sale_id) < (?, ?)
>  ORDER BY sale_date DESC, sale_id DESC
>  FETCH FIRST 10 ROWS ONLY
> ```

### Why it is fast AND stable
> "The database can use the `SALE_DATE < ?` condition for index access. That means that the database can
> truly skip the rows from the previous pages."

> "You will also get stable results if new rows are inserted."

### Row values syntax + dialect gap + OR-expansion fallback
> "The `where` clause uses the little-known 'row values' syntax... It combines multiple values into a
> logical unit."

> "Even though the row values syntax is part of the SQL standard, only a few databases support it."

> "Nevertheless it is possible to use an approximated variant of the seek method with databases that do
> not properly support the row values."

> ```sql
> WHERE sale_date <= ?
>   AND NOT (sale_date = ? AND sale_id >= ?)
> ```

**Why it matters:** The whole keyset/seek pattern, in Winand's own SQL: a row-value predicate on
`(sort_key, id)`, a total `ORDER BY sort_key, id`, `FETCH FIRST n ROWS ONLY`, and a composite index that
turns the predicate into an index range scan (O(rows-returned), not O(offset)). The last block is the
verbatim source for the "databases lacking row-value comparison need a manual expansion" portability note,
which routes to the indexing skill (index dependence) and the dialect map (SQLite/old-MySQL gaps).

---

## 3. PostgreSQL — Row Constructor / Row-Wise Comparison

URL: https://www.postgresql.org/docs/current/functions-comparisons.html (§ Row Constructor Comparison)

### Lexicographic, left-to-right, stop at first decisive pair
> "For the `<`, `<=`, `>` and `>=` cases, the row elements are compared left-to-right, stopping as soon
> as an unequal or null pair of elements is found."

### NULL behavior
> "If either of this pair of elements is null, the result of the row comparison is unknown (null);
> otherwise comparison of this pair of elements determines the result."

> "For example, `ROW(1,2,NULL) < ROW(1,3,0)` yields true, not null, because the third pair of elements
> are not considered."

> "Two rows are considered equal if all their corresponding members are non-null and equal; the rows are
> unequal if any corresponding members are non-null and unequal; otherwise the result of the row
> comparison is unknown (null)."

**Why it matters:** Verbatim mechanics of `(a,b) > (x,y)` — element-by-element, left to right, stopping at
the first decisive pair. This is exactly the lexicographic ordering keyset needs, and it is the clean
equivalent of the error-prone `(a > x) OR (a = x AND b > y)`. The NULL rule (a NULL in the decisive pair
makes the whole comparison UNKNOWN, which `WHERE` drops) is the trap to flag and routes back to the
foundation skill's three-valued logic. The `ROW(1,2,NULL) < ROW(1,3,0)` example proves comparison stops
before the NULL when an earlier pair already decides.

---

## 4. PostgreSQL — VALUES Lists

URL: https://www.postgresql.org/docs/current/sql-values.html

> "VALUES computes a row value or set of row values specified by value expressions."

> "When more than one row is specified, all the rows must have the same number of elements."

> "It is most commonly used to generate a \"constant table\" within a larger command, but it can be used
> on its own."

> "`VALUES` can also be used where a sub-`SELECT` might be written, for example in a `FROM` clause:"

> "Within larger commands, `VALUES` is syntactically allowed anywhere that `SELECT` is."

**Why it matters:** Backs the `VALUES` row-list section — a multi-row `VALUES` is a literal table you can
INSERT, JOIN against, or filter with, the set-based alternative to N single-row round trips or a giant
`OR`/`IN` chain. Pairs naturally with multi-row `IN ((..),(..))`, which is the same row-list machinery in
predicate position.

---

## 5. PostgreSQL — SELECT (LIMIT / OFFSET / FETCH FIRST / WITH TIES)

URL: https://www.postgresql.org/docs/current/sql-select.html (§ LIMIT Clause)

### Standard vs ad-hoc syntax
> ```
> OFFSET start { ROW | ROWS }
> FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } { ONLY | WITH TIES }
> ```
> ```
> LIMIT { count | ALL }
> OFFSET start
> ```

> "SQL:2008 introduced a different syntax to achieve the same result, which PostgreSQL also supports."

(FETCH FIRST/NEXT is the SQL-standard form; LIMIT/OFFSET is the PostgreSQL/MySQL ad-hoc form.)

### WITH TIES
> "The `WITH TIES` option is used to return any additional rows that tie for the last place in the result
> set according to the `ORDER BY` clause; `ORDER BY` is mandatory in this case, and `SKIP LOCKED` is not
> allowed."

### A total ORDER BY is mandatory for correct paging
> "When using `LIMIT`, it is a good idea to use an `ORDER BY` clause that constrains the result rows into
> a unique order. Otherwise you will get an unpredictable subset of the query's rows — you might be asking
> for the tenth through twentieth rows, but tenth through twentieth in what ordering?"

### OFFSET still pays for skipped rows
> "If a `LIMIT` is used, locking stops once enough rows have been returned to satisfy the limit (but note
> that rows skipped over by `OFFSET` will get locked)."

**Why it matters:** Verbatim standard syntax for the `OFFSET ... FETCH FIRST n ROWS [ONLY|WITH TIES]`
surface and the `ROW|ROWS` / `FIRST|NEXT` synonyms. The "unique order" quote is the hard requirement that a
correct page needs a *total* ORDER BY (tiebreak on a unique column) — the same undefined-order rule the
foundation skill owns. "rows skipped over by OFFSET will get locked" corroborates that OFFSET does work
proportional to the offset.

---

## 6. antonz.org — LIMIT vs. FETCH in SQL (Anton Zhiyanov)

URL: https://antonz.org/sql-fetch/

### LIMIT is not in the standard; FETCH FIRST is the equivalent
> "There is no `limit` clause in the SQL standard"

> "`fetch first N rows only` does exactly what `limit N` does."

### Synonyms
> "`next` here is just a syntactic sugar, a synonym for `first`"

> "`row` and `rows` are also synonyms."

### WITH TIES
> "select anyone with the same salary as the last (5th) employee. Here comes `with ties`"

### Database support
> "The following DBMS support `fetch`: PostgreSQL 8.4+, Oracle 12c+, MS SQL 2012+, DB2 9+. However, only
> Oracle supports `percent` fetching. MySQL and SQLite do not support `fetch` at all."

**Why it matters:** Verbatim basis for "LIMIT is non-standard but ubiquitous; FETCH FIRST is the standard
keyword." The DB-support sentence is the portability anchor: MySQL and SQLite do **not** implement FETCH
FIRST, so portable real-world code often must use `LIMIT` even though it is non-standard — route the full
table to `sql-standard-vs-dialect-map`.

---

## Synthesis — what the skill must nail

- **(a) keyset beats OFFSET:** `WHERE (sort_key, id) > (:k, :id) ORDER BY sort_key, id FETCH FIRST n ROWS
  ONLY` — anchored on the last row's values, index-supported, O(rows-returned). [Source 2]
- **(b) OFFSET is O(offset) scan-and-discard AND unstable:** server fetches+orders skipped rows then drops
  them; an insert/delete between page loads shifts the window → skipped or duplicated rows. [Source 1, 5]
- **(c) row value comparison `(a,b) > (x,y)` is lexicographic** (left-to-right, stop at first decisive
  pair) and is the clean equivalent of `(a > x) OR (a = x AND b > y)`; recommend row-value where supported;
  a NULL in the decisive pair makes it UNKNOWN. SQLite/older-MySQL lack row-value comparison → manual
  expansion / dialect map. [Source 3, 2]
- **(d) standard `OFFSET ... FETCH FIRST n ROWS [ONLY|WITH TIES]` vs non-standard `LIMIT`;** FIRST/NEXT and
  ROW/ROWS are synonyms; MySQL/SQLite only have LIMIT. [Source 5, 6]
- **(e) a TOTAL, stable ORDER BY (tiebreak on a unique column) is mandatory** for correct paging — without
  it you get "an unpredictable subset"; this is the foundation skill's undefined-order rule. [Source 5]
- **(f) `VALUES` row-lists** are a literal table for set-based bulk INSERT/JOIN/filter; multi-row
  `IN ((..),(..))` is the same row-list machinery in predicate position. [Source 4, 3]
