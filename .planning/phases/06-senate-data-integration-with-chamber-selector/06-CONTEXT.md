# Phase 6: Senate Data Integration - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Add Senato della Repubblica stenographic records (XIX legislatura) to the existing Neo4j graph alongside Camera dei Deputati data. Create a dedicated Senate XML parser (`senate_parser.py`) that produces the same output format as the Camera parser. Extend the schema with a `chamber` property on Deputy/Session nodes to distinguish Camera from Senato. Add a chamber selector UI component (Camera / Senato / Both) above the chat input that filters retrieval queries.

</domain>

<decisions>
## Implementation Decisions

### Senate Data Source & Parser

- **Data source:** senato.it open data portal (stenographic XML records)
- **Legislature:** XIX only (current, consistent with Camera)
- **Parser:** Dedicated `build/senate_parser.py` module — separate from `xml_parser.py` (Camera)
- **Output format:** Same dict structure as Camera parser (`parse_xml_file()` returns same keys: `seduta`, `dibattiti`, `fasi`, `interventi`, `votazioni`)
- **db_builder.py receives uniform data** — no Camera/Senato distinction at the write level, only the `chamber` property differs
- **Makefile target:** `make db-senate` for Senate-only build, `make db-all` builds both

### Chamber Selector UI

- **Component:** Dropdown/toggle above the chat input area
- **Options:** Camera | Senato | Entrambi (Both)
- **Default:** Entrambi (Both) — search across both chambers
- **Behavior:** Selection persists per session (localStorage). Filters the retrieval query.
- **i18n:** Labels translated in both Italian and English (Phase 5 infrastructure)
- **Backend propagation:** Chamber selection sent to backend via query parameter or request body field

### Schema & Naming

- **Deputy distinction:** Same `Deputy` label with `chamber: "camera" | "senato"` property
- **Session distinction:** Same `Session` label with `chamber: "camera" | "senato"` property (already has `chamber` field from Phase 1)
- **GovernmentMember:** No `chamber` property — government members are cross-chamber
- **ParliamentaryGroup:** Senato has different group names than Camera — new groups created as needed
- **Committee:** Senato committees are different from Camera — new nodes created
- **Retrieval filter:** Dense and graph channels add `WHERE s.chamber IN $chambers` clause when not "both"

### Claude's Discretion
- Senate XML format analysis and parser implementation details
- Senate download URL patterns and error handling
- How to handle Senate-specific metadata differences
- Exact chamber selector component design (toggle vs dropdown)
- Whether Senate embeddings need separate pre-calculation or can share the existing pipeline
- How to handle cross-chamber government members in retrieval results

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Build pipeline (being extended)
- `build/xml_parser.py` — Camera parser (pattern to follow for senate_parser.py)
- `build/db_builder.py` — Database writer (must handle chamber property)
- `build/build_and_update.py` — CLI entry point (add Senate build path)
- `build/download.py` — Camera XML download (pattern for Senate download)
- `build/config.yaml` — Build configuration

### Backend retrieval (needs chamber filter)
- `backend/app/services/neo4j_client.py` — Vector search (add chamber filter)
- `backend/app/services/retrieval/dense_channel.py` — Dense retrieval (add WHERE clause)
- `backend/app/services/retrieval/graph_channel.py` — Graph retrieval (add WHERE clause)
- `backend/app/services/retrieval/sparse_channel.py` — BM25 retrieval (add WHERE clause)

### Frontend (needs chamber selector)
- `frontend/src/hooks/use-chat.ts` — Chat hook (pass chamber selection to backend)
- `frontend/src/components/chat/ChatArea.tsx` — Chat area (place chamber selector)
- `frontend/src/components/chat/ChatInput.tsx` — Chat input area
- `frontend/messages/it.json` + `en.json` — Locale files (add chamber selector labels)

### Prior decisions
- `.planning/phases/01-build-pipeline/01-CONTEXT.md` — Schema design, modular build architecture
- `.planning/phases/04-enrichment/04-CONTEXT.md` — BM25+RRF retrieval, SPARQL ingestion patterns

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `xml_parser.py` — Camera parser pattern (StenograficoParser class) — follow same interface for Senate
- `db_builder.py` — Already has `chamber` field on Session nodes — just needs Senate value
- `download.py` — XML download with retry logic — reuse pattern for Senate
- `chunker.py` — Language-agnostic sentence splitting — works for Senate text too
- `ner.py` — NER extraction — works for Senate chunks identically

### Established Patterns
- Build pipeline: parser → chunker → db_builder → precalculate_embeddings
- Retrieval channels: each has a Cypher query with WHERE clauses — add chamber filter
- Frontend: `use-chat.ts` sends request body to backend — add chamber field

### Integration Points
- `build_and_update.py` — Add `do_build_senate()` or extend `do_build()` with chamber param
- `Makefile` — Add `db-senate` target alongside `db-all`
- Frontend chat request — Add `chamber` field to ChatRequest model
- Backend retrieval — All 3 channels need `s.chamber IN $chambers` filter

</code_context>

<specifics>
## Specific Ideas

- The Senate parser will be simpler than the Camera parser (Senate XML is typically less nested)
- The chamber selector should feel like a natural part of the chat interface, not a separate settings page
- When "Both" is selected, results from both chambers are interleaved by relevance score

</specifics>

<deferred>
## Deferred Ideas

- Historical legislatures (XVIII and earlier) — future milestone
- Cross-chamber analysis features (comparing Camera vs Senato positions) — future milestone
- Joint session (seduta comune) handling — rare, defer

</deferred>

---

*Phase: 06-senate-data-integration*
*Context gathered: 2026-04-04*
