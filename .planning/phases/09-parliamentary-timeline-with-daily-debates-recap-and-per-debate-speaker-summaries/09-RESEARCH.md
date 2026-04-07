# Phase 9: Parliamentary Timeline — Research

**Researched:** 2026-04-07
**Domain:** Full-stack feature: build pipeline (summary generation), FastAPI REST endpoints, Next.js browsable timeline UI
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Timeline Navigation**
- New dedicated page at `/timeline`, added to sidebar navigation
- Scrollable vertical list, most recent sessions first, infinite scroll
- Reuse existing ChamberSelector component (Camera/Senato/Both) at the top
- Session cards are collapsible — each day's session is a card with expandable debates inside
- Cursor-based pagination (use last session date as cursor) for stable infinite scroll
- CalendarDays icon in sidebar, positioned after Search and before Rankings

**Search & Filtering**
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

**Daily Session Card (Collapsed)**
- AI-generated 2-3 sentence recap of the day's key topics
- Stats line: debate count, vote count, speech count
- List of debate titles (collapsed, expandable)
- Recap in user's current language (IT or EN) — both pre-generated at build time

**Debate Detail (Expanded)**
- AI-generated 3-5 sentence debate recap
- Parliamentary acts shown as badge/tags under debate title (using existing Debate-[:DISCUSSES]->ParliamentaryAct edges)
- Vote outcomes as expandable summary line ("Votes: 3 roll calls") — click to see each vote's subject, outcome, and counts
- Phase structure shown as sub-list: phase names with speech counts (e.g., "General Discussion (12 speeches)") — not expandable to individual speeches
- "Ask about this" button that opens new chat pre-filled with a natural language question derived from debate metadata

**Speaker Summaries**
- All speakers shown (no limit), listed in speaking order (chronological)
- Flat list (not grouped by party)
- Each speaker row: name, party badge, speech count, phase participation tags
- speakingRole shown as badge (e.g., "Relatore", "Presidente")
- Government members get a distinct badge/icon (shield)
- Click speaker → expandable inline AI position summary (2-3 sentences)
- Speaker name links to /ranking page with that speaker pre-selected

**Summary Generation (Build Pipeline)**
- Pre-computed at build time — stored as Neo4j node properties
- Separate `make generate-summaries` target (not part of db-all); `make db-full` runs both in sequence
- Both Italian and English summaries generated at build time (recapIt/recapEn on Session and Debate nodes)
- gpt-4.1-mini model for all summary generation
- Session recap: 2-3 sentences, stored as Session.recapIt / Session.recapEn
- Debate recap: 3-5 sentences, stored as Debate.recapIt / Debate.recapEn
- Speaker summaries: one per (speaker, debate) pair, combining all speeches in that debate — stored as SpeakerDebateSummary node with summaryIt, summaryEn, speechCount, phases[] properties, linked via (Speaker)-[:HAS_DEBATE_SUMMARY]->(SpeakerDebateSummary)
- SpeakerDebateSummary stores party snapshot at debate time
- Short debates (<3 speeches): skip AI summary, show metadata only
- Resumable: skip sessions that already have recapIt set
- Dry run mode: `make generate-summaries DRY_RUN=1` shows estimated token count, API cost, and counts
- Uniform processing for both Camera and Senate data (single script, uses chamber property)
- Sessions without summaries gracefully degrade to metadata-only display

**Backend API**
- Three endpoints:
  1. `GET /api/timeline?before=<date>&limit=20&chamber=both&search=keyword&from=date&to=date`
  2. `GET /api/timeline/debates/{id}`
  3. `GET /api/timeline/speakers/{debateId}/{speakerId}`
- Session list includes debate titles and stats (no extra requests for collapsed view)
- Locale via HTTP header (same as Phase 5 chat endpoint) — returns recapIt or recapEn based on locale
- Speaker detail returns summary only (no raw speech text)

**Empty & Loading States**
- Skeleton cards (3-4) while initial sessions load
- Explainer text above loading area
- Small spinner at bottom during infinite scroll fetch
- "No sessions found" message with clear filters button for empty search results
- Floating "back to top" button when scrolled deep
- Sessions without summaries show metadata-only with "Summary not yet generated" note

