# Research: `sql-data-modification` skill

Core DML (INSERT / UPDATE / DELETE) for a standard-SQL skills plugin. Gap found in audit:
the plugin had `sql-merge-and-upsert` but no skill owning the foundational write statements,
the missing-WHERE catastrophe, set-based vs row-by-row DML, RETURNING, or TRUNCATE-vs-DELETE.

Each entry: URL, section, verbatim quote, why it matters for the skill. Accessed 2026-06-26.

---

## 1. PostgreSQL — INSERT

URL: https://www.postgresql.org/docs/current/sql-insert.html

**Synopsis (set-based VALUES grammar).**
> `{ DEFAULT VALUES | VALUES ( { expression | DEFAULT } [, ...] ) [, ...] | query }`

Why it matters: the `[, ...]` after the parenthesised row shows the multi-row VALUES form is
first-class — multiple rows in *one* statement, not a loop. The `| query` arm is `INSERT ... SELECT`.

**Single-row VALUES.**
> "Insert a single row into table `films`: `INSERT INTO films VALUES ('UA502', 'Bananas', 105, ...);`"

**Multi-row VALUES (one statement, set-based).**
> "To insert multiple rows using the multirow `VALUES` syntax:
> `INSERT INTO films (code, title, did, date_prod, kind) VALUES ('B6717', 'Tampopo', 110, '1985-02-10', 'Comedy'), ('HG120', 'The Dinner Game', 140, DEFAULT, 'Comedy');`"

Why it matters: the centerpiece of the set-based-over-looped section. One statement, one round
trip, one transaction — vs N single-row INSERTs. Note `DEFAULT` usable per-value.

**INSERT ... SELECT for bulk copy.**
> "This example inserts some rows into table `films` from a table `tmp_films` with the same column
> layout as `films`: `INSERT INTO films SELECT * FROM tmp_films WHERE date_prod < '2004-05-07';`"

Why it matters: the canonical bulk-copy / ETL form — the engine does the loop set-based.

**DEFAULT VALUES.**
> "`DEFAULT VALUES` — All columns will be filled with their default values, as if `DEFAULT` were
> explicitly specified for each column." / "`INSERT INTO films DEFAULT VALUES;`"

**RETURNING (generated keys without a second round trip).**
> "The optional `RETURNING` clause causes `INSERT` to compute and return value(s) based on each row
> actually inserted (or updated, if an `ON CONFLICT DO UPDATE` clause was used). This is primarily
> useful for obtaining values that were supplied by defaults, such as a serial sequence number."
> Example: `INSERT INTO distributors (did, dname) VALUES (DEFAULT, 'XYZ Widgets') RETURNING did;`

Why it matters: the RETURNING section — fetch the just-generated id in the same statement.

---

## 2. PostgreSQL — UPDATE

URL: https://www.postgresql.org/docs/current/sql-update.html

**SET semantics.**
> "UPDATE changes the values of the specified columns in all rows that satisfy the condition. Only
> the columns to be modified need be mentioned in the `SET` clause; columns not explicitly modified
> retain their previous values."

**WHERE — and the missing-WHERE catastrophe.**
> "Only rows for which this expression returns `true` will be updated."

Why it matters: the converse is the centerpiece catastrophe — with no WHERE, *every* row satisfies
(there is no condition to fail), so the whole table is rewritten. (DELETE doc states the all-rows
case explicitly; UPDATE is symmetric.)

**FROM is a PostgreSQL extension (non-standard join-update).**
> "There are two ways to modify a table using information contained in other tables in the database:
> using sub-selects, or specifying additional tables in the `FROM` clause."
> "When a `FROM` clause is present, what essentially happens is that the target table is joined to
> the tables mentioned in the _from_item_ list, and each output row of the join represents an update
> operation for the target table."
> "This command conforms to the SQL standard, except that the `FROM` and `RETURNING` clauses are
> PostgreSQL extensions, as is the ability to use `WITH` with `UPDATE`."

Why it matters: nails claim (c) — `UPDATE ... FROM` is NON-STANDARD. Cite the explicit "FROM ... are
PostgreSQL extensions" line.

