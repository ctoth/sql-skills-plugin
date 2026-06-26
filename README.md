# SQL Skills

A [Claude Code](https://claude.com/claude-code) plugin: **32 agent skills for writing correct, portable, modern standard SQL** (ISO/IEC 9075).

The skills are vendor-neutral. They teach the *standard* spelling of each feature, name the correctness traps that LLMs reliably fall into, and map portability per feature across PostgreSQL, MySQL/MariaDB, SQLite, SQL Server, Oracle, and DuckDB — so the SQL an agent writes is right the first time and runs where you need it.

## What this is for

Most SQL goes wrong in the same handful of places: a filter on the null-able side of an outer join silently demoting it to an inner join; `NOT IN` collapsing to zero rows because one value was NULL; `FLOAT` chosen for money; string-interpolated queries inviting injection. These skills auto-invoke when Claude is about to write or edit the relevant SQL and steer it onto the standard, safe path.

Each skill is self-contained: a `SKILL.md` with the rule and worked examples, plus a `references/common-mistakes.md` catalog of the specific antipatterns it guards against.

## Installation

Add the marketplace, then install the `sql` plugin:

```
/plugin marketplace add ctoth/sql-skills-plugin
/plugin install sql@sql-skills-marketplace
```

The skills activate automatically based on what you're writing — no commands to remember. You can also browse them with `/plugin`.

## Skills

### Foundations — the correctness floor
| Skill | Covers |
| --- | --- |
| `sql-relational-and-null-discipline` | Results are unordered sets; NULL means *unknown* and propagates UNKNOWN; WHERE/HAVING/ON keep only TRUE, CHECK rejects only FALSE. |
| `sql-select-and-query-processing` | Logical clause-evaluation order (FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → UNION → ORDER BY → FETCH) and what each clause can reference. |
| `sql-style-and-naming` | Readable, reviewable SQL — plus the `'literal'` vs `"identifier"` quoting trap. |

### Querying
| Skill | Covers |
| --- | --- |
| `sql-joins` | Join composition and the outer-join-filter-in-WHERE bug that silently drops rows. |
| `sql-subqueries-and-exists` | Correlated subqueries and the `NOT IN` + NULL trap; `NOT EXISTS` as the safe anti-join. |
| `sql-aggregation-and-grouping` | The functional-dependency GROUP BY rule, WHERE vs HAVING, `FILTER (WHERE …)`, GROUPING SETS/ROLLUP/CUBE. |
| `sql-window-functions` | Window functions and the frame-clause traps (`ROWS` vs `RANGE`, default frames). |
| `sql-set-operations` | UNION vs UNION ALL (and the dedup cost trap), INTERSECT, EXCEPT. |
| `sql-cte-and-recursion` | `WITH` for decomposition and `WITH RECURSIVE` with termination/cycle guards. |
| `sql-lateral-and-correlated-derived` | `LATERAL` — a FROM-clause derived table that can reference preceding items. |
| `sql-expressions-case-and-functions` | Portable scalar expressions: `CASE`, `COALESCE`, `NULLIF`, `||`, standard string functions. |
| `sql-pattern-matching-and-collation` | `LIKE`/`ESCAPE` and how collation silently changes case sensitivity across engines. |

### Data types & expressions
| Skill | Covers |
| --- | --- |
| `sql-data-types-and-numerics` | Type selection by meaning — exact `NUMERIC(p,s)` for money, never binary `FLOAT`. |
| `sql-datetime-and-intervals` | Temporal types, `TIMESTAMP WITH TIME ZONE`, typed literals, intervals. |
| `sql-json` | Standard SQL/JSON (`JSON_VALUE`/`JSON_QUERY`/`JSON_EXISTS`/`JSON_TABLE`) over vendor operators. |

### Writing data
| Skill | Covers |
| --- | --- |
| `sql-data-modification` | INSERT/UPDATE/DELETE as set-based operations, not row-by-row loops. |
| `sql-merge-and-upsert` | Atomic upsert instead of a race-prone SELECT-then-INSERT-or-UPDATE. |
| `sql-generated-and-identity-columns` | `GENERATED … AS IDENTITY` and computed columns over vendor `SERIAL`/`AUTO_INCREMENT`. |

### Schema & integrity
| Skill | Covers |
| --- | --- |
| `sql-schema-design-and-normalization` | The 1NF→BCNF ladder, natural vs surrogate keys, junction tables, deliberate denormalization. |
| `sql-constraints-and-integrity` | Push integrity into the schema as declarative constraints, not app-side re-checks. |
| `sql-views-and-introspection` | Views as stored queries, updatability rules, `CHECK OPTION`, and `INFORMATION_SCHEMA`. |

### Performance & scale
| Skill | Covers |
| --- | --- |
| `sql-indexing-and-sargability` | Index design and the sargability rule — keep the indexed column bare in predicates. |
| `sql-explain-and-set-based-thinking` | Think in sets, not rows; measure the plan instead of guessing. |
| `sql-pagination-and-keyset` | Scalable keyset pagination and the row-value machinery behind it. |

### Security & transactions
| Skill | Covers |
| --- | --- |
| `sql-injection-and-parameterization` | The non-negotiable rule: bind every value; never interpolate SQL. |
| `sql-privileges-and-access-control` | `GRANT`/`REVOKE`, roles, and default-deny access. |
| `sql-transactions-and-isolation` | Transaction boundaries, savepoints, the four isolation levels, DDL transactionality. |

### Patterns & modern SQL
| Skill | Covers |
| --- | --- |
| `sql-gaps-and-islands` | The canonical runs-and-gaps pattern, set-based instead of procedural. |
| `sql-temporal-tables` | SQL:2011 system-versioned tables and `FOR SYSTEM_TIME` queries. |
| `sql-match-recognize` | SQL:2016 `MATCH_RECOGNIZE` row-pattern recognition for time-series. |
| `sql-property-graph-queries` | SQL:2023 SQL/PGQ — property graphs over relational tables. |

### Portability
| Skill | Covers |
| --- | --- |
| `sql-standard-vs-dialect-map` | The portability index — each standard feature mapped to engine support and spelling. |

## Layout

```
.claude-plugin/marketplace.json     marketplace manifest
plugins/sql/
  .claude-plugin/plugin.json        plugin manifest
  skills/<skill-name>/
    SKILL.md                        rule + worked examples
    references/common-mistakes.md   antipattern catalog
reports/                            research, planning, and audit notes
```
