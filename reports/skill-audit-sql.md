# Skill Audit — Standard SQL Skills Plugin

**Target:** `plugins/sql/skills/` (30 skills)
**Bar:** `golang-skills-plugin/.../go-error-handling/SKILL.md`
**Plan:** `reports/skill-plan-sql.md`
**Auditor role:** analyst (find problems, not approve)
**Date:** 2026-06-26

---

## Overall verdict: **PASS**

The set meets and in several places exceeds the golang `go-error-handling` bar. All 30 skills are structurally complete, richly cited against primary sources, carry WRONG/RIGHT (or an equivalent BEFORE/AFTER) code, a portability block, a "Who suffers" section, a routing section, and `${CLAUDE_SKILL_DIR}/references/...` pointers. Cross-link integrity is **perfect** within the plugin: every `sql-*` name referenced in any SKILL body resolves to a real sibling directory (zero broken links). The two MVCC-boundary skills stay cleanly in their lane and route theory out without duplicating the anomaly catalog.

There are **no must-fix errors**. The findings below are warnings and polish: a handful of un-revalidated `modern-sql.com/feature/*` citations in the same URL family as two that were already found dead and replaced, two reference-style skills that use a non-`WRONG/RIGHT` code idiom, unverifiable external `mvcc-*` link targets, and two genuine coverage gaps (core DML; pattern-matching/collation).

---

## Per-skill verdict table

Legend: FM = frontmatter quality, Cite = citation density (spot-checked where noted), W/R = WRONG/RIGHT present, Struct = pull-quote + foundation link + portability + who-suffers + routing + reference pointers, Refs = common-mistakes.md + sources.yaml present & non-trivial.

| # | Skill | FM | Cite | W/R | Struct | Refs | Verdict | Note |
|---|-------|----|----|----|----|----|---------|------|
| 1 | sql-relational-and-null-discipline | ✓ | ✓ deep-read | ✓ (6) | ✓ | ✓ 193/98 | **PASS** | exemplary; policy root |
| 2 | sql-select-and-query-processing | ✓ | ✓ | ✓ (3) | ✓ | ✓ 168/46 | **PASS** | |
| 3 | sql-joins | ✓ | ✓ | ✓ (5) | ✓ | ✓ 141/50 | **PASS** | |
| 4 | sql-subqueries-and-exists | ✓ | ✓ | ✓ (2) | ✓ | ✓ 133/34 | **PASS** | |
| 5 | sql-aggregation-and-grouping | ✓ | ✓ | ✓ (4) | ✓ | ✓ 255/61 | **PASS** | uses /feature/filter (see W2) |
| 6 | sql-set-operations | ✓ | ✓ | ✓ (4) | ✓ | ✓ 226/41 | **PASS** | |
| 7 | sql-window-functions | ✓ | ✓ deep-read | ✓ (5) | ✓ | ✓ 180/69 | **PASS** | default-frame/LAST_VALUE claim correct & cited (PG+SQLite) |
| 8 | sql-cte-and-recursion | ✓ | ✓ | ✓ (3) | ✓ | ✓ 212/45 | **PASS** | uses /feature/with (see W2) |
| 9 | sql-lateral-and-correlated-derived | ✓ | ✓ | ✓ (3) | ✓ | ✓ 289/52 | **PASS** | |
| 10 | sql-expressions-case-and-functions | ✓ | ✓ | ✓ (7) | ✓ | ✓ 227/38 | **PASS** | |
| 11 | sql-datetime-and-intervals | ✓ | ✓ | ✓ (5) | ✓ | ✓ 207/40 | **PASS** | |
| 12 | sql-data-types-and-numerics | ✓ | ✓ | ✓ (6) | ✓ | ✓ 167/44 | **PASS** | |
| 13 | sql-constraints-and-integrity | ✓ | ✓ | ✓ (5) | ✓ | ✓ 177/38 | **PASS** | |
| 14 | sql-injection-and-parameterization | ✓ | ✓ deep-read | ✓ (5) | ✓ | ✓ 174/38 | **PASS** | OWASP-anchored; "value never enters SQL text" |
| 15 | sql-merge-and-upsert | ✓ | ✓ | ✓ (4) | ✓ | ✓ 133/37 | **PASS** | routes concurrency to txns/MVCC |
| 16 | sql-pagination-and-keyset | ✓ | ✓ deep-read | ✓ (2) | ✓ | ✓ 190/62 | **PASS** | absorbs row-values per plan #5; sources.yaml clean |
| 17 | sql-generated-and-identity-columns | ✓ | ✓ | ✓ (3) | ✓ | ✓ 149/58 | **PASS** | |
| 18 | sql-views-and-introspection | ✓ | ✓ | ✓ (3) | ✓ | ✓ 173/40 | **PASS** | |
| 19 | sql-json | ✓ | ✓ | ✓ (5) | ✓ | ✓ 181/59 | **PASS** | |
| 20 | sql-gaps-and-islands | ✓ | ✓ deep-read | ✓ (2) | ✓ | ✓ 205/60 | **PASS** | distinct from window-functions; routes mechanics there |
| 21 | sql-schema-design-and-normalization | ✓ | ✓ | ✓ (5) | ✓ | ✓ 169/57 | **PASS** | |
| 22 | sql-style-and-naming | ✓ | ✓ | ✓ (5) | ✓ | ✓ 161/47 | **PASS** | |
| 23 | sql-transactions-and-isolation | ✓ | ✓ deep-read | ✓ (1) | ✓ | ✓ 183/64 | **PASS** | MVCC boundary clean (see §MVCC) |
| 24 | sql-privileges-and-access-control | ✓ | ✓ | ✓ (5) | ✓ | ✓ 148/67 | **PASS** | |
| 25 | sql-indexing-and-sargability | ✓ | ✓ | ✓ (4) | ✓ | ✓ 198/104 | **PASS** | |
| 26 | sql-explain-and-set-based-thinking | ✓ | ✓ | ✓ (3) | ✓ | ✓ 198/55 | **PASS** | |
| 27 | sql-standard-vs-dialect-map | ✓ | ✓ deep-read | ~ matrix | ✓ | ✓ 222/225 + dialect-matrix.md | **PASS** | reference matrix; no literal WRONG/RIGHT by design (W3) |
| 28 | sql-temporal-tables | ✓ | ✓ deep-read | ✓ (2) | ✓ | ✓ 195/49 | **PASS** | MVCC boundary clean (see §MVCC) |
| 29 | sql-match-recognize | ✓ | ✓ deep-read | ~ BEFORE/AFTER | ✓ | ✓ 140/33 | **PASS** | low-portability flagged; uses BEFORE/AFTER not WRONG/RIGHT (W3) |
| 30 | sql-property-graph-queries | ✓ | ✓ deep-read | ✓ (2) | ✓ | ✓ 125/69 | **PASS** | bleeding-edge flagged; routes portable fallback to CTE |

