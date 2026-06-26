# Common LATERAL & Correlated-Derived Mistakes

Anti-patterns in LLM-generated SQL around `LATERAL` and correlated derived tables, each with wrong/right
code and a primary-source citation. The policy root (`sql-lateral-and-correlated-derived`) states the
rules; this file holds the high-frequency failure modes. All RIGHT examples are standard/portable SQL;
non-standard spellings are flagged and routed to `sql-standard-vs-dialect-map`.

---

## 1. A plain derived table that tries to see a sibling column

**The problem:** The model writes a `FROM` subquery that references the outer table and expects it to work — but "without `LATERAL`, each subquery is evaluated independently and so cannot cross-reference any other `FROM` item" ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)). The derived table "is illegal in SQL-92 because derived tables cannot depend on other tables in the same `FROM` clause" ([MySQL — Lateral Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html)). It fails to parse: `invalid reference to FROM-clause entry` / `Unknown column 'c.id'`.

```sql
-- WRONG — the derived table references c.id, but a plain FROM subquery can't see c
SELECT c.id, recent.total
FROM customers c
JOIN (SELECT SUM(amount) AS total
      FROM orders o
      WHERE o.customer_id = c.id) recent ON true;   -- c.id not in scope

-- RIGHT — LATERAL puts the preceding FROM items in scope, evaluated per outer row
SELECT c.id, recent.total
FROM customers c
JOIN LATERAL (SELECT SUM(amount) AS total
             FROM orders o
             WHERE o.customer_id = c.id) recent ON true;
```

*Source: [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html); [MySQL — Lateral Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html). Depth: this skill, §1.*

---

## 2. Forgetting `ON true` on the lateral join

**The problem:** The model writes `JOIN LATERAL (...)` and stops, or invents an `ON` predicate that re-duplicates the correlation already inside the body. With `LATERAL` the correlation predicate lives *inside* the subquery (`WHERE o.customer_id = c.id`), so the join itself has nothing left to match on — the idiom is the trivial truth `ON true`. The PostgreSQL example is `... LEFT JOIN LATERAL get_product_names(m.id) pname ON true` ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)). Omitting the `ON` clause is a syntax error for a `JOIN`.

```sql
-- WRONG — a JOIN requires an ON clause; this won't parse
SELECT c.id, o.id AS order_id
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 1 ROWS ONLY
) o;

-- RIGHT — the correlation is inside the body, so the join predicate is just ON true
SELECT c.id, o.id AS order_id
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 1 ROWS ONLY
) o ON true;
```

*Source: [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html). Depth: this skill, §2.*

---

## 3. Stuffing "top N" into a scalar subquery in the SELECT list

**The problem:** Asked for "the latest 3 orders per customer," the model reaches for a correlated scalar subquery — but a scalar subquery may return *one* row and *one* column. `FETCH FIRST 3 ROWS` inside it overflows: `more than one row returned by a subquery used as an expression`. Top-N-per-group is the task of accessing "the first or top n rows for every unique value of a given column" ([Wikibooks — Top N per Group](https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group)), and the standard answer is a lateral join.

```sql
-- WRONG — a scalar subquery returns at most one value; "top 3" overflows it
SELECT c.id,
       (SELECT o.id FROM orders o WHERE o.customer_id = c.id
        ORDER BY o.created_at DESC FETCH FIRST 3 ROWS ONLY) AS recent_orders
FROM customers c;

-- RIGHT — LEFT JOIN LATERAL returns the whole top-N set as rows
SELECT c.id, o.id AS order_id, o.created_at, o.amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.created_at, o.amount
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 3 ROWS ONLY
) o ON true;
```

*Source: [Wikibooks — Top N per Group](https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group); [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html). Depth: this skill, §2.*

---

## 4. N correlated scalar subqueries instead of one LATERAL

**The problem:** A dashboard wants several facts about each customer's most recent order, and the model grows one correlated scalar subquery *per column* — each independently re-scanning `orders`. `LATERAL` is "evaluated using that row['s] ... values" once per outer row and hands back many columns at a time ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)). Besides the N+1 scans, the per-column copies can silently return fields from *different* rows if any `ORDER BY` isn't identical and total.

