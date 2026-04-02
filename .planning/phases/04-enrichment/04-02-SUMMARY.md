---
phase: 04-enrichment
plan: "02"
subsystem: database
tags: [sparql, neo4j, python, dati.camera.it, vote-ingestion, committee-enrichment]

requires:
  - phase: 01-build-pipeline
    provides: "Deputy, Session, Vote, Committee nodes with MEMBER_OF_COMMITTEE relationships"

provides:
  - "sparql_ingester.py with SparqlIngester class for vote and committee role enrichment"
  - "IndividualVote nodes linked via VOTED and ON_VOTE relationships"
  - "MEMBER_OF_COMMITTEE.officerRole enrichment from SPARQL endpoint"
  - "make enrich-sparql and make enrich-sparql-test Makefile targets"

affects:
  - 04-enrichment
  - authority-scoring

tech-stack:
  added: ["urllib (stdlib SPARQL HTTP), no new external deps"]
  patterns:
    - "SPARQL pagination with LIMIT/OFFSET loop until page < PAGE_SIZE"
    - "URI regex parsing for deputato.rdf -> persona.rdf ID conversion"
    - "Module-level _sparql_get helper swallowed all exceptions, returns []"
    - "Batch preparation helpers (_prepare_vote_batch, _prepare_committee_role_batch) are pure and testable without DB"

key-files:
  created:
    - build/sparql_ingester.py
    - build/tests/test_sparql_ingester.py
  modified:
    - Makefile

key-decisions:
  - "Used stdlib urllib instead of requests to avoid new dependencies"
  - "sparql_dep_uri_to_neo4j_id uses _19 legislature suffix as match anchor — returns None for any non-19th-legislature URI"
  - "IndividualVote.id format: iv_{person_id}_{session_num}_{vote_num} — deterministic for MERGE idempotency"
  - "Vote batch preparation skips rows with unparseable votazione URI rather than crashing"
  - "Committee role matching uses toLower CONTAINS on c.name — fuzzy match handles label variations"
  - "BATCH_SIZE=500 for Neo4j writes, SPARQL_PAGE_SIZE=1000 for HTTP pagination"

requirements-completed: [ENR-01, ENR-02]

duration: 2min
completed: 2026-04-02
---

# Phase 4 Plan 02: SPARQL Ingester Summary

**SPARQL ingester fetching per-deputy vote records and committee officer roles from dati.camera.it, writing IndividualVote nodes with VOTED/ON_VOTE edges and enriching MEMBER_OF_COMMITTEE with officerRole using stdlib urllib and LIMIT/OFFSET pagination.**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-02T21:51:06Z
- **Completed:** 2026-04-02T21:53:47Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments

- SparqlIngester class with ingest_votes (paginated SPARQL, IndividualVote MERGE) and ingest_committee_roles (officerRole SET on existing relationships)
- 33 unit tests covering URI parsing, outcome mapping, batch preparation, pagination, and timeout handling — all passing with mocked HTTP
- make enrich-sparql and make enrich-sparql-test Makefile targets with Neo4j readiness wait loop

## Task Commits

Each task was committed atomically:

1. **RED — Failing tests for SparqlIngester** - `b2a2c92` (test)
2. **GREEN — sparql_ingester.py implementation** - `257db0b` (feat)
3. **Task 2 — Makefile enrich-sparql targets** - `bb4cac5` (feat)

## Files Created/Modified

- `build/sparql_ingester.py` - SparqlIngester class, URI helpers, _sparql_get, CLI entry point
- `build/tests/test_sparql_ingester.py` - 33 unit tests with mocked HTTP, all green
- `Makefile` - enrich-sparql and enrich-sparql-test targets added after Database Pipeline section

## Decisions Made

- stdlib urllib used to avoid adding requests/httpx as build dependency — consistent with zero-extra-deps philosophy in the build directory
- IndividualVote.id = `iv_{person_id}_{session}_{vote}` makes MERGE idempotent across re-runs
- SPARQL vote batch preparation skips rows with unparseable votazione URI silently (counted as skipped in stats) — avoids crashing entire deputy on one bad row
- Committee matching uses `toLower(c.name) CONTAINS toLower(row.committeeName)` — SPARQL labels can differ slightly from Neo4j stored names

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required. dati.camera.it SPARQL endpoint is public (no API key needed).

## Next Phase Readiness

- IndividualVote nodes ready for ActsComponent scoring in authority pipeline
- Committee officerRole available for CommitteeComponent weighting
- Run `make enrich-sparql-test` with 5 deputies to smoke-test before full run

---
*Phase: 04-enrichment*
*Completed: 2026-04-02*

## Self-Check: PASSED

Files verified:
- build/sparql_ingester.py: FOUND
- build/tests/test_sparql_ingester.py: FOUND
- Makefile (enrich-sparql target): FOUND

Commits verified:
- b2a2c92 (RED tests): FOUND
- 257db0b (feat implementation): FOUND
- bb4cac5 (Makefile targets): FOUND
