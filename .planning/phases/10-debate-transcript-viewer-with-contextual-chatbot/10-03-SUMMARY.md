---
phase: 10-debate-transcript-viewer-with-contextual-chatbot
plan: 03
subsystem: api
tags: [fastapi, openai, sse, neo4j, vector-search, rag, streaming]

# Dependency graph
requires:
  - phase: 10-debate-transcript-viewer-with-contextual-chatbot
    plan: 01
    provides: transcript_service.py foundation and transcript router with 3 GET endpoints

provides:
  - POST /api/transcript/{debate_id}/chat SSE endpoint with debate-scoped RAG
  - debate_chat_streaming async generator in transcript_service.py
  - TranscriptChatRequest and TranscriptChatMessage Pydantic models

affects:
  - 10-04 (frontend chatbot panel consumes this SSE endpoint)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Debate-scoped vector search: top_k=100 over full index, then debate_id filter in MATCH clause, LIMIT 15 final results
    - SSE generator as async generator function (no pipeline semaphore — independent of main query pipeline)
    - Citations deduplication by speech_id with sequential [1], [2] index numbers before text generation

key-files:
  created: []
  modified:
    - backend/app/models/transcript.py
    - backend/app/services/transcript_service.py
    - backend/app/routers/transcript.py

key-decisions:
  - "debate_chat_streaming runs independently of _pipeline_semaphore — debate chat is per-debate scope, not global pipeline"
  - "top_k=100 for vector index query to ensure enough debate-specific chunks survive the debate_id MATCH filter, then LIMIT 15 final"
  - "Citations yielded as SSE event before text streaming begins — frontend can render citation panel immediately"
  - "Locale derived from Accept-Language header (same pattern as other transcript endpoints)"

patterns-established:
  - "Debate-scoped RAG: CALL db.index.vector.queryNodes followed by MATCH with debate_id filter"
  - "SSE generator: progress -> citations -> chunk stream -> done (no pipeline semaphore)"

requirements-completed: [TR-03]

# Metrics
duration: 5min
completed: 2026-04-08
---

# Phase 10 Plan 03: Debate Chat SSE Endpoint Summary

**Debate-scoped RAG chatbot via POST /api/transcript/{debate_id}/chat streaming SSE — embeds query, retrieves debate-only chunks from Neo4j vector index, generates answer with speech citations using gpt-4.1-mini**

## Performance

- **Duration:** 5 min
- **Started:** 2026-04-08T12:18:00Z
- **Completed:** 2026-04-08T12:21:38Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Added TranscriptChatMessage and TranscriptChatRequest Pydantic models to models/transcript.py
- Implemented debate_chat_streaming async generator in transcript_service.py with full RAG pipeline: embed -> vector search with debate_id filter -> citations -> streaming generation
- Registered POST /{debate_id}/chat endpoint in transcript router returning SSE StreamingResponse

## Task Commits

Each task was committed atomically:

1. **Task 1: Add TranscriptChatRequest model and debate_chat_streaming service function** - `e65114e` (feat)
2. **Task 2: Add POST chat endpoint to transcript router** - `d79de75` (feat)

## Files Created/Modified
- `backend/app/models/transcript.py` - Added TranscriptChatMessage and TranscriptChatRequest models
- `backend/app/services/transcript_service.py` - Added debate_chat_streaming async generator with full debate-scoped RAG pipeline
- `backend/app/routers/transcript.py` - Added StreamingResponse import, TranscriptChatRequest import, and POST /{debate_id}/chat endpoint

## Decisions Made
- debate_chat_streaming runs independently of _pipeline_semaphore — per-debate scope is isolated from the global query pipeline
- top_k=100 for vector index to ensure enough debate-specific chunks survive the debate_id MATCH filter, then LIMIT 15 final results
- Citations yielded as dedicated SSE event before text streaming — frontend can render citation panel without waiting for full answer
- Locale derived from Accept-Language header, consistent with the three existing GET endpoints in the router

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- NumPy 2.x incompatibility in anaconda environment prevented live Python import verification. Used source-file AST inspection instead (established pattern from STATE.md decisions). All acceptance criteria verified via AST parse and string search.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- POST /api/transcript/{debate_id}/chat endpoint is ready for frontend integration
- Frontend chatbot panel (Plan 04 or later) can POST to this endpoint with query + history array
- SSE events: progress, citations (array with speech_id + speaker + party + chunk_text[:200]), chunk, done, error
- No blockers

---
*Phase: 10-debate-transcript-viewer-with-contextual-chatbot*
*Completed: 2026-04-08*