(W/R column shows count of "WRONG" markers in the SKILL body from the grep sweep.)

---

## Frontmatter & description quality (check 1, 2)

- **All 30**: `name` matches directory; `allowed-tools: Read, Glob, Grep`; `compatibility: "Claude Code, Codex CLI, Gemini CLI"` present.
- **All 30** descriptions carry an explicit `Auto-invokes when ...` trigger clause.
- **No description contains "this skill"** (the banned standalone phrase). The 40+ "this skill" hits from grep are all body prose ("this skill owns ...", "Source provenance for every claim in this skill") — legitimate.
- **Length**: descriptions run ~150–220 words — rich, multi-clause, trigger-laden, modeled on the golang style. None exceeds the ~600-word flag; none is too thin. The advanced/low-portability three (temporal, match-recognize, property-graph) correctly embed an explicit low/very-low-portability + "confirm engine support" caveat directly in the description — a good trigger surface that pre-warns the agent.
- Descriptions are genuinely good *trigger surfaces*: they name the exact statements/operators (`OVER`, `PARTITION BY`, `NOT IN`, `START TRANSACTION`, `GRAPH_TABLE`) and the symptom phrasings ("why does my query return no rows", "filter on row_number", "what did this row look like last March") an agent would actually hit.

## Citations (check 3)

Spot-checked 8 skills in full (foundation, window-functions, transactions, temporal-tables, gaps-and-islands, standard-vs-dialect-map, property-graph, injection) plus two `sources.yaml`. Every factual claim carries an inline primary-source link with a quoted fragment — PostgreSQL docs, SQLite docs, MySQL docs, modern-sql.com, OWASP, Oracle docs, MariaDB/Microsoft Learn for temporal, Trino/Oracle/arXiv for the advanced three. The window-functions default-frame claim (the centerpiece) is correctly quoted from **both** PostgreSQL and SQLite, which is exactly the rigor the bar wants. `sources.yaml` files use the correct shape (`name` / `url` / `type` / `accessed` / `sections_used`), and several add a `note:` field documenting source substitutions. **No skill asserts facts without citation** in the sampled set, and the structural markers indicate the same pattern across all 30.

## WRONG/RIGHT code & SQL correctness (check 4)

