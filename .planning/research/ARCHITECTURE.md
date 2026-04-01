# Architecture Patterns

**Domain:** Parliamentary RAG system — full codebase refactoring
**Researched:** 2026-04-02

---

## 1. Current Architecture Map

### Top-Level Directory Layout

```
ParliamentRAG/
├── build/                  # DB build pipeline (standalone Python scripts)
├── backend/
│   ├── app/
│   │   ├── main.py         # FastAPI entrypoint
│   │   ├── config.py       # Settings + YAML loader
│   │   ├── key_pool.py     # OpenAI key round-robin
│   │   ├── models/         # Pydantic models (6 files)
│   │   ├── routers/        # FastAPI routers (11 files)
│   │   └── services/       # Business logic (5 subpackages + 4 top-level files)
│   ├── scripts/            # Utility scripts (5 files)
│   └── config/             # YAML config files
├── frontend/
│   └── src/
│       ├── app/            # Next.js App Router pages (7 pages)
│       ├── components/     # React components (7 subfolders)
│       ├── hooks/          # Custom hooks (3 hooks)
│       ├── lib/            # API clients (5 files)
│       ├── types/          # TypeScript types (5 files)
│       └── config/         # Frontend config
└── data/                   # Downloaded XML + CSV sources
```

### Build Pipeline (`build/`)

| File | Role | Status |
|------|------|--------|
| `build_and_update.py` | Main orchestrator: full rebuild or incremental update. English schema. Production path. | Active |
| `ingest_stenografici.py` | `StenograficoIngester` class — XML parsing, text preprocessing, chunking. Has Italian `save_to_neo4j()` dead path. | Partially active (parser reused) |
| `ingest_atti_parlamentari.py` | `AttiParlamentariIngester` — Camera API, parliamentary acts. | Active |
| `app_config.py` | Italian-language role mapping: `GOVERNMENT_ROLES`, `PARLIAMENT_ROLES`, `CAPIGRUPPO`, `COMMISSION_ROLES`. | Active |
| `initialize_db.py` | Constraints + indexes setup. | Active |
| `neo4j_helper.py` | Low-level Neo4j utility (used by older scripts). | Partially used |
| `embedding_service.py` | SQLite embedding cache (`embeddings_cache.db`, 344 MB). | Active |
| `precalculate_embeddings.py` | Batch pre-calculation of all node embeddings. | Active |
| `precalculate_baseline_experts.py` | Pre-computes baseline expert panel for eval set. | Active |
| `create_vector_index.py` | Creates `chunk_embedding_index` vector index. | Active |
| `download_deputies_csv.py` | Downloads deputy CSV from dati.camera.it. | Active |
| `download_new_xmls.py` | Downloads new stenographic XML files from Camera API. | Active |
| `populate_ruoli.py` | Assigns institutional roles from `app_config.py` to Neo4j nodes. | Active |
| `migrate_foti.py` | One-shot migration: moves Foti from Deputy to GovernmentMember. | Dead code (one-shot complete) |

**Critical build issue:** `build_and_update.py` imports `StenograficoIngester` from `ingest_stenografici.py` for the XML parser only. `ingest_stenografici.py` also contains a full Italian-schema `save_to_neo4j()` method and Italian constraints/indexes (`Seduta`, `Dibattito`, `Fase`, `Intervento`, `Votazione`) — this code is dead in production but still executes if the file is imported carelessly.

### Backend Services (`backend/app/services/`)

| Subpackage | Files | Role |
|------------|-------|------|
| `retrieval/` | `engine.py`, `dense_channel.py`, `graph_channel.py`, `merger.py`, `query_rewriter.py`, `commission_matcher.py` | Dual-channel retrieval: vector search + graph traversal |
| `generation/` | `pipeline.py`, `analyst.py`, `sectional.py`, `integrator.py`, `surgeon.py`, `synthesis.py`, `citation_registry.py`, `coherence_validator.py`, `evidence_first_writer.py`, `position_brief.py`, `reported_speech.py` | 4-stage LLM generation pipeline |
| `authority/` | `scorer.py`, `components.py`, `coalition_logic.py` | Query-dependent authority scoring |
| `compass/` | `pipeline.py`, `scorer.py`, `anchors.py`, `clustering.py`, `axis_labeling.py`, `reference_axes.py` | Ideological positioning (2D PCA on text embeddings) |
| `citation/` | `sentence_extractor.py` | Sentence boundary extraction, chunk salience scoring |
| `neo4j_client.py` | — | Neo4j connection pool, singleton pattern |
| `deps.py` | — | Lazy singleton initialization for all services |
| `task_store.py` | — | In-memory task cancellation registry |
| `log_service.py` | — | (Unused in middleware, referenced but not hooked in) |

