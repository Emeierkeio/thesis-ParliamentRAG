# Roadmap: ParliamentRAG Full Codebase Refactoring

## Overview

A bottom-up refactoring ordered by the dependency graph: build pipeline first (produces clean English-only schema), then backend services and routers (all Cypher consumers updated atomically with the schema change), then frontend (independent of backend internals), then retrieval and graph enrichment on top of the clean foundation. Every layer builds on the one below it, with a single atomic deploy boundary between Phase 1 and Phase 2.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [x] **Phase 1: Build Pipeline** - Clean English-only build pipeline with full data extraction (completed 2026-04-02)
- [x] **Phase 2: Backend** - Services, routers, scripts, and code quality — deployed atomically with Phase 1 schema (completed 2026-04-02)
- [x] **Phase 3: Frontend** - Strict TypeScript, English naming, clean components (completed 2026-04-02)
- [x] **Phase 4: Enrichment** - Hybrid retrieval (BM25+RRF) and graph enrichment (SPARQL, NER) (completed 2026-04-02)

## Phase Details

### Phase 1: Build Pipeline
**Goal**: The build pipeline produces a clean, English-only Neo4j schema with no dead code, no redundant properties, and full data extraction (Vote nodes, Debate-to-Act links, Speaker roles, Phase types)
**Depends on**: Nothing (first phase)
**Requirements**: BUILD-01, BUILD-02, BUILD-03, BUILD-04, BUILD-05, BUILD-06, BUILD-07, BUILD-08, BUILD-09, DATA-01, DATA-02, DATA-03, DATA-04
**Success Criteria** (what must be TRUE):
  1. Running `make db-all` rebuilds the entire database from scratch and completes without errors
  2. The rebuilt database contains no Italian-schema properties (`start_char_raw`, `preprocessed_text`, `complete_date`) and all Chunk/Speech/Session nodes have only the new camelCase property names
  3. Vote nodes exist in the graph with HAS_VOTE edges from Session nodes (corrected from Debate — votes are session-level in XML)
  4. Debate nodes have DISCUSSES edges to ParliamentaryAct nodes for debates that reference acts in their XML
  5. Speech nodes carry a `speakingRole` property for speeches with an institutional role in the XML
**Plans**: 5 plans

Plans:
- [ ] 01-01-PLAN.md — Extract XML parser and create test infrastructure
- [ ] 01-02-PLAN.md — Create chunker module and config file
- [ ] 01-03-PLAN.md — Build database builder with UNWIND batch writes
- [ ] 01-04-PLAN.md — Wire modules into build_and_update.py entry point
- [ ] 01-05-PLAN.md — Integration verification against Neo4j

### Phase 2: Backend
**Goal**: All backend layers (services, routers, scripts) are clean, correctly typed, and deploy atomically with the new schema — all Cypher consumers updated, business logic extracted into services, dependency injection fixed, cross-layer coupling violations resolved
**Depends on**: Phase 1 (schema must exist before Cypher updates are deployed)
**Requirements**: SVC-01, SVC-02, SVC-03, SVC-04, SVC-05, SVC-06, API-01, API-02, API-03, API-04, API-05, SCR-01, SCR-02, SCR-03, QA-01, QA-02, QA-03
**Success Criteria** (what must be TRUE):
  1. The backend starts and all API endpoints respond correctly after the new schema database is connected (no `null` returns from renamed properties)
  2. No cross-layer import violations exist: evaluation router does not import from survey router, seed script does not import from chat router
  3. `search.py` uses the shared Neo4j client from `deps.py` (no duplicate connection pool)
  4. Expert computation exists in exactly one place (`services/experts.py`) — chat and query routers both delegate to it
  5. A smoke test suite runs against the critical paths (build pipeline, retrieval, evaluation) and passes
**Plans**: 6 plans

Plans:
- [ ] 02-01-PLAN.md — Test infrastructure, SSE contract doc, and dead Cypher property removal
- [ ] 02-02-PLAN.md — Rewrite DI to typed Depends(), fix duplicate Neo4j pools
- [ ] 02-03-PLAN.md — Unified expert service extraction, cross-layer import fixes
- [ ] 02-04-PLAN.md — Evaluation service extraction from router
- [ ] 02-05-PLAN.md — New data endpoints (votes, acts), script DI refactoring
- [ ] 02-06-PLAN.md — Code quality sweep, comprehensive tests (80+), SSE contract verification

### Phase 3: Frontend
**Goal**: The Next.js frontend has strict TypeScript (no `any`), English naming throughout, and clean component structure with no dead code
**Depends on**: Phase 2 (API contract must be stable before formalizing TypeScript types)
**Requirements**: FE-01, FE-02, FE-03
**Success Criteria** (what must be TRUE):
  1. `tsc --strict` compiles the frontend with zero type errors (no `any` escapes)
  2. All Italian-language route paths, variable names, and comments are replaced with English equivalents
  3. The `/valutazione` → `/evaluation` rename works with a redirect so existing bookmarks do not break
