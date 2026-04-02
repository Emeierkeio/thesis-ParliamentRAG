---
phase: 03-frontend
plan: 02
subsystem: ui
tags: [nextjs, typescript, react, routes, barrel-exports, dead-code]

# Dependency graph
requires:
  - phase: 03-frontend-01
    provides: strict TypeScript types (BalanceMetrics, Message.committeeMatches) modified in this plan
provides:
  - English route paths /evaluation, /rankings, /explore with client-side redirects from old paths
  - Clean English-only code identifiers throughout frontend
  - No dead code (formatEventForFrontend, formatExpert, formatCitation deleted)
  - Barrel index.ts exports for evaluation, graph, search, settings, ui component folders
affects: [03-frontend-03, any future plans referencing BalanceMetrics or route paths]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Route rename pattern: copy page content to new path, replace old page with router.replace redirect stub"
    - "Wire-format vs local identifiers: SSE field names stay Italian (maggioranza_percentage), local TypeScript identifiers become English"
    - "CoalitionFilter maps local majority/opposition to wire maggioranza/opposizione in filter predicate"

key-files:
  created:
    - frontend/src/app/evaluation/page.tsx
    - frontend/src/app/rankings/page.tsx
    - frontend/src/app/explore/page.tsx
    - frontend/src/components/evaluation/index.ts
    - frontend/src/components/graph/index.ts
    - frontend/src/components/search/index.ts
    - frontend/src/components/settings/index.ts
    - frontend/src/components/ui/index.ts
  modified:
    - frontend/src/types/chat.ts
    - frontend/src/hooks/use-chat.ts
    - frontend/src/app/api/chat/route.ts
    - frontend/src/config/index.ts
    - frontend/src/lib/api.ts
    - frontend/src/lib/graph-api.ts
    - frontend/src/app/ranking/page.tsx (now redirect stub)
    - frontend/src/app/valutazione/page.tsx (now redirect stub)
    - frontend/src/app/explorer/page.tsx (now redirect stub)
    - frontend/src/components/layout/Sidebar.tsx
    - frontend/src/components/chat/MessageBubble.tsx
    - frontend/src/types/survey.ts

key-decisions:
  - "Wire-format SSE values (maggioranza_percentage, opposizione_percentage, commissioni as wire field) preserved as-is; only local TypeScript identifiers translated to English"
  - "CoalitionFilter type uses English values (majority/opposition); filter predicate maps to wire values maggioranza/opposizione"
  - "ChatHistoryItem.commissioni kept as-is (backend API wire format field returned from /history endpoint)"

patterns-established:
  - "Redirect stubs: old route page replaced with useEffect + router.replace, no SSR redirect"
  - "Barrel export per component folder: exact named exports mirrored from source files"

requirements-completed: [FE-02, FE-03]

# Metrics
duration: 10min
completed: 2026-04-02
---

# Phase 3 Plan 02: Italian-to-English Cleanup and Route Renames Summary

**English route structure (/evaluation, /rankings, /explore) with client-side redirects, all Italian code identifiers translated, dead formatEventForFrontend/formatExpert/formatCitation functions deleted, and 5 barrel index.ts files added**

## Performance

- **Duration:** ~10 min
- **Started:** 2026-04-02T18:24:00Z
- **Completed:** 2026-04-02T18:32:39Z
- **Tasks:** 2
- **Files modified:** 20 (12 modified, 8 created)

## Accomplishments
- Renamed /valutazione -> /evaluation, /ranking -> /rankings, /explorer -> /explore with client-side redirect stubs on old paths
- Translated all Italian code identifiers: BalanceMetrics properties (maggioranzaPercentage -> majorityPercentage, opposizionePercentage -> oppositionPercentage), Message.commissioni -> committeeMatches, local variables commList/topComm/commResult/magg/opp
- Deleted 3 dead functions from route.ts (formatEventForFrontend, formatExpert, formatCitation — ~110 lines), removed addUserMessage from use-chat.ts, removed 3 empty dead blocks
- Created barrel index.ts for all 5 missing component folders: evaluation, graph, search, settings, ui
- tsc --noEmit exits 0, zero any types (preserved from Plan 01)

