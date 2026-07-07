---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: 07
subsystem: ui
tags: [next-intl, react, typescript, votes, compass, rankings, pca, rice-cohesion]

# Dependency graph
requires:
  - phase: 14-05
    provides: "votes-api.ts client (getVoteCompass, getVoteCohesion) + VoteCompass/Cohesion i18n keys"
  - phase: 14-02
    provides: "GET /api/compass/votes backend endpoint"
  - phase: 14-03
    provides: "GET /api/rankings/votes cohesion endpoint"
provides:
  - "Compass page segmented 'Assi testuali | Assi di voto' toggle (F2)"
  - "Vote compass SVG scatter with party-colored dots and variance explained"
  - "Rankings page 'Metriche di voto' toggle revealing Rice cohesion column (F3)"
  - "Graceful degradation: vote columns hidden (not zeroed) when data unavailable"
affects: [14-08, vote-intelligence-frontend]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "axisMode state ('text' | 'vote') pattern for toggling between text and vote-derived visualizations"
    - "riceByParty Map (uppercase party → rice) for O(1) deputy-to-cohesion lookup"
    - "riceAvailable gate (available:true AND data present) prevents rendering empty vote columns"

key-files:
  created: []
  modified:
    - frontend/src/app/compass/page.tsx
    - frontend/src/app/rankings/page.tsx

key-decisions:
  - "VoteCompassScatter uses config.politicalGroups for party dot colors — mixed-case Neo4j party names match the mixed-case config keys added in earlier phases"
  - "fetchVoteCompass defaults to legislature=19, chamber='camera' — compass page has no chamber selector; 'both' invalid for vote compass per Pitfall 1"
  - "riceByParty lookup normalizes to uppercase — deputy.group from backend is uppercase; cohesion.parties from Neo4j is mixed-case, so both normalized to uppercase for matching"
  - "Vote columns hidden entirely (not zeroed) when cohesion.available===false — Pitfall 3 guard maintained"
  - "handleReset and restoreCompassEntry both reset axisMode to 'text' — prevents stale vote mode state on new text searches"

patterns-established:
  - "Segmented toggle (text | vote) added to existing controls bar as shrink-0 element — minimal layout disruption"
  - "riceAvailable derived from cohesion state: available===true AND parties.length>0 — single source of truth for column visibility"

requirements-completed: [VI-02, VI-03]

# Metrics
duration: 4min
completed: 2026-07-07
---

# Phase 14 Plan 07: Vote Compass Toggle + Rankings Cohesion Summary

**Vote-axis compass toggle (F2) and Rice cohesion rankings column (F3) wired to shared votes-api.ts client with graceful degradation when individual-vote data is absent**

## Performance

- **Duration:** 4 min
- **Started:** 2026-07-07T00:04:16Z
- **Completed:** 2026-07-07T00:08:48Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- Compass page gains a segmented "Assi testuali | Assi di voto" toggle in the controls bar; switching to vote mode fetches `/api/compass/votes` and renders a party-colored SVG scatter with variance explained footer
- Rankings page gains a "Metriche di voto" toggle button in the filter bar; when enabled it fetches `/api/rankings/votes` and adds a Rice cohesion column to the desktop grid (header + per-row percentage)
- Both features degrade cleanly: vote columns/scatter hidden entirely (not zeroed) when `available === false`; unavailable note shown to user

## Task Commits

1. **Task 1: F2 — vote/text axis toggle on compass page** - `6d594e5` (feat)
2. **Task 2: F3 — cohesion columns/toggle on rankings page** - `bd6d3c5` (feat)

## Files Created/Modified
- `frontend/src/app/compass/page.tsx` — Added axisMode state, segmented toggle, VoteCompassScatter SVG component, handleAxisModeChange callback
- `frontend/src/app/rankings/page.tsx` — Added showVoteMetrics state, Cohesion translation, vote metrics toggle button, riceByParty lookup, conditional Rice column in header and RankingRow

## Decisions Made
- VoteCompassScatter renders inside compass/page.tsx (not a separate component file) — keeps the feature self-contained and avoids an extra file for a single-use component
- riceByParty lookup normalizes both sides to uppercase for reliable party name matching between backend wire format and cohesion response
- Vote compass defaults to legislature=19, chamber="camera" — no chamber selector on compass page; "both" is invalid (Pitfall 1)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None - TypeScript and ESLint both clean (zero errors, only pre-existing warnings).

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- F2 and F3 complete; both consume the shared votes-api.ts client and i18n keys from 14-05
- Ready for 14-08 (vote explorer page) which builds on the same votes-api.ts infrastructure

---
*Phase: 14-vote-intelligence*
*Completed: 2026-07-07*
