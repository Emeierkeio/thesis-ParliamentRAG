---
phase: 03-frontend
plan: 01
subsystem: frontend/types
tags: [typescript, strict-mode, sse-types, type-safety]
dependency_graph:
  requires: []
  provides: [SSE discriminated union types, ChatHistoryItem interface, typed history callback, zero-any codebase]
  affects: [frontend/src/types, frontend/src/hooks, frontend/src/components, frontend/src/app]
tech_stack:
  added: []
  patterns: [discriminated union types, Record<string,unknown> for polymorphic details, runtime narrowing for unknown, double-cast via unknown for structural incompatibilities]
key_files:
  created:
    - frontend/src/types/sse.ts
  modified:
    - frontend/src/types/chat.ts
    - frontend/src/types/survey.ts
    - frontend/src/types/api.ts
    - frontend/src/hooks/use-chat.ts
    - frontend/src/lib/api.ts
    - frontend/src/components/chat/CompassCard.tsx
    - frontend/src/components/chat/MessageBubble.tsx
    - frontend/src/components/graph/GraphVisualizer.tsx
    - frontend/src/components/shared/HistoryModal.tsx
    - frontend/src/components/shared/ProgressIndicator.tsx
    - frontend/src/components/settings/SettingsModal.tsx
    - frontend/src/components/survey/SurveyModal.tsx
    - frontend/src/app/ranking/page.tsx
    - frontend/src/app/explorer/page.tsx
    - frontend/src/app/compass/page.tsx
    - frontend/src/app/valutazione/page.tsx
decisions:
  - StepResult.details typed as Record<string,unknown> (not a union) — simpler and sufficient since consumers do runtime narrowing anyway
  - GraphRecord typed as Record<string,unknown> — avoids complex index signature union; internal string extraction uses strProp helper
  - fgRef typed as MutableRefObject<ForceGraphMethods|undefined> — library expects undefined not null
  - mapRawToConfig(raw: unknown) uses explicit cast per field — avoids any while preserving exact field mapping
  - BalanceMetrics passed as details via double-cast (as unknown as Record<string,unknown>) — BalanceMetrics lacks index signature
metrics:
  duration: 70min
  completed: 2026-04-02T18:22:38Z
  tasks_completed: 2
  files_modified: 17
---

# Phase 3 Plan 1: TypeScript Strict Mode — Eliminate all any Summary

Zero `any` occurrences across all 15 frontend source files; full SSE discriminated union created from frozen SSE_CONTRACT.md.

## What Was Built

Created `frontend/src/types/sse.ts` with a complete SSE event discriminated union (17 interfaces + `SSEEvent` union type) derived from `SSE_CONTRACT.md`. Updated 4 type files and 13 consumer files to replace every `any` occurrence with proper TypeScript types. `tsc --noEmit` now compiles with zero errors.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create SSE type interfaces and fix type-level any in type files | 62dc52a | sse.ts (new), chat.ts, survey.ts, api.ts |
| 2 | Eliminate all any from hooks, components, and page files | d179329 | use-chat.ts, api.ts, CompassCard.tsx, MessageBubble.tsx, GraphVisualizer.tsx, HistoryModal.tsx, ProgressIndicator.tsx, SettingsModal.tsx, SurveyModal.tsx, ranking/page.tsx, explorer/page.tsx, compass/page.tsx, valutazione/page.tsx |

## Key Design Decisions

1. **`StepResult.details?: Record<string, unknown>`** — The plan suggested a union type `StepResultDetails` but the union couldn't satisfy `Record<string, unknown>` without an index signature on each member. Collapsed to `Record<string, unknown>` with runtime narrowing in consumers (Rule 1 fix).

2. **`GraphRecord = Record<string, unknown>`** — The plan suggested a typed interface with string/number/boolean fields. The switch-based caption logic accesses many optional string props; using `Record<string, unknown>` with a `strProp()` helper function is safer and avoids widening the index signature.

3. **`mapRawToConfig` runtime narrowing** — `raw: unknown` with explicit field-by-field extraction rather than a single cast, matching the plan's intent.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Citation interface needed `institutional_role` field**
- **Found during:** Task 1 (tsc check)
- **Issue:** CitationCard.tsx and MessageBubble.tsx accessed `citation.institutional_role` which was missing after removing the `[key: string]: any` index signature
- **Fix:** Added `institutional_role?: string | null` to Citation interface
- **Files modified:** frontend/src/types/chat.ts

**2. [Rule 1 - Bug] BalanceMetrics not assignable to Record<string,unknown>**
- **Found during:** Task 2 (tsc check)
- **Issue:** `BalanceMetrics` interface lacks an index signature, cannot be passed as `details` in StepResult without cast
- **Fix:** Used `as unknown as Record<string, unknown>` at the single assignment site in use-chat.ts
- **Files modified:** frontend/src/hooks/use-chat.ts

**3. [Rule 1 - Bug] fgRef type mismatch with ForceGraph2D**
- **Found during:** Task 2 (tsc check)
- **Issue:** Library expects `MutableRefObject<ForceGraphMethods | undefined>` not `RefObject<ForceGraphMethods | null>`
- **Fix:** Changed ref initialization to `useRef<ForceGraphMethods | undefined>(undefined) as MutableRefObject<...>`
- **Files modified:** frontend/src/components/graph/GraphVisualizer.tsx

**4. [Rule 1 - Bug] SurveyResponse not directly castable to Record<string, ABRating>**
- **Found during:** Task 2 (tsc check)
- **Issue:** TypeScript requires double-cast via unknown for structurally incompatible types
- **Fix:** `(item.human as unknown as Record<string, ABRating | undefined>)`
- **Files modified:** frontend/src/app/valutazione/page.tsx

## Verification Results

```
tsc --noEmit: 0 errors
grep ": any" src/: 0 matches
grep "as any" src/: 0 matches
frontend/src/types/sse.ts: exists, exports SSEEvent union
```

## Self-Check: PASSED
