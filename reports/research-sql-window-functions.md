# Research: sql-window-functions

Primary-source research for the `sql-window-functions` skill. Each entry: URL, section, verbatim
quote, and why it matters for the skill. Accessed 2026-06-26.

---

## Source 1 — PostgreSQL: Window Functions (tutorial)

URL: https://www.postgresql.org/docs/current/tutorial-window.html

### Quote A — windows don't collapse rows (section: intro)
> "A _window function_ performs a calculation across a set of table rows that are somehow related to the current row. This is comparable to the type of calculation that can be done with an aggregate function. However, window functions do not cause rows to become grouped into a single output row like non-window aggregate calls would. Instead, the rows retain their separate identities."

**Why it matters:** This is the *defining* property and the PARTITION BY vs GROUP BY distinction. GROUP BY collapses N rows to one per group; a window keeps all N rows and attaches a per-row computed value. The single most common conceptual error LLMs make is treating `PARTITION BY` like `GROUP BY`.

### Quote B — PARTITION BY semantics (section: intro)
> "The `PARTITION BY` clause within `OVER` divides the rows into groups, or partitions, that share the same values of the `PARTITION BY` expression(s). For each row, the window function is computed across the rows that fall into the same partition as the current row."

**Why it matters:** Defines partition = the set the function sees; without PARTITION BY the whole result is one partition.

### Quote C — cannot filter on a window in WHERE/GROUP BY/HAVING (CENTERPIECE for §6)
> "Window functions are permitted only in the `SELECT` list and the `ORDER BY` clause of the query. They are forbidden elsewhere, such as in `GROUP BY`, `HAVING` and `WHERE` clauses. This is because they logically execute after the processing of those clauses."

**Why it matters:** This is the exact wording behind the classic `WHERE row_number() = 1` error. The cause is logical execution order, not a syntax quirk.

### Quote D — the fix is a sub-select (CTE/subquery)
> "If there is a need to filter or group rows after the window calculations are performed, you can use a sub-select."

PG's own top-N-per-group example:
```sql
SELECT depname, empno, salary, enroll_date
FROM
  (SELECT depname, empno, salary, enroll_date,
     row_number() OVER (PARTITION BY depname ORDER BY salary DESC, empno) AS pos
     FROM empsalary
  ) AS ss
WHERE pos < 3;
```

**Why it matters:** Canonical top-N-per-group pattern. Note the total ORDER BY (salary DESC, empno) to break ties deterministically — ties back to the foundation skill (§1, undefined order).

### Quote E — windows run after aggregates
> "Also, window functions execute after non-window aggregate functions. This means it is valid to include an aggregate function call in the arguments of a window function, but not vice versa."

**Why it matters:** Explains why `SUM(SUM(x)) OVER (...)` is legal but you cannot nest a window inside an aggregate.

---

## Source 2 — PostgreSQL: Window Functions (function reference)

URL: https://www.postgresql.org/docs/current/functions-window.html

### Quote A — ranking functions (the tie-behavior trio)
> `row_number` "Returns the number of the current row within its partition, counting from 1."
> `rank` "Returns the rank of the current row, with gaps; that is, the `row_number` of the first row in its peer group."
> `dense_rank` "Returns the rank of the current row, without gaps; this function effectively counts peer groups."

**Why it matters:** The exact tie semantics. ROW_NUMBER gives every row a distinct number (arbitrary among peers); RANK leaves gaps after ties (1,1,3); DENSE_RANK does not (1,1,2). Picking the wrong one is a silent off-by-one in leaderboards/top-N.

### Quote B — ntile
> `ntile` "Returns an integer ranging from 1 to the argument value, dividing the partition as equally as possible."

### Quote C — lag / lead (offset)
> `lag` "Returns _value_ evaluated at the row that is _offset_ rows before the current row within the partition; if there is no such row, instead returns _default_ ... If omitted, _offset_ defaults to 1 and _default_ to `NULL`."
> `lead` "Returns _value_ evaluated at the row that is _offset_ rows after the current row within the partition..."

**Why it matters:** The `default` third arg avoids NULL at partition edges (first row's LAG, last row's LEAD).

