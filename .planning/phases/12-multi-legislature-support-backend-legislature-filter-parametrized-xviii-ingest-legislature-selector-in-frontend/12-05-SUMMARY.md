---
phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
plan: "05"
subsystem: frontend
tags: [legislature-selector, i18n, use-chat, ChatArea, URL-persistence]
dependency_graph:
  requires: ["12-01"]
  provides: ["LEG-05", "legislature-selector-ui"]
  affects: ["frontend/src/hooks/use-chat.ts", "frontend/src/components/chat/ChatArea.tsx", "frontend/src/app/home/page.tsx", "frontend/src/app/chat/[id]/page.tsx"]
tech_stack:
  added: []
  patterns: ["segmented-control-selector", "localStorage+URL-param-persistence", "next-intl-dynamic-key"]
key_files:
  created:
    - frontend/src/components/chat/LegislatureSelector.tsx
  modified:
    - frontend/src/hooks/use-chat.ts
    - frontend/src/components/chat/ChatArea.tsx
    - frontend/src/app/home/page.tsx
    - frontend/src/app/chat/[id]/page.tsx
    - frontend/messages/it.json
    - frontend/messages/en.json
decisions:
  - "[12-05]: LegislatureSelector uses numeric 18|19 type (not string) matching plan spec; t(String(option)) call avoids TS key union issue"
  - "[12-05]: badgeKey computed as badge_${chamberPart}_${legislature} dynamic string; cast as never for next-intl key union"
  - "[12-05]: Legislature hydration added inside existing chamber hydration useEffect (single effect, eslint-disable comment already present)"
metrics:
  duration: "3min"
  completed: "2026-07-03"
  tasks_completed: 3
  files_modified: 7
---

# Phase 12 Plan 05: Legislature Selector Frontend Summary

LegislatureSelector segmented control (XVIII|XIX) added to chat UI with localStorage+URL persistence, fetch payload inclusion, and legislature-aware welcome badges mirroring the existing ChamberSelector pattern exactly.

## Tasks Completed

| # | Name | Commit | Files |
|---|------|--------|-------|
| 1 | Create LegislatureSelector component + i18n namespace | 9dbcca8 | LegislatureSelector.tsx, it.json, en.json |
| 2 | Add legislature state, persistence, and fetch payload to use-chat.ts | 4b967a5 | use-chat.ts |
| 3 | Wire LegislatureSelector into ChatArea + pages + legislature-aware badge | 936585b | ChatArea.tsx, home/page.tsx, chat/[id]/page.tsx, it.json, en.json |

## Deviations from Plan

None — plan executed exactly as written.

## Decisions Made

1. **LegislatureSelector key cast:** `t(String(option) as "18" | "19")` used inside the component to satisfy TS union type for next-intl; `t(badgeKey as never)` in WelcomeScreen for dynamic key.
2. **Legislature hydration in single useEffect:** Added legislature hydration lines at the end of the existing chamber hydration `useEffect` (sharing the `// eslint-disable-next-line` comment) to keep a single effect for both selectors and avoid duplicating the disable comment.

## Self-Check: PASSED

- LegislatureSelector.tsx: FOUND
- Commit 9dbcca8: FOUND
- Commit 4b967a5: FOUND
- Commit 936585b: FOUND
