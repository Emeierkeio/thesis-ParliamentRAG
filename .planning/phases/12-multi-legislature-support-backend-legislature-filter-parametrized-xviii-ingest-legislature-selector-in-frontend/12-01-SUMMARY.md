---
phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
plan: "01"
subsystem: backend-retrieval
tags: [legislature-filter, retrieval, cypher, pydantic]
dependency_graph:
  requires: []
  provides: [legislature-filter-in-retrieval-pipeline]
  affects: [backend/app/routers/query.py, backend/app/routers/chat.py, backend/app/services/retrieval/engine.py, backend/app/services/retrieval/dense_channel.py, backend/app/services/retrieval/sparse_channel.py, backend/app/services/retrieval/ner_channel.py, backend/app/services/retrieval/graph_channel.py]
tech_stack:
  added: []
  patterns: [additive-kwarg-default-19, cypher-where-filter, source-inspection-tests]
key_files:
  created:
    - backend/tests/test_query_router.py
    - backend/tests/test_retrieval_channels.py
  modified:
    - backend/app/routers/query.py
    - backend/app/routers/chat.py
    - backend/app/services/retrieval/dense_channel.py
    - backend/app/services/retrieval/sparse_channel.py
    - backend/app/services/retrieval/ner_channel.py
    - backend/app/services/retrieval/graph_channel.py
    - backend/app/services/retrieval/engine.py
decisions:
  - "[12-01]: legislature kwarg defaults to 19 on every signature — unmigrated callers are unaffected and result sets are unchanged while DB is 100% legislature=19"
  - "[12-01]: AND s.legislature = $legislature appended to existing WHERE s.chamber IN $chambers in all channels — purely additive, no restructuring needed"
  - "[12-01]: graph_channel uses session_conditions list (same pattern as chambers) for both _get_chunks_by_entity and _get_chunks_from_signatories"
metrics:
  duration: "4 min"
  completed: "2026-07-03"
  tasks_completed: 3
  files_modified: 9
---

# Phase 12 Plan 01: Legislature Filter in Retrieval Pipeline Summary

Add an orthogonal `legislature: int = 19` filter to all request models and all four retrieval channels so every Cypher query is scoped to a single legislature via `AND s.legislature = $legislature`.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Wave 0 — failing source-inspection tests | 95552fe | test_query_router.py, test_retrieval_channels.py |
| 2 | Add legislature field to QueryRequest/ChatRequest + propagate | 1ec7086 | query.py, chat.py |
| 3 | Add legislature filter to all four channels + engine | 38aba57 | dense/sparse/ner/graph_channel.py, engine.py |

## What Was Built

- `QueryRequest.legislature: int = Field(default=19)` in `query.py`; propagated to `services["retrieval"].retrieve()` via `legislature=_legislature`
- `ChatRequest.legislature: int = Field(default=19)` in `chat.py`; propagated in both `process_chat_background` and `process_chat_streaming` retrieve_sync calls
- All four retrieval channels (`DenseChannel`, `SparseChannel`, `NERChannel`, `GraphChannel`) now accept `legislature: int = 19` and embed `AND s.legislature = $legislature` in their Cypher WHERE clauses
- `graph_channel._get_chunks_by_entity` and `_get_chunks_from_signatories` both receive `legislature` via a `session_conditions` list, producing 2 Cypher occurrences (total 5 across all channels)
- `engine.retrieve_sync` and `engine.retrieve` both accept `legislature: int = 19` and thread it through to every channel closure
- Two source-inspection test files added using the established `Path(__file__).parent.parent / ...` pattern (no live imports, no scipy)

## Verification

- `python -m pytest tests/test_query_router.py tests/test_retrieval_channels.py -q` → 5 passed
- Manual grep: 5 Cypher occurrences of `s.legislature =` across all channels (dense×1, sparse×1, ner×1, graph×2)
- Full suite: 246 passed, 4 pre-existing unrelated failures (test_experts, test_sse_contract, test_translation_service — not caused by this plan)

## Deviations from Plan

None — plan executed exactly as written.

## Notes

- Zero behavior change today: all 1,111 existing Session nodes have `legislature=19`, so the `AND s.legislature = $legislature` filter matches every node in the current DB
- The filter becomes active the moment XVIII (legislature=18) data is ingested in Plan 04
- SSE event payloads and field names unchanged (API-03 contract honored)

## Self-Check: PASSED

- test_query_router.py: FOUND
- test_retrieval_channels.py: FOUND
- Commit 95552fe: FOUND
- Commit 1ec7086: FOUND
- Commit 38aba57: FOUND
