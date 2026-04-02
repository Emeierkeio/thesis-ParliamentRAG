# Codebase Structure

**Analysis Date:** 2026-04-01

## Directory Layout

```
/Users/mirkotritella/Desktop/ParliamentRAG/
├── backend/                           # FastAPI + Neo4j backend
│   ├── app/
│   │   ├── main.py                    # FastAPI application entry point
│   │   ├── config.py                  # Configuration management (YAML loader)
│   │   ├── key_pool.py                # OpenAI API key management
│   │   ├── middleware/                # Custom middleware (logging, CORS)
│   │   ├── models/                    # Pydantic data models
│   │   │   ├── evidence.py            # UnifiedEvidence schema with citation integrity
│   │   │   ├── authority.py           # Authority score models
│   │   │   ├── compass.py             # Ideology score models
│   │   │   ├── evaluation.py          # Evaluation/survey models
│   │   │   ├── survey.py              # User evaluation survey models
│   │   │   ├── query.py               # Query/response models
│   │   │   └── __init__.py
│   │   ├── routers/                   # FastAPI route handlers (HTTP endpoints)
│   │   │   ├── chat.py                # Main chat endpoint (SSE streaming)
│   │   │   ├── query.py               # Alternative query endpoint
│   │   │   ├── history.py             # Conversation history retrieval
│   │   │   ├── evaluation.py          # Dashboard metrics and A/B evaluations
│   │   │   ├── survey.py              # User survey submission and analysis
│   │   │   ├── evidence.py            # Evidence detail endpoint
│   │   │   ├── config.py              # Configuration GET/PUT/reload
│   │   │   ├── authority.py           # Authority rankings by topic
│   │   │   ├── compass.py             # Standalone compass endpoint
│   │   │   ├── graph.py               # Graph exploration endpoint
│   │   │   ├── search.py              # Parliamentary record search
│   │   │   └── __init__.py            # Router imports
│   │   └── services/                  # Core business logic
│   │       ├── deps.py                # Lazy service initialization singleton
│   │       ├── neo4j_client.py        # Neo4j database abstraction
│   │       ├── task_store.py          # Task cancellation tracking
│   │       ├── retrieval/             # Dual-channel evidence retrieval
│   │       │   ├── engine.py          # Main orchestrator (dense + graph + merge)
│   │       │   ├── dense_channel.py   # Vector similarity search
│   │       │   ├── graph_channel.py   # Metadata-driven graph retrieval
│   │       │   ├── merger.py          # Channel score merging and deduplication
│   │       │   ├── query_rewriter.py  # Optional query expansion
│   │       │   ├── commission_matcher.py  # Commission reference matching
│   │       │   └── __init__.py
│   │       ├── generation/            # 4-stage generation pipeline with citation integrity
│   │       │   ├── pipeline.py        # Main orchestrator (Analyst → Sectional → Integrator → Surgeon)
│   │       │   ├── analyst.py         # Stage 1: Claim decomposition
│   │       │   ├── sectional.py       # Stage 2: Per-party sections
│   │       │   ├── integrator.py      # Stage 3: Narrative coherence
│   │       │   ├── surgeon.py         # Stage 4: Citation extraction (offset-based)
│   │       │   ├── citation_registry.py  # Citation tracking through pipeline
│   │       │   ├── coherence_validator.py # Semantic alignment verification
│   │       │   ├── synthesis.py       # Convergence/divergence analysis
│   │       │   ├── reported_speech.py # Indirect speech handling
│   │       │   ├── evidence_first_writer.py # Evidence prioritization
│   │       │   ├── position_brief.py  # Stance extraction
│   │       │   └── __init__.py
│   │       ├── authority/             # Query-dependent authority scoring
│   │       │   ├── scorer.py          # Main scoring orchestrator
│   │       │   ├── components.py      # Score components (Prof, Edu, Committee, Acts, Interventions, Role)
│   │       │   ├── coalition_logic.py # Government/majority/opposition classification
│   │       │   └── __init__.py
│   │       ├── compass/               # Ideological positioning analysis
│   │       │   ├── pipeline.py        # Compass orchestrator
│   │       │   ├── scoring.py         # Speaker ideology scoring
│   │       │   ├── clustering.py      # Party cluster detection
│   │       │   ├── reference_axes.py  # Reference axis definitions
│   │       │   ├── anchors.py         # Anchor point computation
│   │       │   ├── axis_labeling.py   # Axis label generation
│   │       │   └── __init__.py
│   │       ├── citation/              # Citation extraction and verification
│   │       │   ├── sentence_extractor.py  # Offset-based quote extraction
│   │       │   └── __init__.py
│   │       └── __init__.py
│   ├── config/                        # Configuration files
│   │   └── default.yaml               # Weights, thresholds, generation params
│   ├── scripts/                       # Utility scripts (data loading, etc.)
│   ├── logs/                          # Application log files (generated at runtime)
│   ├── evaluation_set.json            # Ground truth topics with baseline Q&A and experts
│   ├── requirements.txt               # Python dependencies
│   ├── Dockerfile                     # Docker build configuration
│   └── venv/                          # Python virtual environment (git-ignored)
│
├── frontend/                          # Next.js 14 + React + TypeScript
│   ├── src/
│   │   ├── app/                       # Next.js 14 App Router
│   │   │   ├── page.tsx               # Home page (main chat interface)
│   │   │   ├── layout.tsx             # Root layout (metadata, fonts, maintenance check)
│   │   │   ├── globals.css            # Global styles
│   │   │   ├── chat/
│   │   │   │   └── [id]/page.tsx      # Chat detail page (load conversation by ID)
│   │   │   ├── valutazione/
│   │   │   │   └── page.tsx           # Evaluation dashboard with metrics and ratings
│   │   │   ├── search/
│   │   │   │   └── page.tsx           # Parliamentary record search UI
│   │   │   ├── ranking/
│   │   │   │   └── page.tsx           # Authority rankings by topic
│   │   │   ├── compass/
│   │   │   │   └── page.tsx           # Standalone ideological compass
│   │   │   └── explorer/
│   │   │       └── page.tsx           # Graph explorer (parliamentary network)
│   │   ├── components/                # Reusable React components
│   │   │   ├── chat/                  # Chat-related components
│   │   │   │   ├── ChatArea.tsx       # Main chat message area
│   │   │   │   ├── ChatInput.tsx      # Message input with send button
│   │   │   │   ├── MessageList.tsx    # Scrollable message history
│   │   │   │   ├── CitationCard.tsx   # Citation display with speaker details
│   │   │   │   ├── ExpertCard.tsx     # Expert/authority display per party
│   │   │   │   ├── CompassCard.tsx    # Inline ideological compass
│   │   │   │   └── ProgressBar.tsx    # Pipeline progress indicator
│   │   │   ├── evaluation/            # Dashboard components
│   │   │   │   ├── EvaluationCharts.tsx  # Chart components (bar, win rate, A/B comparison)
│   │   │   │   └── MetricCard.tsx     # Metric display card
│   │   │   ├── survey/                # Survey/A/B evaluation modal
│   │   │   │   ├── SurveyModal.tsx    # Main A/B rating modal
│   │   │   │   └── RatingScale.tsx    # Likert scale for ratings
│   │   │   ├── layout/                # Layout structure
│   │   │   │   ├── Sidebar.tsx        # Navigation sidebar with history
│   │   │   │   └── MobileMenuButton.tsx
│   │   │   ├── shared/                # Shared UI components
│   │   │   │   ├── HistoryModal.tsx   # Conversation history modal
│   │   │   │   ├── Card.tsx           # Generic card container
│   │   │   │   ├── Button.tsx         # Button component
│   │   │   │   ├── Badge.tsx          # Badge/label component
│   │   │   │   └── Tabs.tsx           # Tabbed interface
│   │   │   ├── ui/                    # Shadcn/ui primitive components
│   │   │   │   ├── card.tsx, button.tsx, tabs.tsx, etc.
│   │   │   ├── graph/                 # Graph explorer components
│   │   │   ├── search/                # Search UI components
│   │   │   ├── settings/              # Settings panel components
│   │   │   └── graph_explorer/        # Interactive network visualization
│   │   ├── hooks/                     # Custom React hooks
│   │   │   ├── use-chat.ts            # Main chat state management (messages, loading, SSE)
│   │   │   ├── use-sidebar.ts         # Sidebar toggle state
│   │   │   ├── use-local-history.ts   # Local history persistence
│   │   │   └── index.ts               # Hook exports
│   │   ├── lib/                       # Utility functions and API clients
│   │   │   ├── api.ts                 # Config API client (GET/PUT /api/config)
│   │   │   ├── evaluation-api.ts      # Evaluation dashboard API
│   │   │   ├── survey-api.ts          # Survey submission API
│   │   │   ├── graph-api.ts           # Graph explorer API
│   │   │   ├── constants.ts           # Global constants (API URLs, etc.)
│   │   │   ├── utils.ts               # Helper utilities (formatting, validation)
│   │   │   └── index.ts               # Exports
│   │   ├── types/                     # TypeScript type definitions
│   │   │   ├── chat.ts                # Message, Citation, Expert, BalanceMetrics types
│   │   │   ├── evaluation.ts          # Dashboard data and evaluation types
│   │   │   ├── survey.ts              # A/B rating and survey types
│   │   │   ├── api.ts                 # API request/response types
│   │   │   └── index.ts               # Type exports
│   │   └── config/                    # Frontend configuration
│   │       └── index.ts               # API base URL, feature flags
│   ├── public/                        # Static assets
│   ├── next.config.js                 # Next.js configuration
│   ├── tsconfig.json                  # TypeScript configuration
│   ├── tailwind.config.ts             # Tailwind CSS configuration
│   ├── package.json                   # Dependencies (Next.js, React, TailwindCSS, etc.)
│   ├── package-lock.json              # Dependency lock file
│   └── .env.local                     # Frontend environment (NEXT_PUBLIC_API_URL)
│
├── neo4j/                             # Neo4j configuration
│   ├── scripts/                       # Data loading scripts
│   └── config/                        # Neo4j server config
│
├── build/                             # Build artifacts (generated)
├── outputs/                           # Evaluation outputs
├── .planning/
│   └── codebase/                      # GSD documentation
│       ├── ARCHITECTURE.md            # Layer design, data flow, abstractions
│       ├── STRUCTURE.md               # This file: directory layout and naming conventions
│       ├── CONVENTIONS.md             # Code style, import patterns, naming
│       ├── TESTING.md                 # Testing framework and patterns
│       ├── STACK.md                   # Tech stack and runtime
│       ├── INTEGRATIONS.md            # External API/service integrations
│       └── CONCERNS.md                # Known issues and technical debt
│
├── .git/                              # Git repository
├── .gitignore                         # Git ignore patterns
├── .env                               # Environment secrets (DO NOT COMMIT)
├── .env.example                       # Example environment template
├── docker-compose.yml                 # Multi-service deployment
├── package.json                       # Root-level Node scripts
├── README.md                          # Project documentation
└── thesis.md, evaluation.md, etc.     # Documentation files
```