### Backend Routers (`backend/app/routers/`)

| Router | Prefix | Role |
|--------|--------|------|
| `chat.py` | `/api` | Main frontend-facing SSE endpoint. Contains `_compute_experts_for_frontend()` — a large helper that also handles expert post-update after citation verification. |
| `query.py` | `/api` | Alternative query endpoint. Contains duplicate expert computation logic (`_compute_experts`) and `_fetch_speaker_details`. |
| `history.py` | `/api` | Chat history CRUD in Neo4j. Contains `_strip_markdown()` and `baseline-experts` endpoint. |
| `evaluation.py` | `/api/evaluation` | Dashboard metrics. Contains `KNOWN_PARTIES`, `PARTY_KEYWORDS`, and all metric computation logic inline. |
| `survey.py` | `/api` | A/B survey storage and retrieval. |
| `authority.py` | `/api/authority` | Ranking by topic. |
| `evidence.py` | `/api/evidence` | Evidence detail lookup. References `start_char_raw`/`end_char_raw`. |
| `search.py` | `/api/search` | Full-text + vector search. Has its own Neo4j client singleton (duplicates `deps.py`). |
| `config.py` | `/api/config` | YAML config read/write. |
| `graph.py` | `/api/graph` | Graph exploration endpoint. |
| `compass.py` | `/api/compass` | Standalone compass. |

### Backend Models (`backend/app/models/`)

| File | Content |
|------|---------|
| `evidence.py` | `UnifiedEvidence`, `IdeologyScore`, `PARTY_DISPLAY_NAMES`, name normalizer functions |
| `query.py` | `QueryRequest`, `QueryResponse` (partially duplicated in routers) |
| `authority.py` | Authority component models |
| `evaluation.py` | `AutomatedMetrics`, `AggregatedMetrics`, `ABComparisonStats`, `CombinedEvaluation`, `EvaluationDashboardData` |
| `survey.py` | `SurveyResponse`, `SurveyStats`, `SimpleRatingResponse` |
| `compass.py` | Compass output models |

### Backend Scripts (`backend/scripts/`)

| File | Dependencies | Role |
|------|-------------|------|
| `compute_baseline_experts.py` | `app.config`, `app.services.neo4j_client`, `app.services.authority.scorer`, `app.services.authority.coalition_logic`, `app.services.retrieval.engine` | Pre-computes baseline expert panel for evaluation set |
| `compute_topic_authority_spread.py` | `app.config`, `app.services.neo4j_client`, `app.services.authority.scorer`, `app.services.retrieval.engine` | Authority spread statistics per topic |
| `seed_evaluation_topic.py` | `app.services.deps`, `app.routers.chat._compute_experts_for_frontend` | Seeds evaluation_set.json with new topics. **Imports router internals — coupling violation.** |
| `enrich_evaluation_set.py` | `app.config`, `app.services.neo4j_client` | Enriches eval set with baseline metrics |
| `export_notebooklm.py` | Standalone | Exports transcripts for NotebookLM |

### Frontend (`frontend/src/`)

