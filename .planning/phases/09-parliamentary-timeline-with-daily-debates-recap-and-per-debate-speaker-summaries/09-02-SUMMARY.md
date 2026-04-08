---
phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
plan: 02
subsystem: api
tags: [fastapi, neo4j, pydantic, timeline, pagination, locale]

requires:
  - phase: 09-01
    provides: SpeakerDebateSummary nodes with summaryIt/summaryEn, Session.recapIt/recapEn, Debate.recapIt/recapEn in Neo4j
provides:
  - "GET /api/timeline — paginated session list with nested debate summaries and stats"
  - "GET /api/timeline/debates/{id} — debate detail with phases, speakers, votes, acts"
  - "GET /api/timeline/speakers/{debateId}/{speakerId} — AI speaker position summary"
  - "Pydantic models: SessionCard, TimelineResponse, DebateDetailResponse, SpeakerSummaryResponse"
  - "Timeline service with locale-aware recap selection and cursor-based pagination"
affects: [09-03-frontend-timeline-page]

tech-stack:
  added: []
  patterns:
    - "Thin router pattern: router delegates all logic to timeline_service module"
    - "Locale-aware responses via Accept-Language header (recapIt/recapEn selection)"
    - "Cursor-based pagination via limit+1 fetch and next_cursor ISO date"
    - "coalesce(dep, gov) pattern for Deputy/GovernmentMember speaker queries"
    - "Source-file inspection tests to avoid scipy/NumPy 2.x import chain"

key-files:
  created:
    - backend/app/models/timeline.py
    - backend/app/services/timeline_service.py
    - backend/app/routers/timeline.py
    - backend/tests/test_timeline.py
  modified:
    - backend/app/main.py

key-decisions:
  - "timeline_service.py as flat module (not package) — consistent with deps.py and other services in this codebase"
  - "Search clause uses EXISTS subquery on debate-level, matching titles, recap text, and speaker names per CONTEXT.md locked decision"
  - "Votes queried via session (Debate<-[:HAS_DEBATE]-Session-[:HAS_VOTE]->Vote) — confirmed by Phase 1 schema decision"
  - "Speakers ordered by min(sp.id) as chronological proxy since speech id encodes order"

patterns-established:
  - "Timeline thin router: locale extracted from header, all computation in timeline_service"
  - "Neo4j date → ISO string: str(record['date']) or toString() in Cypher"

requirements-completed: [TL-03, TL-04, TL-05]

duration: 2min
completed: 2026-04-08
---

# Phase 9 Plan 02: Backend Timeline API Summary

**Three FastAPI timeline endpoints with Pydantic models, locale-aware Neo4j service, cursor pagination, and 18 passing smoke tests**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-08T01:48:07Z
- **Completed:** 2026-04-08T01:50:25Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments
- Created `backend/app/models/timeline.py` with 8 Pydantic models covering all three endpoint response shapes
- Created `backend/app/services/timeline_service.py` with three async functions: `get_sessions()`, `get_debate_detail()`, `get_speaker_summary()` — handles locale, cursor pagination, speaker dual-type (Deputy/GovernmentMember), and SpeakerDebateSummary lookup
- Created `backend/app/routers/timeline.py` with three GET endpoints under `/api/timeline` prefix, registered in `main.py`
- Added `backend/tests/test_timeline.py` with 18 smoke tests using source-file inspection pattern (no scipy/NumPy import chain), all green

## Task Commits

Each task was committed atomically:

1. **Task 1: Create Pydantic models and timeline service** - `de28e30` (feat)
2. **Task 2: Create timeline router, register in main.py, add smoke tests** - `d26ed90` (feat)

**Plan metadata:** (docs commit follows)

_Note: Task 1 used TDD — tests written first (RED), then models + service implemented (GREEN)_

## Files Created/Modified
- `backend/app/models/timeline.py` — 8 Pydantic v2 models: SessionCard, DebateSummary, TimelineResponse, PhaseInfo, VoteInfo, ActInfo, SpeakerInfo, DebateDetailResponse, SpeakerSummaryResponse
- `backend/app/services/timeline_service.py` — get_sessions(), get_debate_detail(), get_speaker_summary(); locale-aware, cursor pagination, coalesce speaker pattern
- `backend/app/routers/timeline.py` — thin router with 3 GET endpoints, Accept-Language locale extraction
- `backend/app/main.py` — added timeline_router import and include_router call
- `backend/tests/test_timeline.py` — 18 smoke tests, all pass in ~0.01s without Neo4j

## Decisions Made
- `timeline_service.py` as flat module (not package) — consistent with other services in this codebase (deps.py, experts.py, etc.)
- Search clause uses an EXISTS subquery pattern at the session WHERE level to allow session-level results to match if any debate matches
- Votes linked via session path (Debate←HAS_DEBATE—Session—HAS_VOTE→Vote) per Phase 1 schema decision that votes are on Session, not Debate
- Speakers ordered chronologically by `min(sp.id)` as a proxy since speech IDs encode sequential order

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None — no external service configuration required.

## Next Phase Readiness
- All three timeline API endpoints are functional and registered
- Frontend (Plan 03) can call:
  - `GET /api/timeline` with `before`, `limit`, `chamber`, `search`, `from_date`, `to_date` params and `Accept-Language` header
  - `GET /api/timeline/debates/{id}` for debate detail
  - `GET /api/timeline/speakers/{debateId}/{speakerId}` for speaker summaries
- Endpoints degrade gracefully if Neo4j has no recapIt/recapEn yet (returns null recap fields)

## Self-Check: PASSED

---
*Phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries*
*Completed: 2026-04-08*