- 28/30 use literal `WRONG`/`RIGHT` comment markers with portable standard SQL; non-standard spellings are explicitly labelled (e.g. window §4 marks the MariaDB `WITH SYSTEM VERSIONING` spelling inline; dialect map rows mark every vendor spelling).
- Sampled SQL is correct: the `ROW_NUMBER`-difference island trick, the `LAST_VALUE` default-frame fix, the `NOT IN`→`NOT EXISTS` anti-join, the keyset `(sort_key, id) > (...)` predicate, the temporal half-open `[start,end)` boundary semantics, and the psycopg `%s`-is-a-bind-placeholder-not-`%`-formatting distinction are all accurate.
- The 2 skills without literal WRONG/RIGHT are by design (see W3).

## Structure (check 5) & references (check 6)

- Pull-quote(s) at top: present in all sampled skills.
- Foundation cross-link `sql-relational-and-null-discipline`: present in all 30.
- Portability block: all 30 (headings vary — "Portability Snapshot" is the norm; low-portability skills use "Portability — LOW/VERY LOW").
- "Who suffers" section: all 30.
- "Routing to related skills": 29/30 — the dialect map (#27) intentionally uses hub phrasing ("the index every other SQL skill's Portability section links to") instead, which is correct for the cross-cutting anchor and not a defect.
- `${CLAUDE_SKILL_DIR}/references/common-mistakes.md` + `sources.yaml`: all 30 present and non-trivial (common-mistakes 125–289 lines; sources 33–225 lines). The dialect map additionally ships `references/dialect-matrix.md` (the exhaustive feature×engine matrix), pointed to from the SKILL — a good split.

## Cross-link integrity (check 7) — **clean**

Extracted every `` `sql-*` `` token from all 30 SKILL bodies (30 distinct names) and diffed against the 30 real directories: **zero referenced names fail to resolve.** No dangling links, no typos, no references to folded/renamed skills. `sql-row-values-and-comparisons` (folded into pagination per plan #5) is correctly absent and never linked.

## MVCC boundary (check 8) — **held**

- **sql-transactions-and-isolation**: States the anomaly catalog in exactly one paragraph (§4), then a blockquoted "⚠ MVCC boundary — route all isolation theory out" callout that explicitly hands `mvcc-isolation-levels-and-anomalies`, `mvcc-snapshot-isolation`, `mvcc-serializable-ssi`, `mvcc-write-skew-and-conflict-materialization`, `mvcc-choosing-isolation` the theory. The line "This skill teaches the names and the SET syntax; that plugin teaches what the levels mean" is the boundary stated crisply. Does **not** duplicate the per-level/per-anomaly matrix. **In lane.**
- **sql-temporal-tables**: §6 "The MVCC Boundary (hard)" splits "language" (this skill: DDL + `FOR SYSTEM_TIME` clauses) from "mechanics" (`mvcc-time-travel-queries`: version storage, snapshot computation, retention/GC horizon). Does not duplicate snapshot mechanics. **In lane.**

---

## Set-level findings

### Consistency — strong
Uniform voice and section architecture across all 30: top pull-quote(s) → foundation-link paragraph → numbered sections with a designated "centerpiece" → Portability → Who Suffers → Routing → Reference Files. The portability idiom "the standard says X; PostgreSQL ✓ / SQLite ~ / MySQL ✗ (spells Y)" is explicitly defined in the dialect map (#27) and used as snapshot tables throughout. This is more internally consistent than a typical multi-author skill set.

### Overlap / duplication — none material
- **gaps-and-islands vs window-functions**: gaps-and-islands is a recipe cookbook that *assumes* and routes frame/`ROWS`-vs-`RANGE` mechanics to window-functions; window-functions routes the consecutive-runs recipe out to gaps-and-islands. Clean complementary split, no duplicated mechanics.
- **pagination vs indexing**: pagination routes "index support that makes keyset O(1)/page" to indexing-and-sargability; indexing routes keyset *usage* back. No overlap.
- **transactions vs merge-and-upsert**: merge routes upsert concurrency/isolation to transactions (+MVCC); transactions routes the check-then-act upsert form to merge. No overlap.
- **expressions vs datetime vs data-types**: each routes the others' territory out explicitly (CAST → expressions, EXTRACT/INTERVAL → datetime, precision/numerics → data-types). Clean.

### Gaps — two worth noting (out-of-scope items already excluded correctly)
The known out-of-scope set (stored procedures/PSM, triggers, FTS/geo/vector, connection pooling) is correctly excluded. Two coverage gaps in *portable standard SQL* are not owned by any skill:

1. **Core DML has no home skill.** There is no dedicated skill for `INSERT` / `UPDATE` / `DELETE` fundamentals — multi-row `INSERT ... VALUES`, `INSERT ... SELECT`, searched `UPDATE`/`DELETE` with subqueries, the standard `MERGE` is covered but plain DML and the (non-standard but ubiquitous) `RETURNING`/`OUTPUT` clause are scattered (VALUES lives in pagination, upsert in merge). An agent asking "how do I bulk-update from another table portably" (`UPDATE ... FROM` PG vs `UPDATE ... JOIN` MySQL — a real portability trap) has no owning skill. **Recommend: consider a `sql-data-modification` skill or fold a DML section into an existing one.**
2. **Pattern matching & collation are unowned.** `LIKE` is mentioned only as a sargability hazard in indexing; `SIMILAR TO`, `LIKE ... ESCAPE`, `_`/`%` semantics, case-sensitivity, and `COLLATE`/collation portability (a notorious cross-engine minefield — `utf8mb4_*` vs `C` vs `en_US`) are not taught anywhere. **Recommend: a short `sql-text-matching-and-collation` skill, or sections in expressions + dialect-map.**

Both are WARN-level gaps, not blockers — they were not in the plan's locked 30 and the plugin is internally complete as scoped.

### Citation health — one watch item
Two `modern-sql.com/feature/*` URLs were found dead during drafting and **correctly** replaced with documented live equivalents (a `note:` in window-functions `sources.yaml` records `/feature/over` → `/caniuse/qualify`; pagination dropped `/feature/fetch-first` for antonz.org + use-the-index-luke). However, **three sibling URLs in the same `/feature/` family remain in use and were not re-validated**: `/feature/match_recognize` (match-recognize, gaps-and-islands, dialect-map), `/feature/with` (cte-and-recursion), `/feature/filter` (aggregation). Given two neighbors in this exact URL family 404'd, these warrant a manual link-check before publishing. (No web access in this audit to confirm live/dead.)

---

## Errors (must-fix)
**None.**

## Warnings (nice-to-fix)
- **W1 — Un-revalidated citations.** Link-check `modern-sql.com/feature/match_recognize`, `/feature/with`, `/feature/filter` (in match-recognize, gaps-and-islands, dialect-map, cte-and-recursion, aggregation, and the corresponding `sources.yaml`/`common-mistakes.md`). Two siblings in this family were already dead. If any 404, substitute a live `modern-sql.com/caniuse/*` or vendor-doc equivalent as was done for `/feature/over`.
- **W2 — External `mvcc-*` link targets unverifiable.** sql-transactions-and-isolation and sql-temporal-tables link `mvcc-isolation-levels-and-anomalies`, `mvcc-snapshot-isolation`, `mvcc-serializable-ssi`, `mvcc-write-skew-and-conflict-materialization`, `mvcc-choosing-isolation`, `mvcc-time-travel-queries` in the sibling `~/code/mvcc-skills-plugin`. These could not be checked against that plugin from here. Confirm those six skill names exist there (the plan's locked-decision #6 listed slightly different names, e.g. it did not name `mvcc-write-skew-and-conflict-materialization` explicitly).
- **W3 — Two reference skills omit literal WRONG/RIGHT.** sql-match-recognize uses BEFORE/AFTER (ugly-portable vs clean) and sql-standard-vs-dialect-map uses ✓/~/✗ matrices instead of WRONG/RIGHT comment markers. Both are defensible for their type (a low-portability feature contrast and a portability index), but they break the otherwise-uniform WRONG/RIGHT idiom. Optional: add at least one explicit WRONG/RIGHT pair to each (e.g. dialect map: `LIMIT 10` shipped to Oracle 11g) for consistency with the other 28.
- **W4 — Coverage gaps.** Core DML (`INSERT`/`UPDATE`/`DELETE`, `UPDATE ... FROM` portability, `RETURNING`) and text-matching/collation (`LIKE`/`SIMILAR TO`/`COLLATE`) are unowned. Consider one or two added skills or sections (see Gaps above).

---

## Evidence trail
Structural sweep covered all 30 SKILL.md via Grep (frontmatter fields, Auto-invokes, "this skill", Who-suffers, Routing, foundation link, `${CLAUDE_SKILL_DIR}` pointers, Portability headings, WRONG markers, `sql-*` link resolution, `mvcc-*` links, `modern-sql.com/feature/*` URLs). References-file existence and line counts verified on disk for all 30. Deep full/partial reads: `sql-relational-and-null-discipline`, `sql-transactions-and-isolation`, `sql-temporal-tables`, `sql-window-functions`, `sql-gaps-and-islands`, `sql-standard-vs-dialect-map`, `sql-property-graph-queries`, `sql-injection-and-parameterization`, plus `sql-match-recognize` head and two `sources.yaml`.