**Mobile Responsiveness**
- Full-width cards, same nesting/expand-collapse behavior on mobile
- Sticky search bar + chamber selector on mobile
- Standard sidebar accessible on timeline page

**Accessibility**
- Standard WAI-ARIA Disclosure pattern: Enter/Space toggles expand/collapse, arrow keys navigate between items
- aria-live="polite" for loading states, search result counts, and empty states

### Claude's Discretion
- Exact card styling, spacing, typography, shadows
- Loading skeleton design details
- Error state handling for API failures
- Exact search debounce timing
- Phase structure visual formatting
- "Back to top" scroll threshold

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

---

## Summary

Phase 9 adds a browsable parliamentary timeline at `/timeline`. It has three distinct sub-domains that must be planned and implemented separately: (1) a build-time summary generation script (`build/generate_summaries.py`) that writes AI recaps to Neo4j, (2) three new FastAPI REST endpoints under `/api/timeline`, and (3) the Next.js frontend page with collapsible session cards, infinite scroll, search/filter, and speaker summaries.

The phase touches the entire stack but is additive — nothing existing is modified except `Sidebar.tsx` (new nav link), `main.py` (new router mount), `Makefile` (two new targets), and translation files. The Neo4j schema gets three new additions: `Session.recapIt/recapEn`, `Debate.recapIt/recapEn`, and a new `SpeakerDebateSummary` node type with `HAS_DEBATE_SUMMARY` relationship.

The highest-risk area is the summary generation script. It must handle: resumability (skip already-done sessions), short-debate skip logic (<3 speeches), dry-run cost estimation, and both Camera and Senato XML in a single pass. The frontend's infinite scroll with cursor-based pagination is the second most complex area, requiring stable cursor handling and correct merge of loaded page results.

**Primary recommendation:** Implement in three sequential waves — (1) build pipeline + Neo4j schema, (2) backend API, (3) frontend. Each wave is independently testable before the next.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| openai (Python) | Already in requirements.txt | gpt-4.1-mini calls for summary generation | Already used for generation pipeline; gpt-4.1-mini locked by Phase 7 decision |
| FastAPI | Already in requirements.txt | Three new timeline endpoints | Established pattern in this codebase |
| neo4j (Python driver) | Already in requirements.txt | Read sessions/debates/speakers, write summary properties | Established pattern |
| Next.js App Router | Already installed | `/timeline` page | Established pattern |
| Shadcn/ui | Already installed | Card, Badge, Button, Skeleton, ScrollArea | All already used in project |
| lucide-react | Already installed | CalendarDays (sidebar), Shield (gov members), ChevronDown (expand) | Already used in Sidebar.tsx |
| next-intl | Already installed | Timeline translation keys | Established i18n pattern from Phase 5 |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| asyncio.gather | stdlib | Parallel OpenAI calls during summary generation | Per-debate parallelism in generate_summaries.py |
| tqdm | Already in build/requirements-build.txt | Progress bar in generate_summaries.py | CLI script feedback |
| IntersectionObserver API (browser) | native | Infinite scroll trigger | Detect when user reaches bottom of list |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| IntersectionObserver for infinite scroll | react-infinite-scroll-component | Native API preferred — no new dep, simpler, project avoids extra dependencies |
| Pre-generated summaries | On-demand generation | Pre-generated is the locked decision; avoids latency spikes on first load |
| Separate IT/EN API calls | Single call with both | Both stored at build time; single endpoint reads correct locale from header — no extra API call needed |

**Installation:** No new dependencies required. All libraries are already present in the project.

---

## Architecture Patterns

### Recommended Project Structure (new files only)

```
build/
└── generate_summaries.py        # New: summary generation script

backend/app/
├── routers/
│   └── timeline.py              # New: three timeline endpoints
├── services/
│   └── timeline/
│       ├── __init__.py          # New: exports TimelineService
│       └── service.py           # New: Neo4j queries for timeline
└── models/
    └── timeline.py              # New: Pydantic models for timeline responses

frontend/src/
├── app/
│   └── timeline/
│       └── page.tsx             # New: /timeline route
├── components/
│   └── timeline/
│       ├── SessionCard.tsx      # New: collapsible day card
│       ├── DebateDetail.tsx     # New: expanded debate view
│       ├── SpeakerRow.tsx       # New: speaker row + inline summary
│       ├── TimelineSearch.tsx   # New: search bar + date pickers + presets
│       └── TimelineSkeleton.tsx # New: loading skeleton
├── hooks/
│   └── use-timeline.ts          # New: fetch state, pagination, search debounce
├── lib/
│   └── timeline-api.ts          # New: API client functions
└── types/
    └── timeline.ts              # New: TypeScript interfaces
```

