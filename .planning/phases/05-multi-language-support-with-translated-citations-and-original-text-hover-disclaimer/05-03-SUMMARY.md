---
phase: 05-multi-language-support-with-translated-citations-and-original-text-hover-disclaimer
plan: 03
subsystem: ui
tags: [next-intl, openai, translation, citations, sse, fastapi, react]

# Dependency graph
requires:
  - phase: 05-01-multi-language-support
    provides: translation service (translate_citation_batch), i18n infrastructure (next-intl)
  - phase: 05-02-multi-language-support
    provides: locale message files (en.json, it.json), CitationCard/TranslationBanner translation keys
provides:
  - End-to-end citation translation: Accept-Language header -> backend -> translated_text fields in SSE
  - CitationCard tooltip showing original Italian text on hover when is_translated=true
  - Globe icon on translated citations (card preview + modal)
  - Dismissable TranslationBanner with localStorage persistence
  - Both chat.py pipelines (background + streaming) and query.py pipeline translate citations
affects: [06-senate-data-integration, any future chat pipeline changes]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Accept-Language header propagated from NEXT_LOCALE cookie through fetch to backend
    - ChatRequest.locale field injected by chat_endpoint from http_request (bypasses Pydantic model limitation)
    - request_locale extracted at top of process_query_streaming from http_request Optional[Request]
    - translate_citation_batch called after _build_verified_citations in both chat.py pipelines and query.py
    - CitationCard uses displayText (translated_text ?? text ?? quote_text) and originalText conditional tooltip
    - TranslationBanner checks locale === 'it' + hasCitations + localStorage dismiss key before rendering

key-files:
  created:
    - frontend/src/components/shared/TranslationBanner.tsx
  modified:
    - frontend/src/types/chat.ts
    - frontend/src/hooks/use-chat.ts
    - frontend/src/components/chat/CitationCard.tsx
    - frontend/src/components/chat/ChatArea.tsx
    - backend/app/routers/chat.py
    - backend/app/routers/query.py

key-decisions:
  - "ChatRequest.locale injected by chat_endpoint from http_request.headers (not sent by client in body) — clean separation between transport and business logic"
  - "request_locale read at top of process_query_streaming (not passed as param) since http_request is already an Optional[Request] param"
  - "TranslationBanner starts dismissed=true to avoid hydration flash, then reads localStorage in useEffect"
  - "Both chat.py pipelines (process_chat_background + process_chat_streaming) translate citations — neither pipeline is skipped"

patterns-established:
  - "Pattern: translation injected as post-processing step before SSE emit, keeping pipeline logic unchanged"
  - "Pattern: CitationCard renders displayText (translated or original) with originalText conditional tooltip — zero branching in rendering logic above this layer"

requirements-completed: [ML-03, ML-04, ML-05]

# Metrics
duration: 5min
completed: 2026-04-04
---

# Phase 5 Plan 03: End-to-End Citation Translation Integration Summary

**End-to-end citation translation: Accept-Language header -> OpenAI GPT-4o-mini -> translated_text in SSE, with globe icon tooltip revealing original Italian and dismissable banner disclaimer**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-04T12:51:00Z
- **Completed:** 2026-04-04T12:53:53Z
- **Tasks:** 2 (+ checkpoint)
- **Files modified:** 7

## Accomplishments

- Both chat.py pipelines (background + streaming) and query.py pipeline now call `translate_citation_batch` before emitting `citation_details` when locale is not Italian
- Frontend sends `Accept-Language` header derived from `NEXT_LOCALE` cookie on every chat request; backend reads it and injects into `ChatRequest.locale`
- CitationCard shows translated text with tooltip revealing original Italian on hover, globe icon on translated citations
- TranslationBanner appears above message area for non-Italian locale when citations exist, dismissable with localStorage persistence

## Task Commits

1. **Task 1: Wire translation into backend pipelines and propagate locale from frontend** - `b90f2ee` (feat)
2. **Task 2: Add citation tooltip, globe icon, and translation banner** - `40cd301` (feat)

## Files Created/Modified

- `frontend/src/types/chat.ts` - Added `translated_text`, `translated_full_text`, `is_translated` optional fields to Citation interface
- `frontend/src/hooks/use-chat.ts` - Added `Accept-Language` header read from NEXT_LOCALE cookie in sendMessage fetch call
- `frontend/src/components/chat/CitationCard.tsx` - displayText/originalText logic, Tooltip with Globe icon on translated citations, modal shows original text section
- `frontend/src/components/shared/TranslationBanner.tsx` - Created: dismissable blue banner with Globe icon + localStorage `translationBannerDismissed` key
- `frontend/src/components/chat/ChatArea.tsx` - Import TranslationBanner, derive hasCitations from last assistant message, render banner above message area
- `backend/app/routers/chat.py` - Added `locale` field to ChatRequest, inject from Accept-Language in chat_endpoint, call translate_citation_batch in both pipelines
- `backend/app/routers/query.py` - Extract request_locale from http_request at top of process_query_streaming, call translate_citation_batch before citation_details yield

## Decisions Made

- `ChatRequest.locale` is injected by `chat_endpoint` from `http_request.headers.get("accept-language")` rather than sent by the client in the request body — keeps transport and business logic cleanly separated
- `request_locale` read at top of `process_query_streaming` (not passed as param) since `http_request` is already an `Optional[Request]` parameter in the function signature
- `TranslationBanner` initializes `dismissed=true` to prevent SSR/hydration flash, then reads localStorage in a `useEffect`
- Both `process_chat_background` and `process_chat_streaming` in chat.py translate citations — neither pipeline is skipped

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required beyond OPENAI_API_KEY already configured in Phase 05-01.

## Next Phase Readiness

- Complete multi-language citation translation flow is live: language selector (Phase 02) -> Accept-Language header -> backend translation -> tooltip + banner in UI
- Task 3 (human verification checkpoint) is blocking — requires visual end-to-end verification with both Italian and English locales
- Ready for human verification: start frontend + backend, switch to English locale, send query, confirm translated citations with globe icon and tooltip

---
*Phase: 05-multi-language-support-with-translated-citations-and-original-text-hover-disclaimer*
*Completed: 2026-04-04*
