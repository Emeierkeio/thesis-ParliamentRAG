# Phase 9: Parliamentary Timeline — Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

A browsable parliamentary timeline at `/timeline` showing daily session recaps with AI-generated summaries, per-debate breakdown with phase structure and speaker lists, and per-debate speaker position summaries. Includes keyword search, date range filtering, and chamber selection. The timeline is read-only — users browse what happened, they don't interact with the RAG pipeline (except via an "Ask about this" cross-link button).

Summary generation happens pre-computed at build time via a separate `make generate-summaries` target, stored as Neo4j properties. Both Italian and English summaries are generated at build time.

</domain>

<decisions>
## Implementation Decisions

### Timeline Navigation
- New dedicated page at `/timeline`, added to sidebar navigation
- Scrollable vertical list, most recent sessions first, infinite scroll
- Reuse existing ChamberSelector component (Camera/Senato/Both) at the top
- Session cards are collapsible — each day's session is a card with expandable debates inside
- Cursor-based pagination (use last session date as cursor) for stable infinite scroll
- CalendarDays icon in sidebar, positioned after Search and before Rankings

### Search & Filtering
- Keyword search bar at the top alongside chamber selector
- Search scope: debate titles + recap text + speaker names
- Server-side API search (not client-side filtering)
- Debounced auto-search (~300ms) — no submit button needed
- Results shown in same timeline layout, filtered (not separate results view)
- Matching keywords highlighted in results
- Optional date range filter with from/to date pickers
- Quick-access preset buttons: Last week, Last month, Last 3 months
- Sessions outside selected date range are hidden (not grayed out)
- Sticky search bar + filters on mobile

### Daily Session Card (Collapsed)
- AI-generated 2-3 sentence recap of the day's key topics
- Stats line: debate count, vote count, speech count
- List of debate titles (collapsed, expandable)
- Recap in user's current language (IT or EN) — both pre-generated at build time

### Debate Detail (Expanded)
- AI-generated 3-5 sentence debate recap
- Parliamentary acts shown as badge/tags under debate title (using existing Debate-[:DISCUSSES]->ParliamentaryAct edges)
- Vote outcomes as expandable summary line ("Votes: 3 roll calls") — click to see each vote's subject, outcome, and counts
- Phase structure shown as sub-list: phase names with speech counts (e.g., "General Discussion (12 speeches)") — not expandable to individual speeches
- "Ask about this" button that opens new chat pre-filled with a natural language question derived from debate metadata (e.g., "What positions were expressed in the debate on Decreto Sicurezza on Apr 7, 2026?")

### Speaker Summaries
- All speakers shown (no limit), listed in speaking order (chronological)
- Flat list (not grouped by party)
- Each speaker row: name, party badge, speech count, phase participation tags
- speakingRole shown as badge (e.g., "Relatore", "Presidente") — data from Phase 1 XML extraction
- Government members (ministers, undersecretaries) get a distinct badge/icon (shield)
- Click speaker → expandable inline AI position summary (2-3 sentences)
- Speaker name links to /ranking page with that speaker pre-selected

### Summary Generation (Build Pipeline)
- Pre-computed at build time — stored as Neo4j node properties
- Separate `make generate-summaries` target (not part of db-all); `make db-full` runs both in sequence
- Both Italian and English summaries generated at build time (recapIt/recapEn on Session and Debate nodes)
- gpt-4.1-mini model for all summary generation
- Session recap: 2-3 sentences, stored as Session.recapIt / Session.recapEn
- Debate recap: 3-5 sentences, stored as Debate.recapIt / Debate.recapEn
- Speaker summaries: one per (speaker, debate) pair, combining all speeches in that debate — stored as SpeakerDebateSummary node with summaryIt, summaryEn, speechCount, phases[] properties, linked via (Speaker)-[:HAS_DEBATE_SUMMARY]->(SpeakerDebateSummary)
- SpeakerDebateSummary stores party snapshot at debate time (in case deputy switches parties)
- Short debates (<3 speeches): skip AI summary, show metadata only
- Resumable: skip sessions that already have recapIt set
- Dry run mode: `make generate-summaries DRY_RUN=1` shows estimated token count, API cost, and counts before generating
- Uniform processing for both Camera and Senate data (single script, uses chamber property)
- Sessions without summaries gracefully degrade to metadata-only display

### Backend API
- Three endpoints:
  1. `GET /api/timeline?before=<date>&limit=20&chamber=both&search=keyword&from=date&to=date` — returns sessions with nested debate titles/stats, cursor-based
  2. `GET /api/timeline/debates/{id}` — returns debate recap, phases, speaker list, votes
  3. `GET /api/timeline/speakers/{debateId}/{speakerId}` — returns speaker AI summary
