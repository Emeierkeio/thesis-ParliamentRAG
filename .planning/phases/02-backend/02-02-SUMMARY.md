---
phase: 02-backend
plan: 02
subsystem: api
tags: [fastapi, di, lru_cache, neo4j, dependency-injection]

# Dependency graph
requires:
  - phase: 02-backend
    provides: Phase 2 context — existing deps.py global pattern to replace
provides:
  - Typed @lru_cache FastAPI Depends() functions for all 5 services
  - Backward-compatible get_services() dict wrapper
  - search.py and evidence.py using shared Neo4j client (no duplicate pools)
  - DI singleton and absence-of-global tests in tests/unit/test_deps.py
affects: [02-backend, any plan that adds routers or tests DI layer]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "@lru_cache() on service factory functions gives FastAPI Depends() singleton semantics without a DI container"
    - "lru_cache.cache_clear() in test setup/teardown prevents cross-test singleton pollution"
    - "Regex ^_neo4j_client\\s*[=:] to assert absence of module-level client globals in router source"

key-files:
  created:
    - backend/tests/unit/test_deps.py
  modified:
    - backend/app/services/deps.py
    - backend/app/routers/search.py
    - backend/app/routers/evidence.py
    - backend/app/main.py
    - backend/tests/conftest.py

key-decisions:
  - "lru_cache provides singleton semantics without a DI container — no extra library needed"
  - "Kept backward-compatible get_services() dict wrapper so unmigrated routers continue to work"
  - "Lifespan shutdown hook calls get_neo4j_client().close() via the shared singleton — no separate client needed"
  - "conftest.py client fixture patches get_neo4j_client (the lru_cache function) not the removed _neo4j_client global"

patterns-established:
  - "All service singletons obtained via typed @lru_cache factory functions in deps.py"
  - "Router endpoints receive services as Depends(get_*) parameters, never call module-level getters directly"

requirements-completed: [SVC-04, SVC-05]

# Metrics
duration: 5min
completed: 2026-04-02
---

# Phase 02 Plan 02: DI Layer Rewrite Summary

**Replaced untyped global-variable singleton pattern with 5 typed @lru_cache FastAPI Depends() functions; eliminated duplicate Neo4j connection pools in search.py and evidence.py**

## Performance

- **Duration:** 5 min
- **Started:** 2026-04-02T15:49:39Z
- **Completed:** 2026-04-02T15:54:00Z
- **Tasks:** 2
- **Files modified:** 5 (+ 1 created)

## Accomplishments

- deps.py rewritten with `get_neo4j_client`, `get_retrieval_engine`, `get_authority_scorer`, `get_ideology_scorer`, `get_generation_pipeline` — all typed and @lru_cache-backed
- search.py and evidence.py no longer maintain their own `_neo4j_client` globals; all endpoints receive the shared client via `Depends(get_neo4j_client)`
- main.py lifespan shutdown now closes the shared Neo4j client via the singleton getter
- 7 passing unit tests in test_deps.py verify singleton behavior, backward compat, and absence of duplicate pools

## Task Commits

Each task was committed atomically:

1. **Task 1: Rewrite deps.py with typed Depends functions** - `b657bcb` (feat)
2. **Task 2: Fix search.py and evidence.py duplicate Neo4j pools + write DI tests** - `4f8b8d7` (feat)

**Plan metadata:** (to be added below)

## Files Created/Modified

- `backend/app/services/deps.py` - Rewritten: 5 @lru_cache factory functions + backward-compat get_services()
- `backend/app/routers/search.py` - Removed get_client() and _neo4j_client global; all 5 endpoints use Depends(get_neo4j_client)
- `backend/app/routers/evidence.py` - Removed get_neo4j() and _neo4j_client global; both endpoints use Depends(get_neo4j_client)
- `backend/app/main.py` - Import get_neo4j_client; lifespan shutdown calls .close()
- `backend/tests/conftest.py` - Updated client fixture to patch get_neo4j_client instead of removed _neo4j_client global
- `backend/tests/unit/test_deps.py` - Created: 7 DI tests (singleton, backward compat, no local globals)

## Decisions Made

- Used `@lru_cache()` on service factory functions — gives singleton semantics with no extra library needed; works with FastAPI's Depends because lru_cache caches the return value on the first call.
- Kept `get_services()` dict wrapper intact to avoid breaking unmigrated routers in a single PR.
- `_ensure_act_vector_index()` in search.py (a non-endpoint helper) calls `get_neo4j_client()` directly — correct since it does not run in a request context.
- conftest.py patched at the function level (`patch("app.services.deps.get_neo4j_client", return_value=...)`) rather than the module attribute, because lru_cache wraps the function and there is no raw `_neo4j_client` global anymore.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed conftest.py to patch new lru_cache function instead of removed global**
- **Found during:** Task 2 (writing test_deps.py)
- **Issue:** conftest.py client fixture patched `app.services.deps._neo4j_client` — a module-level global that no longer exists after the deps.py rewrite. Would have caused the shared test fixture to fail.
- **Fix:** Updated patch target to `app.services.deps.get_neo4j_client` with `return_value=mock_neo4j`.
- **Files modified:** `backend/tests/conftest.py`
- **Verification:** All 7 DI tests pass; conftest fixture is used by integration tests.
- **Committed in:** `4f8b8d7` (Task 2 commit)

**2. [Rule 1 - Bug] Tightened test_search_router_no_local_client regex to avoid false positive**
- **Found during:** Task 2 (first test run)
- **Issue:** `assert "_neo4j_client" not in source` matched the import line `from ..services.deps import get_neo4j_client` (substring), causing a false failure.
- **Fix:** Replaced string-in check with `re.search(r"^_neo4j_client\s*[=:]", source, re.MULTILINE)` to match only module-level assignment/annotation.
- **Files modified:** `backend/tests/unit/test_deps.py`
- **Verification:** 7/7 tests pass.
- **Committed in:** `4f8b8d7` (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (both Rule 1 — bugs introduced by the refactor itself)
**Impact on plan:** Both fixes were necessary for the tests to be correct. No scope creep.

## Issues Encountered

- The system Python (`/opt/anaconda3`) has a NumPy 2.x / scipy 1.x binary incompatibility that prevents importing neo4j/scipy. This is a pre-existing environment issue unrelated to this plan. All verifications ran with the project's venv at `backend/venv/bin/python`.

## Next Phase Readiness

- DI foundation is in place; future plans can migrate remaining routers (graph.py, authority.py, etc.) to use Depends(get_*) in the same pattern established here.
- graph.py and authority.py still have module-level `_neo4j_client` globals — deferred to a later plan per scope boundary.

---
*Phase: 02-backend*
*Completed: 2026-04-02*