| Directory | Content |
|-----------|---------|
| `app/` | Next.js App Router: `page.tsx` (home/chat), `chat/[id]/page.tsx` (shared link), `search/page.tsx`, `explorer/page.tsx`, `ranking/page.tsx`, `compass/page.tsx`, `valutazione/page.tsx` (eval dashboard) |
| `components/chat/` | `ChatArea`, `ChatInput`, `MessageBubble`, `CitationCard`, `ExpertCard`, `CompassCard`, `TopicStatsModal` |
| `components/survey/` | `SurveyModal`, `CitationReviewStep`, `StarRating` |
| `components/evaluation/` | `EvaluationCharts` |
| `components/search/` | `DeputySelector`, `GroupSelector`, `ResultsList`, `ResultDetailDialog` |
| `components/shared/` | `HistoryModal`, `ProgressIndicator` |
| `components/settings/` | `SettingsModal`, `GraphicalEditors` |
| `components/layout/` | `Sidebar` |
| `components/graph/` | `GraphVisualizer` |
| `hooks/` | `use-chat.ts`, `use-local-history.ts`, `use-sidebar.ts` |
| `lib/` | `api.ts` (config API), `evaluation-api.ts`, `graph-api.ts`, `survey-api.ts`, `utils.ts`, `constants.ts` |
| `types/` | `chat.ts`, `api.ts`, `evaluation.ts`, `survey.ts`, `index.ts` |
| `app/api/` | Next.js API proxy routes (pass-through to backend) |

---

## 2. Problems Found in Current Architecture

### Build Pipeline

**P1: Two ingestion paths with a shared class boundary.**
`build_and_update.py` (English schema, production) imports `StenograficoIngester` from `ingest_stenografici.py` which also contains Italian-schema `save_to_neo4j()`, Italian constraints (`Seduta`, `Dibattito`, etc.), and Italian indexes. These are dead in production but still in the codebase.

**P2: Dead properties throughout the backend.**
`start_char_raw` and `end_char_raw` on `Chunk` nodes are referenced in 12 Cypher queries across 6 backend files (`evidence.py`, `neo4j_client.py`, `chat.py` x2, `engine.py` x2, `graph_channel.py`, `dense_channel.py`) and in `backend/scripts`. The properties have known off-by-one errors and are documented as unused by the backend — but the backend queries them anyway, silently getting wrong offsets and falling back to `chunk_text`.

**P3: `migrate_foti.py` is one-shot dead code.**
The Foti migration has already run. The file should be removed.

**P4: Mixed Italian/English in build config.**
`app_config.py` (build) uses Italian keys (`GOVERNMENT_ROLES`, `CAPIGRUPPO`, etc.) and Italian values. The backend `app/config.py` is English. These are separate files but the build depends on `app_config.py` for role assignment.

**P5: `alignment_map` dead logic in chunking.**
`ingest_stenografici.py` computes `alignment_map` during preprocessing. This map was the basis for `start_char_raw`/`end_char_raw` — which are being removed. The alignment map computation is dead once the properties are removed.

### Backend Services

**P6: Duplicate Neo4j client singletons.**
`search.py` router maintains its own `_neo4j_client` global instead of using `deps.py`. This creates a second Neo4j connection pool in the same process.

**P7: Expert computation split across two routers.**
`chat.py` has `_compute_experts_for_frontend()` (the real implementation, ~200 lines). `query.py` has `_compute_experts()` (a parallel implementation). `seed_evaluation_topic.py` imports directly from `chat.py` router internals — a scripts→routers coupling violation.

**P8: Evaluation router contains domain logic inline.**
`evaluation.py` contains `KNOWN_PARTIES`, `PARTY_KEYWORDS`, and all metric computation functions as module-level code inside a router file. This logic belongs in a service.

**P9: `history.py` contains a `baseline-experts` endpoint.**
The baseline-experts endpoint retrieves pre-computed baseline expert data from `evaluation_set.json`. This is evaluation-domain logic living in the history router.

**P10: Italian strings in production code.**
`main.py` logging setup comments are Italian. `maintenance_middleware` returns Italian error messages. `query.py` SSE messages are Italian (`"Avvio retrieval..."`, `"Trovate {n} evidenze"`, etc.). Italian is present throughout routers.

**P11: Pydantic models duplicated in routers.**
`QueryRequest`, `CitationInfo`, `ExpertInfo`, `QueryResponse` are defined in `query.py` rather than in `models/query.py`. Similar duplication in `chat.py`.