```sql
-- WRONG — three correlated subqueries, three scans, fields can drift out of sync
SELECT c.id,
       (SELECT o.id         FROM orders o WHERE o.customer_id = c.id ORDER BY o.created_at DESC FETCH FIRST 1 ROWS ONLY) AS last_id,
       (SELECT o.amount     FROM orders o WHERE o.customer_id = c.id ORDER BY o.created_at DESC FETCH FIRST 1 ROWS ONLY) AS last_amount,
       (SELECT o.created_at FROM orders o WHERE o.customer_id = c.id ORDER BY o.created_at DESC FETCH FIRST 1 ROWS ONLY) AS last_at
FROM customers c;

-- RIGHT — one lateral body, one ORDER BY, all columns from the same chosen row
SELECT c.id, last.id AS last_id, last.amount AS last_amount, last.created_at AS last_at
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.amount, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 1 ROWS ONLY
) last ON true;
```

*Source: [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html). Depth: this skill, §4; correlated subqueries owned by `sql-subqueries-and-exists`.*

---

## 5. Inner `JOIN LATERAL` silently dropping childless parents

**The problem:** The model uses `JOIN LATERAL` (or the bare-comma form) for "top N per customer" and is surprised that customers with zero orders vanish. An inner lateral join drops a parent whose body produces no rows. PostgreSQL notes it is "often particularly handy to `LEFT JOIN` to a `LATERAL` subquery, so that source rows will appear in the result even if the `LATERAL` subquery produces no rows for them" ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html)).

```sql
-- WRONG — customers with no orders disappear from the result
SELECT c.id, o.id AS order_id
FROM customers c
JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 3 ROWS ONLY
) o ON true;

-- RIGHT — LEFT JOIN LATERAL keeps every customer, NULL-extended when there are no orders
SELECT c.id, o.id AS order_id
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 3 ROWS ONLY
) o ON true;
```

*Source: [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html). Depth: this skill, §2.*

---

## 6. Non-deterministic "latest" from a non-total `ORDER BY`

**The problem:** The lateral body orders by a non-unique column (`created_at`) only, so when two rows tie the "latest" is whichever the plan happens to surface — and it can change between runs or between the per-column copies of §4. "If sorting is not chosen, the rows will be returned in an unspecified order ... it must not be relied on" ([PostgreSQL — Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html)); `FETCH FIRST n` is only deterministic on a *total* order. Tie-break on a unique column.

```sql
-- WRONG — ties on created_at make "the latest" undefined
SELECT c.id, o.id, o.amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.amount FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC          -- not a total order
    FETCH FIRST 1 ROWS ONLY
) o ON true;

-- RIGHT — tie-break on a unique column for a deterministic pick
SELECT c.id, o.id, o.amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.amount FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC   -- total order
    FETCH FIRST 1 ROWS ONLY
) o ON true;
```

*Source: [PostgreSQL — Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html). Depth: foundation `sql-relational-and-null-discipline`, §1; applied here in §2.*

---

## 7. Writing `LATERAL` before a set-returning function (or a comma-join that drops empties)

**The problem:** Two opposite errors around table functions in `FROM`. (a) The model thinks it *must* write `LATERAL` before `unnest(...)`; in fact "for functions the key word is optional ... the function's arguments can contain references to columns provided by preceding `FROM` items in any case" — it is "a noise word" there ([PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html); [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html)). (b) The model uses the bare-comma form (an implicit `CROSS JOIN LATERAL`) and loses parent rows whose array is empty.

```sql
-- WRONG — orders with an empty/NULL tag_ids array silently vanish (implicit CROSS JOIN LATERAL)
SELECT o.id, t.tag_id
FROM orders o, unnest(o.tag_ids) AS t(tag_id);

-- RIGHT — implicit lateral is fine when you WANT to drop empties (no LATERAL keyword needed)
SELECT o.id, t.tag_id
FROM orders o, unnest(o.tag_ids) AS t(tag_id);   -- keeps only orders with >=1 tag

-- RIGHT — to keep orders with empty arrays, LEFT JOIN LATERAL ... ON true
SELECT o.id, t.tag_id
FROM orders o
LEFT JOIN LATERAL unnest(o.tag_ids) AS t(tag_id) ON true;
```

