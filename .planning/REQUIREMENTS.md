# Requirements: ParliamentRAG Full Refactoring

**Defined:** 2026-04-02
**Core Value:** A clean, correct, and explainable codebase that is easy to maintain, extend, and reason about

## v1 Requirements

### Build Pipeline

- [x] **BUILD-01**: Build pipeline produces English-only Neo4j schema (camelCase properties, PascalCase labels, SCREAMING_SNAKE_CASE relationships)
- [x] **BUILD-02**: Remove Italian-schema dead code from `ingest_stenografici.py` (save_to_neo4j, Italian constraints/indexes)
- [x] **BUILD-03**: Extract XML parser class from `ingest_stenografici.py` into standalone module (decouple from save logic)
- [x] **BUILD-04**: Remove redundant Chunk properties (`startCharRaw`, `endCharRaw`) and dead alignment_map code
- [x] **BUILD-05**: Remove redundant Speech property (`preprocessedText` — keep only `text` as preprocessed)
- [x] **BUILD-06**: Remove redundant Session property (`completeDate` — use only Neo4j Date `date`)
- [x] **BUILD-07**: Use UNWIND batch writes for bulk ingestion (replace per-item MERGE loops)
- [x] **BUILD-08**: Use managed transactions (`execute_read`/`execute_write`) instead of auto-commit `session.run()`
- [x] **BUILD-09**: Single `make db-all` target that rebuilds entire DB from scratch with fresh CSV downloads

### Data Extraction

- [x] **DATA-01**: Persist Vote nodes (renamed from Votazione) with HAS_VOTE relationship from Debate
- [x] **DATA-02**: Extract `<argomenti>` metadata to create Debate-[:DISCUSSES]->ParliamentaryAct edges
- [x] **DATA-03**: Extract speaker institutional role from `<emphasis>` tag into Speech.speakingRole property
- [x] **DATA-04**: Parse Phase title patterns into Phase.phaseType enum (e.g., "government_opinion", "vote_declaration", "general_discussion")

### Backend Services

- [x] **SVC-01**: Update all Cypher queries across 6+ modules to match new camelCase property names
- [x] **SVC-02**: Extract expert computation into `services/experts.py` (unify `chat.py` and `query.py` duplicate implementations)
- [x] **SVC-03**: Extract evaluation business logic into `services/evaluation_service.py` (decouple from router)
- [x] **SVC-04**: Replace `get_services()` dict with FastAPI `Depends()` typed dependency injection
- [x] **SVC-05**: Fix search.py duplicate Neo4j connection pool (use shared client from deps.py)
- [x] **SVC-06**: Clean naming, type hints, English docstrings across all service modules

### Backend API

- [x] **API-01**: Refactor routers to thin wrappers around services (no business logic in routers)
- [x] **API-02**: Fix cross-router import violations (evaluation.py→survey.py, seed script→chat.py)
- [x] **API-03**: Freeze SSE event contract (document all 18 yield sites, ensure no behavioral changes)
- [x] **API-04**: Clean endpoint naming, Pydantic v2 models, consistent error handling
- [x] **API-05**: Preserve API response shapes (frontend contract must not break)

### Backend Scripts

- [x] **SCR-01**: Refactor utility scripts with consistent naming, docstrings, error handling
- [x] **SCR-02**: Fix `seed_evaluation_topic.py` router-import coupling (use extracted service)
- [x] **SCR-03**: Scripts use shared Neo4j client instead of creating own connections

### Frontend

- [x] **FE-01**: Strict TypeScript — no `any` types, proper interfaces for all API responses
- [x] **FE-02**: Clean component structure — remove dead code, consistent naming
- [x] **FE-03**: English route paths and variable names where currently Italian

### Retrieval Enrichment

- [x] **RET-01**: Add BM25 sparse retrieval channel via Neo4j full-text index (zero new dependencies)
- [x] **RET-02**: Implement Reciprocal Rank Fusion (RRF) merger for hybrid dense+sparse+graph retrieval

### Graph Enrichment

