# External Integrations

**Analysis Date:** 2026-04-01

## APIs & External Services

**OpenAI API (LLM & Embeddings):**
- **Service:** OpenAI (primary and only LLM provider)
- **What it's used for:**
  - Text embeddings (dense vector retrieval)
  - LLM inference for multi-stage generation pipeline
  - Models: gpt-4o (analyst, writer, integrator roles)
- **SDK/Client:** `openai >= 1.10.0`
- **Auth:** Environment variable `OPENAI_API_KEY` (supports comma-separated list for load distribution)
- **Implementation locations:**
  - `backend/app/key_pool.py` - API key management and rotation
  - `backend/app/services/retrieval/engine.py` - Embedding generation
  - `backend/app/services/generation/analyst.py` - Analyst LLM calls
  - `backend/app/services/generation/evidence_first_writer.py` - Writer LLM calls
  - `backend/app/services/generation/integrator.py` - Integrator LLM calls
  - `backend/app/services/generation/coherence_validator.py` - Validation LLM calls
  - `backend/app/routers/search.py` - Query embedding generation
  - `backend/app/routers/authority.py` - Topic embedding for authority ranking

## Data Storage

**Databases:**
- **Type/Provider:** Neo4j 5.15.0+ (Graph database)
  - Connection: `NEO4J_URI` (default: `bolt://localhost:7689`)
  - Client: Python neo4j driver 5.15.0+
  - Features:
    - Vector index (`chunk_embedding_index`) for dense semantic search
    - APOC and Graph Data Science plugins enabled in Docker Compose
    - Custom constraints for ChatHistory and SurveyEvaluation nodes

**File Storage:**
- **Type:** Local filesystem only
  - `backend/evaluation_set.json` - Ground truth evaluation queries with baseline answers and expert assignments
  - `backend/logs/` - Rotating log files (app_TIMESTAMP.log, debug_TIMESTAMP.log)
  - `backend/config/default.yaml` - System configuration (weights, thresholds, generation parameters)
  - `backend/config/commissioni_topics.yaml` - Parliamentary commission topic mappings
  - `neo4j/` - Database volumes mounted by Docker Compose:
    - `neo4j_data/` - Database data files
    - `neo4j_logs/` - Neo4j logs
    - `neo4j_import/` - Data import directory
    - `neo4j_plugins/` - Custom plugins

**Caching:**
- **Type:** None - No external caching service
- **In-memory:** Neo4j vector index warmed up on application startup (`_warmup_neo4j_index` in `backend/app/main.py`)

## Authentication & Identity

**Auth Provider:**
- **Type:** Custom implementation with optional evaluator tracking
- **Approach:**
  - Backend does not enforce authentication on API endpoints (CORS open to "*")
  - Chat history identified by session ID (UUID)
  - A/B evaluation assignments tracked per evaluator (optional `evaluator_id`)
  - Neo4j credentials required: `NEO4J_USER` and `NEO4J_PASSWORD`
  - OpenAI API authentication via `OPENAI_API_KEY`

## Monitoring & Observability

**Error Tracking:**
- **Type:** None detected
- **Approach:** Built-in Python logging only

**Logs:**
- **Framework:** Python standard logging module
- **Configuration:** `backend/app/main.py` - `setup_logging()`
- **Log files:**
  - `backend/logs/app_TIMESTAMP.log` - INFO+ level, operation logs (20 MB rotating, 10 backups)
  - `backend/logs/debug_TIMESTAMP.log` - DEBUG+ level, diagnostic logs (50 MB rotating, 5 backups)
- **Console:** INFO+ to stdout during development
- **Noisy library suppression:** httpx, httpcore, urllib3, openai, neo4j.notifications set to WARNING
- **Debug modules:** Specific modules elevated to DEBUG:
  - `app.services.generation.integrator` - Citation context issues
  - `app.services.generation.coherence_validator` - Embedding scoring details

## CI/CD & Deployment

**Hosting:**
- **Type:** Docker-based containerized deployment
- **Backend:** Dockerfile at `backend/Dockerfile` - FastAPI application
- **Database:** Docker Compose service (neo4j:5.15.0)
- **Frontend:** Next.js with `output: 'standalone'` for Docker deployment

**CI Pipeline:**
- **Type:** Not detected in codebase
- **Build tooling:** npm scripts and Python pytest

## Environment Configuration

**Required env vars (CRITICAL):**
- `OPENAI_API_KEY` - OpenAI API key (no default, application will fail without this)
- `NEO4J_PASSWORD` - Neo4j database password (no default)

**Optional env vars with defaults:**
- `NEO4J_URI` - Default: `bolt://localhost:7689`
- `NEO4J_USER` - Default: `neo4j`
- `NEXT_PUBLIC_API_URL` - Default: `http://localhost:8000/api`
- `ENVIRONMENT` - Default: `development`
- `LOG_LEVEL` - Default: `INFO`
- `WORKERS` - Default: `4`
- `DEBUG` - Default: `False`

**Secrets location:**
- `.env` file (root or `backend/` directory, gitignored)
- `.env.example` - Template for reference (safe to commit)
- **Never committed:** `.env`, `*.key`, `*.pem`, credentials files

## Webhooks & Callbacks

**Incoming:**
- **Type:** None detected

**Outgoing:**
- **Type:** None - No external webhook notifications
- **Server-Sent Events (SSE):** Backend streams responses to frontend
  - `/api/query` - Query processing with streaming updates (experts, analysis, response)
  - `/api/chat` - Frontend-compatible chat endpoint (streaming)
  - Frontend routes (`src/app/api/chat/route.ts`) bridge SSE from backend to client

## Data Pipelines & Integration Points

**Query Processing Pipeline:**
1. Frontend sends query to `POST /api/chat` (Next.js route handler)
2. Route handler proxies to backend `POST /api/query`
3. Backend processes via `process_query_streaming()` in `backend/app/routers/query.py`:
   - Embed query with OpenAI
   - Retrieve dense candidates (Neo4j vector index)
   - Retrieve graph candidates (Neo4j Cypher)
   - Merge results with scoring
   - Compute experts (top authority per party)
   - Run LLM generation pipeline (analyst → writer → integrator)
   - Verify citations and emit experts update SSE
4. Frontend receives SSE stream and updates UI progressively

**Evaluation Features:**
- A/B evaluation system via `SurveyEvaluation` Neo4j nodes
- Ground truth baselines loaded from `evaluation_set.json`
- Baseline experts pre-computed with query-specific authority scores
- Dashboard aggregates metrics from evaluation database

**History & Persistence:**
- Chat sessions stored as `ChatHistory` nodes in Neo4j
- Expert assignments and generated responses persisted per chat
- Survey evaluations linked to chats via `SurveyEvaluation` nodes
- Constraint: unique index on ChatHistory.id

---

*Integration audit: 2026-04-01*