**Plans**: 2 plans

Plans:
- [ ] 03-01-PLAN.md — SSE type interfaces and strict TypeScript (zero any)
- [ ] 03-02-PLAN.md — Italian-to-English renames, route migration, barrel exports, dead code removal

### Phase 4: Enrichment
**Goal**: The system retrieves more relevant results via BM25 sparse channel merged with RRF, and the graph contains per-deputy vote records, committee officer roles, and NER-extracted law/person references on chunks
**Depends on**: Phase 2 (clean services layer required before adding new retrieval channels and ingestion modules)
**Requirements**: RET-01, RET-02, ENR-01, ENR-02, ENR-03, ENR-04
**Success Criteria** (what must be TRUE):
  1. Queries for specific deputy names or decree numbers ("decreto 231") return more relevant chunks than before due to BM25 sparse channel contributing to RRF-merged results
  2. The graph contains individual deputy vote records linked from the SPARQL ingest (`make enrich-sparql`)
  3. Chunk nodes carry `lawRefs` and `personRefs` arrays populated by NER at ingestion time
  4. Entity-filtered retrieval in `graph_channel.py` uses `lawRefs`/`personRefs` for targeted traversal
**Plans**: 3 plans

Plans:
- [ ] 04-01-PLAN.md — BM25 sparse channel, RRF merger, fulltext index in build pipeline
- [ ] 04-02-PLAN.md — SPARQL ingestion (per-deputy votes + committee officer roles)
- [ ] 04-03-PLAN.md — NER extraction module, build pipeline integration, entity-filtered retrieval

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6

**Critical Deploy Note:** Phase 1 and Phase 2 must be deployed as a unit. Do not run the Phase 1 rebuilt database against the old (pre-Phase-2) backend — schema properties will return `null` silently.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Build Pipeline | 4/5 | Complete    | 2026-04-02 |
| 2. Backend | 5/6 | Complete    | 2026-04-02 |
| 3. Frontend | 2/2 | Complete    | 2026-04-02 |
| 4. Enrichment | 2/3 | Complete    | 2026-04-02 |

### Phase 5: Multi-language Support

**Goal:** The application supports Italian (default) and English with i18n infrastructure, parliamentary citations are translated on-the-fly via OpenAI when user language ≠ Italian, tooltip hover shows original Italian text, and a dual-layer disclaimer (banner + globe icon) communicates machine translation
**Requirements**: ML-01, ML-02, ML-03, ML-04, ML-05
**Depends on:** Phase 4
**Success Criteria** (what must be TRUE):
  1. Language selector allows switching between Italian and English
  2. All UI text renders correctly in both languages via i18n translation keys
  3. Citations in non-Italian mode show translated text with original Italian on hover
  4. Banner disclaimer appears above response area when language ≠ Italian
  5. Globe icon appears on each translated citation
**Plans**: 3 plans

Plans:
- [ ] 05-01-PLAN.md — i18n infrastructure (next-intl, locale files, language selector) + backend translation service
- [ ] 05-02-PLAN.md — Extract all hardcoded Italian UI strings to translation keys
- [ ] 05-03-PLAN.md — Citation translation wiring (backend + frontend), tooltip, banner, globe icon

### Phase 6: Senate Data Integration

**Goal:** The system ingests Senato della Repubblica stenographic records (XIX legislatura) alongside Camera data, with a chamber selector (Camera / Senato / Both) in the UI that filters retrieval queries
**Requirements**: SEN-01, SEN-02, SEN-03, SEN-04, SEN-05
**Depends on:** Phase 5
**Success Criteria** (what must be TRUE):
  1. Senate stenographic XML files are downloaded and parsed by a dedicated `senate_parser.py`
  2. Senate Session, Debate, Phase, Speech, Chunk nodes exist in Neo4j with `chamber: "senato"`
  3. Chamber selector in the UI allows switching between Camera / Senato / Both
  4. Default "Both" retrieval returns results from both chambers ranked by relevance
  5. `make db-senate` builds Senate data, `make db-all` builds both
**Plans**: 3 plans

Plans:
- [ ] 06-01-PLAN.md — Senate AKN parser, download script, build_and_update integration, Makefile targets
- [ ] 06-02-PLAN.md — Backend retrieval chamber filter (all 3 channels + ChatRequest extension)
- [ ] 06-03-PLAN.md — Frontend ChamberSelector component, wiring to use-chat.ts, locale keys

### Phase 7: Pipeline optimization — analyze and optimize the full retrieval-to-generation pipeline (retrieval, generation, ideological compass) for cost, latency, and quality

**Goal:** [To be planned]
**Requirements**: TBD
**Depends on:** Phase 6
**Plans:** 3/3 plans complete

Plans:
- [ ] TBD (run /gsd:plan-phase 7 to break down)

### Phase 8: Senate individual vote scraping from senato.it HTML pages

**Goal:** [To be planned]
**Requirements**: TBD
**Depends on:** Phase 7
**Plans:** 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 8 to break down)
