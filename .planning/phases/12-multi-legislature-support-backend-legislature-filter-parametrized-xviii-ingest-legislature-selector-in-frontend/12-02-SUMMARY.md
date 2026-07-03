---
phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
plan: 02
subsystem: api
tags: [neo4j, fastapi, timeline, legislature, cypher]

# Dependency graph
requires:
  - phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
    provides: get_sessions Cypher and timeline router that this plan extends with legislature filter
provides:
  - "GET /api/timeline accepts legislature: int = 19 query param and filters Session nodes by s.legislature"
  - "get_sessions service signature extended with legislature: int = 19"
  - "Source-inspection tests (RED then GREEN) for the legislature filter"
affects:
  - 12-multi-legislature-support (future plans: XVIII ingest, frontend selector)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "legislature param threaded from FastAPI query param → service kwarg → Cypher params dict → WHERE clause"
    - "Debate-detail and speaker-summary endpoints left unfiltered (keyed by unique leg18_/leg19_ prefixed ids)"

key-files:
  created:
    - backend/tests/test_timeline_legislature.py
  modified:
    - backend/app/routers/timeline.py
    - backend/app/services/timeline_service.py

key-decisions:
  - "[Phase 12-02]: legislature: int = 19 placed in router signature after chamber param — aligns with chamber ordering and leaves debate-detail/speaker-summary endpoints unchanged (id-scoped)"
  - "[Phase 12-02]: AND s.legislature = $legislature added as first condition after chamber filter in Cypher WHERE — most selective filter first"

patterns-established:
  - "Source-inspection test written RED before implementation, then implementation makes it GREEN"

requirements-completed: [LEG-02]

# Metrics
duration: 1min
completed: 2026-07-03
---

# Phase 12 Plan 02: Timeline Legislature Filter Summary

**`legislature: int = 19` query param added to GET /api/timeline, forwarded to get_sessions Cypher WHERE `AND s.legislature = $legislature`, keeping timeline scoped to current legislature by default**

## Performance

- **Duration:** 1 min
- **Started:** 2026-07-03T16:07:32Z
- **Completed:** 2026-07-03T16:08:20Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Created source-inspection test file (RED) asserting `legislature: int = 19` in router and service, `s.legislature = $legislature` in Cypher, and `"legislature": legislature` in params dict
- Added `legislature: int = 19` query param to `get_timeline` router function; forwarded as `legislature=legislature` kwarg to `get_sessions`
- Added `legislature: int = 19` to `get_sessions` signature, `AND s.legislature = $legislature` to Cypher WHERE, and `"legislature": legislature` to params dict
- Debate-detail and speaker-summary endpoints left untouched (already scoped by unique `leg18_`/`leg19_` prefixed IDs)
- All 20 timeline tests pass GREEN; 7 pre-existing failures unrelated to this plan remain unchanged

## Task Commits

Each task was committed atomically:

1. **Task 1: Wave 0 — failing source-inspection test for timeline legislature filter** - `aebab87` (test)
2. **Task 2: Add legislature param to timeline router and service Cypher** - `7fd73f2` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified
- `backend/tests/test_timeline_legislature.py` — Source-inspection tests: router has `legislature: int = 19`, service has `s.legislature = $legislature` in Cypher and `"legislature": legislature` in params
- `backend/app/routers/timeline.py` — Added `legislature: int = 19` query param after `chamber`, forwarded to `get_sessions`
- `backend/app/services/timeline_service.py` — Added `legislature: int = 19` to signature, `AND s.legislature = $legislature` to WHERE, `"legislature": legislature` to params dict

## Decisions Made
- `legislature: int = 19` placed after `chamber` in both router and service signatures — mirrors existing param ordering and makes the default legislature explicit in the function signature
- `AND s.legislature = $legislature` inserted as the first AND condition after the initial `WHERE ($chambers IS NULL OR ...)` — evaluated before the date and search conditions, which is more selective
- Debate-detail and speaker-summary endpoints intentionally left without a legislature filter — they are keyed by unique debate IDs that embed the legislature (`leg18_...` vs `leg19_...`), per RESEARCH Pitfall 4

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- Full test suite showed 7 pre-existing failures: 3 anticipate future Phase 12 plans (query router, retrieval channels, engine legislature threading); 4 are pre-existing bugs in experts, SSE contract, and translation service. None caused by this plan's changes.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Timeline endpoint now exposes `?legislature=19` (or any int) — ready for Plan 12-03 (XVIII ingest) to populate `legislature=18` Session nodes and Plan 12-04/05 (frontend selector) to pass the param
- Pre-existing test failures in experts, SSE contract, and translation should be addressed in a dedicated maintenance plan

---
*Phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend*
*Completed: 2026-07-03*
