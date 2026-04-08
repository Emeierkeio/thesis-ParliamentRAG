---
phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
plan: 03
subsystem: frontend-data-layer
tags: [typescript, react, hooks, i18n, timeline, api-client]
dependency_graph:
  requires: ["09-02"]
  provides: ["09-04", "09-05"]
  affects: ["frontend/src/types", "frontend/src/lib", "frontend/src/hooks", "frontend/messages"]
tech_stack:
  added: []
  patterns: ["IntersectionObserver infinite scroll", "debounced search", "locale cookie reading", "Accept-Language header"]
key_files:
  created:
    - frontend/src/types/timeline.ts
    - frontend/src/lib/timeline-api.ts
    - frontend/src/hooks/use-timeline.ts
  modified:
    - frontend/messages/en.json
    - frontend/messages/it.json
decisions:
  - "useRef typed as `ReturnType<typeof setTimeout> | undefined` initialized to undefined for React 19 compatibility"
  - "RefObject<HTMLDivElement | null> used (not HTMLDivElement) to match React 19 useRef return type"
  - "Timeline i18n section appended after HistoryModal to preserve alphabetical ordering within file"
metrics:
  duration: "2min"
  completed: "2026-04-08"
  tasks_completed: 2
  files_created: 3
  files_modified: 2
---

# Phase 09 Plan 03: Frontend Data Layer for Parliamentary Timeline Summary

**One-liner:** TypeScript interfaces mirroring Pydantic models, API client with locale-aware headers, and a hook with IntersectionObserver infinite scroll and 300ms search debounce — plus 27 i18n keys in both locales.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create TypeScript types and API client | 73d1f41 | frontend/src/types/timeline.ts, frontend/src/lib/timeline-api.ts |
| 2 | Create use-timeline hook and add i18n translation keys | 13def7a | frontend/src/hooks/use-timeline.ts, frontend/messages/en.json, frontend/messages/it.json |

## Artifacts

- **`frontend/src/types/timeline.ts`** — Exports `TimelineSession`, `DebateSummary`, `TimelineResponse`, `PhaseInfo`, `VoteInfo`, `ActInfo`, `SpeakerInfo`, `DebateDetailResponse`, `SpeakerSummaryResponse`, `TimelineFilters`
- **`frontend/src/lib/timeline-api.ts`** — Exports `getTimelineSessions`, `getDebateDetail`, `getSpeakerSummary` — all read NEXT_LOCALE cookie and send Accept-Language header
- **`frontend/src/hooks/use-timeline.ts`** — Exports `useTimeline` with sessions, loading states, infinite scroll trigger, 300ms search debounce, filter management, and `hasActiveFilters` flag

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] React 19 useRef type compatibility**
- **Found during:** Task 2 TypeScript compilation
- **Issue:** `useRef<ReturnType<typeof setTimeout>>()` requires an argument in React 19; `RefObject<HTMLDivElement>` is incompatible with `useRef` return type (now `RefObject<T | null>`)
- **Fix:** Changed `useRef<ReturnType<typeof setTimeout>>()` to `useRef<ReturnType<typeof setTimeout> | undefined>(undefined)`; changed `RefObject<HTMLDivElement>` interface to `RefObject<HTMLDivElement | null>`
- **Files modified:** frontend/src/hooks/use-timeline.ts
- **Commit:** 13def7a (included in same task commit)

## Self-Check: PASSED

Files exist:
- frontend/src/types/timeline.ts: FOUND
- frontend/src/lib/timeline-api.ts: FOUND
- frontend/src/hooks/use-timeline.ts: FOUND

Commits:
- 73d1f41: FOUND
- 13def7a: FOUND
