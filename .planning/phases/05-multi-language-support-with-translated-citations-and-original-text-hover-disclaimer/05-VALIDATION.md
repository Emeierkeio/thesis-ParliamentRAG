---
phase: 5
slug: multi-language-support
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-04
---

# Phase 5 — Validation Strategy

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | tsc --noEmit + grep assertions + pytest (backend) |
| **Quick run command** | `cd frontend && npx tsc --noEmit 2>&1 | tail -5` |
| **Full suite command** | `cd frontend && npx tsc --noEmit && cd ../backend && python -m pytest tests/ -q` |
| **Estimated runtime** | ~15 seconds |

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Language selector switches UI text | ML-01 | Visual verification | Toggle language, verify all labels change |
| Citations show translated text in English mode | ML-03 | Requires running pipeline + OpenAI | Send query in English mode, check citations are translated |
| Tooltip shows original Italian on hover | ML-04 | Visual/interaction verification | Hover over translated citation, verify Italian original appears |
| Banner + globe icon visible | ML-05 | Visual verification | Switch to English, verify banner and 🌐 icons |

## Validation Sign-Off

- [ ] `tsc --noEmit` exits 0
- [ ] All hardcoded Italian strings extracted to locale files
- [ ] Backend tests pass
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
