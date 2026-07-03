---
phase: 12
slug: multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-07-03
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest (backend), jest/vitest per frontend config |
| **Config file** | backend: existing `backend/tests/` structure; frontend: `frontend/package.json` test script |
| **Quick run command** | `cd backend && python -m pytest tests/ -x -q` |
| **Full suite command** | `cd backend && python -m pytest tests/ -q` |
| **Estimated runtime** | ~30 seconds (backend quick) |

---

## Sampling Rate

- **After every task commit:** Run `cd backend && python -m pytest tests/ -x -q`
- **After every plan wave:** Run `cd backend && python -m pytest tests/ -q` (+ `npx tsc --noEmit` in frontend for Plan 3)
- **Before `/gsd:verify-work`:** Full suite green + manual Cypher counts verified
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 12-01-* | 01 | 1 | P1-R1 QueryRequest legislature field | unit | `pytest tests/test_query_router.py::test_legislature_field -x` | ❌ W0 | ⬜ pending |
| 12-01-* | 01 | 1 | P1-R2 channels accept legislature kwarg | unit | `pytest tests/test_retrieval_channels.py::test_legislature_param -x` | ❌ W0 | ⬜ pending |
| 12-01-* | 01 | 1 | P1-R3 default-19 regression (same results) | integration | `pytest tests/test_retrieval_integration.py::test_legislature_default_same -x` | ❌ W0 | ⬜ pending |
| 12-01-* | 01 | 1 | P1-R4 timeline default filter | integration | `pytest tests/test_timeline_legislature.py::test_default_filter -x` | ❌ W0 | ⬜ pending |
| 12-02-* | 02 | 1 | timeline legislature filter | integration | `pytest tests/test_timeline_legislature.py -x` | ❌ W0 | ⬜ pending |
| 12-03-* | 03 | 1 | session dedup by legislature + loaders | integration | `pytest tests/test_db_builder.py::test_existing_session_numbers_by_legislature -x` | ❌ W0 | ⬜ pending |
| 12-04-* | 04 | 2 | XVIII ingest counts | smoke | make exit code + Cypher gate (manual detail below) | n/a | ⬜ pending |
| 12-05-* | 05 | 2 | frontend selector + payload | typecheck | `cd frontend && npx tsc --noEmit` | ✅ | ⬜ pending |
| 12-06-* | 06 | 3 | E2E pre-checks | integration | `pytest tests/ -q` + `npx tsc --noEmit` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `backend/tests/test_query_router.py` — `test_legislature_field` (Pydantic default 19)
- [ ] `backend/tests/test_retrieval_channels.py` — `test_legislature_param` per channel class
- [ ] `backend/tests/test_retrieval_integration.py` — `test_legislature_default_same` (live DB; skip-marked if Neo4j down)
- [ ] `backend/tests/test_timeline_legislature.py` — default + explicit legislature filter
- [ ] `backend/tests/test_db_builder.py` — `test_existing_session_numbers_by_legislature`

*Frontend: no jest test infra confirmed — `npx tsc --noEmit` is the automated gate; UI behavior manual.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| XVIII ingest completeness | P2-R3 | Requires 2-4h ingest run | `cypher-shell`: `MATCH (s:Session {legislature: 18}) RETURN s.chamber, count(*)` → camera 741, senato 459 |
| XVIII query E2E | P4 | Live pipeline + UI | Select XVIII in UI, ask "decreti COVID", verify citations dated 2018-2022; switch XIX, verify unchanged |
| XIX regression after ingest | P1-R3 | Live comparison | Same query pre/post ingest with legislature=19 → same citations |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
