---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: 01
subsystem: api
tags: [neo4j, fastapi, rice-index, vote-analytics, cohesion]

# Dependency graph
requires:
  - phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages
    provides: IndividualVote nodes, VOTED/ON_VOTE/HAS_VOTE graph edges
  - phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
    provides: chamber+legislature filter pattern enforced on all Session Cypher queries
provides:
  - votes_service.py with 8 functions: rice_index, mean_rice, has_individual_votes,
    get_party_cohesion, get_deputy_vote_stats, search_votes, get_vote_facts, get_vote_coherence
  - test_vote_services.py with 8 passing unit tests (no DB, no scipy)
affects:
  - 14-02 (compass vote pipeline — imports from votes_service)
  - 14-03 (votes router — imports search_votes, get_party_cohesion, get_deputy_vote_stats)
  - 14-04 (DirectWriter F4 injection — imports get_vote_facts)
  - 14-05 (F1 coherence SSE — imports get_vote_coherence)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "votes_service.py as single source of truth — all downstream plans import from here"
    - "availability guard pattern: has_individual_votes() check before Rice computation (returns {available: False} instead of zeros)"
    - "coalesce(a.title, d.title, v.subject) label hierarchy to avoid generic 'Votazione' labels"
    - "Phase 12 rule enforced: every Cypher filters s.chamber + s.legislature"
    - "MEMBER_OF_GROUP date-scoped at vote date: mg.start_date <= s.date AND (mg.end_date IS NULL OR mg.end_date >= s.date)"

key-files:
  created:
    - backend/app/services/votes_service.py
    - backend/tests/unit/test_vote_services.py
  modified: []

key-decisions:
  - "14-01: All 8 vote analytics functions written in one module (votes_service.py) for single import point in all downstream plans"
  - "14-01: get_party_cohesion returns {available: False, reason: individual_votes_pending} — never zeros — when IndividualVote data absent (Pitfall 3)"
  - "14-01: get_vote_coherence groups results by session_id and filters null party entries from party_breakdown"
  - "14-01: search_votes uses $min_margin IS NULL OR margin >= $min_margin pattern to allow optional server-side margin filter"

patterns-established:
  - "availability guard pattern — check has_individual_votes() before Rice queries; return {available: False} not zero"
  - "label coalesce hierarchy — coalesce(a.title, d.title, v.subject) on every vote Cypher"

requirements-completed: [VI-01, VI-03, VI-04, VI-05]

# Metrics
duration: 3min
completed: 2026-07-06
---

# Phase 14 Plan 01: Vote Analytics Service Summary

**Rice index math + 8 graph analytics functions in a single votes_service.py module, with full degradation guard when IndividualVote data is absent**

## Performance

- **Duration:** 3 min
- **Started:** 2026-07-06T23:40:15Z
- **Completed:** 2026-07-06T23:43:25Z
- **Tasks:** 3 (all complete)
- **Files modified:** 2 (created)

## Accomplishments
- Implemented `rice_index` and `mean_rice` pure Python functions (no scipy, no numpy) with TDD coverage
- Added F3 functions: `has_individual_votes` availability guard, `get_party_cohesion`, `get_deputy_vote_stats`
- Added F5/F4/F1 functions: `search_votes`, `get_vote_facts`, `get_vote_coherence`
- All 8 unit tests pass (pure math + empty-input guards) with no DB or scipy dependency

## Task Commits

Each task was committed atomically:

1. **Task 1 (TDD RED): Wave-0 test scaffold** - `500454f` (test)
2. **Tasks 1/2/3 (TDD GREEN + impl): All 8 vote analytics functions** - `bb382a1` (feat)

_Note: Tasks 2 and 3 were implemented in the same module as Task 1 (all in votes_service.py), so GREEN commit covers all three tasks._

**Plan metadata:** See final docs commit below.

## Files Created/Modified
- `backend/app/services/votes_service.py` — 438 lines; Rice math, party cohesion, deputy stats, search, facts, coherence
- `backend/tests/unit/test_vote_services.py` — 80 lines; 8 unit tests, all pure (no DB)

## Decisions Made
- All 3 tasks implemented in votes_service.py in a single pass — the plan's module structure made this natural; all functions share Cypher patterns and the same neo4j injection signature
- `get_party_cohesion` calls `has_individual_votes` first and short-circuits to `{"available": False, "reason": "individual_votes_pending"}` instead of returning zeros — follows degradation rule from research
- `get_vote_coherence` groups Cypher rows by `session_id` in Python after fetching — cleaner than a complex Cypher aggregation
- `search_votes` uses `$min_margin IS NULL OR margin >= $min_margin` in the Cypher WITH+WHERE block to allow optional margin filtering server-side (Pitfall 4)

## Deviations from Plan

None - plan executed exactly as written. All 8 functions implemented per research Cypher specs. All 8 tests pass. Acceptance criteria verified programmatically.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- `votes_service.py` exports all 8 functions downstream plans depend on
- Plan 14-02 (compass vote pipeline) can import `search_votes` and the Rice math
- Plan 14-03 (votes router) can import `search_votes`, `get_party_cohesion`, `get_deputy_vote_stats`
- Plan 14-04 (DirectWriter F4 injection) can import `get_vote_facts`
- Plan 14-05 (F1 coherence SSE) can import `get_vote_coherence`

---
*Phase: 14-vote-intelligence*
*Completed: 2026-07-06*
