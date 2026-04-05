---
phase: 07-pipeline-optimization
plan: 02
subsystem: generation-config, query-router, chat-router, tests
tags: [cost-reduction, latency, gpt-4.1-mini, embedding-reuse, parallel-execution]
dependency_graph:
  requires: [07-01]
  provides: [gpt-4.1-mini-config, embedding-reuse, parallel-authority-compass]
  affects: [query.py, chat.py, default.yaml, benchmark]
tech_stack:
  added: []
  patterns: [asyncio.gather for parallel CPU tasks, retrieval_result dict reuse]
key_files:
  created:
    - backend/tests/unit/test_latency_fixes.py
  modified:
    - backend/config/default.yaml
    - backend/app/routers/query.py
    - backend/app/routers/chat.py
decisions:
  - "gpt-4.1-mini selected for all three generation stages — ~12x cost reduction vs gpt-4o ($0.20/$0.80 vs $2.50/$10.00)"
  - "query_embedding reused from retrieval_result dict added in Plan 07-01 — eliminates ~300ms embed_query round-trip"
  - "asyncio.gather for authority+compass: both are CPU-bound blocking tasks with no dependency on each other — safe to parallelize"
  - "Expert computation kept sequential after gather — it depends on authority_scores being ready"
  - "compass_result exception handling preserved after gather — compass failure must not abort the pipeline"
metrics:
  duration: 8min
  completed_date: "2026-04-05"
  tasks_completed: 2
  files_modified: 4
---

# Phase 7 Plan 02: Model Swap to gpt-4.1-mini and Latency Optimizations Summary

**One-liner:** Switched all three generation stages to gpt-4.1-mini and eliminated redundant embed_query calls by reusing query_embedding from retrieval result, with authority+compass running in parallel via asyncio.gather.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Model swap + latency optimizations | cd8b78b | backend/config/default.yaml, backend/app/routers/query.py, backend/app/routers/chat.py |
| 2 | Latency optimization tests | 00ed413 | backend/tests/unit/test_latency_fixes.py |

## Changes Made

### Task 1: Model Swap and Latency Optimizations

**Part A — default.yaml:**
- `analyst`, `writer`, `integrator` all changed from `"gpt-4o"` to `"gpt-4.1-mini"`
- Cost impact: ~12x reduction on generation tokens ($2.50/$10.00 → $0.20/$0.80 per 1M tokens)

**Part B — query.py (streaming path):**
- Removed redundant `embed_query` call (~300ms saved per request)
- `query_embedding = retrieval_result["query_embedding"]` — reuses embedding computed during retrieval in Plan 07-01
- Authority and compass now execute in parallel: `asyncio.gather(authority_task, compass_task)`
- Combined progress steps 3+4 into single message since they execute concurrently
- Removed the client disconnect check between authority and compass (they now run together)
- Expert computation still sequential after gather (depends on authority_scores)
- Compass exception handling preserved after gather

**Part B — query.py (non-streaming path):**
- Same embed_query removal and retrieval_result reuse applied to the synchronous code path

**Part C — chat.py:**
- Both occurrences of embed_query replaced with `retrieval_result["query_embedding"]`
- First occurrence: streaming chat endpoint (~line 262)
- Second occurrence: high-quality chat endpoint (~line 734)

### Task 2: Latency Fix Tests

Created `backend/tests/unit/test_latency_fixes.py` with 6 source-file inspection tests:

1. `test_query_py_no_double_embed` — asserts embed_query absent from query.py
2. `test_chat_py_no_double_embed` — asserts embed_query absent from chat.py
3. `test_query_py_uses_retrieval_result_embedding` — asserts retrieval_result["query_embedding"] present in query.py
4. `test_chat_py_uses_retrieval_result_embedding` — asserts retrieval_result["query_embedding"] present in chat.py
5. `test_query_py_parallel_authority_compass` — asserts asyncio.gather present in query.py
6. `test_models_are_gpt41_mini` — asserts all three model keys in default.yaml use gpt-4.1-mini

All 6 tests pass.

## Verification Results

```
6 passed in 0.01s
grep -c "gpt-4.1-mini" config/default.yaml → 3
grep -c "embed_query" app/routers/query.py → 0
grep -c "embed_query" app/routers/chat.py → 0
asyncio.gather(authority_task, compass_task) → present in query.py
```

## Deviations from Plan

None — plan executed exactly as written.

## Key Decisions

- **gpt-4.1-mini selected** for all three generation stages — ~12x cost reduction vs gpt-4o ($0.20/$0.80 vs $2.50/$10.00).
- **query_embedding reuse** from retrieval_result dict added in Plan 07-01 — eliminates ~300ms embed_query round-trip per request.
- **asyncio.gather** for authority+compass: both are CPU-bound blocking tasks with no mutual dependency — safe to parallelize.
- **Expert computation kept sequential** after gather — it depends on authority_scores being ready from authority_all.
- **Compass exception handling preserved** after gather — compass failure must not abort the pipeline.

## Self-Check: PASSED

All created files exist. Both task commits verified in git log:
- cd8b78b: feat(07-02): model swap to gpt-4.1-mini + latency optimizations
- 00ed413: test(07-02): add latency fix inspection tests