## Directory Purposes

**`backend/app/`:**
- Purpose: Core FastAPI application
- Contains: HTTP routers, Pydantic models, business logic services
- Key files: `main.py` (entry point), `config.py` (configuration), `models/` (schema), `routers/` (HTTP handlers), `services/` (logic)

**`backend/app/routers/`:**
- Purpose: HTTP API endpoints
- Contains: FastAPI route handlers with request/response validation
- Key files:
  - `chat.py` – Main entry point for user queries (SSE streaming)
  - `query.py` – Alternative query endpoint
  - `history.py` – Conversation history retrieval
  - `evaluation.py` – Dashboard metrics computation
  - `survey.py` – User A/B evaluation storage and aggregation

**`backend/app/services/`:**
- Purpose: Core business logic in isolated, reusable modules
- Contains: Retrieval, generation, authority scoring, ideology compass
- Structure: Each major component (retrieval, generation, etc.) is a subpackage with internal modules
- Key feature: `deps.py` ensures all services are singletons (shared across requests)

**`backend/app/services/retrieval/`:**
- Purpose: Dual-channel evidence retrieval orchestration
- Contains: Dense channel (vector search), graph channel (metadata), merger, query rewriter
- Entry: `engine.py::RetrievalEngine.retrieve_sync()` called from routers
- Output: List of UnifiedEvidence records with authority scores