*Source: [PostgreSQL — LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html); [PostgreSQL — SELECT](https://www.postgresql.org/docs/current/sql-select.html). Depth: this skill, §3.*

---

## 8. Reinventing top-N-per-group with window-filter gymnastics when LATERAL is simpler

**The problem:** Starting from a small or filtered set of outer rows, the model ranks the *entire* child table with `ROW_NUMBER()` and filters — scanning all of `orders` to keep three rows per a handful of customers. The window form is the right tool for a report "across the whole table," but when you start from a few outer rows, `LATERAL` probes the indexed child per row instead ([Wikibooks — Top N per Group](https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group)).

```sql
-- WRONG (for a small outer set) — ranks ALL orders, then throws most away
SELECT id, customer_id, created_at, amount
FROM (
    SELECT o.*,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC, o.id DESC) AS rn
    FROM orders o
) ranked
WHERE customer_id IN (1, 2, 3) AND rn <= 3;

-- RIGHT — probe the indexed child only for the customers you care about
SELECT c.id, o.id AS order_id, o.created_at, o.amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.created_at, o.amount
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 3 ROWS ONLY
) o ON true
WHERE c.id IN (1, 2, 3);
```

*Source: [Wikibooks — Top N per Group](https://en.wikibooks.org/wiki/Structured_Query_Language/Retrieve_Top_N_Rows_per_Group). Depth: this skill, §5; the windowing side owned by `sql-window-functions`.*

---

## 9. Confusing `OUTER APPLY` with `LEFT JOIN` (and `CROSS APPLY` with `INNER JOIN`)

**The problem:** Porting to/from SQL Server, the model treats `APPLY` as an ordinary join with an `ON` predicate, or maps `CROSS APPLY` to `LEFT JOIN`. The "LATERAL keyword (in PostgreSQL/MySQL) or CROSS APPLY (in SQL Server/Oracle)" name the *same* operation ([SQL Boy — LATERAL Joins and CROSS APPLY](https://www.hisqlboy.com/blog/understanding-sql-lateral-joins)). The mapping is mechanical: `CROSS APPLY` = `[CROSS] JOIN LATERAL (...) ON true` (drops unmatched parents); `OUTER APPLY` = `LEFT JOIN LATERAL (...) ON true` (keeps them, NULL-extended). `APPLY` takes no `ON` clause.

```sql
-- WRONG — CROSS APPLY is not a LEFT JOIN; this drops customers with no orders,
-- and APPLY does not take an ON clause
SELECT c.id, o.id AS order_id
FROM customers c
CROSS APPLY (
    SELECT TOP (3) o.id, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
) o ON true;

-- RIGHT (SQL Server) — OUTER APPLY keeps childless customers; no ON clause
SELECT c.id, o.id AS order_id
FROM customers c
OUTER APPLY (
    SELECT TOP (3) o.id, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
) o;
```

*Source: [SQL Boy — LATERAL Joins and CROSS APPLY](https://www.hisqlboy.com/blog/understanding-sql-lateral-joins). Depth: this skill, §6; full dialect table owned by `sql-standard-vs-dialect-map`.*

---

## 10. Using `LATERAL` / `APPLY` on an engine that has neither (SQLite)

**The problem:** The model emits a `LEFT JOIN LATERAL` (or `CROSS APPLY`) query for SQLite, where it simply won't parse — SQLite has no `LATERAL` and no `APPLY`. The portable rewrite is a window function (`ROW_NUMBER() OVER (PARTITION BY ...)`) or a correlated subquery. (`LATERAL` is also gated by version: MySQL 8.0.14+, Oracle 12c+ ([MySQL — Lateral Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html)).)

```sql
-- WRONG — SQLite cannot parse LATERAL (nor APPLY)
SELECT c.id, o.id AS order_id
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC, o.id DESC
    FETCH FIRST 3 ROWS ONLY
) o ON true;

-- RIGHT — window-function rewrite is the portable top-N-per-group on SQLite
SELECT id, customer_id, created_at
FROM (
    SELECT o.id, o.customer_id, o.created_at,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC, o.id DESC) AS rn
    FROM orders o
) ranked
WHERE rn <= 3;
```

*Source: [MySQL — Lateral Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html). Depth: this skill, §7; version gates and the SQLite gap owned by `sql-standard-vs-dialect-map`.*
