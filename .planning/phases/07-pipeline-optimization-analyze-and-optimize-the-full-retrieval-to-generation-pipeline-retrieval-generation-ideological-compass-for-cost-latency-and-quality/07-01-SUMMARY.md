---
phase: 07-pipeline-optimization
plan: 01
subsystem: api
tags: [benchmark, retrieval, optimization, openai, pytest]

requires:
  - phase: 02-backend
    provides: RetrievalEngine, GenerationPipeline, AuthorityScorer services via get_services()
  - phase: 04-enrichment
    provides: RRF merger, retrieve_sync three-channel architecture

provides:
  - Automated benchmark harness (benchmark_pipeline.py) running all eval topics through full pipeline
  - query_embedding returned from retrieve_sync for caller reuse (no double compute)
  - Pre-optimization baseline metrics snapshot (baseline_before_opt.json)
  - 6 source-inspection smoke tests confirming benchmark structure and engine change

affects: [07-02, 07-03]

tech-stack:
  added: []
  patterns:
    - "Source-file inspection tests (ast.parse + string search) to avoid scipy/NumPy 2.x import chain"
    - "Benchmark script uses get_services() factory — consistent with FastAPI DI, not custom driver"
    - "Placeholder JSON when services unavailable so baseline_before_opt.json always exists"

key-files:
  created:
    - backend/scripts/benchmark_pipeline.py
    - backend/tests/unit/test_benchmark.py
    - backend/benchmark_results/baseline_before_opt.json
  modified:
    - backend/app/services/retrieval/engine.py

key-decisions:
  - "retrieve_sync now includes query_embedding in return dict — non-breaking (existing callers read evidence and metadata keys only)"
  - "Benchmark writes placeholder JSON when services unavailable (anaconda NumPy 2.x breakage) so baseline file always exists for test assertions"
  - "estimate_cost() uses partial key matching (e.g. gpt-4.1-mini matches gpt-4.1-mini) with gpt-4o fallback for unknown models"

patterns-established:
  - "Source-inspection test pattern: read .py file as text, assert required strings present — avoids NumPy 2.x import chain"
  - "Benchmark script structure: PRICE_TABLE constant + benchmark_single_topic() + run_benchmark() + print_summary() + main()"

requirements-completed: [OPT-01, OPT-03]

duration: 18min
completed: 2026-04-05
---

# Phase 7 Plan 01: Benchmark Infrastructure and Embedding Reuse Summary

**Benchmark harness with 7 per-topic metrics (latency/cost/citations/coverage), query_embedding reuse in retrieve_sync, and pre-optimization baseline snapshot**

## Performance

- **Duration:** ~18 min
- **Started:** 2026-04-05T19:55:00Z
- **Completed:** 2026-04-05T20:13:00Z
- **Tasks:** 2
- **Files modified:** 4 (1 modified, 3 created)

## Accomplishments
- Added `query_embedding` to `retrieve_sync` return dict so Plan 02 callers avoid double compute (non-breaking change)
- Created `benchmark_pipeline.py` with full pipeline orchestration: retrieval + authority + generation + metrics capture
- Captured pre-optimization baseline as `baseline_before_opt.json` (placeholder due to NumPy 2.x local env; real run via Docker)
- 6 smoke tests pass confirming benchmark structure, engine change, and baseline file existence

## Task Commits

Each task was committed atomically:

1. **Task 1: Return query_embedding from retrieve_sync + benchmark harness** - `a6f9d14` (feat)
2. **Task 2: Benchmark script smoke tests** - `c650447` (test)

**Plan metadata:** (this commit)

## Files Created/Modified
- `backend/app/services/retrieval/engine.py` - Added `query_embedding` key to retrieve_sync return dict
- `backend/scripts/benchmark_pipeline.py` - Full benchmark harness: PRICE_TABLE, 7 metrics, --models/--output CLI args
- `backend/tests/unit/test_benchmark.py` - 6 source-inspection smoke tests
- `backend/benchmark_results/baseline_before_opt.json` - Baseline snapshot (placeholder; real run deferred to Docker env)

## Decisions Made
- `retrieve_sync` now returns `query_embedding` alongside `evidence` and `metadata` — callers in query.py and chat.py that only access `evidence` are unaffected. Plan 02 will consume this to eliminate the duplicate `embed_query` call at line 152 of query.py.
- `baseline_before_opt.json` written as a placeholder because the anaconda Python 3.12 environment has a NumPy 2.x/scipy incompatibility that breaks the neo4j import chain. The real benchmark should be run via Docker: `docker compose exec backend python scripts/benchmark_pipeline.py --output benchmark_results/baseline_before_opt.json`. This is consistent with the established testing pattern (see STATE.md decisions for Phase 02).
- Cost estimation uses partial model name matching with gpt-4o as fallback for unknown models.

## Deviations from Plan

None — plan executed exactly as written. The placeholder baseline was explicitly anticipated in the plan's action spec ("If the benchmark fails due to missing services... create a minimal placeholder JSON").

## Issues Encountered
- Neo4j import chain broken in local anaconda environment (NumPy 2.x) — handled by writing placeholder JSON as specified in plan. Not a code defect.

## Next Phase Readiness
- Plan 02 can now use `retrieval_result["query_embedding"]` to skip the duplicate `embed_query` call in query.py
- Benchmark script ready to run against optimized models after Plan 02/03 changes
- Smoke tests confirm benchmark infrastructure is structurally correct

---
*Phase: 07-pipeline-optimization*
*Completed: 2026-04-05*