**Correlated-subquery is the portable form.**
> "Because of this indeterminacy, referencing other tables only within sub-selects is safer, though
> often harder to read and slower than using a join."

Why it matters: the portable RIGHT alternative — `SET col = (SELECT ... WHERE correlated)` plus a
`WHERE EXISTS` guard. PG itself recommends sub-selects for safety.

**RETURNING.**
> "The optional `RETURNING` clause causes `UPDATE` to compute and return value(s) based on each row
> actually updated."

---

## 3. PostgreSQL — DELETE

URL: https://www.postgresql.org/docs/current/sql-delete.html

**Missing-WHERE catastrophe — stated explicitly.**
> "If the `WHERE` clause is absent, the effect is to delete all rows in the table. The result is a
> valid, but empty table."

Why it matters: the verbatim spine of the centerpiece. A `DELETE FROM t;` is a silent table wipe,
not an error.

**USING is non-standard.**
> "There are two ways to delete rows in a table using information contained in other tables in the
> database: using sub-selects, or specifying additional tables in the `USING` clause."
> "This syntax is not standard."
> Example: `DELETE FROM films USING producers WHERE producer_id = producers.id AND producers.name = 'foo';`

**Standard / portable alternative.**
> "A more standard way to do it is: `DELETE FROM films WHERE producer_id IN (SELECT id FROM producers
> WHERE name = 'foo');`"

Why it matters: claim (c) for DELETE — `USING` non-standard, the `WHERE ... IN (SELECT ...)` /
`WHERE EXISTS` correlated form is portable, and PG labels it "more standard."

**TRUNCATE is faster.**
> "`TRUNCATE` provides a faster mechanism to remove all rows from a table."

**RETURNING.**
> "The optional `RETURNING` clause causes `DELETE` to compute and return value(s) based on each row
> actually deleted."

---

## 4. PostgreSQL — Returning Data From Modified Rows

URL: https://www.postgresql.org/docs/current/dml-returning.html

**Avoids the extra round trip.**
> "Sometimes it is useful to obtain data from modified rows while they are being manipulated. The
> `INSERT`, `UPDATE`, `DELETE`, and `MERGE` commands all have an optional `RETURNING` clause that
> supports this. Use of `RETURNING` avoids performing an extra database query to collect the data,
> and is especially valuable when it would otherwise be difficult to identify the modified rows
> reliably."

Why it matters: the core value proposition + the race argument — fetching a just-inserted id with a
*separate* SELECT is both an extra round trip AND unreliable to identify.

**Generated keys.**
> "when using a `serial` column to provide unique identifiers, `RETURNING` can return the ID assigned
> to a new row"
> Example: `INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool') RETURNING id;`

**SQL:2023 OLD/NEW (data change delta-table direction).**
> "In each of these commands, it is also possible to explicitly return the old and new content of the
> modified row. For example: `UPDATE products SET price = price * 1.10 WHERE price <= 99.99 RETURNING
> name, old.price AS old_price, new.price AS new_price, new.price - old.price AS price_change;`"
> "This syntax for returning old and new values is available in `INSERT`, `UPDATE`, `DELETE`, and
> `MERGE` commands, but typically old values will be `NULL` for an `INSERT`, and new values will be
> `NULL` for a `DELETE`."

Why it matters: claim (d) — RETURNING is non-standard but the SQL:2023 standard added a form
(OLD/NEW rows of a "data change delta table"); PG 18 exposes it as `old.`/`new.`.

---

## 5. SQLite — INSERT

URL: https://www.sqlite.org/lang_insert.html

**Three forms — VALUES (multi-row), SELECT, DEFAULT VALUES.**
> "The first form (with the "VALUES" keyword) creates one or more new rows in an existing table."
> "The second form of the INSERT statement contains a SELECT statement instead of a VALUES clause. A
> new entry is inserted into the table for each row of data returned by executing the SELECT statement."
> "The third form of an INSERT statement is with DEFAULT VALUES. The INSERT ... DEFAULT VALUES
> statement inserts a single new row into the named table."

Why it matters: confirms the same three INSERT forms exist in SQLite — portable across PG and SQLite.
"creates one or more new rows" = multi-row VALUES is standard SQLite too.

