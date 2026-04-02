# Requirements: ParliamentRAG Full Refactoring

**Defined:** 2026-04-02
**Core Value:** A clean, correct, and explainable codebase that is easy to maintain, extend, and reason about

## v1 Requirements

### Build Pipeline

- [ ] **BUILD-01**: Build pipeline produces English-only Neo4j schema (camelCase properties, PascalCase labels, SCREAMING_SNAKE_CASE relationships)
- [ ] **BUILD-02**: Remove Italian-schema dead code from `ingest_stenografici.py` (save_to_neo4j, Italian constraints/indexes)
- [ ] **BUILD-03**: Extract XML parser class from `ingest_stenografici.py` into standalone module (decouple from save logic)
- [ ] **BUILD-04**: Remove redundant Chunk properties (`startCharRaw`, `endCharRaw`) and dead alignment_map code
- [ ] **BUILD-05**: Remove redundant Speech property (`preprocessedText` — keep only `text` as preprocessed)
- [ ] **BUILD-06**: Remove redundant Session property (`completeDate` — use only Neo4j Date `date`)
- [ ] **BUILD-07**: Use UNWIND batch writes for bulk ingestion (replace per-item MERGE loops)
- [ ] **BUILD-08**: Use managed transactions (`execute_read`/`execute_write`) instead of auto-commit `session.run()`
- [ ] **BUILD-09**: Single `make db-all` target that rebuilds entire DB from scratch with fresh CSV downloads

### Data Extraction

- [ ] **DATA-01**: Persist Vote nodes (renamed from Votazione) with HAS_VOTE relationship from Debate
- [ ] **DATA-02**: Extract `<argomenti>` metadata to create Debate-[:DISCUSSES]->ParliamentaryAct edges
- [ ] **DATA-03**: Extract speaker institutional role from `<emphasis>` tag into Speech.speakingRole property
- [ ] **DATA-04**: Parse Phase title patterns into Phase.phaseType enum (e.g., "government_opinion", "vote_declaration", "general_discussion")

### Backend Services

- [ ] **SVC-01**: Update all Cypher queries across 6+ modules to match new camelCase property names
- [ ] **SVC-02**: Extract expert computation into `services/experts.py` (unify `chat.py` and `query.py` duplicate implementations)
- [ ] **SVC-03**: Extract evaluation business logic into `services/evaluation_service.py` (decouple from router)
- [ ] **SVC-04**: Replace `get_services()` dict with FastAPI `Depends()` typed dependency injection
- [ ] **SVC-05**: Fix search.py duplicate Neo4j connection pool (use shared client from deps.py)
- [ ] **SVC-06**: Clean naming, type hints, English docstrings across all service modules

### Backend API

- [ ] **API-01**: Refactor routers to thin wrappers around services (no business logic in routers)
- [ ] **API-02**: Fix cross-router import violations (evaluation.py→survey.py, seed script→chat.py)
- [ ] **API-03**: Freeze SSE event contract (document all 18 yield sites, ensure no behavioral changes)
- [ ] **API-04**: Clean endpoint naming, Pydantic v2 models, consistent error handling
- [ ] **API-05**: Preserve API response shapes (frontend contract must not break)

### Backend Scripts

- [ ] **SCR-01**: Refactor utility scripts with consistent naming, docstrings, error handling
- [ ] **SCR-02**: Fix `seed_evaluation_topic.py` router-import coupling (use extracted service)
- [ ] **SCR-03**: Scripts use shared Neo4j client instead of creating own connections

### Frontend

- [ ] **FE-01**: Strict TypeScript — no `any` types, proper interfaces for all API responses
- [ ] **FE-02**: Clean component structure — remove dead code, consistent naming
- [ ] **FE-03**: English route paths and variable names where currently Italian

### Retrieval Enrichment

- [ ] **RET-01**: Add BM25 sparse retrieval channel via Neo4j full-text index (zero new dependencies)
- [ ] **RET-02**: Implement Reciprocal Rank Fusion (RRF) merger for hybrid dense+sparse+graph retrieval

### Graph Enrichment

