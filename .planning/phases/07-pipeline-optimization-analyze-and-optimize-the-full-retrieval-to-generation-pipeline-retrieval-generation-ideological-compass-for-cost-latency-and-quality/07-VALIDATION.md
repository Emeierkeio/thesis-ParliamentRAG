---
phase: 07
slug: pipeline-optimization
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-05
---

# Phase 07 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 7.x |
| **Config file** | backend/pytest.ini or pyproject.toml |
| **Quick run command** | `cd backend && python -m pytest tests/ -x -q --timeout=30` |
| **Full suite command** | `cd backend && python -m pytest tests/ -v --timeout=60` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd backend && python -m pytest tests/ -x -q --timeout=30`
- **After every plan wave:** Run `cd backend && python -m pytest tests/ -v --timeout=60`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 0 | Benchmark harness | integration | `python -m pytest tests/test_benchmark.py` | ❌ W0 | ⬜ pending |
| 07-02-01 | 02 | 1 | Model swap | integration | `python -m pytest tests/test_model_swap.py` | ❌ W0 | ⬜ pending |
| 07-03-01 | 03 | 1 | Latency fixes | unit | `python -m pytest tests/test_latency.py` | ❌ W0 | ⬜ pending |
| 07-04-01 | 04 | 2 | NER channel | unit | `python -m pytest tests/test_ner_channel.py` | ❌ W0 | ⬜ pending |
| 07-05-01 | 05 | 2 | RRF tuning | integration | `python -m pytest tests/test_rrf_sweep.py` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `backend/tests/test_benchmark.py` — benchmark harness stubs
- [ ] `backend/tests/conftest.py` — shared fixtures (if not already present)
- [ ] Benchmark script infrastructure for before/after comparison

*If none: "Existing infrastructure covers all phase requirements."*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Generation quality comparison | Quality baseline | Subjective assessment of output coherence | Run same query before/after model swap, compare narratives |
| Compass visual accuracy | Compass review | Political spectrum positioning is subjective | Verify compass output for known political positions |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