### Pattern 1: Build Pipeline Script Structure

The generate_summaries.py script follows the same pattern as other build scripts (e.g., `sparql_ingester.py`, `precalculate_baseline_experts.py`).

**What:** Standalone Python script that reads from Neo4j, calls OpenAI, writes properties back to Neo4j. Uses the shared Neo4j driver pattern (accepts `--neo4j-uri`, `--neo4j-user`, `--neo4j-password` CLI args). Runs in the `build/` directory, uses `build/requirements-build.txt`.

**Key decisions:**
- Skip sessions where `Session.recapIt IS NOT NULL` (resumability)
- Skip debates with fewer than 3 speeches (no AI summary; store a sentinel like `recapIt=""` or simply leave null and handle gracefully in API)
- `DRY_RUN=1` mode: count sessions/debates/speakers, estimate tokens, print cost, exit without writing
- SpeakerDebateSummary node: `id = f"{debate_id}_{speaker_id}"` for idempotent MERGE
- Parallel per-session: gather speaker summary calls concurrently with asyncio; rate-limit to avoid 429s

```python
# Source: Pattern from build/sparql_ingester.py
import argparse
from neo4j import GraphDatabase
from openai import AsyncOpenAI

async def generate_for_session(session_data, client, tx_fn):
    """Generate and store all summaries for one session."""
    # 1. Generate session recap (IT + EN in one call or two calls)
    # 2. For each debate: generate debate recap (IT + EN)
    # 3. For each (debate, speaker) pair with >= 3 speeches: generate speaker summary
    # 4. Write all to Neo4j in one transaction per session
    pass
```

**Makefile targets:**
```makefile
generate-summaries: db-install ## Generate AI summaries (resumable)
    @$(PYTHON) $(BUILD_DIR)/generate_summaries.py \
        --neo4j-uri $(NEO4J_LOCAL) \
        --neo4j-user $(NEO4J_USER) \
        --neo4j-password $(NEO4J_PASS) \
        $(if $(DRY_RUN),--dry-run,)

db-full: db-all generate-summaries ## Full DB + summaries (one-shot)
```

### Pattern 2: FastAPI Timeline Router

Follows the thin-router pattern from `backend/app/routers/data.py` — router delegates to a `TimelineService` in `services/timeline/`.

**Endpoint 1 — Session list:**
```python
# Source: established pattern in backend/app/routers/data.py
@router.get("/api/timeline")
async def get_timeline(
    before: Optional[str] = None,       # ISO date cursor (e.g. "2026-03-15")
    limit: int = 20,
    chamber: str = "both",
    search: Optional[str] = None,
    from_date: Optional[str] = None,
    to_date: Optional[str] = None,
    request: Request = None,
    neo4j: Neo4jClient = Depends(get_neo4j_client),
) -> TimelineResponse:
    locale = request.headers.get("Accept-Language", "it")[:2]
    return await timeline_service.get_sessions(
        before=before, limit=limit, chamber=chamber,
        search=search, from_date=from_date, to_date=to_date, locale=locale
    )
```

**Locale header pattern** (established in Phase 5, confirmed in `use-chat.ts` line 187):
```
frontend reads NEXT_LOCALE cookie → sends Accept-Language: en|it header
backend reads request.headers.get("Accept-Language", "it")[:2]
```

**Endpoint 2 — Debate detail:**
```python
@router.get("/api/timeline/debates/{debate_id}")
async def get_debate_detail(debate_id: str, ...) -> DebateDetailResponse:
    ...
```

**Endpoint 3 — Speaker summary (lazy load on click):**
```python
@router.get("/api/timeline/speakers/{debate_id}/{speaker_id}")
async def get_speaker_summary(debate_id: str, speaker_id: str, ...) -> SpeakerSummaryResponse:
    ...
```

### Pattern 3: Neo4j Schema Extensions

