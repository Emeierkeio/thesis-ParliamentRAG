---
phase: 06-senate-data-integration-with-chamber-selector
plan: "03"
subsystem: frontend/chat
tags: [chamber-selector, i18n, localStorage, sse, react]
dependency_graph:
  requires: [06-02]
  provides: [chamber-filter-ui, chamber-in-request-body]
  affects: [frontend/src/components/chat/ChatArea.tsx, frontend/src/hooks/use-chat.ts]
tech_stack:
  added: []
  patterns: [segmented-button, localStorage-persistence, hook-state]
key_files:
  created:
    - frontend/src/components/chat/ChamberSelector.tsx
  modified:
    - frontend/messages/it.json
    - frontend/messages/en.json
    - frontend/src/components/chat/ChatArea.tsx
    - frontend/src/hooks/use-chat.ts
    - frontend/src/app/page.tsx
    - frontend/src/app/chat/[id]/page.tsx
decisions:
  - ChamberSelector placed in sticky header below ChatInput row, right-aligned — visible on every message state
  - chamber/onChamberChange props added to ChatArea (not managed internally) — follows existing prop-down pattern
  - chat/[id]/page.tsx also updated so shared-chat links render the selector
metrics:
  duration: ~6min
  completed: 2026-04-04T15:59:22Z
  tasks_completed: 2
  files_modified: 6
---

# Phase 06 Plan 03: ChamberSelector UI Component Summary

ChamberSelector segmented button (Camera / Senato / Entrambi) rendered in chat header, persisted in localStorage, and sent to backend as `chamber` field in every request body.

## Tasks Completed

| # | Name | Commit | Key Files |
|---|------|--------|-----------|
| 1 | ChamberSelector component + locale keys | 30e3a89 | ChamberSelector.tsx, it.json, en.json |
| 2 | Wire ChamberSelector into ChatArea and use-chat.ts | f22209e | ChatArea.tsx, use-chat.ts, page.tsx, chat/[id]/page.tsx |

## What Was Built

**ChamberSelector component** (`frontend/src/components/chat/ChamberSelector.tsx`):
- Segmented button group with 3 options: camera / senato / both
- Uses `useTranslations("ChamberSelector")` for full i18n support
- Styled with existing Tailwind/shadcn conventions, no animations

**Locale keys** added to both `it.json` and `en.json`:
- Italian: Camera / Senato / Entrambi / Ramo del Parlamento
- English: Chamber of Deputies / Senate / Both / Parliamentary Chamber

**use-chat.ts** additions:
- `chamber` state initialized from `localStorage.getItem("parliamentRAG.chamber")` with `"both"` default
- `useEffect` persists any chamber change to localStorage
- `chamber` included in fetch body: `{ query, task_id, chamber }`
- `chamber` and `setChamber` returned from the hook

**ChatArea.tsx** additions:
- `chamber` and `onChamberChange` props added to `ChatAreaProps` interface
- `ChamberSelector` imported and rendered in sticky header, right-aligned below the input row

**page.tsx and chat/[id]/page.tsx**: destructure `chamber`/`setChamber` from `useChat()` and pass to `ChatArea`.

## Deviations from Plan

**1. [Rule 2 - Missing functionality] Updated chat/[id]/page.tsx**
- Found during: Task 2
- Issue: The plan specified `frontend/src/app/chat/page.tsx` but this path does not exist; the main chat page is `src/app/page.tsx`. The shared-chat page at `src/app/chat/[id]/page.tsx` also uses ChatArea and would have had a TypeScript error without the required new props.
- Fix: Updated `chat/[id]/page.tsx` as well as `page.tsx` — both now pass `chamber` and `onChamberChange` to ChatArea.
- Files modified: `frontend/src/app/chat/[id]/page.tsx`
- Commit: f22209e

## Success Criteria Met

- [x] ChamberSelector renders Camera/Senato/Both as segmented buttons
- [x] Default is "both" (Entrambi in Italian)
- [x] Selection persists in localStorage under "parliamentRAG.chamber"
- [x] Chamber value included in fetch body sent to backend
- [x] Labels translated in both it.json and en.json
- [x] Zero TypeScript errors

## Self-Check: PASSED

Files created:
- FOUND: frontend/src/components/chat/ChamberSelector.tsx

Commits exist:
- FOUND: 30e3a89 (Task 1)
- FOUND: f22209e (Task 2)