- [x] **ENR-01**: SPARQL ingestion from dati.camera.it — per-deputy individual vote records
- [x] **ENR-02**: SPARQL ingestion from dati.camera.it — committee officer roles with dates
- [x] **ENR-03**: NER at ingestion time on chunks — extract LAW and PERSON entity references
- [x] **ENR-04**: Store NER results as Chunk properties (lawRefs, personRefs) for entity-filtered retrieval

### Code Quality

- [x] **QA-01**: Add smoke tests for critical paths (build pipeline, retrieval, evaluation) — zero current coverage
- [x] **QA-02**: Python code follows best practices: type hints, English docstrings, no dead code
- [x] **QA-03**: Consistent naming conventions across entire codebase (camelCase Neo4j, snake_case Python, camelCase TypeScript)

### Multi-language Support

- [x] **ML-01**: i18n infrastructure with next-intl or equivalent — Italian (default) + English locale files
- [x] **ML-02**: Extract all hardcoded Italian UI text to translation keys across all pages and components
- [x] **ML-03**: On-the-fly citation translation via OpenAI when user language ≠ Italian
- [ ] **ML-04**: Tooltip hover on translated citations showing original Italian text
- [ ] **ML-05**: Dual-layer disclaimer: dismissable banner + permanent globe icon (🌐) on translated citations

### Senate Data Integration

- [ ] **SEN-01**: Dedicated Senate XML parser (`senate_parser.py`) producing same output format as Camera parser
- [ ] **SEN-02**: Senate data download and ingestion into Neo4j with `chamber: "senato"` on all nodes
- [ ] **SEN-03**: Chamber selector UI component (Camera / Senato / Both) above chat input, default "Both"
- [x] **SEN-04**: All retrieval channels (dense, sparse, graph) filter by `chamber` when not "Both"
- [ ] **SEN-05**: `make db-senate` target for Senate-only build, `make db-all` builds both chambers

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
| BUILD-01 | Phase 1 | Complete |
| BUILD-02 | Phase 1 | Complete |
| BUILD-03 | Phase 1 | Complete |
| BUILD-04 | Phase 1 | Complete |
| BUILD-05 | Phase 1 | Complete |
| BUILD-06 | Phase 1 | Complete |
| BUILD-07 | Phase 1 | Complete |
| BUILD-08 | Phase 1 | Complete |
| BUILD-09 | Phase 1 | Complete |
| DATA-01 | Phase 1 | Complete |
| DATA-02 | Phase 1 | Complete |
| DATA-03 | Phase 1 | Complete |
| DATA-04 | Phase 1 | Complete |
| SVC-01 | Phase 2 | Complete |
| SVC-02 | Phase 2 | Complete |
| SVC-03 | Phase 2 | Complete |
| SVC-04 | Phase 2 | Complete |
| SVC-05 | Phase 2 | Complete |
| SVC-06 | Phase 2 | Complete |
| API-01 | Phase 2 | Complete |
| API-02 | Phase 2 | Complete |
| API-03 | Phase 2 | Complete |
| API-04 | Phase 2 | Complete |
| API-05 | Phase 2 | Complete |
| SCR-01 | Phase 2 | Complete |
| SCR-02 | Phase 2 | Complete |
| SCR-03 | Phase 2 | Complete |
| QA-01 | Phase 2 | Complete |
| QA-02 | Phase 2 | Complete |
| QA-03 | Phase 2 | Complete |
| FE-01 | Phase 3 | Complete |
| FE-02 | Phase 3 | Complete |
| FE-03 | Phase 3 | Complete |
| RET-01 | Phase 4 | Complete |
| RET-02 | Phase 4 | Complete |
| ENR-01 | Phase 4 | Complete |
| ENR-02 | Phase 4 | Complete |
| ENR-03 | Phase 4 | Pending |
| ENR-04 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 39 total
- Mapped to phases: 39
- Unmapped: 0

---
*Requirements defined: 2026-04-02*
*Last updated: 2026-04-02 — Traceability updated to match 4-phase coarse roadmap*