### Quote D — first_value / last_value / nth_value (value functions, frame-sensitive)
> `first_value` "Returns _value_ evaluated at the row that is the first row of the window frame."
> `last_value` "Returns _value_ evaluated at the row that is the last row of the window frame."
> `nth_value` "Returns _value_ evaluated at the row that is the _n_'th row of the window frame (counting from 1); returns `NULL` if there is no such row."

### Quote E — THE LAST_VALUE SURPRISE (centerpiece)
> "Note that `first_value`, `last_value`, and `nth_value` consider only the rows within the 'window frame', which by default contains the rows from the start of the partition through the last peer of the current row. This is likely to give unhelpful results for `last_value` and sometimes also `nth_value`. You can redefine the frame by adding a suitable frame specification (`RANGE`, `ROWS` or `GROUPS`) to the `OVER` clause."

**Why it matters:** PG itself warns LAST_VALUE "is likely to give unhelpful results." LAST_VALUE over the default frame returns the *current row's* value (the frame ends at the current row's last peer), NOT the partition maximum. This is the silent wrong-number bug. Fix: explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, or just `MAX() OVER (...)`. Note: row_number, rank, dense_rank, ntile, lag, lead are NOT frame-sensitive — only first/last/nth_value are.

---

## Source 3 — PostgreSQL: Window Function Calls (syntax / frame clause)

URL: https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS

### Quote A — frame grammar
```
{ RANGE | ROWS | GROUPS } frame_start [ frame_exclusion ]
{ RANGE | ROWS | GROUPS } BETWEEN frame_start AND frame_end [ frame_exclusion ]
```
where frame_start/frame_end ∈ { UNBOUNDED PRECEDING | offset PRECEDING | CURRENT ROW | offset FOLLOWING | UNBOUNDED FOLLOWING }
and frame_exclusion ∈ { EXCLUDE CURRENT ROW | EXCLUDE GROUP | EXCLUDE TIES | EXCLUDE NO OTHERS }

### Quote B — THE DEFAULT FRAME (verbatim, the centerpiece)
> "The default framing option is `RANGE UNBOUNDED PRECEDING`, which is the same as `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. With `ORDER BY`, this sets the frame to be all rows from the partition start up through the current row's last `ORDER BY` peer. Without `ORDER BY`, this means all rows of the partition are included in the window frame, since all rows become peers of the current row."

**Why it matters:** The exact source for both centerpiece traps. (a) With ORDER BY, default frame ends at "the current row's last ORDER BY peer" → LAST_VALUE = current row, not partition end. (b) RANGE lumps peers: "all rows from the partition start up through the current row's last ORDER BY peer" — when sort keys tie, every tied row gets the *same* running total (the total through the whole peer group), so running totals jump in steps instead of incrementing per row.

### Quote C — ROWS vs RANGE vs GROUPS
- ROWS: "the frame starts or ends the specified number of rows before or after the current row"; CURRENT ROW = the current row.
- GROUPS: "the frame starts or ends the specified number of _peer groups_ before or after the current row's peer group"; a peer group is "a set of rows that are equivalent in the `ORDER BY` ordering"; requires an ORDER BY.
- RANGE: "require that the `ORDER BY` clause specify exactly one column"; offset is "the maximum difference between the value of that column in the current row and its value in preceding or following rows of the frame" (interval for datetime columns).

**Why it matters:** ROWS counts physical rows (per-row running total); RANGE counts by value and includes all peers (steps on ties). For a per-row running total over a non-unique sort key, you MUST use ROWS.

### Quote D — EXCLUDE
> EXCLUDE CURRENT ROW (excludes current row), EXCLUDE GROUP (current row + ordering peers), EXCLUDE TIES (peers but not the current row), EXCLUDE NO OTHERS (default, excludes nothing).

### Quote E — named WINDOW
> "`OVER wname` is not exactly equivalent to `OVER (wname ...)`; the latter implies copying and modifying the window definition, and will be rejected if the referenced window specification includes a frame clause."

**Why it matters:** The `WINDOW w AS (...)` clause lets you define a window once and reuse it across several SELECT-list functions — DRY and prevents subtly divergent copies.

---

## Source 4 — SQLite: Window Functions

URL: https://www.sqlite.org/windowfunctions.html

