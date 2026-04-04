---
phase: 6
slug: senate-data-integration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-04
---

# Phase 6 — Validation Strategy

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest (build + backend) + tsc --noEmit (frontend) |
| **Quick run command** | `cd build && python -m pytest tests/test_senate_parser.py -x -q` |
| **Full suite command** | `cd build && python -m pytest tests/ -v && cd ../backend && python -m pytest tests/ -v && cd ../frontend && npx tsc --noEmit` |
| **Estimated runtime** | ~30 seconds |

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `make db-senate` completes | SEN-02 | Requires live Neo4j + network | Run `make db-senate`, verify no errors |
| Senate data in Neo4j | SEN-02 | Requires DB inspection | `MATCH (s:Session {chamber:"senato"}) RETURN count(s)` > 0 |
| Chamber selector works | SEN-03 | Visual UI verification | Toggle Camera/Senato/Both, verify results change |
| `make db-all` builds both | SEN-05 | Requires full rebuild | Run `make db-all`, verify both chambers present |

## Validation Sign-Off

- [ ] All automated tests pass
- [ ] `tsc --noEmit` exits 0
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
