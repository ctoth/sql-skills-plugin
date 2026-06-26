# Skillbuilder orchestration notes — SQL skills plugin

Goal: comprehensive SOTA *standard* SQL skill set (more comprehensive than golang's 25), heavily cited
to primary sources (ISO 9075 summaries, Postgres/SQLite docs, modern-sql.com, use-the-index-luke,
SQL Antipatterns), portable-only. Vendor specifics deferred to future per-DB plugins.

## Layout
- Plugin root: ~/code/sql-skills-plugin
- Skills: plugins/sql/skills/<name>/SKILL.md (+ references/common-mistakes.md, sources.yaml, domain refs)
- Reports: reports/
- Model reference for quality bar: ~/code/golang-skills-plugin/plugins/golang/skills/go-error-handling

## Quality bar (from golang plugin)
- Frontmatter: name, description (rich auto-invoke triggers), allowed-tools: Read, Glob, Grep, compatibility line.
- Every factual claim cites a primary-source URL with a quoted fragment.
- WRONG/RIGHT code blocks. A "who suffers" human-experience section. Cross-routing to related skills.
- references/: common-mistakes.md + sources.yaml (+ domain-specific tables).

## Phase status
- [x] Phase 1 research -> reports/skill-research-sql.md (510 lines). Foundation = sql-relational-and-null-discipline.
- [ ] Phase 2 plan -> reports/skill-plan-sql.md  (in progress)
- [ ] Phase 3 research+draft per skill (foundation first, then batches of 2-3)
- [ ] Phase 4 audit + coverage dimensions

## Key decisions
- Foundation/policy-root skill: sql-relational-and-null-discipline (the go-idiomatic-discipline analogue).
- Second cross-cutting anchor: sql-standard-vs-dialect-map (the go-version-feature-map analogue).
- mvcc boundary: sql-transactions-and-isolation owns SQL surface only; routes anomaly theory to mvcc plugin.
- Portability block required in every skill: "standard says X; Postgres ✓, SQLite ~, MySQL ✗ (spells Y)".

## STATUS: Phase 3 starting. Plan = reports/skill-plan-sql.md (30 skills, specs in §2, build order §3).
Decisions taken (no gate): 30 skills, output plugins/sql/skills/, all 3 advanced skills full files,
row-values folded into pagination, gaps-and-islands standalone, stored procs out.
Build order: foundation solo -> Wave1 batches 1-5 -> Wave2 batches 6-11 (dialect-map=batch10 after others).
NEXT: foundation researcher -> draft -> use as reference impl for all later drafters.

## PROGRESS LOG (Phase 3)
Pattern: combined research+draft agents in batches of 3; agents write files only (NO git); I commit per batch.
Several modern-sql.com/feature/* URLs are 404 now — agents substituted live equivalents and noted it. Fine.
API "Overloaded" hit 2 agents (lateral, datetime) — both had written research report to disk; I patched
the missing pieces with a small follow-up agent. Resilience design (research-to-disk-first) works.

Wave 1 COMPLETE & committed (16 skills): foundation + batches 1-5.
22 skills committed through Batch 7. Now dispatching Batches 8+9 (transactions, privileges, indexing, explain).
schema-design came in at 300 lines (8 WRONG/RIGHT pairs) — accepted, comprehensive. All else within range.
Wave 2 progress:
- [x] Batch 6: generated-and-identity-columns(187), views-and-introspection(186), json(255) — committing now
- [ ] Batch 7: schema-design-and-normalization(#21), style-and-naming(#22), gaps-and-islands(#20)
- [ ] Batch 8: transactions-and-isolation(#23, mvcc boundary), privileges-and-access-control(#24)
- [ ] Batch 9: indexing-and-sargability(#25), explain-and-set-based-thinking(#26)
- [ ] Batch 10: standard-vs-dialect-map(#27)  <- draft LAST so it indexes settled spellings
- [ ] Batch 11: temporal-tables(#28), match-recognize(#29), property-graph-queries(#30)
Then: plugin.json + marketplace.json (specs in plan §near end), Phase 4 audit, coverage-dimensions check.

## COMPLETE. 32 skills (30 planned + 2 gap-closers from audit W4: data-modification, pattern-matching-and-collation).
Phase 4 audit: PASS, no must-fix errors. Warnings W1/W2 resolved (modern-sql/filter live; all 6 mvcc cross-link
targets verified to exist). W4 (DML + pattern-matching gap) closed with the 2 new skills.
Metadata committed. All 32 structurally valid; cross-link integrity confirmed by audit.
Audit report: reports/skill-audit-sql.md. Coverage-dimensions assessment delivered in final report.
Testing Reality + Process Integration dimensions deliberately partial — deep gaps there (migration tooling,
online DDL, ORM integration, pgTAP/dbt) are NON-STANDARD/app-layer, correctly deferred to future vendor plugins
per the "standard SQL that works anywhere" constraint.

## Subagent preamble (MANDATORY on every dispatch)
"IMPORTANT: You are a subagent. Execute immediately. Do NOT restate the task."
