---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: 02
subsystem: backend
tags: [compass, pca, vote-intelligence, numpy, caching]
dependency_graph:
  requires: []
  provides: [GET /api/compass/votes, party-vote-matrix, SVD-PCA-2D, TTL-cache]
  affects: [backend/app/routers/compass.py, backend/app/services/compass/vote_pipeline.py]
tech_stack:
  added: []
  patterns: [numpy SVD-PCA, in-process TTL dict cache, executor-wrapped sync in async endpoint]
key_files:
  created:
    - backend/app/services/compass/vote_pipeline.py
    - backend/tests/unit/test_vote_compass.py
  modified:
    - backend/app/routers/compass.py
decisions:
  - "numpy SVD-PCA only (no scipy) — anaconda NumPy 2.x/scipy incompatibility; vote_pipeline.py has zero scipy imports"
  - "Test file pre-stubs compass package submodules in sys.modules to prevent scipy crash from __init__.py import chain (established project pattern)"
  - "pca_2d returns full variance_explained list (not sliced to 2); _compute_vote_compass slices to [:2] in response payload"
  - "chamber default 'camera' not 'both' — vote compass mixes political systems if chambers combined (Pitfall 1)"
  - "available:false + reason:individual_votes_pending when < 3 parties have IndividualVote data (Pitfall 3)"
metrics:
  duration_minutes: 4
  tasks_completed: 2
  files_created: 2
  files_modified: 1
  completed_date: "2026-07-06"
---

# Phase 14 Plan 02: Vote-Based Compass (F2) Summary

**One-liner:** Party×vote matrix reduced to 2D via numpy SVD-PCA, cached per legislature/chamber, exposed at GET /api/compass/votes separate from text compass.

## What Was Built

### Task 1 — vote_pipeline.py (TDD)

`backend/app/services/compass/vote_pipeline.py`:

- `PARTY_VOTE_MATRIX_QUERY` — Cypher with chamber+legislature filter on Session, date-scoped MEMBER_OF_GROUP, aggregates IndividualVote by party majority per vote.
- `build_party_vote_matrix(rows)` — favor>against→1.0, against>favor→0.0, equal/missing→0.5. Initialised to 0.5 (np.full). Returns (matrix, party_labels, vote_ids).
- `pca_2d(matrix)` — centres per vote column, applies np.linalg.svd, returns coords shape (n_parties, 2) and variance_explained list. Guard: < 2 rows or < 2 cols → zeros + empty list.
- `get_vote_compass(legislature, chamber, neo4j)` — TTL cache wrapper using `_vote_compass_cache` dict, 1-hour TTL.
- `_compute_vote_compass(legislature, chamber, neo4j)` — queries DB, builds matrix, runs PCA, returns available:false when < 3 parties have data.

Test file `backend/tests/unit/test_vote_compass.py`:
- 5 tests: `test_party_vote_matrix`, `test_pca_shape`, `test_pca_shape_guard_too_small`, `test_compass_cache`, `test_compass_cache_expires`.
- Pre-stubs compass package submodules in sys.modules to bypass scipy/NumPy 2.x crash.
- All pass with numpy only.

### Task 2 — GET /api/compass/votes endpoint

`backend/app/routers/compass.py`:
- Added `from ..services.compass.vote_pipeline import get_vote_compass`.
- New `@router.get("/compass/votes")` endpoint: runs `get_vote_compass` in executor (async-safe), defaults legislature=19, chamber="camera".
- Existing `@router.post("/compass")` text compass untouched.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Test assertion `len(variance_explained) <= 2` was stricter than spec**
- **Found during:** Task 1 GREEN phase — `pca_2d` returns all singular value variances; plan only requires `sum <= 1.0 + 1e-9`.
- **Fix:** Removed extra `len <= 2` assertion from `test_pca_shape`. The implementation correctly slices to `[:2]` inside `_compute_vote_compass` for the API response.
- **Files modified:** `backend/tests/unit/test_vote_compass.py`
- **Commit:** a4bacf7

## Self-Check: PASSED