**P12: `log_service.py` exists but is not wired.**
The middleware directory has `analytics.py` cached but no middleware is registered in `main.py` beyond CORS and the maintenance check.

### Backend Scripts

**P13: `seed_evaluation_topic.py` imports router internals.**
`from app.routers.chat import _compute_experts_for_frontend` — a private function exported from a router. Expert computation should live in a service, not a router.

### Frontend

**P14: `valutazione` (Italian) page name.**
The evaluation dashboard page is at `/valutazione` — Italian URL in an otherwise mixed-language codebase. The refactoring should decide on English-only URLs.

**P15: Italian type comments.**
`types/chat.ts` has Italian inline comments (`"Metadati opzionali per le risposte dell'assistente"`, etc.).

**P16: `[key: string]: any` on Citation type.**
The `Citation` interface has an escape-hatch index signature that bypasses TypeScript strictness.

---

## 3. Dependency Graph

Dependencies flow strictly downward. Arrows show "depends on".

```
build/
  build_and_update.py
    → ingest_stenografici.StenograficoIngester  (parser only)
    → ingest_atti_parlamentari.AttiParlamentariIngester
    → app_config.py                              (role assignment)
    → initialize_db.py                           (constraints)
    → embedding_service.py                       (embeddings cache)

backend/app/
  main.py
    → config.py
    → routers/* (all 11 routers)
    → services/deps.py                           (via lifespan warmup)

  routers/
    chat.py
      → services/deps.py                         (get_services)
      → services/neo4j_client.py
      → services/compass/
      → services/retrieval/commission_matcher.py
      → services/task_store.py
      → config.py, key_pool.py

    query.py
      → services/neo4j_client.py
      → services/authority/coalition_logic.py
      → services/deps.py
      → config.py

    history.py
      → services/neo4j_client.py (get_neo4j_client)

    evaluation.py
      → models/evaluation.py
      → models/survey.py
      → routers/survey.py (_load_surveys, _calculate_stats)  [cross-router import]

    survey.py
      → services/neo4j_client.py (get_neo4j_client)

    authority.py
      → services/deps.py

    evidence.py
      → services/neo4j_client.py
      → config.py

    search.py
      → services/neo4j_client.py                [own singleton, not via deps]
      → config.py, key_pool.py

    config.py, graph.py, compass.py
      → services/deps.py or services/neo4j_client.py

  services/
    deps.py
      → neo4j_client.py
      → retrieval/ (RetrievalEngine)
      → authority/ (AuthorityScorer)
      → compass/ (IdeologyScorer)
      → generation/ (GenerationPipeline)

    retrieval/engine.py
      → neo4j_client.py
      → key_pool.py
      → retrieval/dense_channel.py
      → retrieval/graph_channel.py
      → retrieval/merger.py
      → retrieval/query_rewriter.py
      → models/evidence.py
      → config.py
      → citation/sentence_extractor.py          [cross-domain: retrieval→citation]

    retrieval/dense_channel.py
      → neo4j_client.py
      → models/evidence.py
      → config.py

    retrieval/graph_channel.py
      → neo4j_client.py
      → models/evidence.py
      → config.py

    generation/pipeline.py
      → generation/analyst.py
      → generation/sectional.py
      → generation/integrator.py
      → generation/surgeon.py
      → generation/synthesis.py
      → generation/citation_registry.py
      → generation/coherence_validator.py
      → config.py
      → citation/sentence_extractor.py

    generation/sectional.py
      → config.py, key_pool.py
      → citation/ (extract_best_sentences, compute_chunk_salience)
      → generation/position_brief.py
      → generation/reported_speech.py

    authority/scorer.py
      → neo4j_client.py
      → authority/coalition_logic.py
      → authority/components.py
      → config.py

  models/
    evidence.py     (no app dependencies — leaf node)
    query.py        (no app dependencies)
    authority.py    (no app dependencies)
    evaluation.py   (no app dependencies)
    survey.py       (no app dependencies)
    compass.py      (no app dependencies)

  config.py         (no app dependencies — leaf node)
  key_pool.py
    → config.py

backend/scripts/
  compute_baseline_experts.py
    → app.config
    → app.services.neo4j_client
    → app.services.authority.scorer
    → app.services.authority.coalition_logic
    → app.services.retrieval.engine

  seed_evaluation_topic.py
    → app.services.deps                          [acceptable]
    → app.routers.chat._compute_experts_for_frontend  [VIOLATION: script→router]

frontend/
  app/*/page.tsx
    → components/*
    → hooks/*
    → lib/api.ts, lib/evaluation-api.ts, etc.

  hooks/use-chat.ts
    → types/
    → config/

  components/*
    → types/
    → lib/utils.ts
    → ui/ (shadcn primitives)
```