## Task Commits

1. **Task 1: Rename Italian code identifiers, translate comments, remove dead code** - `427d448` (feat)
2. **Task 2: Rename routes with redirects and add barrel exports** - `4ce001d` (feat)

**Plan metadata:** (created in this summary commit)

## Files Created/Modified
- `frontend/src/app/evaluation/page.tsx` - Full evaluation dashboard (moved from valutazione)
- `frontend/src/app/rankings/page.tsx` - Full rankings page (moved from ranking)
- `frontend/src/app/explore/page.tsx` - Full graph explorer page (moved from explorer)
- `frontend/src/app/valutazione/page.tsx` - Client-side redirect stub to /evaluation
- `frontend/src/app/ranking/page.tsx` - Client-side redirect stub to /rankings
- `frontend/src/app/explorer/page.tsx` - Client-side redirect stub to /explore
- `frontend/src/components/evaluation/index.ts` - Barrel export for EvaluationCharts functions
- `frontend/src/components/graph/index.ts` - Barrel export for GraphVisualizer
- `frontend/src/components/search/index.ts` - Barrel export for search components
- `frontend/src/components/settings/index.ts` - Barrel export for SettingsModal and GraphicalEditors
- `frontend/src/components/ui/index.ts` - Barrel export for all 15 shadcn/ui primitives
- `frontend/src/types/chat.ts` - BalanceMetrics renamed, Message.commissioni -> committeeMatches
- `frontend/src/hooks/use-chat.ts` - Variable renames, dead code removed, comments translated
- `frontend/src/app/api/chat/route.ts` - Dead functions deleted, comments translated
- `frontend/src/components/chat/MessageBubble.tsx` - Updated BalanceMetrics property names
- `frontend/src/components/layout/Sidebar.tsx` - /ranking -> /rankings (2 occurrences)
- `frontend/src/config/index.ts` - Docstring translated
- `frontend/src/lib/api.ts` - Error message translated
- `frontend/src/lib/graph-api.ts` - Error message translated
- `frontend/src/types/survey.ts` - Comment translated

## Decisions Made
- Wire-format SSE values preserved: `maggioranza_percentage`, `opposizione_percentage`, `commissioni` as SSE type name, and `"maggioranza"`/`"opposizione"` coalition string comparisons against wire data all stay Italian because they are frozen backend contract values
- CoalitionFilter local type changed from `"maggioranza"/"opposizione"` to `"majority"/"opposition"`; filter predicate maps to wire values
- `ChatHistoryItem.commissioni` kept Italian — it is a backend API response field, not a local identifier

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] MessageBubble.tsx also referenced old BalanceMetrics property names**
- **Found during:** Task 1 (BalanceMetrics rename)
- **Issue:** Plan listed 14 cascading references but did not explicitly name MessageBubble.tsx; tsc caught 5 errors in that file
- **Fix:** Updated 3 references to opposizionePercentage and 2 to maggioranzaPercentage in MessageBubble.tsx
- **Files modified:** frontend/src/components/chat/MessageBubble.tsx
- **Verification:** tsc --noEmit exits 0
- **Committed in:** 427d448 (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (Rule 1 - cascading rename)
**Impact on plan:** Required fix — plan noted ~14 cascading references; tsc correctly caught the remaining ones.

## Issues Encountered
None beyond the MessageBubble.tsx cascade (handled above).

## Next Phase Readiness
- FE-02 and FE-03 complete: no dead code, barrel exports in all 9 component folders, English routes and identifiers
- Plan 03 (if exists) can proceed — all type renames and route changes are committed

---
*Phase: 03-frontend*
*Completed: 2026-04-02*