### Quote A — version support
> "Window function support was first added to SQLite with release version 3.25.0 (2018-09-15)."
> "In SQLite version 3.28.0 (2019-04-16), windows function support was extended to include the EXCLUDE clause, GROUPS frame types, window chaining, and support for '<expr> PRECEDING' and '<expr> FOLLOWING' boundaries in RANGE frames."

### Quote B — same default frame as PG
> "The default frame-spec is: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW EXCLUDE NO OTHERS. The default means that aggregate window functions read all rows from the beginning of the partition up to and including the current row and its peers."

**Why it matters:** Confirms the default frame is identical across PG and SQLite → the LAST_VALUE trap and RANGE-on-ties trap are portable, not PG-specific.

### Quote C — frame types & EXCLUDE & 11 functions
> Three frame types ROWS/GROUPS/RANGE; four EXCLUDE forms (NO OTHERS / CURRENT ROW / GROUP / TIES); 11 built-in window functions: row_number, rank, dense_rank, percent_rank, cume_dist, ntile, lag, lead, first_value, last_value, nth_value.

**Why it matters:** Full feature parity with the standard incl. GROUPS/EXCLUDE since 3.28.

---

## Source 5 — modern-sql.com (vendor-neutral portability + QUALIFY)

`/feature/over` returned HTTP 404 (page moved/removed as of 2026-06-26). Substituted two live
modern-sql caniuse pages that carry the portability matrix and the window-in-WHERE angle.

URL: https://modern-sql.com/caniuse/qualify

### Quote A — windows cannot go in WHERE/HAVING; QUALIFY can
> "Neither the `where` nor the `having` clause allow window functions—but the `qualify` clause does."

**Why it matters:** Vendor-neutral confirmation of the PG rule. QUALIFY is the convenience filter (filter directly on a window result) but is **non-standard** — per the matrix it is supported by DuckDB, H2, and others (notably Snowflake/BigQuery/Teradata), and NOT by PostgreSQL, MySQL, MariaDB, SQL Server, or Oracle. So the portable answer remains the wrap-in-subquery/CTE pattern.

URL: https://modern-sql.com/caniuse/over_rows_between

### Quote B — ROWS framing + portability matrix
> "The `over` accepts `rows` framing, which limits the scope of the window functions to the rows between the specified range of rows relative to the current row." "Meaningful framing requires an `order by` clause in `over` as well."

Support: PostgreSQL ✓ 9.0+, SQLite ✓, MySQL ✓ 8.0.11+, MariaDB ✓ 10.2+, SQL Server ✓ 2012+, Oracle ✓ 11gR1+.

**Why it matters:** Portability table for the skill. Window functions are SQL:2003; broadly available. MySQL added them only in 8.0 (2018) — pre-8.0 MySQL has no window functions at all (route emulation to the dialect-map). Full GROUPS/EXCLUDE: PostgreSQL (11+) and SQLite (3.28+); MySQL 8 lacks GROUPS and EXCLUDE.

---

## Synthesis — the points the skill must nail

1. **Windows don't collapse rows** (Src1-A); PARTITION BY ≠ GROUP BY (Src1-B).
2. **Ranking tie behavior**: ROW_NUMBER distinct / RANK gaps / DENSE_RANK no gaps (Src2-A).
3. **Offset (LAG/LEAD)** with default arg; **value (FIRST/LAST/NTH_VALUE)** frame-sensitive (Src2-C/D).
4. **DEFAULT FRAME = RANGE ... CURRENT ROW with ORDER BY** (Src3-B, Src4-B) → LAST_VALUE returns current row, not partition max (Src2-E). Fix: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` or `MAX() OVER`.
5. **ROWS vs RANGE on ties** (Src3-B/C): RANGE includes the whole peer group → running total steps/jumps; ROWS counts physical rows → per-row total. Use ROWS for running totals on non-unique sort keys.
6. **Cannot filter a window in WHERE/HAVING** (Src1-C, Src5-A) → wrap in CTE/subquery (Src1-D); QUALIFY is non-standard.
7. **Named WINDOW clause** to share a definition (Src3-E).
8. **EXCLUDE / GROUPS** brief (Src3-C/D, Src4-C).
9. **Portability** (Src5-B): SQL:2003; PG full incl GROUPS/EXCLUDE; SQLite full since 3.28; MySQL 8+/MariaDB 10.2+ (no GROUPS/EXCLUDE on MySQL 8); pre-8 MySQL none.