**Key coupling violations to fix:**

1. `evaluation.py` router imports `_load_surveys` and `_calculate_stats` from `survey.py` router (cross-router import).
2. `seed_evaluation_topic.py` script imports `_compute_experts_for_frontend` from `chat.py` router.
3. `search.py` maintains its own Neo4j singleton instead of using `deps.py`.
4. `retrieval/engine.py` imports from `citation/sentence_extractor.py` (cross-domain, acceptable but worth noting).

---

## 4. Target Architecture

### Recommended Target Layout

```
ParliamentRAG/
├── build/
│   ├── pipeline/                     # Renamed from flat files
│   │   ├── orchestrator.py           # Was: build_and_update.py
│   │   ├── xml_parser.py             # Was: ingest_stenografici.py (parser only, no Italian save path)
│   │   ├── acts_ingester.py          # Was: ingest_atti_parlamentari.py
│   │   ├── role_config.py            # Was: app_config.py (English keys)
│   │   ├── db_setup.py               # Was: initialize_db.py
│   │   ├── embedding_cache.py        # Was: embedding_service.py
│   │   ├── embeddings.py             # Was: precalculate_embeddings.py
│   │   └── downloaders.py            # Was: download_deputies_csv.py + download_new_xmls.py
│   └── requirements-build.txt
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── key_pool.py
│   │   ├── models/
│   │   │   ├── evidence.py
│   │   │   ├── query.py              # Move QueryRequest/Response from routers here
│   │   │   ├── authority.py
│   │   │   ├── evaluation.py
│   │   │   ├── survey.py
│   │   │   └── compass.py
│   │   ├── routers/
│   │   │   ├── chat.py               # Main SSE endpoint — thin, delegates to services
│   │   │   ├── query.py              # Alternative endpoint — thin
│   │   │   ├── history.py
│   │   │   ├── evaluation.py         # Thin router only; logic moved to services/evaluation.py
│   │   │   ├── survey.py
│   │   │   ├── authority.py
│   │   │   ├── evidence.py
│   │   │   ├── search.py             # Uses deps.py singleton (not own)
│   │   │   ├── config.py
│   │   │   ├── graph.py
│   │   │   └── compass.py
│   │   └── services/
│   │       ├── deps.py               # Single source of singletons
│   │       ├── neo4j_client.py
│   │       ├── task_store.py
│   │       ├── experts.py            # NEW: extracted expert computation (from chat.py + query.py)
│   │       ├── evaluation_service.py # NEW: metric computation logic (from evaluation.py router)
│   │       ├── retrieval/
│   │       ├── generation/
│   │       ├── authority/
│   │       ├── compass/
│   │       └── citation/
│   ├── scripts/
│   │   ├── compute_baseline_experts.py
│   │   ├── compute_topic_authority_spread.py
│   │   ├── seed_evaluation_topic.py  # Import from services/experts.py (not routers)
│   │   ├── enrich_evaluation_set.py
│   │   └── export_notebooklm.py
│   └── config/
│
└── frontend/
    └── src/
        ├── app/
        │   ├── page.tsx              # home/chat
        │   ├── chat/[id]/page.tsx
        │   ├── search/page.tsx
        │   ├── explorer/page.tsx
        │   ├── ranking/page.tsx
        │   ├── compass/page.tsx
        │   └── evaluation/page.tsx   # Was: valutazione/ (English URL)
        ├── components/               # Unchanged structure
        ├── hooks/
        ├── lib/
        ├── types/                    # Strict types, no `[key: string]: any`
        └── config/
```

