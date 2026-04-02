---
phase: 1
slug: build-pipeline
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 7.x (existing in requirements.txt) |
| **Config file** | none — Wave 0 creates `build/tests/conftest.py` |
| **Quick run command** | `cd build && python -m pytest tests/ -x -q` |
| **Full suite command** | `cd build && python -m pytest tests/ -v` |
| **Estimated runtime** | ~15 seconds (no Neo4j needed for unit tests) |

---

## Sampling Rate

- **After every task commit:** Run `cd build && python -m pytest tests/ -x -q`
- **After every plan wave:** Run `cd build && python -m pytest tests/ -v`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | BUILD-01 | integration | `python -m pytest tests/test_schema.py` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | BUILD-03 | unit | `python -m pytest tests/test_xml_parser.py` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | BUILD-04 | unit | `python -m pytest tests/test_chunker.py` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | BUILD-07 | unit | `python -m pytest tests/test_db_builder.py` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | DATA-01 | unit | `python -m pytest tests/test_xml_parser.py::test_vote_parsing` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | DATA-02 | unit | `python -m pytest tests/test_xml_parser.py::test_argomenti_parsing` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | DATA-03 | unit | `python -m pytest tests/test_xml_parser.py::test_speaker_role` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | DATA-04 | unit | `python -m pytest tests/test_xml_parser.py::test_phase_type` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `build/tests/__init__.py` — package marker
- [ ] `build/tests/conftest.py` — shared fixtures (sample XML snippets, mock data)
- [ ] `build/tests/test_xml_parser.py` — stubs for XML parsing tests (votes, argomenti, speaker roles, phase types)
- [ ] `build/tests/test_chunker.py` — stubs for chunking logic tests
- [ ] `build/tests/test_db_builder.py` — stubs for batch write tests (mock Neo4j driver)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `make db-all` completes end-to-end | BUILD-09 | Requires running Neo4j Docker + real data download | Run `make db-all` against local Neo4j, verify no errors |
| Vote nodes visible in Neo4j browser | DATA-01 | Requires DB inspection | `MATCH (v:Vote) RETURN count(v)` should be > 0 |
| DISCUSSES edges exist | DATA-02 | Requires DB inspection | `MATCH ()-[:DISCUSSES]->() RETURN count(*)` should be > 0 |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
