---
phase: 06-senate-data-integration-with-chamber-selector
plan: 02
subsystem: api
tags: [retrieval, neo4j, cypher, fastapi, pydantic, chamber-filter]

# Dependency graph
requires:
  - phase: 06-senate-data-integration-with-chamber-selector
    provides: Session.chamber property populated in Neo4j for Camera and Senato sessions

provides:
  - ChatRequest.chamber field (default "both") enabling frontend to select camera/senato/both
  - QueryRequest.chamber field with same default
  - WHERE s.chamber IN $chambers filtering in all 3 retrieval channels (dense, sparse, graph)
  - chambers parameter threaded through engine.retrieve_sync() and retrieve() to all channel calls

affects:
  - 06-senate-data-integration-with-chamber-selector plan 03 (frontend chamber selector UI)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Chamber filter as list[str] (not enum) so default ['camera','senato'] is a no-op for existing callers"
    - "Backend converts 'both' string to list at router layer before passing down to engine"

key-files:
  created: []
  modified:
    - backend/app/routers/chat.py
    - backend/app/routers/query.py
    - backend/app/services/retrieval/engine.py
    - backend/app/services/retrieval/dense_channel.py
    - backend/app/services/retrieval/sparse_channel.py
    - backend/app/services/retrieval/graph_channel.py

key-decisions:
  - "chambers passed as list[str] not str enum — callers set default ['camera','senato'] locally so missing callers still get both chambers"
  - "graph_channel uses session_conditions list instead of appending to date_clause string — allows chamber condition to always be first, dates appended only when provided"

patterns-established:
  - "chambers default ['camera','senato'] at every layer (router, engine, channels) ensures backward compat"
  - "graph_channel._get_chunks_by_entity uses params['chambers'] inline rather than f-string interpolation"

requirements-completed: [SEN-04]

# Metrics
duration: 4min
completed: 2026-04-04
---

# Phase 6 Plan 02: Chamber Filtering in Retrieval Channels Summary

**WHERE s.chamber IN $chambers added to all 3 Neo4j retrieval channels (dense/sparse/graph) with ChatRequest.chamber field defaulting to "both"**

## Performance

- **Duration:** 4 min
- **Started:** 2026-04-04T15:50:36Z
- **Completed:** 2026-04-04T15:54:17Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- All 3 retrieval channels (dense, sparse, graph) accept `chambers: list[str] | None` and inject `WHERE s.chamber IN $chambers` into their Cypher queries
- `ChatRequest` gains `chamber: str = Field(default="both")` — zero breaking change for existing clients
- `QueryRequest` (legacy endpoint) gains same field for consistency
- `RetrievalEngine.retrieve_sync()` and `retrieve()` both thread `chambers` from top-level call through to all channel calls
- Default behavior (both) returns all results with no performance penalty — `["camera", "senato"]` covers all sessions

## Task Commits

Each task was committed atomically:

1. **Task 1: Add chambers parameter to all 3 retrieval channels** - `cac008c` (feat)
2. **Task 2: Thread chambers through engine, ChatRequest, and query router** - `1fda532` (feat)

**Plan metadata:** (docs commit — see below)

## Files Created/Modified
- `backend/app/services/retrieval/dense_channel.py` - chambers param + WHERE s.chamber IN $chambers after Session MATCH
- `backend/app/services/retrieval/sparse_channel.py` - chambers param + WHERE s.chamber IN $chambers after Session MATCH
- `backend/app/services/retrieval/graph_channel.py` - chambers param on retrieve(), _get_chunks_from_signatories(), _get_chunks_by_entity(); uses session_conditions list pattern
- `backend/app/services/retrieval/engine.py` - chambers param on retrieve_sync() and retrieve(); threads to dense, sparse, graph channel calls
- `backend/app/routers/chat.py` - ChatRequest.chamber field; chambers list computed in both background and streaming handlers
- `backend/app/routers/query.py` - QueryRequest.chamber field; chambers threading in both streaming and sync paths

## Decisions Made
- `chambers` passed as `list[str]` rather than enum: each layer defaults to `["camera", "senato"]` independently so any call site that doesn't pass the arg still gets full coverage — no silent filtering.
- `graph_channel` switched from appending `s.chamber IN $chambers` to an `AND` on the existing date clause to using a `session_conditions` list that always starts with the chamber condition. This avoids a logic edge case where `date_clause = "1=1"` would be replaced with `"1=1 AND s.chamber IN $chambers"` (harmless but ugly).

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered
- Live import test (`python -c "from app.routers.chat import ChatRequest..."`) failed due to known NumPy 2.x / scipy binary incompatibility in the anaconda environment (documented in STATE.md). Verified via source-file inspection instead, consistent with established project pattern.

## User Setup Required
None — no external service configuration required.

## Next Phase Readiness
- Backend chamber filtering is complete. Frontend can now send `chamber: "camera"`, `"senato"`, or `"both"` in the ChatRequest body and retrieval will be scoped accordingly.
- Phase 6 Plan 03 (frontend chamber selector UI) can proceed immediately.

---
*Phase: 06-senate-data-integration-with-chamber-selector*
*Completed: 2026-04-04*
