---
phase: 02-backend
plan: 03
subsystem: api
tags: [experts, refactoring, dependency-injection, import-hygiene, tdd]

# Dependency graph
requires:
  - phase: 02-backend
    plan: 02
    provides: "deps.py DI layer with get_neo4j_client singleton"
  - phase: 02-backend
    plan: 01
    provides: "evidence.py schema and UnifiedEvidence model"
provides:
  - "Unified expert computation in services/experts.py (combined formula 0.70*authority+0.30*similarity)"
  - "patch_experts_for_cited_speakers() public function"
  - "survey_helpers.py shared service breaking evaluation→survey cross-router coupling"
  - "Zero cross-router import violations (API-02, SCR-02)"
affects: [02-04, 03-backend, evaluation, survey, seed-scripts]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Service layer owns all cross-cutting business logic; routers are thin delegates"
    - "Shared helpers extracted to services/ to break cross-router imports"
    - "TDD for service extraction: tests written first, then implementation"

key-files:
  created:
    - "backend/app/services/experts.py - Unified compute_experts() + patch_experts_for_cited_speakers()"
    - "backend/app/services/survey_helpers.py - load_surveys, calculate_stats, load_evaluation_set_raw"
    - "backend/tests/unit/test_experts.py - 10 tests for expert computation service"
    - "backend/tests/unit/test_import_violations.py - 6 import boundary enforcement tests"
  modified:
    - "backend/app/routers/chat.py - Removed 3 local expert functions (~290 lines); delegating to service"
    - "backend/app/routers/query.py - Removed _compute_experts, _fetch_speaker_details, inline patch (~200 lines); delegating to service"
    - "backend/app/routers/evaluation.py - Fixed API-02: all survey imports now from services.survey_helpers"
    - "backend/scripts/seed_evaluation_topic.py - Fixed SCR-02: imports compute_experts from service"

key-decisions:
  - "Combined ranking formula 0.70*authority + 0.30*similarity is the single canonical implementation (from chat.py)"
  - "query.py's authority_only formula preserved via ranking_formula='authority_only' param"
  - "survey_helpers.py uses public names (load_surveys, calculate_stats) with private aliases for backward compat"
  - "asyncio.run() used in tests (not deprecated asyncio.get_event_loop().run_until_complete)"

patterns-established:
  - "Extract-then-delegate: extract function verbatim to service, replace local with import+call"
  - "Cross-layer imports broken by creating services/ module that both sides import from"

requirements-completed: [SVC-02, API-02, SCR-02]

# Metrics
duration: 25min
completed: 2026-04-02
---

# Phase 02 Plan 03: Expert Service Extraction Summary

**Expert computation unified in services/experts.py (combined formula 0.70*auth+0.30*sim), chat.py and query.py delegate to the service, and all cross-router import violations (API-02, SCR-02) fixed via services/survey_helpers.py**

## Performance

- **Duration:** ~25 min
- **Started:** 2026-04-02T16:00:00Z
- **Completed:** 2026-04-02T16:25:00Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments
- Created `services/experts.py` as the single source of truth for expert computation with the canonical `0.70 * authority + 0.30 * similarity` formula, `party_changed` handling, and `patch_experts_for_cited_speakers()`
- Removed ~490 lines of duplicated expert code from chat.py (3 functions) and query.py (2 functions + 100-line inline patch block)
- Fixed all 3 cross-layer import violations: evaluation.py no longer imports from survey.py, seed script no longer imports from chat.py
- 16 new unit tests passing (10 expert service, 6 import boundary)

## Task Commits

1. **Task 1: Create unified experts.py service with tests** - `3a86840` (feat)
2. **Task 2: Wire routers and scripts to use experts service, fix all import violations** - `9ab23fd` (feat)

**Plan metadata:** (pending docs commit)

## Files Created/Modified
- `backend/app/services/experts.py` - Unified compute_experts() + patch_experts_for_cited_speakers() + _fetch_speaker_details()
- `backend/app/services/survey_helpers.py` - Shared helpers extracted from survey.py for evaluation.py to import
- `backend/tests/unit/test_experts.py` - 10 TDD tests: one-per-party, combined formula, authority_only, party_changed, frozen fields, GovernmentMember exclusion, patch mismatch/no-mismatch
- `backend/tests/unit/test_import_violations.py` - 6 enforcement tests for layer boundaries
- `backend/app/routers/chat.py` - Import + 3 delegate calls replacing local definitions
- `backend/app/routers/query.py` - Import + 3 delegate calls replacing local definitions
- `backend/app/routers/evaluation.py` - 3 inline imports replaced with services.survey_helpers
- `backend/scripts/seed_evaluation_topic.py` - Service import replacing router import

## Decisions Made
- Used `asyncio.run()` in tests (not deprecated `get_event_loop().run_until_complete()`)
- The `_canned_speaker_details` test helper takes `(neo4j_client, speaker_id)` - matching the actual `_fetch_speaker_details(neo4j_client, speaker_id)` signature
- `survey_helpers.py` exposes public names (`load_surveys`, `calculate_stats`) plus private aliases (`_load_surveys`, `_calculate_stats`) for backward compat with any remaining direct references
- evaluation.py had 3 cross-router imports total (not just the module-level one): fixed all via inline replacement

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed test mock signature mismatch**
- **Found during:** Task 1 (TDD RED→GREEN transition)
- **Issue:** `_canned_speaker_details(speaker_id)` test helper had 1 arg but `_fetch_speaker_details` is called as `(neo4j_client, speaker_id)` - `side_effect` would fail with "takes 1 positional argument but 2 were given"
- **Fix:** Updated helper signature to `_canned_speaker_details(neo4j_client, speaker_id)`; also replaced deprecated `asyncio.get_event_loop().run_until_complete()` with `asyncio.run()`
- **Files modified:** `backend/tests/unit/test_experts.py`
- **Verification:** 10/10 tests pass
- **Committed in:** `3a86840` (Task 1 commit)

**2. [Rule 1 - Bug] Found 3 inline cross-router imports in evaluation.py (not just 1)**
- **Found during:** Task 2 (acceptance criteria verification)
- **Issue:** Plan mentioned one module-level `from app.routers.survey import _load_surveys, _calculate_stats`; evaluation.py also had 2 additional inline imports inside functions: `_load_evaluation_set_raw` and `_get_ab_assignment, _deblind_preference`
- **Fix:** Added `load_evaluation_set_raw`, `_get_ab_assignment`, `_deblind_preference` to `survey_helpers.py`; replaced all 3 inline imports in evaluation.py
- **Files modified:** `backend/app/services/survey_helpers.py`, `backend/app/routers/evaluation.py`
- **Verification:** `grep "from app.routers" evaluation.py` returns zero matches; 6/6 violation tests pass
- **Committed in:** `9ab23fd` (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (2x Rule 1 - bugs caught during testing)
**Impact on plan:** Both fixes necessary for correctness. No scope creep.

## Issues Encountered
- The `test_deps.py` suite has a pre-existing ERROR (scipy/NumPy 2.x incompatibility in `TestGetNeo4jClientSingleton`) that was already present before this plan. Not caused by these changes. Our 16 new tests all pass cleanly.

## Next Phase Readiness
- Expert computation is now centralized and testable; Plan 04 (evaluation service) can import from services/experts.py
- Import violation enforcement tests will catch future layer breaches automatically
- `survey_helpers.py` is ready to be extended in Plan 04 when evaluation service is built

---
*Phase: 02-backend*
*Completed: 2026-04-02*