**`backend/app/services/generation/`:**
- Purpose: 4-stage text generation with citation integrity
- Contains: Analyst (claims), Sectional (per-party), Integrator (narrative), Surgeon (citations)
- Critical: Citation integrity system (registry, coherence validator, offset-based extraction)
- Entry: `pipeline.py::GenerationPipeline.generate()` called after retrieval

**`backend/app/services/authority/`:**
- Purpose: Query-dependent authority scoring
- Contains: Scorer (main), components (6 scoring aspects), coalition logic (government/opposition classification)
- Key: Scores are **per-query**, not global – computed against query embedding
- Output: Authority_score field in each evidence record

**`backend/app/models/`:**
- Purpose: Pydantic schemas for type safety and validation
- Key files:
  - `evidence.py` – UnifiedEvidence with quote_text (verbatim) vs chunk_text (preview)
  - `survey.py` – A/B evaluation models
  - `evaluation.py` – Dashboard and metric models
  - `authority.py` – Authority score breakdown models

**`backend/config/`:**
- Purpose: YAML-based configuration (not secrets)
- Contains: `default.yaml` with all weights, thresholds, generation parameters
- Usage: Loaded once at startup, reloadable via `POST /api/config/reload`

**`frontend/src/app/`:**
- Purpose: Next.js 14 App Router pages
- Contains: Page components (page.tsx) for each route
- Key pages:
  - `page.tsx` – Home (main chat interface)
  - `valutazione/page.tsx` – Evaluation dashboard
  - `search/page.tsx` – Parliamentary record search
  - `ranking/page.tsx` – Expert rankings