### Component Boundaries

| Component | Responsibility | Must Not Touch |
|-----------|---------------|----------------|
| `build/pipeline/xml_parser.py` | Parse XML, preprocess text, chunk speeches | Neo4j write, Italian schema, `save_to_neo4j` |
| `build/pipeline/orchestrator.py` | Coordinate full rebuild or incremental update | Parsing internals |
| `backend/app/services/retrieval/` | Embed query, retrieve chunks (dense + graph), merge, expand | LLM calls, generation |
| `backend/app/services/authority/` | Score speaker authority for a given query | Retrieval, generation |
| `backend/app/services/generation/` | 4-stage LLM pipeline, citations | Retrieval, authority scoring |
| `backend/app/services/experts.py` | Compute expert panel per party from evidence | Generation internals |
| `backend/app/services/evaluation_service.py` | Compute all evaluation metrics | HTTP routing |
| `backend/app/routers/*` | HTTP request/response, SSE formatting | Business logic (delegate to services) |
| `backend/scripts/*` | Offline utilities | HTTP routing internals |
| `frontend/src/hooks/` | API calls, state management | Component rendering logic |
| `frontend/src/components/` | Render UI | Direct API calls (use hooks/lib) |
| `frontend/src/lib/` | API client functions | State management |

### Data Flow

```
User Query (HTTP/SSE)
  └── router/chat.py or router/query.py
        ├── services/retrieval/engine.py
        │     ├── dense_channel.py    → Neo4j vector index → chunks
        │     ├── graph_channel.py    → Neo4j graph traversal → chunks
        │     └── merger.py          → merged + expanded evidence list
        ├── services/authority/scorer.py
        │     └── components.py      → per-speaker authority scores [0,1]
        ├── services/experts.py      → expert panel (one per party)
        ├── services/compass/pipeline.py → 2D ideological positions
        └── services/generation/pipeline.py
              ├── analyst.py         → claim decomposition
              ├── sectional.py       → per-party sections
              ├── integrator.py      → coherent narrative
              └── surgeon.py        → verbatim citation insertion
                    → SSE stream to frontend

DB Build Flow
  orchestrator.py
    ├── downloaders.py     → data/xml/, data/csv/
    ├── xml_parser.py      → Session/Debate/Phase/Speech/Chunk nodes (English schema)
    ├── acts_ingester.py   → ParliamentaryAct nodes
    ├── role_config.py     → role assignments on Deputy/GovernmentMember
    ├── embedding_cache.py → SQLite cache (344 MB)
    └── embeddings.py      → pre-compute all node embeddings → Neo4j

Scripts Flow (offline)
  compute_baseline_experts.py
    → services/authority/scorer.py   (reuses app service)
    → services/retrieval/engine.py   (reuses app service)
    → evaluation_set.json            (writes pre-computed baseline experts)

  seed_evaluation_topic.py
    → services/experts.py            (reuses extracted service, not router)
```

---

## 5. Schema Changes (Build → Backend Contract)

Properties being removed in the refactoring and what replaces them:

| Node | Current Property | Action | Backend Impact |
|------|-----------------|--------|----------------|
| `Chunk` | `start_char_raw` | Remove | Replace all 12 Cypher references with `span_start` (new name, correct offsets) |
| `Chunk` | `end_char_raw` | Remove | Replace all 12 Cypher references with `span_end` |
| `Speech` | `text` (raw) + `preprocessed_text` | Merge: keep only `text` (preprocessed) | Update queries that distinguish them |
| `Session` | `complete_date` (string) + `date` (Neo4j Date) | Remove `complete_date` | Audit all `complete_date` references |

Rename mapping (Italian schema → English, only applies if `ingest_stenografici` Italian path is ever used — being removed):

| Old (Italian) | New (English) |
|--------------|--------------|
| `Seduta` | `Session` |
| `Dibattito` | `Debate` |
| `Fase` | `Phase` |
| `Intervento` | `Speech` |
| `Votazione` | `Vote` |

---

## 6. Anti-Patterns to Avoid