New properties added to existing nodes (no schema migration needed — Neo4j is schemaless):
```cypher
-- Session node: add recapIt, recapEn
MATCH (s:Session {id: $id})
SET s.recapIt = $recap_it, s.recapEn = $recap_en

-- Debate node: add recapIt, recapEn
MATCH (d:Debate {id: $id})
SET d.recapIt = $recap_it, d.recapEn = $recap_en

-- New SpeakerDebateSummary node
MERGE (sds:SpeakerDebateSummary {id: $id})
SET sds.summaryIt = $summary_it,
    sds.summaryEn = $summary_en,
    sds.speechCount = $speech_count,
    sds.phases = $phases,
    sds.partySnapshot = $party_snapshot,
    sds.debateId = $debate_id,
    sds.speakerId = $speaker_id
WITH sds
MATCH (sp:Deputy|GovernmentMember {id: $speaker_id})
MERGE (sp)-[:HAS_DEBATE_SUMMARY]->(sds)
WITH sds
MATCH (d:Debate {id: $debate_id})
MERGE (sds)-[:FOR_DEBATE]->(d)
```

**Key Cypher for session list (with cursor):**
```cypher
MATCH (s:Session)
WHERE s.chamber IN $chambers
  AND ($before IS NULL OR s.date < date($before))
  AND ($from_date IS NULL OR s.date >= date($from_date))
  AND ($to_date IS NULL OR s.date <= date($to_date))
WITH s ORDER BY s.date DESC LIMIT $limit
MATCH (s)-[:HAS_DEBATE]->(d)
OPTIONAL MATCH (d)-[:HAS_PHASE]->(p)-[:CONTAINS_SPEECH]->(sp)
WITH s, d, count(DISTINCT sp) AS speechCount
OPTIONAL MATCH (s)-[:HAS_VOTE]->(v)
WITH s, collect(DISTINCT {id: d.id, title: d.title, speechCount: speechCount}) AS debates,
     count(DISTINCT v) AS voteCount
RETURN s, debates, voteCount, size(debates) AS debateCount
ORDER BY s.date DESC
```

**Search extension (add to WHERE clause):**
```cypher
AND ($search IS NULL OR
     toLower(d.title) CONTAINS toLower($search) OR
     toLower(s.recapIt) CONTAINS toLower($search) OR
     toLower(s.recapEn) CONTAINS toLower($search))
```

Note: Full-text search on recap text will be slow without an index. For Phase 9, CONTAINS is acceptable since it runs against already-filtered rows (date + chamber). Add a full-text index if performance is a concern.

### Pattern 4: Frontend Infinite Scroll with Cursor

```typescript
// Source: established pattern; IntersectionObserver is standard browser API
const observerRef = useRef<IntersectionObserver | null>(null);
const loadMoreRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  observerRef.current = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting && hasMore && !isFetchingMore) {
        fetchMoreSessions();
      }
    },
    { threshold: 0.1 }
  );
  if (loadMoreRef.current) observerRef.current.observe(loadMoreRef.current);
  return () => observerRef.current?.disconnect();
}, [hasMore, isFetchingMore]);
```

Cursor state in `use-timeline.ts`:
```typescript
const [cursor, setCursor] = useState<string | null>(null); // ISO date of last loaded session
const [sessions, setSessions] = useState<TimelineSession[]>([]);
const [hasMore, setHasMore] = useState(true);

async function fetchMoreSessions() {
  const data = await getTimelineSessions({ before: cursor, ...filters });
  setSessions(prev => [...prev, ...data.sessions]);
  setCursor(data.nextCursor ?? null);
  setHasMore(data.hasMore);
}
```

### Pattern 5: Sidebar Integration

Adding the Timeline nav link to `Sidebar.tsx` follows the exact same `NavButton` pattern already used for Search, Rankings, and Compass. The `CalendarDays` icon is already imported at line 35 of `Sidebar.tsx` (used in the "data updated at" footer row — will need to import a different icon or reuse it for the nav item).

```typescript
// Add after Search NavButton, before Rankings NavButton:
<NavButton
  item={{
    icon: CalendarDays,
    label: t('parliamentaryTimeline'),
    href: "/timeline",
    isActive: pathname === "/timeline",
    onClick: () => navTo("/timeline")
  }}
  isCollapsed={isCollapsed}
  disabled={false}
/>
```

