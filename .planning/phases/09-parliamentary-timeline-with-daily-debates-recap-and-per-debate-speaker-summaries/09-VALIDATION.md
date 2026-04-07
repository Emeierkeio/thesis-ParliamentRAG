---
phase: 9
slug: parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-07
---

# Phase 9 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 7.x (backend) / manual verification (frontend) |
| **Config file** | backend/pytest.ini or none — Wave 0 installs |
| **Quick run command** | `pytest backend/tests/test_timeline.py -x -q` |
| **Full suite command** | `pytest backend/tests/ -x -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `pytest backend/tests/test_timeline.py -x -q`
- **After every plan wave:** Run `pytest backend/tests/ -x -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |

*Status: pending · green · red · flaky*

---

## Wave 0 Requirements

- [ ] `backend/tests/test_timeline.py` — stubs for timeline API endpoints
- [ ] `backend/tests/conftest.py` — shared fixtures (if not exists)

*If none: "Existing infrastructure covers all phase requirements."*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Timeline page renders with session cards | UI | Visual rendering | Navigate to /timeline, verify session cards display |
| Infinite scroll loads more sessions | UI | Browser interaction | Scroll to bottom, verify new sessions load |
| Search filters debates by keyword | UI | Browser interaction | Type keyword, verify filtered results |
| Speaker summary expands inline | UI | Browser interaction | Click speaker, verify summary appears |
| Mobile responsive layout | UI | Device testing | Resize browser, verify full-width cards |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
