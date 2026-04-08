---
phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
plan: "04"
subsystem: ui

tags: [react, next.js, typescript, shadcn, lucide-react, next-intl, collapsible, timeline]

# Dependency graph
requires:
  - phase: 09-03
    provides: "TypeScript interfaces (timeline.ts) and API client functions (timeline-api.ts)"

provides:
  - "TimelineSkeleton: 4-card loading placeholder component"
  - "TimelineSearch: keyword input + date presets + date range pickers with i18n"
  - "SpeakerRow: collapsible speaker row with lazy-loaded AI summary, party/role/shield badges"
  - "DebateDetail: lazy-loaded debate view with phases, votes, acts, speakers, and Ask-about-this nav"
  - "SessionCard: top-level collapsible session card with nested per-debate Collapsibles and keyword highlighting"

affects: [09-05, timeline-page]

# Tech tracking
tech-stack:
  added: ["@radix-ui/react-collapsible (already present via shadcn)", "skeleton shadcn component (installed during execution)"]
  patterns:
    - "Lazy-load-on-first-expand: DebateDetail and SpeakerRow fetch data only when Collapsible opens and cache via component state"
    - "WAI-ARIA Disclosure: aria-expanded + aria-controls + role=region on every Collapsible pair"
    - "useId() for stable ARIA IDs per component instance"
    - "highlightText helper: case-insensitive regex split for keyword highlighting via <mark> elements"

key-files:
  created:
    - frontend/src/components/timeline/TimelineSkeleton.tsx
    - frontend/src/components/timeline/TimelineSearch.tsx
    - frontend/src/components/timeline/SpeakerRow.tsx
    - frontend/src/components/timeline/DebateDetail.tsx
    - frontend/src/components/timeline/SessionCard.tsx
    - frontend/src/components/ui/skeleton.tsx
  modified: []

key-decisions:
  - "skeleton.tsx installed via npx shadcn add skeleton (was missing from ui components)"
  - "DebateDetail uses useEffect on mount (not lazy-on-expand) since it is already rendered inside a parent Collapsible"
  - "DebateRow inner component inside SessionCard handles per-debate expand and conditionally renders DebateDetail only when open"
  - "SpeakerRow caches summary result in component state; re-expand never re-fetches (idle/loading/loaded/error union)"

patterns-established:
  - "Lazy-load-on-first-expand pattern: fetch in onOpenChange, cache in local state with 4-state union"
  - "Inner component pattern: DebateRow defined in SessionCard module for per-debate collapsible to avoid prop drilling"
  - "highlightText(text, term): returns ReactNode using regex split — safe for all special characters via escape"

requirements-completed: [TL-06, TL-08]

# Metrics
duration: 15min
completed: 2026-04-08
---

# Phase 09 Plan 04: Timeline UI Components Summary

**Five React timeline components with collapsible drill-down, lazy API fetching, WAI-ARIA Disclosure, i18n, and keyword search highlighting**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-04-08T01:55:28Z
- **Completed:** 2026-04-08T01:58:30Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Five timeline components in `frontend/src/components/timeline/` with full TypeScript types
- Collapsible session/debate/speaker hierarchy with WAI-ARIA Disclosure pattern on every level
- Lazy-load pattern: SpeakerRow fetches `getSpeakerSummary`, DebateDetail fetches `getDebateDetail` only on first expand with local caching
- Keyword highlighting via `highlightText` helper applied to session recap and debate titles
- TimelineSearch with 3 date presets (7/30/90 days), native date range inputs, active preset tracking, and clear functionality

## Task Commits

Each task was committed atomically:

1. **Task 1: Create TimelineSkeleton, TimelineSearch, and SpeakerRow** - `042585e` (feat)
2. **Task 2: Create DebateDetail and SessionCard** - `e129487` (feat)

## Files Created/Modified
- `frontend/src/components/timeline/TimelineSkeleton.tsx` - 4 shimmer placeholder cards matching SessionCard layout
- `frontend/src/components/timeline/TimelineSearch.tsx` - Search + date presets + date pickers with i18n and accessible labels
- `frontend/src/components/timeline/SpeakerRow.tsx` - Collapsible speaker row with lazy AI summary, party/role/shield badges, /ranking link
- `frontend/src/components/timeline/DebateDetail.tsx` - Full debate detail: recap, phases, votes collapsible, acts badges, speakers list, Ask-about-this button
- `frontend/src/components/timeline/SessionCard.tsx` - Top-level session card with nested per-debate Collapsibles and highlightText
- `frontend/src/components/ui/skeleton.tsx` - Skeleton shadcn component (installed during execution as it was missing)

## Decisions Made
- `skeleton.tsx` installed via `npx shadcn add skeleton` (was not in UI components despite plan's prerequisite check instructing this)
- `DebateDetail` triggers fetch in `useEffect` on mount (not in `onOpenChange`) since it is already rendered conditionally inside `DebateRow`'s `CollapsibleContent` — avoids double-render race
- `DebateRow` defined as inner component within `SessionCard.tsx` module to handle per-debate expand state without cluttering the session component props
- Summary state uses a discriminated union (`idle | loading | loaded | error`) for type-safe render branches

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Installed missing skeleton shadcn component**
- **Found during:** Task 1 (TimelineSkeleton creation)
- **Issue:** `frontend/src/components/ui/skeleton.tsx` did not exist; plan's prerequisite check specified this case
- **Fix:** Ran `cd frontend && npx shadcn add skeleton` before creating components
- **Files modified:** `frontend/src/components/ui/skeleton.tsx`
- **Verification:** File created, `export { Skeleton }` confirmed, TypeScript check passes
- **Committed in:** `042585e` (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking — missing dependency)
**Impact on plan:** The skeleton installation was explicitly anticipated as a prerequisite in the plan. No scope creep.

## Issues Encountered
None - all components compiled cleanly with zero TypeScript errors.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All 5 timeline components ready for composition in Plan 05 (timeline page)
- Components import cleanly from `@/types/timeline` and `@/lib/timeline-api` (built in Plan 03)
- i18n keys used match those already defined in `messages/en.json` and `messages/it.json`
- No blockers

---
*Phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries*
*Completed: 2026-04-08*
