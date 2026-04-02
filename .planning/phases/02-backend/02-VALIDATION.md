---
phase: 2
slug: backend
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 7.x + pytest-asyncio + httpx (TestClient) |
| **Config file** | `backend/tests/conftest.py` (Wave 0 creates) |
| **Quick run command** | `cd backend && python -m pytest tests/ -x -q --ignore=tests/integration` |
| **Full suite command** | `cd backend && python -m pytest tests/ -v` |
| **Estimated runtime** | ~30 seconds (unit tests with mocks) |

---

## Sampling Rate

- **After every task commit:** Run quick command
- **After every plan wave:** Run full suite
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| TBD | TBD | 0 | QA-01 | scaffold | `pytest tests/ --collect-only` | ❌ W0 | ⬜ pending |
| TBD | TBD | 1 | SVC-01 | unit | `pytest tests/test_cypher.py -v` | ❌ W0 | ⬜ pending |
| TBD | TBD | 1 | SVC-02 | unit | `pytest tests/test_experts.py -v` | ❌ W0 | ⬜ pending |
| TBD | TBD | 1 | SVC-03 | unit | `pytest tests/test_evaluation_service.py -v` | ❌ W0 | ⬜ pending |
| TBD | TBD | 2 | API-01 | unit | `pytest tests/test_routers.py -v` | ❌ W0 | ⬜ pending |
| TBD | TBD | 2 | API-03 | unit | `pytest tests/test_sse_contract.py -v` | ❌ W0 | ⬜ pending |
| TBD | TBD | 3 | SCR-01 | unit | `pytest tests/test_scripts.py -v` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `backend/tests/__init__.py` — package marker
- [ ] `backend/tests/conftest.py` — shared fixtures (mock Neo4j driver, mock OpenAI, TestClient)
- [ ] `backend/tests/test_cypher.py` — stubs for Cypher query update verification
- [ ] `backend/tests/test_experts.py` — stubs for expert service tests
- [ ] `backend/tests/test_evaluation_service.py` — stubs for evaluation service tests
- [ ] `backend/tests/test_routers.py` — stubs for thin router tests
- [ ] `backend/tests/test_sse_contract.py` — stubs for SSE event contract verification
- [ ] `backend/tests/test_scripts.py` — stubs for script tests

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Backend starts with new schema DB | SVC-01 | Requires running Neo4j with Phase 1 data | `make dev`, verify no startup errors |
| SSE streaming works end-to-end | API-03 | Requires full pipeline with OpenAI | Send a chat query via frontend, verify response streams |
| Evaluation dashboard loads correctly | SVC-03 | Requires ChatHistory data in Neo4j | Navigate to /valutazione, verify metrics display |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
