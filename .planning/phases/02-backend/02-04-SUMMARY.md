---
phase: 02-backend
plan: "04"
subsystem: backend/evaluation
tags: [refactor, service-extraction, tdd, evaluation-metrics]
dependency_graph:
  requires: ["02-01", "02-02", "02-03"]
  provides: ["evaluation_service.py metric computation"]
  affects: ["backend/app/routers/evaluation.py", "backend/app/services/evaluation_service.py"]
tech_stack:
  added: []
  patterns: ["service-layer extraction", "TDD red-green", "verbatim function move"]
key_files:
  created:
    - backend/app/services/evaluation_service.py
    - backend/tests/unit/test_evaluation_service.py
  modified:
    - backend/app/routers/evaluation.py
decisions:
  - "_compute_automated_metrics and all metric helpers extracted verbatim to evaluation_service.py preserving all three historical bug fixes"
  - "Router retains Neo4j I/O helpers (_fetch_all_chats, _fetch_chunk_texts) as they belong in the HTTP layer"
  - "evaluation_service exports _build_expert_full_lookup, ALL_PARTIES, KNOWN_PARTIES, _count_parties_in_text so the router can use them without re-defining"
metrics:
  duration: 6min
  completed: "2026-04-02T16:17:55Z"
  tasks_completed: 2
  files_changed: 3
---

# Phase 02 Plan 04: Evaluation Service Extraction Summary

**One-liner:** Verbatim extraction of 400+ lines of metric computation from evaluation.py into evaluation_service.py with TDD test coverage verifying all three historical bug fixes.

## What Was Built

`evaluation_service.py` now owns all metric computation logic:
- `_compute_automated_metrics()` — per-chat metrics (party coverage, citation fidelity, authority utilization, authority discrimination, response completeness, per-group authority breakdown)
- `_compute_baseline_authority_from_precomputed()` — bug fix #1: uses query-specific precomputed scores, not global max
- `_compute_baseline_authority_full()` — fallback text-matching for baseline authority
- `_compute_aggregated()` — aggregate metrics with confidence intervals
- `_build_expert_full_lookup()`, `_count_parties_in_text()`, CI helpers

`evaluation.py` router is now a thin HTTP wrapper (509 lines, down from 959). It handles Neo4j I/O, request parsing, and response formatting. All metric computation calls delegate to the service.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Extract evaluation service and write tests (TDD) | 6a037b4 | evaluation_service.py, test_evaluation_service.py |
| 2 | Thin evaluation.py router to use service | 4108927 | evaluation.py |

## Verification

- `wc -l evaluation.py` = 509 (< 600 required)
- `wc -l evaluation_service.py` = 517 (> 200 required)
- `grep -c "def _compute" evaluation.py` = 0 (all moved to service)
- `pytest tests/unit/test_evaluation_service.py` = 9 passed

## Decisions Made

1. **Verbatim extraction** — All metric computation functions moved with zero logic changes. Historical bug fixes (party_top_expert fallback, precomputed baseline experts, expert_full_lookup fallback) preserved exactly as in MEMORY.md.

2. **Router retains data-access helpers** — `_fetch_all_chats`, `_fetch_chunk_texts`, `_load_simple_ratings`, and `_get_client` remain in the router because they perform Neo4j I/O and are part of the HTTP orchestration layer. The service is kept stateless and dependency-free.

3. **Service exports constants** — `ALL_PARTIES`, `KNOWN_PARTIES`, `_count_parties_in_text`, and `_build_expert_full_lookup` exported from the service so the router does not re-define them. Single source of truth.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

All files and commits verified present:
- backend/app/services/evaluation_service.py — FOUND
- backend/tests/unit/test_evaluation_service.py — FOUND
- commit 6a037b4 — FOUND
- commit 4108927 — FOUND
