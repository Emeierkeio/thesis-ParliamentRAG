---
phase: 02-backend
plan: 05
subsystem: api
tags: [fastapi, neo4j, di, scripts, endpoints]

# Dependency graph
requires:
  - phase: 02-backend
    provides: "get_neo4j_client DI function in app.services.deps"
  - phase: 02-backend
    provides: "Vote nodes and DISCUSSES relationships from Phase 1 db_builder"
provides:
  - "GET /api/data/sessions/{id}/votes endpoint"
  - "GET /api/data/debates/{id}/acts endpoint"
  - "Scripts use shared Neo4j client (no own driver creation)"
  - "Source-inspection tests for new endpoints and script DI compliance"
affects: [03-frontend, evaluation, data-access]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "sys.modules stub pattern for FastAPI TestClient tests in scipy/NumPy 2.x broken anaconda environment"
    - "Scripts call get_neo4j_client() from app.services.deps instead of Neo4jClient(settings...)"

key-files:
  created:
    - backend/app/routers/data.py
    - backend/tests/unit/test_new_endpoints.py
    - backend/tests/unit/test_scripts.py
  modified:
    - backend/app/main.py
    - backend/scripts/compute_baseline_experts.py
    - backend/scripts/enrich_evaluation_set.py

key-decisions:
  - "TestClient functional tests use sys.modules stubs (pre-registered before importlib load) to bypass scipy/NumPy 2.x import chain — consistent with existing project pattern for this environment"
  - "data.py router registered directly in main.py (not via routers/__init__.py) to keep __init__.py unchanged"

patterns-established:
  - "Source-inspection tests (test_scripts.py, test_new_endpoints.py structural section) for contracts that cannot use live imports in this environment"
  - "sys.modules stub injection pattern for minimal FastAPI TestClient apps avoiding the full compass/scipy import chain"

requirements-completed: [API-04, API-05, SCR-01, SCR-03]

# Metrics
duration: 5min
completed: 2026-04-02
---

# Phase 2 Plan 05: Data Endpoints and Script DI Refactor Summary

**New FastAPI endpoints expose Phase 1 Vote nodes and DISCUSSES relationships; all 3 backend scripts migrated from own Neo4jClient instantiation to shared get_neo4j_client() from deps.py**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-02T16:20:00Z
- **Completed:** 2026-04-02T16:25:04Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Created `backend/app/routers/data.py` with GET `/api/data/sessions/{id}/votes` and GET `/api/data/debates/{id}/acts`, both using `Depends(get_neo4j_client)` for DI
- Registered `data_router` in `backend/app/main.py` alongside existing routers
- Refactored `compute_baseline_experts.py` and `enrich_evaluation_set.py` to call `get_neo4j_client()` instead of `Neo4jClient(settings.neo4j_uri, ...)` — zero own-driver creation in any script
- Confirmed `seed_evaluation_topic.py` already uses `get_services()` from deps (no change needed)
- Verified zero `start_char_raw` / `end_char_raw` dead properties in all scripts
- Added 14 tests in `test_new_endpoints.py` (9 structural + 4 functional) and 8 tests in `test_scripts.py`

## Task Commits

Each task was committed atomically:

1. **Task 1: Create new data endpoints for votes and debate-act links** - `b2a1fcf` (feat)
2. **Task 2: Refactor scripts to use shared Neo4j client + clean up** - `6c3deb5` (feat)

**Plan metadata:** TBD (docs commit below)

## Files Created/Modified
- `backend/app/routers/data.py` - New router: GET /api/data/sessions/{id}/votes, GET /api/data/debates/{id}/acts
- `backend/app/main.py` - Added data_router import and include_router
- `backend/scripts/compute_baseline_experts.py` - Replaced own Neo4jClient with get_neo4j_client() from deps
- `backend/scripts/enrich_evaluation_set.py` - Replaced own Neo4jClient with get_neo4j_client() from deps
- `backend/tests/unit/test_new_endpoints.py` - 14 tests: structural source-inspection + functional with sys.modules stubs
- `backend/tests/unit/test_scripts.py` - 8 tests: no own driver, uses deps, no dead properties

## Decisions Made
- Used `sys.modules` stub injection to pre-register fake `app.services.deps` and `app.services.neo4j_client` modules before loading `app.routers.data` via `importlib.util`, avoiding the compass/scipy import chain. This is consistent with the project's established "source-file inspection instead of live import" pattern for this NumPy 2.x broken anaconda environment.
- Registered `data_router` directly in `main.py` (not via `routers/__init__.py`) to leave the existing package init unchanged.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] TestClient tests needed sys.modules stubs to bypass scipy/NumPy import chain**
- **Found during:** Task 1 (test_new_endpoints.py creation)
- **Issue:** Any import of `app.routers.data` triggers `app.routers.__init__` → `query.py` → `deps.py` → `compass.py` → `scipy` — which fails in this environment with `ImportError: numpy.core.multiarray failed to import`
- **Fix:** Added `_build_test_app_with_stubs()` helper that pre-registers stub modules for `app.services.deps` and `app.services.neo4j_client` in `sys.modules` before loading the data router via `importlib.util.spec_from_file_location`. Structural tests remain pure source-inspection.
- **Files modified:** `backend/tests/unit/test_new_endpoints.py`
- **Verification:** 14 tests pass
- **Committed in:** `b2a1fcf` (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (Rule 1 - environment compatibility)
**Impact on plan:** Required test adaptation for the known NumPy 2.x environment constraint. No scope creep. Production code unchanged.

## Issues Encountered
- Pre-existing `test_deps.py` failure (scipy/NumPy 2.x incompatibility) — out of scope per STATE.md "Pydantic model tests use source-file inspection (not live import) to avoid scipy/numpy NumPy 2.x incompatibility" note. Not introduced by this plan.

## Next Phase Readiness
- All new data endpoints (`/api/data/sessions/{id}/votes`, `/api/data/debates/{id}/acts`) are ready for frontend consumption
- All scripts (`compute_baseline_experts.py`, `enrich_evaluation_set.py`, `seed_evaluation_topic.py`) use shared Neo4j client — no connection doubling
- Phase 2 backend plans complete

---
*Phase: 02-backend*
*Completed: 2026-04-02*
