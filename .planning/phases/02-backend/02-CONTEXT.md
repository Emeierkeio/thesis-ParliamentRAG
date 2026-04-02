# Phase 2: Backend - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Clean all backend layers (services, routers, scripts) and deploy atomically with the Phase 1 Neo4j schema. Update all Cypher queries to match the new English-only camelCase schema, extract duplicated business logic into dedicated service modules, fix FastAPI dependency injection, resolve cross-layer coupling violations, add comprehensive unit tests, and expose new data (Votes, DISCUSSES, speakingRole, phaseType) via basic API endpoints.

</domain>

<decisions>
## Implementation Decisions

### Cypher Query Updates

- **Remove `start_char_raw`/`end_char_raw` completely** from all Cypher queries, Pydantic models, and API responses. The frontend does not use `span_start`/`span_end`. Do not keep as 0 placeholder — remove the fields entirely.
- **`i.text` is now preprocessed only** — update silently. Remove any comments referencing "raw text" or "preprocessed_text" distinction. `sp.text` / `i.text` is simply "the speech text".
- **Expose new schema data via basic endpoints:**
  - GET endpoint for votes (linked to sessions)
  - GET endpoint for act-debate links (DISCUSSES relationships)
  - speakingRole and phaseType available in existing query responses where Speech/Phase are returned
- **Blast radius files for Cypher updates:** `neo4j_client.py`, `graph_channel.py`, `evidence.py`, `query.py`, `chat.py`, `search.py`, `scorer.py`, `evaluation.py`, `compute_baseline_experts.py`, `enrich_evaluation_set.py`

### Service Extraction

- **Two new service modules:**
  - `backend/app/services/experts.py` — Unified expert computation (merge `_compute_experts_for_frontend()` from chat.py and `_compute_experts()` from query.py into one)
  - `backend/app/services/evaluation_service.py` — Evaluation business logic extracted from evaluation.py router
- **FastAPI DI migration:** Replace `get_services()` dict in deps.py with typed `Depends()` functions using `@lru_cache` for singleton semantics. Standard FastAPI pattern.
- **Thin routers:** Routers contain ONLY input validation + service call + output formatting. Zero business logic in routers. All query building, score computation, expert matching moves to services.
- **Fix coupling violations:**
  - evaluation.py must NOT import from survey.py router
  - seed_evaluation_topic.py must NOT import from chat.py router
  - search.py must use shared Neo4j client from deps.py (no duplicate connection pool)

### SSE Event Contract

- **Document ALL 18 event types BEFORE touching query.py** — create an SSE contract document listing every event name, payload shape, and emission order
- **Event types and payload field names are FROZEN** — no renames, no restructuring
- **Keep snake_case in SSE payloads** — `authority_score`, `group`, `first_name` etc. stay as-is. The frontend reads these. Changing them breaks the contract.
- **Keep dual `experts` emission** — first before generation, second after citation verification. The frontend overwrites first with second. This is intentional and must not change.
- **SSE logic can move to a service** but the yield points and event structure must remain identical

### Smoke Tests

- **Comprehensive unit tests** — not just smoke tests. Full coverage of service functions, with mocked Neo4j driver for logic tests.
- **Test location:** `backend/tests/` (separate directory in backend root)
- **Strategy:** Mix of mock + integration:
  - Unit tests with mock Neo4j driver for all service logic
  - Integration tests with `@pytest.mark.integration` for real Cypher queries
  - Integration tests skipped by default (need running Neo4j)
- **Target:** ~100+ tests covering services, routers (via TestClient), and scripts
- **Critical paths to cover:**
  - Expert computation (the unified service)
  - Evaluation metrics computation
  - Retrieval pipeline (dense + graph channels)
  - Authority scoring
  - SSE event emission order and payload shapes