- [ ] **ENR-01**: SPARQL ingestion from dati.camera.it — per-deputy individual vote records
- [ ] **ENR-02**: SPARQL ingestion from dati.camera.it — committee officer roles with dates
- [ ] **ENR-03**: NER at ingestion time on chunks — extract LAW and PERSON entity references
- [ ] **ENR-04**: Store NER results as Chunk properties (lawRefs, personRefs) for entity-filtered retrieval

### Code Quality

- [ ] **QA-01**: Add smoke tests for critical paths (build pipeline, retrieval, evaluation) — zero current coverage
- [ ] **QA-02**: Python code follows best practices: type hints, English docstrings, no dead code
- [ ] **QA-03**: Consistent naming conventions across entire codebase (camelCase Neo4j, snake_case Python, camelCase TypeScript)

## v2 Requirements

### Advanced Enrichment

- **ENR-V2-01**: Wikidata biographical enrichment for deputies
- **ENR-V2-02**: Normattiva.it ELI URI linking for enacted laws
- **ENR-V2-03**: Entity linking (map mentions to knowledge base IDs via ReLiK)
- **ENR-V2-04**: Citation graph (Speech→ParliamentaryAct explicit references)

### Retrieval

- **RET-V2-01**: Query-time NER for entity-aware search filtering
- **RET-V2-02**: Stance detection on political topics (requires Italian-specific model training)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Full Akoma Ntoso XML migration | Existing XML schema captures all structural info; overkill for this project |
| Changing embedding model/dimensions | Keep text-embedding-3-small @ 1536d; model change is a separate project |
| Frontend UI/UX redesign | Only code cleanup, not visual changes |
| Relation extraction (who supports what) | Vote nodes provide definitive support/opposition data; NLP extraction unreliable for formal text |
| Coreference resolution | Dense embeddings handle coreference implicitly; marginal value for formal parliamentary text |
| Topic modeling for debate categorization | Phase.phaseType enum + EuroVoc descriptors cover categorization needs |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| BUILD-01 | Phase 1 | Pending |
| BUILD-02 | Phase 1 | Pending |
| BUILD-03 | Phase 1 | Pending |
| BUILD-04 | Phase 1 | Pending |
| BUILD-05 | Phase 1 | Pending |
| BUILD-06 | Phase 1 | Pending |
| BUILD-07 | Phase 1 | Pending |
| BUILD-08 | Phase 1 | Pending |
| BUILD-09 | Phase 1 | Pending |
| DATA-01 | Phase 1 | Pending |
| DATA-02 | Phase 1 | Pending |
| DATA-03 | Phase 1 | Pending |
| DATA-04 | Phase 1 | Pending |
| SVC-01 | Phase 2 | Pending |
| SVC-02 | Phase 2 | Pending |
| SVC-03 | Phase 2 | Pending |
| SVC-04 | Phase 2 | Pending |
| SVC-05 | Phase 2 | Pending |
| SVC-06 | Phase 2 | Pending |
| API-01 | Phase 2 | Pending |
| API-02 | Phase 2 | Pending |
| API-03 | Phase 2 | Pending |
| API-04 | Phase 2 | Pending |
| API-05 | Phase 2 | Pending |
| SCR-01 | Phase 2 | Pending |
| SCR-02 | Phase 2 | Pending |
| SCR-03 | Phase 2 | Pending |
| QA-01 | Phase 2 | Pending |
| QA-02 | Phase 2 | Pending |
| QA-03 | Phase 2 | Pending |
| FE-01 | Phase 3 | Pending |
| FE-02 | Phase 3 | Pending |
| FE-03 | Phase 3 | Pending |
| RET-01 | Phase 4 | Pending |
| RET-02 | Phase 4 | Pending |
| ENR-01 | Phase 4 | Pending |
| ENR-02 | Phase 4 | Pending |
| ENR-03 | Phase 4 | Pending |
| ENR-04 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 39 total
- Mapped to phases: 39
- Unmapped: 0

---
*Requirements defined: 2026-04-02*
*Last updated: 2026-04-02 — Traceability updated to match 4-phase coarse roadmap*