Note: `CalendarDays` is already imported for both the desktop and mobile sidebars (line 35). The existing usage is for the "data updated at" label in the footer — it renders as a small decorative icon. Adding it as a nav button is fine; it won't clash because one is inside a `NavButton` (with active state) and the other is in the footer info row.

### Anti-Patterns to Avoid

- **Don't eager-load speaker summaries:** Speaker summaries are loaded on-demand when the user clicks a speaker row (separate API call). Do not include them in the session list endpoint or debate detail endpoint.
- **Don't run summary generation synchronously during API requests:** All summaries are pre-computed. The API simply reads from Neo4j.
- **Don't use `s.date` as a string for cursor directly:** Neo4j returns dates as `neo4j.time.Date` objects. Convert to ISO string (`str(record["date"])`) before returning from API.
- **Don't clear the session list on search:** When the search term changes, reset `cursor = null` and `sessions = []`, then refetch. Appending to stale results causes display corruption.
- **Don't use `MATCH (sp:Deputy|GovernmentMember)` union syntax in older Neo4j:** Use two separate OPTIONAL MATCH clauses or `MATCH (sp) WHERE sp:Deputy OR sp:GovernmentMember` depending on Neo4j version.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Search debounce | Custom setTimeout management | `useEffect` + `setTimeout` + cleanup ref (minimal, built-in) | No lib needed — single `useRef<ReturnType<typeof setTimeout>>` pattern |
| Keyword highlight | Custom regex replace | Simple string split on match, wrap in `<mark>` | Only needs to highlight exact keyword in title/recap text |
| Infinite scroll detection | Polling or scroll event listener | `IntersectionObserver` | Browser-native, zero-jank, already understood by the project |
| OpenAI retry | Custom backoff | `openai` library's built-in retry (max_retries=3) | Already handles 429/5xx; don't re-implement |
| Cost estimation (dry run) | Tokenizer library | `len(text) / 4` approximation | Good enough for cost estimates; avoids `tiktoken` dep |

**Key insight:** This phase is additive. All the complex infrastructure (OpenAI client, Neo4j client, Shadcn/ui components, i18n, chamber filtering) already exists. Don't re-implement any of it.

---

## Common Pitfalls

### Pitfall 1: Neo4j Date Type in Python
**What goes wrong:** `s.date` is returned as `neo4j.time.Date`, not a Python `datetime.date`. Calling `.isoformat()` raises AttributeError.
**Why it happens:** The Neo4j driver uses its own date type.
**How to avoid:** Cast with `str(record["date"])` — `neo4j.time.Date` has a `__str__` that returns ISO format. Or use `record["date"].to_native()` to get a Python `datetime.date`.
**Warning signs:** AttributeError mentioning `neo4j.time.Date` in backend logs.

### Pitfall 2: Cursor Collision When Multiple Sessions on the Same Date
**What goes wrong:** If two sessions share the same date, using `date < cursor` skips them both after the first page.
**Why it happens:** Cursor is the date of the last loaded session.
**How to avoid:** Use `date < cursor OR (date = cursor AND session_number < last_number)` — i.e., composite cursor on (date, session_number). Or accept the edge case (rare for parliamentary sessions to share exact dates).
**Warning signs:** Sessions disappear from the timeline when paginating past a date boundary.

### Pitfall 3: SpeakerDebateSummary Linked to Both Deputy and GovernmentMember
**What goes wrong:** A speaker may be a `Deputy` node OR a `GovernmentMember` node. A single `MATCH (sp:Deputy {id: $id})` will miss government members.
**Why it happens:** The schema has two distinct node types for speakers.
**How to avoid:** Use `OPTIONAL MATCH (d:Deputy {id: $id}) OPTIONAL MATCH (g:GovernmentMember {id: $id}) WITH coalesce(d, g) AS sp` pattern when linking `HAS_DEBATE_SUMMARY`.
**Warning signs:** Government ministers have no HAS_DEBATE_SUMMARY relationship even though the script appears to succeed.

### Pitfall 4: `make generate-summaries` Accidentally Re-runs for Already-Done Sessions
**What goes wrong:** If resumability check is missing, re-running the script regenerates summaries for all sessions (wastes API cost).
**Why it happens:** The script doesn't check existing `recapIt` before calling OpenAI.
**How to avoid:** Always query `WHERE s.recapIt IS NULL` before processing. In dry-run mode, count both total and pending sessions to show how much work remains.