### Anti-Pattern 1: Cross-Router Imports
**What:** `evaluation.py` imports `_load_surveys`, `_calculate_stats` from `survey.py`. `seed_evaluation_topic.py` imports `_compute_experts_for_frontend` from `chat.py`.
**Why bad:** Creates tight coupling between modules that should be independent. Breaking one router breaks another.
**Instead:** Extract shared logic into service modules (`evaluation_service.py`, `experts.py`). Routers import from services; scripts import from services.

### Anti-Pattern 2: Duplicate Client Singletons
**What:** `search.py` maintains `_neo4j_client: Optional[Neo4jClient] = None` at module level.
**Why bad:** Two Neo4j connection pools in one process, different initialization order, potential inconsistency.
**Instead:** All routers use `deps.py` via `get_services()`. One pool, one initialization point.

### Anti-Pattern 3: Business Logic in Router Files
**What:** `evaluation.py` contains `KNOWN_PARTIES`, `PARTY_KEYWORDS`, and all metric computation. `chat.py` contains `_compute_experts_for_frontend()` (200+ lines).
**Why bad:** Routers should validate input, call services, format output. Embedding business rules in routers makes them untestable and locks logic behind HTTP concerns.
**Instead:** Routers are thin. Business logic lives in `services/`.

### Anti-Pattern 4: Dead Code in Production-Path Files
**What:** `ingest_stenografici.py` contains Italian `save_to_neo4j()`, Italian constraints, Italian indexes — none of which are called by `build_and_update.py`.
**Why bad:** Confusing to read; risks being accidentally invoked; inflates the module with ~300 lines that have no runtime effect.
**Instead:** Extract only what `build_and_update.py` actually uses (the `StenograficoIngester` parser class) into `xml_parser.py`. Delete the rest.

### Anti-Pattern 5: Schema Property Names Leaking Across Layers
**What:** `start_char_raw`/`end_char_raw` are Neo4j property names that appear verbatim in Cypher strings embedded in 6 different Python files. Renaming requires a grep-and-replace across the full backend.
**Instead:** Define Cypher query strings as named constants or in a dedicated query module. Schema property names appear in exactly one place.

---

## 7. Recommended Refactoring Order

The ordering is determined by the dependency graph: lower layers must be refactored before higher layers that depend on them.

### Phase 1: Build Pipeline
**Rationale:** The build pipeline is independent of the backend. Refactoring it first produces a clean DB schema (English-only, no redundant properties). All subsequent backend work targets the clean schema — no need to update queries twice.

Steps within Phase 1:
1. Extract `xml_parser.py` from `ingest_stenografici.py` (parser class only, no Italian save path).
2. Rename `build_and_update.py` to `orchestrator.py`, update import to `xml_parser`.
3. Translate `app_config.py` to English keys and rename to `role_config.py`.
4. Remove `migrate_foti.py` (dead one-shot script).
5. Consolidate `download_deputies_csv.py` + `download_new_xmls.py` into `downloaders.py`.
6. Remove `start_char_raw`/`end_char_raw` from chunking output; emit `span_start`/`span_end` with correct offsets.
7. Remove `preprocessed_text` from Speech (keep only `text`).
8. Remove `complete_date` from Session (keep only `date`).
9. Rename `Votazione` → `Vote`.
10. Add a `Makefile` target `db-all` that runs full rebuild.

### Phase 2: Backend Service Layer (Models + Services)
**Rationale:** Services are the foundation for routers and scripts. Cleaning them before routers avoids double-touching the same logic.

Steps within Phase 2:
1. Update all Cypher queries to use new property names (`span_start`/`span_end`, `text`, `date`). Files: `neo4j_client.py`, `dense_channel.py`, `graph_channel.py`, `engine.py` x2, `evidence.py`, `chat.py` x2.
2. Extract `_compute_experts_for_frontend()` (from `chat.py`) and `_compute_experts()` (from `query.py`) into a unified `services/experts.py`.
3. Extract `KNOWN_PARTIES`, `PARTY_KEYWORDS`, and metric computation from `evaluation.py` router into `services/evaluation_service.py`.
4. Remove duplicate `_load_surveys`/`_calculate_stats` cross-router import; `evaluation.py` calls `evaluation_service.py`.
5. Fix `search.py` to use `deps.py` singleton instead of its own.
6. Move `QueryRequest`, `CitationInfo`, `ExpertInfo`, `QueryResponse` into `models/query.py`.
7. Add type hints throughout services (Python 3.11 style).
8. Translate all Italian strings in services (log messages, comments, docstrings).

