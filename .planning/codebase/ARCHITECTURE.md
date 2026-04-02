# Architecture

**Analysis Date:** 2026-04-01

## Pattern Overview

**Overall:** Multi-View Retrieval-Augmented Generation (RAG) system with dual-channel retrieval, query-dependent authority scoring, and iterative generation pipeline with citation integrity.

**Key Characteristics:**
- Modular service architecture with clear separation of concerns (retrieval, authority, generation, evaluation)
- Dual-channel retrieval: dense vector search + graph-based evidence discovery
- Query-specific authority scoring (not global) applied per-retrieval
- 4-stage generation pipeline with citation integrity verification
- SSE streaming responses for real-time UI feedback
- Both synchronous and asynchronous pipeline execution paths

## Layers

**Presentation Layer:**
- Purpose: User interface and real-time interaction
- Location: `frontend/src/app/`, `frontend/src/components/`
- Contains: Next.js pages, React components, hooks for chat/survey/evaluation
- Depends on: Frontend API client (`frontend/src/lib/api.ts`), types (`frontend/src/types/`)
- Used by: End users via browser

**API Layer:**
- Purpose: HTTP endpoints for chat, search, evaluation, and configuration
- Location: `backend/app/routers/` (query.py, chat.py, history.py, evaluation.py, survey.py, etc.)
- Contains: FastAPI routers with request/response models
- Depends on: Service layer, middleware, configuration
- Used by: Frontend, external integrations

**Service Layer:**
- Purpose: Core business logic encapsulation
- Location: `backend/app/services/`
- Contains: Specialized subsystems
  - `retrieval/` - Dual-channel retrieval orchestration
  - `generation/` - 4-stage generation pipeline
  - `authority/` - Query-dependent authority scoring
  - `compass/` - Ideological positioning analysis
  - `citation/` - Citation extraction and verification
  - `neo4j_client.py` - Database abstraction
  - `deps.py` - Lazy service initialization

**Data Layer:**
- Purpose: Database access and data structures
- Location: `backend/app/models/`, Neo4j database
- Contains: Pydantic models, Neo4j queries, evidence schema
- Depends on: Neo4j client, configuration

## Data Flow

**Query Pipeline:**

1. **User Query Entry** → `POST /api/chat` or `POST /api/query`
   - Location: `backend/app/routers/chat.py::process_query_streaming`
   - Validates request, assigns task_id, acquires pipeline semaphore

2. **Query Embedding** → `RetrievalEngine.embed_query()`
   - Calls OpenAI text-embedding-3-small API
   - Returns 1536-dimensional embedding vector

3. **Dual-Channel Retrieval** → `RetrievalEngine.retrieve_sync()`
   - **Dense Channel**: Vector similarity search on Neo4j chunks
     - Location: `backend/app/services/retrieval/dense_channel.py`
     - Returns top-k matches by cosine similarity
   - **Graph Channel**: Metadata-driven retrieval via acts, commission references
     - Location: `backend/app/services/retrieval/graph_channel.py`
     - Returns evidence linked through semantic relationships
   - **Merger**: Combines results using weighted channel scores
     - Location: `backend/app/services/retrieval/merger.py`
     - Deduplicates and ranks by unified score

4. **Authority Scoring** → `AuthorityScorer.score_evidence()`
   - Location: `backend/app/services/authority/scorer.py`
   - **Per-query** computation (not global)
   - Components (each [0,1]): ProfessionComponent, EducationComponent, CommitteeComponent, ActsComponent, InterventionsComponent, RoleComponent
   - Location: `backend/app/services/authority/components.py`
   - Stored in evidence objects before generation

5. **Expert Selection** → `_compute_experts()` in `backend/app/routers/query.py`
   - Picks top-authority speaker per party from retrieved pool
   - Stores in `chatData.experts` (query-specific, one per party)
   - Later updated if cited speakers differ (expert-update block)

