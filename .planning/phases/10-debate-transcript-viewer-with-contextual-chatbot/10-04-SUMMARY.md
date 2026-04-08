---
phase: 10-debate-transcript-viewer-with-contextual-chatbot
plan: "04"
subsystem: ui
tags: [nextjs, react, collapsible, accordion, transcript, deep-link, lazy-load, shadcn]

# Dependency graph
requires:
  - phase: 10-02
    provides: transcript-api.ts (getTranscriptSpeeches, getSpeechText), transcript types

provides:
  - /transcript/[debateId] dynamic route page with header, breadcrumb, and 60/40 two-panel layout
  - TranscriptPanel component with deep-link support and targetSpeechId citation wiring
  - SpeechRow accordion with lazy text loading, party/role/government badges, copy-link
  - PhaseHeader phase dividers with data-phase-id attribute for TranscriptMiniMap IntersectionObserver
  - highlight-pulse CSS keyframe animation for citation scroll feedback

affects:
  - 10-05 (TranscriptMiniMap uses PhaseHeader data-phase-id and TranscriptPanel in page)
  - 10-06 (TranscriptChatbot wires into targetSpeechId/onTargetConsumed props in TranscriptPanel)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Lazy accordion text loading: fetch on first open, idle/loading/loaded/error state machine
    - Deep-link via window.location.hash read after speeches array is populated
    - targetSpeechId/onTargetConsumed prop pair for parent-to-panel scroll commands
    - highlight-pulse CSS animation via classList.add/remove for citation feedback

key-files:
  created:
    - frontend/src/app/transcript/[debateId]/page.tsx
    - frontend/src/components/transcript/TranscriptPanel.tsx
    - frontend/src/components/transcript/SpeechRow.tsx
    - frontend/src/components/transcript/PhaseHeader.tsx
  modified:
    - frontend/src/app/globals.css

key-decisions:
  - "page.tsx, TranscriptPanel.tsx, globals.css already committed in Plan 05 scaffolding commit (823a603) — no duplicate commit needed for Task 1 files"
  - "SpeechRow uses Collapsible controlled mode (isOpen/onOpenChange props) — TranscriptPanel manages open state centrally so only one speech is open at a time"
  - "PhaseHeader includes data-phase-id from the start to avoid cross-plan file conflicts with Plan 05 MiniMap"
  - "TranscriptPanel includes targetSpeechId/onTargetConsumed from the start to avoid cross-plan file conflicts with Plan 06 chatbot"

patterns-established:
  - "Transcript component controlled accordion: parent holds openSpeechId, children receive isOpen+onOpenChange"
  - "Deep-link pattern: useEffect on speechesLoaded flag, hash read from window.location.hash"

requirements-completed: [TR-01, TR-05, TR-10, TR-11]

# Metrics
duration: 2min
completed: 2026-04-08
---

# Phase 10 Plan 04: Transcript Page Route and Core Viewer Components Summary

**Lazy-loading SpeechRow accordion and PhaseHeader dividers with deep-link, copy-link, and chatbot citation wiring hooks for the two-panel transcript viewer**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-08T12:20:53Z
- **Completed:** 2026-04-08T12:22:30Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- SpeechRow accordion lazy-loads speech text via getSpeechText on first expand, with idle/loading/loaded/error state machine and retry on error
- PhaseHeader renders institutional blue accent dividers with data-phase-id attribute pre-wired for Plan 05 MiniMap IntersectionObserver
- TranscriptPanel manages centralized open state, deep-link hash reading after speeches load, and targetSpeechId/onTargetConsumed props pre-wired for Plan 06 chatbot citation scrolling

## Task Commits

Each task was committed atomically:

1. **Task 1: Create page route, TranscriptPanel, and highlight-pulse CSS** - `823a603` (feat - committed by Plan 05 scaffolding)
2. **Task 2: Create SpeechRow accordion and PhaseHeader components** - `39e4a43` (feat)

**Plan metadata:** (docs commit below)

## Files Created/Modified

- `frontend/src/app/transcript/[debateId]/page.tsx` - Dynamic route page with debate header, breadcrumb, and 60/40 two-panel layout
- `frontend/src/components/transcript/TranscriptPanel.tsx` - Left panel container with phase dividers, deep-link, and targetSpeechId prop for chatbot wiring
- `frontend/src/components/transcript/SpeechRow.tsx` - Per-speech collapsible with lazy text loading, badges, copy-link button
- `frontend/src/components/transcript/PhaseHeader.tsx` - Phase divider with border-l-[3px] border-primary accent and data-phase-id attribute
- `frontend/src/app/globals.css` - Added highlight-pulse @keyframes animation for citation scroll feedback

## Decisions Made

- `page.tsx`, `TranscriptPanel.tsx`, and `globals.css` were already committed in the Plan 05 scaffolding commit (823a603). Plan 05 ran before Plan 04 (plans executed out of sequence) and created these files as scaffolding to support the MiniMap integration. The content matches Plan 04's specification exactly, so no duplicate commit was needed.
- SpeechRow uses Collapsible in controlled mode (isOpen/onOpenChange props) — TranscriptPanel manages openSpeechId centrally so only one speech is expanded at a time (accordion behavior).
- PhaseHeader includes data-phase-id from the start to avoid cross-plan file conflicts with Plan 05 TranscriptMiniMap.
- TranscriptPanel includes targetSpeechId/onTargetConsumed from the start to avoid cross-plan file conflicts with Plan 06 chatbot.

## Deviations from Plan

None — plan executed exactly as written. The only notable observation is that Task 1 files (page.tsx, TranscriptPanel.tsx, globals.css) were already correctly committed by the Plan 05 agent which ran before Plan 04. The content matched the Plan 04 specification, so no re-commit was necessary.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- TranscriptPanel, SpeechRow, and PhaseHeader are ready for Plan 05 TranscriptMiniMap integration (data-phase-id attribute already in PhaseHeader DOM)
- TranscriptPanel targetSpeechId/onTargetConsumed props ready for Plan 06 chatbot citation click wiring
- TypeScript compiles with zero errors across all transcript components

---
*Phase: 10-debate-transcript-viewer-with-contextual-chatbot*
*Completed: 2026-04-08*