### Pitfall 5: Search Resets Losing the Page Position
**What goes wrong:** When the user types in the search bar, the session list resets to page 1. If the user was deep in the list, the scroll position jumps to the top unexpectedly.
**Why it happens:** Correct behavior — the search filters require a fresh fetch from the start. But if the reset is not handled cleanly, the old data briefly flashes before the new data loads.
**How to avoid:** On search term change: immediately set `sessions = []` and `cursor = null` in the same state update, then trigger fetch. Use a loading skeleton to mask the transition.

### Pitfall 6: CalendarDays Already Used in Sidebar Footer
**What goes wrong:** Adding CalendarDays as a nav icon when it's already used as a decorative icon in the footer may create visual ambiguity.
**Why it happens:** The icon was imported for the data-update date display.
**How to avoid:** The CONTEXT.md explicitly specifies CalendarDays for the sidebar nav. Use it. The footer usage is a small icon in a text row — stylistically distinct from a nav button.

### Pitfall 7: OpenAI Rate Limits During Bulk Generation
**What goes wrong:** Generating summaries for all sessions in parallel hits the OpenAI TPM (tokens per minute) limit, causing widespread 429 errors.
**Why it happens:** Large number of concurrent requests.
**How to avoid:** Use `asyncio.Semaphore(10)` to cap concurrency at ~10 parallel calls. The `openai` library's built-in retry handles transient 429s. Log progress every N sessions.

---

## Code Examples

### Locale Detection in Backend Endpoint

```python
# Source: established pattern from backend/app/routers/chat.py (Phase 5)
from fastapi import Request

@router.get("/api/timeline")
async def get_timeline(
    request: Request,
    neo4j: Neo4jClient = Depends(get_neo4j_client),
    ...
):
    locale = request.headers.get("Accept-Language", "it")[:2]
    recap_field = "recapEn" if locale == "en" else "recapIt"
    ...
```

### Locale Sending in Frontend

```typescript
// Source: frontend/src/hooks/use-chat.ts line 182-187 (Phase 5 pattern)
const locale = document.cookie
  .split('; ')
  .find(c => c.startsWith('NEXT_LOCALE='))
  ?.split('=')[1] || 'it';

const response = await fetch('/api/timeline', {
  headers: { 'Accept-Language': locale }
});
```

### ChamberSelector Reuse

```typescript
// Source: frontend/src/components/chat/ChatArea.tsx — ChamberSelector already exists
import { ChamberSelector } from "@/components/chat/ChamberSelector";

// In timeline page:
<ChamberSelector chamber={chamber} onChamberChange={setChamber} />
```

### Expand/Collapse with WAI-ARIA Disclosure

```typescript
// Source: WAI-ARIA Authoring Practices — Disclosure pattern
<button
  aria-expanded={isOpen}
  aria-controls={`debate-${debate.id}`}
  onClick={() => setIsOpen(!isOpen)}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      setIsOpen(!isOpen);
    }
  }}
>
  {debate.title}
</button>
<div
  id={`debate-${debate.id}`}
  hidden={!isOpen}
  role="region"
>
  ...
</div>
```

### Summary Generation OpenAI Call Pattern

```python
# Source: gpt-4.1-mini locked by Phase 7 decision; pattern from generation pipeline
async def generate_session_recap(session_data: dict, client: AsyncOpenAI) -> tuple[str, str]:
    """Generate IT and EN recaps for a session. Returns (recap_it, recap_en)."""
    prompt_it = f"""Scrivi un riassunto di 2-3 frasi della sessione parlamentare del {session_data['date']}.
    Argomenti trattati: {', '.join(session_data['debate_titles'])}
    Il riassunto deve essere in italiano, chiaro e conciso."""

    prompt_en = f"""Write a 2-3 sentence summary of the parliamentary session of {session_data['date']}.
    Topics discussed: {', '.join(session_data['debate_titles'])}
    The summary must be in English, clear and concise."""

    it_resp, en_resp = await asyncio.gather(
        client.chat.completions.create(
            model="gpt-4.1-mini",
            messages=[{"role": "user", "content": prompt_it}],
            max_tokens=200,
        ),
        client.chat.completions.create(
            model="gpt-4.1-mini",
            messages=[{"role": "user", "content": prompt_en}],
            max_tokens=200,
        )
    )
    return it_resp.choices[0].message.content, en_resp.choices[0].message.content
```