6. **Ideology Scoring** → `IdeologyScorer.score_evidence()`
   - Location: `backend/app/services/compass/scorer.py`
   - Computes left/center/right positioning per speaker/chunk
   - Contributes to ideological compass visualization

7. **Generation Pipeline** → `GenerationPipeline.generate()`
   - Location: `backend/app/services/generation/pipeline.py`

   **Stage 1 (Analyst)**: Query → Atomic claims
   - Location: `backend/app/services/generation/analyst.py`
   - Decomposes query into evidence requirements

   **Stage 2 (Sectional Writer)**: Claims + Evidence → Per-party sections
   - Location: `backend/app/services/generation/sectional.py`
   - Generates parallel sections for each party's viewpoint

   **Stage 3 (Integrator)**: Sections → Coherent narrative
   - Location: `backend/app/services/generation/integrator.py`
   - Merges sections into unified response with guard against citation loss

   **Stage 4 (Surgeon)**: Narrative → Verbatim citations
   - Location: `backend/app/services/generation/surgeon.py`
   - Extracts exact quotes using offset-based extraction (NO fuzzy matching)

8. **Citation Integrity System** → Throughout pipeline
   - **Citation Registry**: Tracks citations through all stages
     - Location: `backend/app/services/generation/citation_registry.py`
   - **Coherence Validator**: Verifies semantic alignment
     - Location: `backend/app/services/generation/coherence_validator.py`
   - **Sentence Extractor**: Offset-based quote extraction
     - Location: `backend/app/services/citation/sentence_extractor.py`

9. **Response Assembly** → SSE events
   - Streams progress events, then metadata, experts, citations, final text
   - Location: `backend/app/routers/chat.py::_stream_response_wrapper`
   - Frontend listens via EventSource and updates UI progressively

**State Management:**
- Chat state: `frontend/src/hooks/use-chat.ts` maintains message history, loading state, progress
- Neo4j state: Persistent storage of parliamentary records, chunks, speaker data, authority scores (pre-computed)
- Config state: Loaded at startup from `backend/config/default.yaml`, reloadable via `GET /api/config/reload`

## Key Abstractions

**UnifiedEvidence:**
- Purpose: Standardized evidence record with query-specific authority
- Location: `backend/app/models/evidence.py`
- Schema: chunk_id, speaker_name, party, authority_score, quote_text (verbatim), chunk_text (preview)
- Critical: quote_text is the ONLY valid citation source (offset-verified)

**Message (Frontend):**
- Purpose: Unified type for user/assistant messages with streaming metadata
- Location: `frontend/src/types/chat.ts`
- Contains: citations[], experts[], compass, balanceMetrics, topicStats
- Accumulates as streaming events arrive

**Expert:**
- Purpose: Per-party authority spokesperson on a query topic
- Location: `frontend/src/types/chat.ts`, populated from backend
- Fields: speaker_id, authority_score, score_breakdown (by component), relevant_speeches_count
- Stored as one per party in chatData.experts

**IdeologyScore:**
- Purpose: Left/center/right positioning for a chunk or speaker
- Location: `backend/app/models/evidence.py`
- Computed per query (not global); influences compass visualization

## Entry Points

**Backend HTTP Entry Points:**

**POST /api/chat:**
- Location: `backend/app/routers/chat.py::stream_chat`
- Triggers: User sends query from chat interface
- Responsibilities:
  - Rate-limit via pipeline semaphore
  - Call process_query_streaming with SSE response
  - Send progress, experts, compass, citations, final text as separate events

**POST /api/query:**
- Location: `backend/app/routers/query.py::query_endpoint`
- Alternative synchronous query entry point
- Less commonly used (chat endpoint preferred)

**POST /api/search:**
- Location: `backend/app/routers/search.py`
- Triggers: User searches parliamentary records
- Returns: Ranked acts/interventions by relevance

**GET /api/history:**
- Location: `backend/app/routers/history.py`
- Retrieves: Past conversations and baseline comparison

**POST /api/evaluation:**
- Location: `backend/app/routers/evaluation.py`
- Triggers: User submits A/B evaluation on survey modal
- Stores: Rating comparisons (system vs baseline)

