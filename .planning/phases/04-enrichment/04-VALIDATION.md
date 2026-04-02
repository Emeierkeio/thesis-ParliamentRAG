---
phase: 4
slug: enrichment
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 7.x (backend tests) + build tests |
| **Config file** | `backend/tests/conftest.py` (existing) |
| **Quick run command** | `cd backend && python -m pytest tests/unit/test_sparse_channel.py tests/unit/test_rrf_merger.py -x -q` |
| **Full suite command** | `cd backend && python -m pytest tests/ -v && cd ../build && python -m pytest tests/ -v` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run quick command
- **After every plan wave:** Run full suite
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Wave 0 Requirements

- [ ] `backend/tests/unit/test_sparse_channel.py` — stubs for BM25 channel tests
- [ ] `backend/tests/unit/test_rrf_merger.py` — stubs for RRF merger tests
- [ ] `build/tests/test_ner.py` — stubs for NER extraction tests
- [ ] `build/tests/test_sparql.py` — stubs for SPARQL ingestion tests

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| BM25 improves exact-term queries | RET-01 | Requires running Neo4j with full data | Query "decreto 231" with/without sparse channel, compare results |
| SPARQL data in graph | ENR-01 | Requires network + Neo4j | `make enrich-sparql` then verify `MATCH (iv:IndividualVote) RETURN count(iv)` > 0 |
| NER fields populated after rebuild | ENR-03 | Requires full rebuild | `make db-all` then `MATCH (c:Chunk) WHERE c.lawRefs IS NOT NULL RETURN count(c)` > 0 |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
