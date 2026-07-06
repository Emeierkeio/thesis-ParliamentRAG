---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: "03"
subsystem: backend-api
tags: [votes, router, fastapi, cohesion, rice-index, f3, f5]
dependency_graph:
  requires: ["14-01"]
  provides: ["14-05", "14-07"]
  affects: ["backend/app/main.py", "backend/app/routers/votes.py"]
tech_stack:
  added: []
  patterns: ["thin-router-pattern", "Depends(get_neo4j_client)", "service-delegation"]
key_files:
  created:
    - backend/app/routers/votes.py
  modified:
    - backend/app/main.py
    - backend/tests/unit/test_routers.py
    - backend/tests/unit/test_cypher_queries.py
decisions:
  - "[14-03]: Optional[str] used instead of str | None union syntax for Python 3.9 compatibility in votes.py"
  - "[14-03]: Source-inspection tests chosen over live TestClient to avoid scipy/NumPy 2.x anaconda import chain"
  - "[14-03]: Router prefix='/api' with relative path /votes and /rankings/votes — consistent with compass and timeline routers"
metrics:
  duration: "2min"
  completed: "2026-07-07"
  tasks_completed: 2
  files_modified: 4
---

# Phase 14 Plan 03: Votes Router Summary

Thin votes router over votes_service, exposing F5 search and F3 cohesion endpoints registered in main.py.

## What Was Built

**backend/app/routers/votes.py** — New router (81 lines) with two endpoints:
- `GET /api/votes` — F5 paginated vote search, delegates to `votes_service.search_votes`
- `GET /api/rankings/votes` — F3 cohesion/rebellion stats, delegates to `votes_service.get_party_cohesion` or `get_deputy_vote_stats` depending on `deputy_id` query param

**backend/app/main.py** — Added `from .routers.votes import router as votes_router` and `app.include_router(votes_router)`.

**Tests (2 new functions)**:
- `test_routers.py::test_votes_endpoint` — source inspection: verifies both route paths, all 3 service delegations, no inline Cypher, main.py registration
- `test_cypher_queries.py::test_votes_search_has_chamber_filter` — source inspection: verifies `s.chamber`, `s.legislature`, `coalesce(a.title, d.title, v.subject)` label hierarchy, and date-scoped `MEMBER_OF_GROUP`

## Verification Results

```
tests/unit/test_routers.py tests/unit/test_cypher_queries.py — 50 passed in 0.07s
```

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

- votes.py: FOUND
- 14-03-SUMMARY.md: FOUND
- commit 52f862d (Task 1): FOUND
- commit 17b6b9f (Task 2): FOUND
