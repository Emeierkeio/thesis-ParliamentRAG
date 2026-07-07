---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: "05"
subsystem: frontend
tags: [votes, explorer, types, i18n, sidebar, api-client]
dependency_graph:
  requires: ["14-02", "14-03"]
  provides: ["votes-types", "votes-api-client", "phase14-i18n", "votes-explorer-page", "sidebar-votes-entry"]
  affects: ["14-06", "14-07"]
tech_stack:
  added: []
  patterns: ["hairline-row table", "filter bar", "pagination", "barrel re-export", "client-side i18n"]
key_files:
  created:
    - frontend/src/types/votes.ts
    - frontend/src/lib/votes-api.ts
    - frontend/src/app/votes/page.tsx
  modified:
    - frontend/src/types/index.ts
    - frontend/src/components/layout/Sidebar.tsx
    - frontend/messages/it.json
    - frontend/messages/en.json
decisions:
  - "votes-api.ts uses config.api.baseUrl (not bare /api path) to support NEXT_PUBLIC_API_URL override"
  - "Vote icon (lucide-react) used for sidebar entry — confirmed available in installed version"
  - "Chamber selector defaults to 'both' on the votes page (unlike vote compass that defaults to 'camera')"
  - "useEffect dependency array uses eslint-disable-line comment to avoid stale closure issue on filter change re-fetch"
metrics:
  duration_minutes: 4
  completed_date: "2026-07-07"
  tasks_completed: 3
  files_changed: 7
---

# Phase 14 Plan 05: Votes Explorer + Shared API Client + i18n + Type Barrel Summary

Votes explorer page (F5) with filterable paginated hairline-row table, shared TypeScript API client for all three Phase-14 vote endpoints, complete i18n for all five Phase-14 namespaces, and sidebar Votazioni entry.

## Tasks Completed

| # | Task | Commit | Files |
|---|------|--------|-------|
| 1 | types/votes.ts + barrel re-export + votes-api.ts + Phase-14 i18n | d613a0a | votes.ts, index.ts, votes-api.ts, it.json, en.json |
| 2 | /votes explorer page — filter bar + hairline-row table | e803e4e | src/app/votes/page.tsx |
| 3 | Sidebar "Votazioni" entry under Strumenti | e35070a | Sidebar.tsx |

## What Was Built

**Types (`frontend/src/types/votes.ts`):** Exports `VoteExplorerEntry`, `VoteSearchResponse`, `PartyCohesion`, `VoteCohesionData`, `DeputyVoteStats`, `VoteCompassParty`, `VoteCompassData`. Re-exported from `frontend/src/types/index.ts` barrel so downstream plans (14-06, 14-07) import from `@/types`.

**API client (`frontend/src/lib/votes-api.ts`):** Three typed async functions — `searchVotes` (GET /api/votes), `getVoteCohesion` (GET /api/rankings/votes), `getVoteCompass` (GET /api/compass/votes). Same `buildHeaders()` / `getLocale()` pattern as `transcript-api.ts`; uses `config.api.baseUrl` for override support.

**i18n:** Five new top-level namespaces (`Votes`, `VoteCoherence`, `VoteFacts`, `VoteCompass`, `Cohesion`) added to both `it.json` and `en.json` with full IT/EN key parity. `Sidebar.votesExplorer` key added (IT: "Votazioni", EN: "Votes").

**Votes explorer page (`frontend/src/app/votes/page.tsx`):** Client component with filter bar (chamber, legislature, date range, outcome toggle, min-margin number input). Hairline-row table with date / chamber / label / outcome / favor% / against% / margin bar columns. Desktop grid layout + mobile stacked layout. Pagination via "Load more" button. Row click navigates to `/transcript/{debate_id}` or `/timeline` fallback. Editorial `[font-family:var(--font-display)]` class on numeric columns.

**Sidebar (`frontend/src/components/layout/Sidebar.tsx`):** `Vote` icon imported from lucide-react; Votazioni NavButton entry added after ideologicalCompass in both desktop and mobile Strumenti sections.

## Deviations from Plan

None — plan executed exactly as written.

## Verification

- `npx tsc --noEmit` exits 0
- `npx eslint src/app/votes/page.tsx src/components/layout/Sidebar.tsx src/lib/votes-api.ts` reports no errors
- IT/EN parity verified for all 5 Phase-14 namespaces via Node.js check
- `export * from "./votes"` confirmed in types/index.ts barrel
- Sidebar contains `/votes` 2× (desktop + mobile blocks)

## Self-Check: PASSED

Files exist:
- frontend/src/types/votes.ts — FOUND
- frontend/src/lib/votes-api.ts — FOUND
- frontend/src/app/votes/page.tsx — FOUND

Commits exist:
- d613a0a — FOUND (feat(14-05): add vote types...)
- e803e4e — FOUND (feat(14-05): add /votes explorer...)
- e35070a — FOUND (feat(14-05): add Votazioni sidebar...)
