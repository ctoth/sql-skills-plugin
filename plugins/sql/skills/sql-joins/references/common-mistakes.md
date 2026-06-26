# Common SQL Join Mistakes

Anti-patterns in LLM-generated SQL around joins, each with wrong/right code and a primary-source
citation. The skill (`sql-joins`) states the rules; this file holds the high-frequency failure modes.
All RIGHT examples are standard/portable SQL; non-standard spellings are flagged and routed to
`sql-standard-vs-dialect-map`. Set-semantics and three-valued-logic basics live in the foundation
skill `sql-relational-and-null-discipline`.

---

## 1. Filtering the outer table in `WHERE` — demoting `LEFT JOIN` to `INNER JOIN`

**The problem:** The model writes a `LEFT JOIN` to keep all left rows, then adds a `WHERE` predicate on a column from the right (null-able) table. After null-extension the right column is NULL, the predicate is UNKNOWN, and `WHERE` drops the row — so the `LEFT JOIN` silently becomes an `INNER JOIN`. "A restriction placed in the `ON` clause is processed _before_ the join, while a restriction placed in the `WHERE` clause is processed _after_ the join ... it matters a lot with outer joins" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)).

```sql
-- WRONG — returns only customers WITH a paid order; customers with none are dropped
SELECT c.id, c.name, o.amount
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE  o.status = 'paid';

-- RIGHT — move the outer-table filter into ON; unmatched customers survive null-extended
SELECT c.id, c.name, o.amount
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'paid';
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN). Depth: this skill, §2.*

---

## 2. `NATURAL JOIN` — joining on every same-named column

**The problem:** The model uses `NATURAL JOIN` for brevity. It joins on "all column names that appear in both input tables," so adding a shared column name later silently changes the join key, and losing all common names makes it "behave like `CROSS JOIN`" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)). The query breaks on a schema migration that never touched it.

```sql
-- WRONG — key set is "whatever columns share a name today"; fragile across schema changes
SELECT * FROM customers NATURAL JOIN orders;

-- RIGHT — explicit ON; intent is stable
SELECT * FROM customers c JOIN orders o ON o.customer_id = c.id;

-- ACCEPTABLE — USING names the column explicitly and de-duplicates it
SELECT * FROM customers JOIN orders USING (customer_id);
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN). Depth: this skill, §3.*

---

## 3. Accidental cross product from a missing or typo'd predicate

**The problem:** The model uses a comma join (`FROM a, b`) and forgets the predicate, or writes an always-true `ON`. The result is the Cartesian product — "If the tables have N and M rows respectively, the joined table will have N \* M rows" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)) — with no error.

```sql
-- WRONG — predicate forgotten: every order paired with every customer (N * M rows)
SELECT * FROM orders o, customers c;

-- RIGHT — explicit JOIN ... ON makes a missing predicate a visible omission
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;

-- RIGHT — when all combinations ARE wanted, say so explicitly
SELECT * FROM sizes CROSS JOIN colors;
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN). Depth: this skill, §5.*

---

## 4. One-to-many fan-out inflating an aggregate

**The problem:** The model joins a one-to-many relationship and aggregates a column from the "one" side. Each one-side row is duplicated once per match — INNER/LEFT give "a row for each row in T2 that satisfies the join condition" ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)) — so the one-side value is summed once per "many" row and the total balloons.

```sql
-- WRONG — SUM(c.credit_limit) counts the limit once PER ORDER, not once per customer
SELECT c.id, SUM(c.credit_limit) AS total_limit, SUM(o.amount) AS spent
FROM   customers c
JOIN   orders o ON o.customer_id = c.id
GROUP  BY c.id;

-- RIGHT — pre-aggregate the "many" side, then join one-to-one
SELECT c.id, c.credit_limit, COALESCE(s.spent, 0) AS spent
FROM   customers c
LEFT JOIN (
    SELECT customer_id, SUM(amount) AS spent FROM orders GROUP BY customer_id
) s ON s.customer_id = c.id;
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN). Depth: this skill, §6; grouping rules owned by `sql-aggregation-and-grouping`.*

---

## 5. `INNER` self-join silently dropping the top of a hierarchy

**The problem:** The model self-joins on `manager_id` with an `INNER JOIN`, dropping the row whose `manager_id IS NULL` (the CEO/root), because `m.id = NULL` is UNKNOWN and never matches.

```sql
-- WRONG — the top-level row (manager_id IS NULL) disappears
SELECT e.name AS employee, m.name AS manager
FROM   employees e JOIN employees m ON m.id = e.manager_id;

-- RIGHT — LEFT JOIN keeps the root row with a NULL manager
SELECT e.name AS employee, m.name AS manager
FROM   employees e LEFT JOIN employees m ON m.id = e.manager_id;
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN). Depth: this skill, §4 and §7.*

---

## 6. Equi-join silently excluding NULL-keyed rows

**The problem:** The model joins on a nullable key and assumes NULL keys match each other. `ON a.k = b.k` evaluates `NULL = NULL` to UNKNOWN, so NULL-keyed rows never match — they are dropped (INNER) or null-extended (LEFT), not paired.

```sql
-- WRONG (assumption) — rows where a.k IS NULL never match rows where b.k IS NULL
SELECT * FROM a JOIN b ON a.k = b.k;

-- RIGHT — null-safe match treats two NULLs as equal (returns true/false, never UNKNOWN)
SELECT * FROM a JOIN b ON a.k IS NOT DISTINCT FROM b.k;
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN); foundation §3 and §8. Depth: this skill, §7; dialect spelling `<=>` owned by `sql-standard-vs-dialect-map`.*

---

## 7. `RIGHT JOIN` where MySQL-portable `FULL JOIN` was meant, or `FULL JOIN` on MySQL

**The problem:** The model writes `FULL JOIN` (to keep unmatched rows on both sides) and ships it to MySQL, which has no `FULL JOIN`. The query errors. FULL is "for each row in T1 that does not satisfy ... a joined row is added with null values ... Also, for each row of T2 that does not satisfy ..." ([PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)).

```sql
-- WRONG on MySQL — FULL JOIN is unsupported
SELECT * FROM a FULL JOIN b ON a.k = b.k;

-- RIGHT (portable emulation) — union a LEFT join with the unmatched RIGHT rows
SELECT * FROM a LEFT JOIN b ON a.k = b.k
UNION ALL
SELECT * FROM a RIGHT JOIN b ON a.k = b.k WHERE a.k IS NULL;
```

*Source: [PostgreSQL — Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN). Depth: this skill, §1 and §8; emulation owned by `sql-standard-vs-dialect-map`.*