### Pydantic Models for Timeline Endpoints

```python
# Source: established pattern in backend/app/models/
from pydantic import BaseModel
from typing import Optional

class DebateSummary(BaseModel):
    id: str
    title: str
    recap: Optional[str] = None      # None if not yet generated
    speech_count: int
    phase_count: int

class SessionCard(BaseModel):
    id: str
    date: str                         # ISO date string
    chamber: str
    number: int
    recap: Optional[str] = None
    debate_count: int
    vote_count: int
    speech_count: int
    debates: list[DebateSummary]

class TimelineResponse(BaseModel):
    sessions: list[SessionCard]
    next_cursor: Optional[str]        # ISO date of last session in this page
    has_more: bool
```

### i18n Keys to Add

```json
// Add to messages/en.json and messages/it.json
"Timeline": {
  "pageTitle": "Parliamentary Timeline",
  "searchPlaceholder": "Search debates, topics, speakers...",
  "chamberLabel": "Chamber",
  "dateFrom": "From",
  "dateTo": "To",
  "lastWeek": "Last week",
  "lastMonth": "Last month",
  "last3Months": "Last 3 months",
  "debateCount": "{count} debates",
  "voteCount": "{count} votes",
  "speechCount": "{count} speeches",
  "summaryNotYetGenerated": "Summary not yet generated",
  "askAboutThis": "Ask about this",
  "speakerSummaryLoading": "Loading summary...",
  "noSessionsFound": "No sessions found",
  "clearFilters": "Clear filters",
  "backToTop": "Back to top",
  "loadingMore": "Loading more sessions...",
  "browseDescription": "Browse parliamentary sessions, debates, and speaker positions",
  "votes": "Votes",
  "phases": "Phases",
  "speakers": "Speakers",
  "governo": "Government",
  "relatore": "Rapporteur",
  "presidente": "President"
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| No timeline feature | Browsable pre-computed timeline | Phase 9 | Users can browse sessions without querying the RAG pipeline |
| Per-request LLM calls | Pre-computed summaries at build time | Phase 9 decision | Zero latency on timeline page, controlled API cost |
| Session/Debate nodes have no recap | Session.recapIt/En, Debate.recapIt/En | Phase 9 | Neo4j schema extended additively |

**Not deprecated in this phase:** Everything existing remains unchanged. Phase 9 is purely additive.

---

## Open Questions

1. **Deputy vs GovernmentMember node for SPOKEN_BY relationship**
   - What we know: `Speech` nodes link to either `Deputy` or `GovernmentMember` via `SPOKEN_BY` relationship (confirmed in `db_builder.py` `_create_spoken_by_relations`).
   - What's unclear: The timeline needs to list speakers per debate. The query must handle both node types. Confirmed from prior bug fix in MEMORY.md that GovernmentMembers use a separate node type.
   - Recommendation: Always use `OPTIONAL MATCH (sp)-[:SPOKEN_BY]->(d:Deputy) OPTIONAL MATCH (sp)-[:SPOKEN_BY]->(g:GovernmentMember) WITH coalesce(d, g) AS speaker` pattern in all timeline queries involving speakers.

2. **Search performance on recap text**
   - What we know: `CONTAINS` in Cypher is a linear scan. The existing full-text index (`chunk_embedding_index`) is on Chunk nodes, not Session/Debate.
   - What's unclear: How many Session/Debate nodes exist after full build (Camera + Senate). If ~2000 sessions, CONTAINS is fine. If ~20000, may need a full-text index.
   - Recommendation: Start with CONTAINS (simpler). Add `CREATE FULLTEXT INDEX timeline_search_index IF NOT EXISTS FOR (d:Debate) ON EACH [d.title, d.recapIt, d.recapEn]` only if search is slow. This is within Claude's Discretion.

3. **SpeakerDebateSummary for Senate speakers**
   - What we know: Senate speakers are `GovernmentMember` or their own senator-specific nodes (from Phase 6 `SenateStenograficoParser`).
   - What's unclear: Whether senators are stored as `Deputy` nodes or a separate `Senator` label.
   - Recommendation: Read `build/senate_parser.py` before implementing the speaker summary linking logic to confirm the node label for Senate speakers. This must be resolved before coding the HAS_DEBATE_SUMMARY MERGE.

---

## Validation Architecture

nyquist_validation is enabled (config.json `workflow.nyquist_validation: true`).

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest 7.4.0+ (in backend/requirements.txt) |
| Config file | none (Wave 0: create pytest.ini or use pyproject.toml) |
| Quick run command | `cd backend && python -m pytest tests/test_timeline.py -x -q` |
| Full suite command | `cd backend && python -m pytest -v --tb=short` |

### Phase Requirements to Test Map
| Behavior | Test Type | Automated Command | Notes |
|----------|-----------|-------------------|-------|
| GET /api/timeline returns session list | smoke | `pytest tests/test_timeline.py::test_timeline_list -x` | Wave 0 gap |
| GET /api/timeline returns recapEn when Accept-Language: en | unit | `pytest tests/test_timeline.py::test_locale_header -x` | Wave 0 gap |
| GET /api/timeline/debates/{id} returns debate detail | smoke | `pytest tests/test_timeline.py::test_debate_detail -x` | Wave 0 gap |
| GET /api/timeline/speakers/{debateId}/{speakerId} returns summary | smoke | `pytest tests/test_timeline.py::test_speaker_summary -x` | Wave 0 gap |
| cursor-based pagination returns non-overlapping pages | unit | `pytest tests/test_timeline.py::test_pagination -x` | Wave 0 gap |
| generate_summaries.py dry-run exits without Neo4j writes | unit | `pytest tests/test_generate_summaries.py::test_dry_run -x` | Wave 0 gap |
| Sessions with recapIt set are skipped (resumability) | unit | `pytest tests/test_generate_summaries.py::test_resumable -x` | Wave 0 gap |

### Sampling Rate
- **Per task commit:** `cd backend && python -m pytest tests/test_timeline.py -x -q`
- **Per wave merge:** `cd backend && python -m pytest -v --tb=short`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `backend/tests/test_timeline.py` — smoke/unit tests for all three endpoints
- [ ] `backend/tests/test_generate_summaries.py` — dry-run and resumability tests
- [ ] `backend/tests/__init__.py` — if not already present
- [ ] `backend/pytest.ini` or add `[tool.pytest.ini_options]` to pyproject.toml — not yet configured

---

## Sources

### Primary (HIGH confidence)
- Direct codebase reading: `build/db_builder.py` — confirmed Session, Debate, Phase, Speech schema and exact property names
- Direct codebase reading: `backend/app/routers/data.py` — confirmed thin-router pattern for new data endpoints
- Direct codebase reading: `backend/app/services/deps.py` — confirmed `get_neo4j_client` Depends pattern
- Direct codebase reading: `frontend/src/components/layout/Sidebar.tsx` — confirmed NavButton pattern, CalendarDays already imported, exact structure for new nav link
- Direct codebase reading: `frontend/src/hooks/use-chat.ts` line 182-187 — confirmed locale cookie → Accept-Language header pattern
- Direct codebase reading: `backend/app/routers/chat.py` — confirmed `request.headers.get("Accept-Language", "it")[:2]` locale extraction
- Direct codebase reading: `Makefile` — confirmed `db-all` target structure; `make db-full` and `make generate-summaries` follow same pattern
- Direct codebase reading: `.planning/codebase/STRUCTURE.md`, `CONVENTIONS.md`, `ARCHITECTURE.md` — confirmed all naming conventions

### Secondary (MEDIUM confidence)
- `09-CONTEXT.md` — all locked decisions verified against the existing codebase for feasibility

### Tertiary (LOW confidence)
- Senator node label type for HAS_DEBATE_SUMMARY linking — not verified; noted as Open Question 3

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries already installed and in use
- Architecture: HIGH — patterns directly read from existing codebase files
- Pitfalls: HIGH — most are confirmed by reading actual code (Neo4j date type, Deputy/GovernmentMember split, CalendarDays reuse)
- Build pipeline structure: HIGH — confirmed by reading Makefile and existing build scripts
- Senate speaker node label: LOW — not verified; requires reading senate_parser.py before implementation

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable stack, 30-day window)
