---
phase: 10-debate-transcript-viewer-with-contextual-chatbot
plan: "02"
subsystem: frontend-foundation
tags: [typescript, api-client, i18n, sidebar, transcript]
dependency_graph:
  requires: []
  provides:
    - frontend/src/types/transcript.ts
    - frontend/src/lib/transcript-api.ts
    - Transcript i18n namespace (en + it)
    - Sidebar /transcript active state
  affects:
    - All subsequent transcript viewer plans (10-03 through 10-06)
tech_stack:
  added: []
  patterns:
    - "Types mirror backend Pydantic models (same as timeline.ts pattern)"
    - "API client with cookie-based locale header (same as timeline-api.ts pattern)"
    - "i18n namespace per feature (Transcript sibling to Timeline, Chat, etc.)"
key_files:
  created:
    - frontend/src/types/transcript.ts
    - frontend/src/lib/transcript-api.ts
  modified:
    - frontend/messages/en.json
    - frontend/messages/it.json
    - frontend/src/components/layout/Sidebar.tsx
decisions:
  - "Followed timeline-api.ts pattern exactly: cookie-based locale, buildHeaders(), individual fetch functions"
  - "Transcript namespace placed as sibling to Timeline in both locale files"
  - "Sidebar fix applied to both mobile and desktop NavButton instances"
metrics:
  duration: "6min"
  completed_date: "2026-04-08"
  tasks_completed: 2
  files_modified: 5
---

# Phase 10 Plan 02: Frontend Foundation — Types, API Client, i18n, Sidebar Summary

**One-liner:** TypeScript interfaces mirroring backend transcript models, three-function API client with locale headers, 26-key i18n namespace in both locales, and sidebar active-state fix for /transcript/* routes.

## Tasks Completed

| # | Task | Commit | Files |
|---|------|--------|-------|
| 1 | Create TypeScript types and API client for transcript | 9adac11 | frontend/src/types/transcript.ts, frontend/src/lib/transcript-api.ts |
| 2 | Add i18n keys and fix sidebar active state | 0c93ef2 | frontend/messages/en.json, frontend/messages/it.json, frontend/src/components/layout/Sidebar.tsx |

## What Was Built

### Task 1: TypeScript Types and API Client

`frontend/src/types/transcript.ts` exports six interfaces:
- `TranscriptSpeechRow` — one row in the speech list (id, phase, speaker info, party, role, is_government_member)
- `TranscriptResponse` — full transcript response with debate metadata and speeches array
- `SpeechTextResponse` — single speech full text (speech_id + text)
- `SuggestionsResponse` — list of suggested questions (string[])
- `TranscriptMessage` — chat message for the session-only chatbot (role, content, optional citations)
- `TranscriptCitation` — citation linking a chatbot answer to a specific speech (index, speech_id, speaker info, chunk_text)

`frontend/src/lib/transcript-api.ts` exports three async functions:
- `getTranscriptSpeeches(debateId)` — fetches `/api/transcript/{id}/speeches`
- `getSpeechText(debateId, speechId)` — fetches `/api/transcript/{id}/speech/{speechId}`
- `getDebateSuggestions(debateId)` — fetches `/api/transcript/{id}/suggestions`

All functions use `Accept-Language` header derived from the `NEXT_LOCALE` cookie, following the exact same pattern as `timeline-api.ts`.

### Task 2: i18n and Sidebar

`frontend/messages/en.json` and `frontend/messages/it.json` each gained a `"Transcript"` namespace with 26 keys covering:
- Breadcrumb text (timeline, session date)
- Panel and chat titles/subtitles
- Chat input (placeholder, send, stop, ask-about-this)
- Search (placeholder, match count, no results)
- Speech states (loading, error)
- Empty states (with/without suggestions)
- Copy-link interaction
- Error messages
- Mobile FAB label

`frontend/src/components/layout/Sidebar.tsx` — the `parliamentaryTimeline` NavButton `isActive` condition updated in both the mobile overlay and desktop sidebar sections:
```
Before: isActive: pathname === "/timeline"
After:  isActive: pathname === "/timeline" || pathname.startsWith("/transcript")
```

## Verification Results

- `npx tsc --noEmit` exits with code 0 (no type errors)
- `grep -c "Transcript" messages/en.json` → 3 (namespace key + 2 value references)
- `grep -c "Transcript" messages/it.json` → 2 (namespace key + 1 value reference)
- `grep -c 'startsWith("/transcript")' src/components/layout/Sidebar.tsx` → 2 (mobile + desktop)

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

- [x] `frontend/src/types/transcript.ts` exists with all 6 exported interfaces
- [x] `frontend/src/lib/transcript-api.ts` exists with 3 exported functions + Accept-Language header
- [x] `frontend/messages/en.json` contains Transcript namespace with readTranscript and chatTitle keys
- [x] `frontend/messages/it.json` contains Transcript namespace with readTranscript and chatTitle keys
- [x] `frontend/src/components/layout/Sidebar.tsx` has 2 occurrences of `startsWith("/transcript")`
- [x] Commits 9adac11 and 0c93ef2 exist in git log
