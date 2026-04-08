---
phase: 10-debate-transcript-viewer-with-contextual-chatbot
plan: "01"
subsystem: api
tags: [fastapi, neo4j, pydantic, openai, transcript]

requires:
  - phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
    provides: timeline_service.py patterns (coalesce dep/gov, locale, Neo4jClient DI)

provides:
  - GET /api/transcript/{debate_id}/speeches — all speeches chronologically with speaker metadata
  - GET /api/transcript/{debate_id}/speech/{speech_id} — single speech text (lazy load)
  - GET /api/transcript/{debate_id}/suggestions — LLM-generated starter questions from debate recap
  - Pydantic models: TranscriptSpeechRow, TranscriptResponse, SpeechTextResponse, SuggestionsResponse

affects:
  - 10-02 (frontend transcript viewer consumes these endpoints)
  - future chatbot plans that use /api/transcript/{debate_id}/suggestions

tech-stack:
  added: []
  patterns:
    - "Async service functions following timeline_service.py conventions (Neo4jClient DI, locale param)"
    - "coalesce(dep, gov) Cypher pattern for Deputy/GovernmentMember speaker nodes"
    - "Lazy speech text loading via dedicated /speech/{id} endpoint"
    - "LLM suggestions with Italian recap fallback and hardcoded question fallback"

key-files:
  created:
    - backend/app/models/transcript.py
    - backend/app/services/transcript_service.py
    - backend/app/routers/transcript.py
  modified:
    - backend/app/main.py

key-decisions:
  - "Italian recap (recapIt) used as universal fallback when locale-specific recap is absent in get_debate_suggestions"
  - "gpt-4.1-mini selected for suggestions (consistent with Phase 7 cost optimization)"
  - "Lazy speech text endpoint pattern: transcript viewer fetches speech text on accordion expand, not on page load"

patterns-established:
  - "Transcript service follows identical DI + locale pattern as timeline_service.py"
  - "LLM errors in get_debate_suggestions are swallowed; fallback questions always returned"

requirements-completed: [TR-02, TR-04]

duration: 8min
completed: 2026-04-08
---

# Phase 10 Plan 01: Transcript Backend API Summary

**Three FastAPI transcript endpoints backed by Neo4j queries and gpt-4.1-mini suggestions, following established timeline_service patterns with coalesce(dep, gov) speaker handling**

## Performance

- **Duration:** 8 min
- **Started:** 2026-04-08T12:09:09Z
- **Completed:** 2026-04-08T12:17:24Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- Created Pydantic v2 models for all three transcript API response shapes
- Implemented transcript service with Neo4j queries (chronological ordering, coalesce speaker pattern, lazy speech text)
- Built LLM-generated starter questions with locale support and graceful fallback
- Registered transcript router in FastAPI app following established Phase 9 pattern

## Task Commits

1. **Task 1: Create Pydantic models and transcript service** - `e5eea6c` (feat)
2. **Task 2: Create transcript router and register in main.py** - `2c7e31b` (feat)

## Files Created/Modified

- `backend/app/models/transcript.py` - TranscriptSpeechRow, TranscriptResponse, SpeechTextResponse, SuggestionsResponse
- `backend/app/services/transcript_service.py` - get_transcript_speeches, get_speech_text, get_debate_suggestions
- `backend/app/routers/transcript.py` - Three GET endpoints at /api/transcript prefix
- `backend/app/main.py` - Added transcript_router import and include_router registration

## Decisions Made

- Italian recap (recapIt) used as universal fallback when locale-specific recap is absent in get_debate_suggestions — avoids empty suggestions for debates only indexed in Italian
- gpt-4.1-mini for suggestions (consistent with Phase 7 cost optimization decision)
- Lazy speech text endpoint pattern: transcript viewer will fetch /speech/{id} on accordion expand rather than loading all speech text upfront on page load

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None - source-file inspection used for verification (standard for this anaconda/NumPy 2.x environment, consistent with STATE.md Phase 02-backend decisions).

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Transcript backend API complete and registered — frontend viewer can consume all three endpoints
- Speech text lazy-loading endpoint ready for accordion pattern in plan 10-02
- Suggestions endpoint ready for chatbot starter UI in plan 10-03 or beyond

---
*Phase: 10-debate-transcript-viewer-with-contextual-chatbot*
*Completed: 2026-04-08*
