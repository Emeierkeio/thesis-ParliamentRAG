---
phase: 10-debate-transcript-viewer-with-contextual-chatbot
plan: 06
subsystem: ui
tags: [react, next.js, sse, streaming, chatbot, typescript]

# Dependency graph
requires:
  - phase: 10-debate-transcript-viewer-with-contextual-chatbot
    plan: 03
    provides: debate chatbot SSE streaming backend endpoint
  - phase: 10-debate-transcript-viewer-with-contextual-chatbot
    plan: 04
    provides: TranscriptPanel with targetSpeechId/onTargetConsumed props, SpeechRow, PhaseHeader
  - phase: 10-debate-transcript-viewer-with-contextual-chatbot
    plan: 05
    provides: TranscriptSearch (containerRef pattern), TranscriptMiniMap, in-transcript search

provides:
  - useTranscriptChat hook: SSE streaming hook for debate chatbot with AbortController and multi-turn history
  - TranscriptChatbot: stateless UI shell (desktop panel + mobile FAB/Sheet) receiving chat state as props
  - SelectionAskButton: floating ask button on text selection that pre-fills chatbot
  - Wired transcript page: single hook instance, citation scroll-to-highlight, select-to-ask
  - DebateDetail entry point: Read transcript button linking to /transcript/{debateId}

affects:
  - future phases using transcript viewer
  - timeline page (DebateDetail now has transcript entry point)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Stateless chatbot UI: hook called at page level, props passed down to avoid duplicate SSE connections
    - TranscriptChatbot handles desktop/mobile internally via hidden lg:flex / lg:hidden CSS classes
    - targetSpeechId state bridges chatbot citation clicks to TranscriptPanel scroll-to-highlight
    - prefillText state bridges SelectionAskButton text selection to chatbot input

key-files:
  created:
    - frontend/src/hooks/use-transcript-chat.ts
    - frontend/src/components/transcript/TranscriptChatbot.tsx
    - frontend/src/components/transcript/SelectionAskButton.tsx
  modified:
    - frontend/src/app/transcript/[debateId]/page.tsx
    - frontend/src/components/timeline/DebateDetail.tsx

key-decisions:
  - "TranscriptChatbot is stateless: hook called at page level (single instance) to prevent duplicate SSE connections on desktop+mobile"
  - "TranscriptChatbot desktop div includes lg:w-2/5 border-l bg-card classes (right panel layout) since it sits inside the flex layout as a sibling"
  - "SelectionAskButton uses mouseup event (not selectionchange) and minimum 10-char threshold"
  - "SelectionAskButton uses onMouseDown with preventDefault to prevent selection clearing when clicking the button"

patterns-established:
  - "Single-hook pattern: streaming hooks called at page level, stateless component receives state as props"
  - "Cross-panel state: targetSpeechId (citation -> scroll) and prefillText (selection -> chatbot) live at page level"

requirements-completed: [TR-06, TR-07, TR-08, TR-13, TR-15]

# Metrics
duration: 3min
completed: 2026-04-08
---

# Phase 10 Plan 06: Transcript Chatbot Wiring Summary

**SSE chatbot hook + stateless panel UI + selection-to-ask + citation-scroll wired into complete transcript viewer, with DebateDetail entry point**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-08T13:05:29Z
- **Completed:** 2026-04-08T13:07:42Z
- **Tasks:** 2 (Task 3 is a human-verify checkpoint)
- **Files modified:** 5

## Accomplishments

- Created `useTranscriptChat` hook: fetch + ReadableStream SSE streaming, AbortController abort, multi-turn history, citations/chunk/error event parsing
- Created `TranscriptChatbot` stateless panel: desktop inline panel + mobile FAB + bottom Sheet, citation `[N]` clickable buttons, suggested starter questions, prefillText handling
- Created `SelectionAskButton`: mouseup-based floating button, containment check, min 10-char threshold, `onMouseDown preventDefault` to preserve selection
- Wired `page.tsx`: single `useTranscriptChat` instance, `targetSpeechId` bridges citation clicks to `TranscriptPanel` scroll-to-highlight, `prefillText` bridges selection to chatbot input
- Added "Read transcript" button to `DebateDetail` with `BookOpen` icon, navigating to `/transcript/{debateId}`

## Task Commits

Each task was committed atomically:

1. **Task 1: Create useTranscriptChat hook and TranscriptChatbot panel** - `eccb89a` (feat)
2. **Task 2: Create SelectionAskButton, wire page assembly, and add DebateDetail entry point** - `3bc32ab` (feat)

**Plan metadata:** (pending — human-verify checkpoint reached before final commit)

## Files Created/Modified

- `frontend/src/hooks/use-transcript-chat.ts` - SSE streaming hook for debate chatbot
- `frontend/src/components/transcript/TranscriptChatbot.tsx` - Stateless chatbot UI (desktop + mobile)
- `frontend/src/components/transcript/SelectionAskButton.tsx` - Floating ask button on text selection
- `frontend/src/app/transcript/[debateId]/page.tsx` - Wired page with all components and single hook instance
- `frontend/src/components/timeline/DebateDetail.tsx` - Added Read transcript button

## Decisions Made

- TranscriptChatbot is stateless — hook called at page level (single instance) to prevent duplicate SSE connections when both desktop panel and mobile sheet exist simultaneously
- TranscriptChatbot desktop div has `hidden lg:flex lg:w-2/5 border-l bg-card flex-col h-full` so it acts as the right panel directly inside the flex layout (no wrapper div needed)
- SelectionAskButton uses `mouseup` (not `selectionchange`) per RESEARCH.md Pitfall 4, with `onMouseDown` + `preventDefault` to avoid selection clearing on button click

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## Next Phase Readiness

- Complete transcript viewer awaiting human-verify checkpoint (Task 3): visual testing of full end-to-end flow, citation scroll, selection ask, mobile bottom sheet
- After human approval, Phase 10 Plan 06 is complete and Phase 10 is fully done

---
*Phase: 10-debate-transcript-viewer-with-contextual-chatbot*
*Completed: 2026-04-08*
