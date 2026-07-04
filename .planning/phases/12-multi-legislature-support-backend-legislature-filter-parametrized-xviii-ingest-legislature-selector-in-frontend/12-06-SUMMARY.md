---
phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
plan: 06
subsystem: verification
tags: [e2e, legislature, retrieval]

requires:
  - phase: 12
    plan: 04
    provides: XVIII data in Neo4j
  - phase: 12
    plan: 05
    provides: frontend legislature selector sending payload

provides:
  - E2E verified legislature scoping in both directions
  - Two leak fixes (commit 5a61bbd~ fix(12-06)): POST /api/query missing legislature forward; _coverage_fill vector search without legislature/chamber filter

affects: [phase 13 (country filter must include coverage_fill from day one)]

tech:
  discovered:
    - engine._coverage_fill was invisible to the plan-time channel audit — any post-merge augmentation query (coverage fill, neighbor expansion) must be included in future dimension filters. _expand_neighbors is safe by construction (NEXT chunks share the speech).
---

# 12-06: End-to-end verification — COMPLETE

One-liner: Legislature dimension verified E2E; two leaks found by the E2E itself and fixed.

## Automated results
- Backend tests: 246 passed; 4 pre-existing failures unrelated to phase 12 (translation/experts/sse tests stale since April refactors)
- Frontend: tsc --noEmit clean
- Timeline: default → XIX 2026 sessions; ?legislature=18 → 2018-2022 sessions ✓

## E2E (API level, POST /api/query)
- legislature=18, "reddito di cittadinanza": citations 2018-07-16 → 2022-06-22, leg19 chunks = 0 ✓
- default (19), "salario minimo": citations 2022-11-29 → 2026-06-10, leg18 chunks = 0 ✓

## Human verification (UI)
Selector visually verified on /home (screenshot). Remaining eyeball check for the user: select XVIII in the UI and confirm citations show 2018-2022 dates.