**`frontend/src/components/`:**
- Purpose: Reusable React components organized by feature domain
- Structure: Each domain (chat, evaluation, survey, layout) has its own subdirectory
- Key:
  - `chat/` – Chat message rendering, experts, citations, compass
  - `evaluation/` – Dashboard charts and metrics
  - `survey/` – A/B rating modal
  - `layout/` – Sidebar, headers
  - `ui/` – Shadcn/ui primitives (button, card, tabs, etc.)

**`frontend/src/hooks/`:**
- Purpose: Custom React hooks for state management and API communication
- Key files:
  - `use-chat.ts` – Main hook managing message state, SSE connection, loading
  - `use-sidebar.ts` – Sidebar collapse/mobile toggle
  - `use-local-history.ts` – Local storage persistence

**`frontend/src/lib/`:**
- Purpose: Utility functions and API clients
- Key files:
  - `api.ts` – Config API client
  - `evaluation-api.ts` – Dashboard data fetching
  - `survey-api.ts` – Survey submission
  - `utils.ts` – String formatting, validation helpers

**`frontend/src/types/`:**
- Purpose: TypeScript interfaces for type safety across frontend
- Key files:
  - `chat.ts` – Message, Citation, Expert, BalanceMetrics
  - `evaluation.ts` – Dashboard data structures
  - `survey.ts` – A/B rating types

## Key File Locations

**Entry Points:**

- **Backend HTTP entry**: `backend/app/main.py` (FastAPI creation, router mounting)
- **Backend business logic entry**: `backend/app/routers/chat.py::stream_chat()` (POST /api/chat)
- **Frontend entry**: `frontend/src/app/page.tsx` (Home page)
- **Configuration entry**: `backend/app/config.py` (Settings + ConfigLoader)

**Configuration:**

- `backend/config/default.yaml` – All runtime parameters (weights, thresholds, models)
- `.env` – Secrets only (NEO4J_PASSWORD, OPENAI_API_KEY)
- `backend/app/config.py` – Settings schema and config loader
- `frontend/src/config/index.ts` – API base URL configuration

**Core Logic:**

- **Retrieval orchestration**: `backend/app/services/retrieval/engine.py`
- **Generation orchestration**: `backend/app/services/generation/pipeline.py`
- **Authority scoring**: `backend/app/services/authority/scorer.py`
- **Ideology compass**: `backend/app/services/compass/pipeline.py`
- **Citation extraction**: `backend/app/services/citation/sentence_extractor.py`

**Testing:**

- Backend: No pytest tests present (integration testing via API)
- Frontend: No test files present (component testing likely via Storybook)
- Integration: Manual API testing via `/docs` (FastAPI auto-generated Swagger UI)

**Service Initialization:**

- `backend/app/services/deps.py` – Lazy initialization of singleton services
- Called by routers via `get_services()` function

## Naming Conventions

**Files:**

- **Router files**: `{domain}.py` (query.py, chat.py, history.py, evaluation.py)
  - Example: `backend/app/routers/chat.py`

- **Service modules**: `{component_name}.py` within domain subdirectories
  - Example: `backend/app/services/retrieval/dense_channel.py`

- **Model files**: `{domain}.py` containing Pydantic classes
  - Example: `backend/app/models/evidence.py`

- **Frontend components**: PascalCase.tsx (e.g., ChatArea.tsx, CitationCard.tsx)
  - Location: `frontend/src/components/{domain}/{ComponentName}.tsx`

- **Frontend pages**: `page.tsx` (Next.js convention)
  - Location: `frontend/src/app/{route}/page.tsx`

- **Hooks**: `use-{hook-name}.ts` (React convention)
  - Example: `frontend/src/hooks/use-chat.ts`

- **API clients**: `{domain}-api.ts`
  - Example: `frontend/src/lib/evaluation-api.ts`

- **Types**: `{domain}.ts` containing TypeScript interfaces
  - Example: `frontend/src/types/chat.ts`

**Directories:**

- **Backend feature domains**: lowercase, plural or singular depending on semantics
  - `retrieval/`, `generation/`, `authority/`, `compass/`, `citation/`, `routers/`, `services/`, `models/`

- **Frontend feature domains**: lowercase, plural
  - `chat/`, `components/`, `evaluation/`, `survey/`, `layout/`, `search/`, `graph/`, `settings/`