### Phase 3: Backend API Routers
**Rationale:** Routers depend on services. After Phase 2, routers can be thinned to pure HTTP handling.

Steps within Phase 3:
1. Thin `chat.py`: delegate to `services/experts.py`, remove local Cypher queries that belong in services.
2. Thin `query.py`: remove duplicated expert computation, use `services/experts.py`.
3. Thin `evaluation.py`: delegate all computation to `services/evaluation_service.py`.
4. Clean endpoint naming (English, consistent).
5. Remove any dead endpoints.
6. Translate Italian error messages and log strings.
7. Remove maintenance message Italian text.

### Phase 4: Backend Scripts
**Rationale:** Scripts import from services and routers. After Phases 2-3, the correct imports exist.

Steps within Phase 4:
1. Fix `seed_evaluation_topic.py` to import from `services/experts.py` (not `routers/chat`).
2. Update all Cypher queries in scripts for new schema properties.
3. Add docstrings, type hints, consistent error handling.

### Phase 5: Frontend
**Rationale:** Frontend depends on API shape (preserved per constraints), not internal backend structure. It can be done in parallel with Phase 4 or after.

Steps within Phase 5:
1. Rename `valutazione/` → `evaluation/`.
2. Translate Italian comments in TypeScript files.
3. Remove `[key: string]: any` from `Citation` interface; add explicit fields.
4. Audit `any` usage across all `.ts`/`.tsx` files.
5. Clean dead/unused component code.

---

## 8. Phase-Specific Warnings

| Phase | Topic | Likely Pitfall | Mitigation |
|-------|-------|---------------|------------|
| 1 | Removing `start_char_raw`/`end_char_raw` | Backend becomes immediately broken until Phase 2 Cypher updates are done | Do Phase 1 and Phase 2 Cypher updates as a single atomic unit; rebuild DB only after backend is also updated |
| 1 | Extracting xml_parser.py | `StenograficoIngester.__init__` takes a Neo4j URI — the parser class is coupled to the DB client even though only parsing is needed | Refactor parser to not require `driver` in `__init__`; pass driver only to the save methods (which will be removed) |
| 2 | Unified experts service | Two implementations exist with slightly different behavior post-citation-update (the second SSE `experts` event in `query.py`). Must preserve this logic exactly. | Read both implementations fully before writing `services/experts.py` |
| 2 | Evaluation service extraction | `evaluation.py` imports `_load_surveys` + `_calculate_stats` from `survey.py`. These are also private functions. | Elevate them to a `services/survey_service.py` first, then both `evaluation.py` and `survey.py` router import from the service. |
| 3 | Thinning `chat.py` | `chat.py` is the largest file (~1400 lines) and contains several inline Cypher queries and the task cancellation flow. | Break into incremental PRs: first extract experts, then Cypher queries, then Italian strings. |
| 4 | Script Cypher queries | `compute_baseline_experts.py` and `enrich_evaluation_set.py` have their own Cypher that also references `start_char_raw`. | Must be updated in Phase 4, not Phase 1 (scripts need the new property names once the DB is rebuilt). |
| 5 | `/valutazione` URL rename | Any external links or bookmarks to `/valutazione` will break | Add a Next.js redirect from `/valutazione` to `/evaluation` |

---

## Sources

- Actual file listing: `find build/ backend/app/ backend/scripts/ frontend/src/ -type f`
- Dependency analysis: direct code inspection of imports in all router and script files
- Property reference audit: `grep -r "start_char_raw\|end_char_raw\|preprocessed_text\|complete_date"` across `backend/app/`
- Cross-router coupling: grep for `from app.routers` in `backend/scripts/`
- Architecture verified against `PROJECT.md` requirements and `MEMORY.md` bug history
