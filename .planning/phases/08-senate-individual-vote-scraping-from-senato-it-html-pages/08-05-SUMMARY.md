---
phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages
plan: "05"
subsystem: infra
tags: [makefile, sparql, senate, camera, vote-enrichment]

# Dependency graph
requires:
  - phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages plan 02
    provides: sparql_ingester.py with --aggregate-only/--skip-aggregate/--legislature/--start-session/--limit-sessions CLI
  - phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages plan 04
    provides: senate_sparql_ingester.py with --aggregate-only/--individual-only/--legislature/--limit-sessions CLI
provides:
  - enrich-votes Makefile target (aggregate ingest both chambers XIX+XVIII)
  - enrich-votes-individual Makefile target (per-parliamentarian long run both chambers)
  - enrich-votes-test Makefile target (smoke 2 Camera sessions + 2 Senate sittings)
affects: [08-06, 08-07, plans that gate long vote runs]

# Tech tracking
tech-stack:
  added: []
  patterns: [enrich-sparql neo4j-wait boilerplate reused verbatim; smoke target skips wait (assumes running)]

key-files:
  created: []
  modified: [Makefile]

key-decisions:
  - "enrich-votes-test skips docker-compose/neo4j-wait — assumes Neo4j already running for quick smoke validation"
  - "--limit-sessions flag confirmed on sparql_ingester.py (Plan 02) — reused for Camera aggregate smoke path"
  - "enrich-votes-individual uses --skip-committees on Camera path to avoid re-running committee role ingest"

patterns-established:
  - "Per-chamber per-legislature invocations chained sequentially within a single target"

requirements-completed: [VOT-07]

# Metrics
duration: 3min
completed: 2026-07-05
---

# Phase 08 Plan 05: Makefile Vote Enrichment Targets Summary

**Three Makefile targets that drive both chamber ingesters across both legislatures: aggregate fast-ish run, long per-parliamentarian run, and a 2-session smoke test**

## Performance

- **Duration:** 3 min
- **Started:** 2026-07-05T00:41:03Z
- **Completed:** 2026-07-05T00:44:00Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments
- Added `enrich-votes` target: starts Neo4j, runs Camera aggregate (XIX from 350, XVIII from 1) then Senate aggregate (XIX, XVIII)
- Added `enrich-votes-individual` target: same Neo4j-wait boilerplate, then Camera --skip-aggregate --skip-committees (XIX) and Senate --individual-only (XIX+XVIII)
- Added `enrich-votes-test` target: no wait (assumes running), 2-session cap on Camera aggregate and Senate aggregate for rapid smoke validation

## Task Commits

1. **Task 1: Add enrich-votes, enrich-votes-individual, enrich-votes-test targets** - `445af90` (feat)

**Plan metadata:** (included in final commit)

## Files Created/Modified
- `Makefile` - 49-line block inserted after enrich-sparql-test: three new targets + section header + .PHONY declaration

## Decisions Made
- `enrich-votes-test` deliberately omits docker compose up and neo4j-wait; the smoke target is meant to validate flags quickly against a DB that is already running (matches plan spec)
- `--skip-committees` added on Camera individual path to avoid redundantly re-running committee role ingest already done by `enrich-sparql`
- `--limit-sessions` confirmed valid flag name in `sparql_ingester.py` (line 747) — plan's conditional note resolved in favor of using it

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Plans 06 and 07 can now invoke `make enrich-votes` (aggregate gate) and `make enrich-votes-individual` (long run gate)
- `make enrich-votes-test` provides a fast smoke path before committing to multi-hour runs

---
*Phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages*
*Completed: 2026-07-05*