- **Feature subdomains**: Related components grouped together
  - Example: `frontend/src/components/chat/` contains ChatArea.tsx, CitationCard.tsx, ExpertCard.tsx, etc.

**Python Code:**

- **Classes**: PascalCase (e.g., `RetrievalEngine`, `GenerationPipeline`, `UnifiedEvidence`)
- **Functions**: snake_case (e.g., `embed_query()`, `retrieve_sync()`, `process_query_streaming()`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `_MAX_CONCURRENT_PIPELINES`, `PARTY_DISPLAY_NAMES`)
- **Private functions**: Prefix with `_` (e.g., `_get_pipeline_semaphore()`)

**TypeScript/React Code:**

- **Components**: PascalCase (e.g., `ChatArea`, `ExpertCard`, `SurveyModal`)
- **Hooks**: `use{CapitalizedName}` (e.g., `useChat`, `useSidebar`)
- **Types/Interfaces**: PascalCase (e.g., `Message`, `Citation`, `Expert`)
- **Constants**: UPPER_SNAKE_CASE or camelCase depending on usage
- **Functions**: camelCase (e.g., `getConfig()`, `updateConfig()`)

## Where to Add New Code

**New Feature (Backend):**

- **Primary code**: `backend/app/routers/{domain}.py` (create new router if needed)
  - Example: To add a new endpoint for "trending topics", create `backend/app/routers/trending.py`

- **Business logic**: `backend/app/services/{domain}/{component}.py` (create new service module)
  - Example: `backend/app/services/trending/analyzer.py`

- **Models**: `backend/app/models/{domain}.py` (add new Pydantic classes)
  - Example: `backend/app/models/trending.py`

- **Route registration**: Add `from .routers.{domain} import router as {domain}_router` in `backend/app/main.py`, then `app.include_router({domain}_router)`

**New Feature (Frontend):**

- **New page**: Create `frontend/src/app/{route}/page.tsx`
  - Example: `frontend/src/app/trending/page.tsx`

- **New component**: Add to `frontend/src/components/{domain}/{ComponentName}.tsx`
  - Example: `frontend/src/components/trending/TrendingChart.tsx`

- **New hook**: Create `frontend/src/hooks/use-{domain}.ts`
  - Example: `frontend/src/hooks/use-trending.ts`

- **API client**: Add to `frontend/src/lib/{domain}-api.ts`
  - Example: `frontend/src/lib/trending-api.ts`

- **Types**: Add to `frontend/src/types/{domain}.ts`
  - Example: `frontend/src/types/trending.ts`

**New Component/Module (Shared):**

- **Utilities**: `backend/app/services/utils/` or `frontend/src/lib/utils.ts`
  - Keep helpers close to where they're used, move to utils only if shared across 3+ places

- **Shared models**: `backend/app/models/` (Pydantic already centralizes schemas)

**Internal Module Exports:**

- **Backend**: Use `__init__.py` to re-export main classes/functions
  - Example: `backend/app/services/retrieval/__init__.py` exports `RetrievalEngine`
  - Routers then import via `from ..services.retrieval import RetrievalEngine`

- **Frontend**: Use `index.ts` barrel files for exports
  - Example: `frontend/src/hooks/index.ts` exports all hooks
  - Components import via `import { useChat } from '@/hooks'`

## Special Directories

**`backend/logs/`:**
- Purpose: Application and debug logs (generated at runtime)
- Generated: Yes (created by `main.py::setup_logging()`)
- Committed: No (in `.gitignore`)
- Content: Rotating log files with timestamps (app_YYYYMMDD_HHMMSS.log, debug_YYYYMMDD_HHMMSS.log)

**`backend/config/`:**
- Purpose: YAML configuration files (not secrets, not code)
- Generated: No
- Committed: Yes
- Content: `default.yaml` with all tunable parameters

**`frontend/public/`:**
- Purpose: Static assets (images, favicons, etc.)
- Generated: No
- Committed: Yes
- Usage: Served by Next.js at root URL

**`neo4j/`:**
- Purpose: Neo4j database scripts and configuration
- Generated: No
- Committed: Yes
- Content: Data loading scripts, server configuration

**`build/` and `outputs/`:**
- Purpose: Artifacts and evaluation results
- Generated: Yes
- Committed: No
- Content: Docker build outputs, evaluation JSON results

**`.planning/codebase/`:**
- Purpose: GSD (Grand System Design) documentation
- Generated: No (written manually by GSD agents)
- Committed: Yes
- Content: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, STACK.md, INTEGRATIONS.md, CONCERNS.md

---

*Structure analysis: 2026-04-01*