- Session list includes debate titles and stats (no extra requests for collapsed view)
- Locale via HTTP header (same as Phase 5 chat endpoint) — returns recapIt or recapEn based on locale
- Speaker detail returns summary only (no raw speech text)

### Empty & Loading States
- Skeleton cards (3-4) while initial sessions load
- Brief explainer text above loading area: "Browse parliamentary sessions, debates, and speaker positions"
- Small spinner at bottom during infinite scroll fetch
- "No sessions found" message with clear filters button for empty search results
- Floating "back to top" button when scrolled deep
- Sessions without summaries show metadata-only with "Summary not yet generated" note

### Mobile Responsiveness
- Full-width cards, same nesting/expand-collapse behavior on mobile
- Sticky search bar + chamber selector on mobile
- Standard sidebar accessible on timeline page (consistent with other pages)

### Accessibility
- Standard WAI-ARIA Disclosure pattern: Enter/Space toggles expand/collapse, arrow keys navigate between items
- aria-live="polite" announcements for loading states, search result counts, and empty states

### Claude's Discretion
- Exact card styling, spacing, typography, shadows
- Loading skeleton design details
- Error state handling for API failures
- Exact search debounce timing
- Phase structure visual formatting
- "Back to top" scroll threshold

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

No external specs — requirements are fully captured in decisions above.

### Codebase References
- `.planning/codebase/STRUCTURE.md` — Directory layout and naming conventions for new files
- `.planning/codebase/CONVENTIONS.md` — Code style patterns (Python snake_case, TypeScript PascalCase, etc.)
- `.planning/codebase/ARCHITECTURE.md` — Layer design and data flow patterns

### Prior Phase Context
- `.planning/phases/05-multi-language-support-with-translated-citations-and-original-text-hover-disclaimer/05-CONTEXT.md` — i18n patterns (cookie-based locale, next-intl, translation keys)
- `.planning/phases/06-senate-data-integration-with-chamber-selector/06-CONTEXT.md` — ChamberSelector component, chamber property filtering
- `.planning/phases/07-pipeline-optimization-analyze-and-optimize-the-full-retrieval-to-generation-pipeline-retrieval-generation-ideological-compass-for-cost-latency-and-quality/07-CONTEXT.md` — gpt-4.1-mini model decision

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `frontend/src/components/layout/Sidebar.tsx` — Sidebar navigation component; add Timeline link here
- `frontend/src/components/chat/ChatArea.tsx` — ChamberSelector already used here; reuse pattern
- `frontend/src/components/ui/` — Shadcn/ui primitives (card, button, badge, tabs, scroll-area, skeleton)
- `frontend/src/hooks/use-sidebar.ts` — Sidebar toggle state management
- `frontend/src/hooks/use-chat.ts` — SSE and state management patterns to reference
- `backend/app/routers/data.py` — Existing data endpoints pattern (sessions/votes, debates/acts)
- `backend/app/services/deps.py` — Dependency injection pattern for new service
- `backend/app/services/neo4j_client.py` — Neo4j query patterns with Session/Debate/Speech traversal

### Established Patterns
- FastAPI router + service separation: thin routers, business logic in services/
- Pydantic models in backend/app/models/ for request/response schemas
- Next.js App Router: page.tsx per route
- Frontend components organized by domain (frontend/src/components/{domain}/)
- API clients in frontend/src/lib/{domain}-api.ts
- Types in frontend/src/types/{domain}.ts
- i18n: next-intl with messages/en.json and messages/it.json
- Locale from HTTP headers (Phase 5 pattern)

### Integration Points
- Sidebar.tsx — add /timeline navigation link with CalendarDays icon
- main.py — register new timeline router
- Neo4j schema — add recapIt/recapEn properties to Session and Debate nodes; add SpeakerDebateSummary node type
- Makefile — add generate-summaries and db-full targets
- messages/en.json + messages/it.json — add timeline translation keys

</code_context>

<specifics>
## Specific Ideas

- Session card layout inspired by the mockup: date header + chamber tag, AI recap below, stats line, then collapsible debate list
- Debate phases shown as a structured sub-list (not just mentioned in AI recap) — "General Discussion (12 speeches)", "Government Opinion (1 speech)", etc.
- "Ask about this" button pre-fills chat with a natural language question like "What positions were expressed in the debate on [topic] on [date]?"
- Speaker names link to /ranking page for cross-feature navigation
- Government members distinguished with shield icon, speaking roles (Relatore, Presidente) as badges

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries*
*Context gathered: 2026-04-07*
