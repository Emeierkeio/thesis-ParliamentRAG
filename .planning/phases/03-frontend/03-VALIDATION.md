---
phase: 3
slug: frontend
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | tsc --noEmit (type checking) + grep-based assertions |
| **Config file** | `frontend/tsconfig.json` (strict: true already set) |
| **Quick run command** | `cd frontend && npx tsc --noEmit 2>&1 | tail -5` |
| **Full suite command** | `cd frontend && npx tsc --noEmit && grep -rn ": any\b" src/ --include="*.ts" --include="*.tsx" | wc -l` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run quick command
- **After every plan wave:** Run full suite
- **Before `/gsd:verify-work`:** Full suite must show 0 `any` and 0 tsc errors
- **Max feedback latency:** 10 seconds

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements — `tsc --noEmit` and grep are sufficient.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| /valutazione redirects to /evaluation | FE-03 | Requires running dev server | Navigate to localhost:3000/valutazione, verify redirect |
| Sidebar links point to new routes | FE-03 | Visual verification | Check all sidebar items navigate correctly |
| UI text still in Italian | FE-02 | Visual spot-check | Browse app, confirm labels/headings are Italian |

---

## Validation Sign-Off

- [ ] `tsc --noEmit` exits 0
- [ ] `grep -rn ": any\b" src/` returns 0 matches
- [ ] No Italian variable names in code
- [ ] All component folders have barrel exports
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
