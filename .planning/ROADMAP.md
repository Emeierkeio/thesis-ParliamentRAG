# Roadmap: ParliamentRAG Full Codebase Refactoring

## Overview

A bottom-up refactoring ordered by the dependency graph: build pipeline first (produces clean English-only schema), then backend services and routers (all Cypher consumers updated atomically with the schema change), then frontend (independent of backend internals), then retrieval and graph enrichment on top of the clean foundation. Every layer builds on the one below it, with a single atomic deploy boundary between Phase 1 and Phase 2.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [ ] **Phase 1: Build Pipeline** - Clean English-only build pipeline with full data extraction
- [ ] **Phase 2: Backend** - Services, routers, scripts, and code quality — deployed atomically with Phase 1 schema
- [ ] **Phase 3: Frontend** - Strict TypeScript, English naming, clean components
- [ ] **Phase 4: Enrichment** - Hybrid retrieval (BM25+RRF) and graph enrichment (SPARQL, NER)

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
**Plans**: TBD

### Phase 3: Frontend
**Goal**: The Next.js frontend has strict TypeScript (no `any`), English naming throughout, and clean component structure with no dead code
**Depends on**: Phase 2 (API contract must be stable before formalizing TypeScript types)
**Requirements**: FE-01, FE-02, FE-03
**Success Criteria** (what must be TRUE):
  1. `tsc --strict` compiles the frontend with zero type errors (no `any` escapes)
  2. All Italian-language route paths, variable names, and comments are replaced with English equivalents
  3. The `/valutazione` → `/evaluation` rename works with a redirect so existing bookmarks do not break
**Plans**: TBD

### Phase 4: Enrichment
**Goal**: The system retrieves more relevant results via BM25 sparse channel merged with RRF, and the graph contains per-deputy vote records, committee officer roles, and NER-extracted law/person references on chunks
**Depends on**: Phase 2 (clean services layer required before adding new retrieval channels and ingestion modules)
**Requirements**: RET-01, RET-02, ENR-01, ENR-02, ENR-03, ENR-04
**Success Criteria** (what must be TRUE):
  1. Queries for specific deputy names or decree numbers ("decreto 231") return more relevant chunks than before due to BM25 sparse channel contributing to RRF-merged results
  2. The graph contains individual deputy vote records linked from the SPARQL ingest (`make enrich-votes`)
  3. Chunk nodes carry `lawRefs` and `personRefs` arrays populated by NER at ingestion time
  4. Entity-filtered retrieval in `graph_channel.py` uses `lawRefs`/`personRefs` for targeted traversal
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

**Critical Deploy Note:** Phase 1 and Phase 2 must be deployed as a unit. Do not run the Phase 1 rebuilt database against the old (pre-Phase-2) backend — schema properties will return `null` silently.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Build Pipeline | 3/5 | In Progress|  |
| 2. Backend | 0/TBD | Not started | - |
| 3. Frontend | 0/TBD | Not started | - |
| 4. Enrichment | 0/TBD | Not started | - |