---

## 6. SQLite — UPDATE

URL: https://www.sqlite.org/lang_update.html

**Missing-WHERE catastrophe — SQLite's own words.**
> "If the UPDATE statement does not have a WHERE clause, all rows in the table are modified by the
> UPDATE."

Why it matters: a second primary source for the centerpiece — the wipe is cross-engine, not a PG quirk.

**UPDATE-FROM is a non-standard extension, since 3.33.0.**
> "The UPDATE-FROM idea is an extension to SQL that allows an UPDATE statement to be driven by other
> tables in the database."
> "UPDATE-FROM is supported beginning in SQLite version 3.33.0 (2020-08-14)."
> "Other relation database engines also implement UPDATE-FROM, but because the construct is not part
> of the SQL standards, each product implements UPDATE-FROM differently. The SQLite implementation
> strives to be compatible with PostgreSQL."

Why it matters: claim (c) — explicitly "not part of the SQL standards" and "each product implements
UPDATE-FROM differently" — the portability warning verbatim.

---

## 7. SQLite — RETURNING

URL: https://www.sqlite.org/lang_returning.html

> "The RETURNING syntax has been supported by SQLite since version 3.35.0 (2021-03-12)."
> "The effect of the RETURNING clause is to cause the statement to return one result row for each
> database row that is deleted, inserted, or updated."
> "RETURNING is not standard SQL. It is an extension. SQLite's syntax for RETURNING is modelled after
> PostgreSQL."
> "The RETURNING clause is designed to provide the application with the values of columns that are
> filled in automatically by SQLite."

Why it matters: claim (d) portability — RETURNING available in SQLite 3.35+, explicitly "not standard
SQL ... modelled after PostgreSQL." Pins the portability row.

---

## Synthesis → skill sections

- **INSERT forms (set-based, not looped).** Single-row VALUES, MULTI-ROW VALUES in one statement,
  INSERT ... SELECT for bulk copy, DEFAULT VALUES. Sources 1, 5. WRONG = a loop of single-row
  INSERTs; RIGHT = one multi-row VALUES / INSERT...SELECT.
- **Missing-WHERE catastrophe (CENTERPIECE).** UPDATE/DELETE with no WHERE rewrites/wipes the whole
  table — DELETE doc "delete all rows" (3), SQLite UPDATE "all rows ... modified" (6). RIGHT = run
  inside an explicit transaction, SELECT the predicate first, then UPDATE/DELETE with the WHERE.
- **Join-updates are non-standard.** `UPDATE ... FROM` (2, 6) and `DELETE ... USING` (3) are vendor
  extensions; the portable form is a correlated subquery in SET and a `WHERE EXISTS` / `WHERE IN
  (SELECT ...)` guard (2 "sub-selects ... safer", 3 "a more standard way").
- **RETURNING.** Non-standard but widespread (PG, SQLite 3.35+, MariaDB, Oracle); fetches generated
  keys without a second round trip and avoids the identify-the-row race (4); SQL:2023 OLD/NEW form (4).
  Route upsert spelling to `sql-merge-and-upsert`.
- **TRUNCATE vs DELETE.** TRUNCATE faster for "remove all rows" (3); resets identity and is often
  auto-commit / non-transactional on MySQL/Oracle → route dialect specifics to dialect-map.
- **Set-based over row-by-row** is the through-line: one statement the engine optimizes and the
  transaction wraps, vs N round trips.

### Portability summary (for the skill table)
| Feature | PostgreSQL | SQLite | MySQL/MariaDB | Oracle |
|---|---|---|---|---|
| Multi-row `VALUES`, `INSERT ... SELECT` | yes | yes | yes | `INSERT ALL` / `SELECT` |
| `RETURNING` | yes | 3.35+ | MariaDB yes / MySQL **no** | yes |
| `UPDATE ... FROM` | yes | 3.33+ | different syntax | no (use subquery) |
| `DELETE ... USING` | yes | (USING n/a; subquery) | different syntax | no |
| `TRUNCATE` transactional | yes (DDL, rollbackable) | no TRUNCATE (use DELETE) | **auto-commit** | **auto-commit** |