### Claude's Discretion
- Exact module file layout within `backend/app/services/`
- How to structure the SSE contract document
- Test fixture design and conftest.py organization
- Order of refactoring within the phase (which files first)
- Whether to use pytest-asyncio for async endpoint tests
- How much of the generation pipeline to refactor vs leave as-is

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Backend services (being refactored)
- `backend/app/services/deps.py` — Current DI pattern (get_services dict) being replaced
- `backend/app/services/neo4j_client.py` — Vector search Cypher with dead properties
- `backend/app/services/retrieval/graph_channel.py` — Graph retrieval with dead properties
- `backend/app/services/retrieval/engine.py` — Retrieval orchestrator
- `backend/app/services/authority/scorer.py` — Authority scoring with text_embedding queries
- `backend/app/services/authority/components.py` — Authority score components [0,1]

### Backend routers (being refactored)
- `backend/app/routers/query.py` — SSE streaming, expert computation, 18 yield sites
- `backend/app/routers/chat.py` — Duplicate expert computation
- `backend/app/routers/evaluation.py` — Evaluation dashboard with cross-router imports
- `backend/app/routers/evidence.py` — Evidence detail with dead Cypher properties
- `backend/app/routers/search.py` — Search with duplicate Neo4j pool
- `backend/app/routers/survey.py` — Survey with imports from evaluation

### Backend scripts (being refactored)
- `backend/scripts/compute_baseline_experts.py` — Baseline experts with Cypher queries
- `backend/scripts/enrich_evaluation_set.py` — Evaluation enrichment with Cypher queries
- `backend/scripts/seed_evaluation_topic.py` — Seeds with chat.py router import

### Models
- `backend/app/models/evidence.py` — UnifiedEvidence with span_start/span_end to remove

### Phase 1 outputs (new schema reference)
- `build/db_builder.py` — Canonical Neo4j schema (property names, relationships, labels)
- `build/xml_parser.py` — Data structures produced by parser
- `.planning/phases/01-build-pipeline/01-CONTEXT.md` — Schema design decisions

### Research
- `.planning/research/ARCHITECTURE.md` — Coupling violations, dependency graph
- `.planning/research/PITFALLS.md` — SSE contract risks, evaluation fragility

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `backend/app/services/neo4j_client.py` — Neo4jClient class with query/vector_search methods. Keep the class, update Cypher strings.
- `backend/app/models/` — Pydantic models for all response types. Update field lists (remove span_start/span_end).
- `backend/config/default.yaml` — Config weights and thresholds. Keep as-is.

### Established Patterns
- SSE streaming via `StreamingResponse` + generator in query.py — this pattern stays, just the generator body may move to a service
- Authority scoring uses query embedding similarity — this pattern is correct, just needs property name updates
- Evaluation uses ChatHistory nodes stored in Neo4j — field names (`first_name`, `last_name`, `group`, `authority_score`) are frozen in stored data

### Integration Points
- Frontend reads SSE events via `use-chat.ts` — frozen contract
- Frontend reads REST responses via fetch — response shapes must not change
- `evaluation_set.json` stores baseline experts — field names frozen in stored data
- Build pipeline produces the DB that backend queries — schema is now Phase 1 output

</code_context>

<specifics>
## Specific Ideas

- The user wants "expert-level" code quality — clean, explainable, best-practice Python
- Comprehensive test coverage (~100+ tests), not just smoke tests
- New basic endpoints for Votes and DISCUSSES data, even if frontend doesn't consume them yet
- The evaluation system is the riskiest refactoring target (400+ lines, multiple historical bug fixes) — handle with extra care

</specifics>

<deferred>
## Deferred Ideas

- BM25 sparse retrieval channel — Phase 4 (RET-01)
- SPARQL enrichment from dati.camera.it — Phase 4 (ENR-01, ENR-02)
- NER entity extraction — Phase 4 (ENR-03, ENR-04)
- Frontend refactoring — Phase 3

</deferred>

---

*Phase: 02-backend*
*Context gathered: 2026-04-02*