**GET /api/config:**
- Location: `backend/app/routers/config.py`
- Returns: Current system configuration (retrieval weights, authority thresholds, generation params)
- Allows: Dynamic tuning via PUT

**GET /api/compass:**
- Location: `backend/app/routers/compass.py`
- Computes: Standalone ideological compass visualization

**Frontend Entry Points:**

**`/` (Home):**
- Location: `frontend/src/app/page.tsx`
- Renders: Chat interface with sidebar, message list, input
- Triggers: SSE streaming via useChat hook

**`/valutazione` (Evaluation):**
- Location: `frontend/src/app/valutazione/page.tsx`
- Renders: Dashboard with A/B ratings, win rates, metrics
- Triggers: getDashboardData() on load

**`/search`:**
- Location: `frontend/src/app/search/page.tsx`
- Renders: Parliamentary record search UI
- Triggers: Search API via useChat

**`/compass`:**
- Location: `frontend/src/app/compass/page.tsx`
- Renders: Standalone ideological compass visualization

**`/ranking`:**
- Location: `frontend/src/app/ranking/page.tsx`
- Renders: Expert authority rankings by topic

## Error Handling

**Strategy:** Multi-level validation with graceful degradation

**Patterns:**

- **Query Validation**: Min 3 chars, max 1000 chars enforced at API level
  - Location: `backend/app/routers/query.py::QueryRequest`

- **Neo4j Failures**: Logged, falls back to empty results
  - Retrieval fails gracefully, generation outputs "no evidence" message
  - Location: `backend/app/routers/chat.py::process_query_streaming` (try/except wraps pipeline)

- **LLM Failures** (OpenAI API): Caught and returned as error event
  - Location: `backend/app/services/generation/pipeline.py` (retry logic with exponential backoff)

- **Citation Integrity Failures**: Citation validity checked post-surgery
  - Offsets must resolve to exact verbatim text
  - Failed citations logged but do not block response
  - Location: `backend/app/services/generation/surgeon.py`

- **Rate Limiting**: Semaphore-based (not per-IP)
  - Max concurrent pipelines: 5 per worker (configurable via env var)
  - Waiting clients notified via "waiting" SSE event
  - Location: `backend/app/routers/chat.py::_pipeline_semaphore`

- **Task Cancellation**: Frontend can cancel via POST /api/chat/cancel
  - Location: `backend/app/services/task_store.py`
  - Checked during generation stages

## Cross-Cutting Concerns

**Logging:**
- **Approach**: Hierarchical file-based + console
  - Two rotating log files per startup: app_TIMESTAMP.log (INFO+), debug_TIMESTAMP.log (DEBUG+)
  - Noisy third-party libs (httpx, urllib3, openai) silenced to WARNING
  - Active investigation modules set to DEBUG explicitly
  - Location: `backend/app/main.py::setup_logging()`

**Validation:**
- **Approach**: Pydantic models at API boundary
  - QueryRequest, ChatRequest, EvaluationRequest all use Pydantic
  - Custom validators for date ranges, embedding dimensions
  - Location: `backend/app/routers/*.py`

**Authentication:**
- **Approach**: None implemented (local/closed system)
- Note: CORS enabled for all origins (not production-safe)
- Location: `backend/app/main.py::CORSMiddleware`

**Performance:**
- **Warmup**: Neo4j vector index warmed on startup (eliminates 15-18s cold start)
  - Location: `backend/app/main.py::_warmup_neo4j_index()`
- **Concurrency**: ThreadPoolExecutor in generation pipeline, async/await in routers
  - Location: `backend/app/services/generation/pipeline.py`, `backend/app/routers/chat.py`
- **Caching**: Config loaded once at startup, reloadable via endpoint
  - Query embeddings not cached (query-specific authority requires fresh computation)
  - Location: `backend/app/config.py::ConfigLoader`

---

*Architecture analysis: 2026-04-01*
