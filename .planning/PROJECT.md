# ParliamentRAG — Full Codebase Refactoring

## What This Is

A comprehensive refactoring of the entire ParliamentRAG codebase: build pipeline, backend services, API routers, utility scripts, and frontend. The project transforms a working but messy system (mixed Italian/English naming, redundant code, fragile logic) into clean, explainable, best-practice code. Normalized English-only Neo4j schema, clean Python services, structured FastAPI routers, and polished React frontend.

## Core Value

A clean, correct, and explainable codebase that is easy to maintain, extend, and reason about — from database build to frontend UI.

## Requirements

### Validated

- Parliament XML stenographic records are parsed and ingested into Neo4j
- Speeches are chunked for RAG retrieval with sentence-aware boundaries
- Text preprocessing removes parentheticals and speaker markers
- Continuation speeches (ellipsis pattern) are merged before chunking
- Deputies, GovernmentMembers, ParliamentaryGroups, Committees are loaded from CSV
- ParliamentaryActs are ingested from Camera API
- Institutional roles are assigned via app_config.py
- Vector index on Chunk.embedding for cosine similarity search
- Embeddings pre-calculated for Chunks, Speeches, Deputies, Committees, ParliamentaryActs
- Backend queries retrieve chunks via dense (vector) and graph channels

### Active

**Build Pipeline:**
- [ ] Normalize entire Neo4j schema to English (node labels, properties, relationships)
- [ ] Remove `ingest_stenografici.py` Italian-schema save path (keep only parser logic)
- [ ] Consolidate build into single clean `build_and_update.py` with English-only output
- [ ] Remove redundant properties: `start_char_raw`/`end_char_raw` on Chunk, `preprocessed_text` on Speech (keep only `text`), `complete_date` on Session (use Neo4j Date)
- [ ] Clean up chunking code: remove dead alignment_map logic, simplify sentence splitting
- [ ] Keep Vote nodes (renamed from Votazione) for future RAG integration
- [ ] Analyze XML stenographic files for additional extractable information (metadata, references, links to acts, etc.)
- [ ] Single `make db-all` target that rebuilds entire DB from scratch with fresh data

**Backend Services:**
- [ ] Refactor retrieval services (dense_channel, graph_channel, engine, merger) — clean naming, structure, dead code
- [ ] Refactor generation services (pipeline, sectional, integrator, etc.) — clean naming, structure
- [ ] Refactor authority scoring services — clean naming, ensure consistency
- [ ] Update all Cypher queries to match new schema property names

**Backend API:**
- [ ] Refactor FastAPI routers (query, chat, history, evaluation, search, authority, config)
- [ ] Clean up endpoint naming, request/response models
- [ ] Remove dead endpoints and unused code

**Backend Scripts:**
- [ ] Refactor utility scripts (seed_eval, compute_baseline, compute_spread, enrich_evaluation_set)
- [ ] Consistent naming, docstrings, error handling

**Frontend:**
- [ ] Refactor Next.js/React components — clean naming, structure, dead code
- [ ] Consistent TypeScript types and interfaces

**Code Quality (all layers):**
- [ ] Python: type hints, English docstrings, no dead code, consistent naming
- [ ] TypeScript: strict types, clean imports, no any
- [ ] Comments only where logic is non-obvious

### Out of Scope

- Votazioni integration into RAG pipeline — separate future milestone
- Changing the RAG retrieval algorithm itself — only cleaning implementation
- Changing embedding dimensions or model — keep OpenAI text-embedding-ada-002 @ 1536d
- Adding new features — this is purely about code quality and correctness
- Redesigning the frontend UI/UX — only code cleanup

## Context

**Current state:** The codebase has two ingestion paths:
1. `ingest_stenografici.py` — Italian schema (Seduta, Dibattito, Fase, Intervento), standalone
2. `build_and_update.py` — English schema (Session, Debate, Phase, Speech), uses parser from #1

Only #2 is used in production. The Italian `save_to_neo4j()` method and its constraints/indexes are dead code.

**Key bugs identified:**
- `start_char_raw`/`end_char_raw` have off-by-one errors and fragile alignment — but they're never used by the backend (it uses `chunk_text` directly), so removing them is the right fix
- `sentence_spans` silently skips sentences it can't find via `text.find()`, causing index misalignment
- Speech stores both `text` (raw) and `preprocessed_text` — only preprocessed is used for chunking/retrieval

**Schema redundancies:**
- Chunk: `start_char_raw`, `end_char_raw` (unused)
- Speech: `text` + `preprocessed_text` (keep only preprocessed as `text`)
- Session: `complete_date` string + `date` Neo4j Date (keep only `date`)
- Speech: `surname_name` (only used for orphan reconciliation fallback — keep but rename)

**XML analysis opportunity:**
The stenographic XML files may contain additional extractable information (references to acts, cross-links between debates, procedural metadata) that isn't currently being captured. Research phase should analyze sample XMLs to identify valuable data.

**Backend files that need query updates:**
- `backend/app/services/neo4j_client.py` — vector search query
- `backend/app/services/retrieval/graph_channel.py` — graph retrieval
- `backend/scripts/compute_baseline_experts.py`
- `backend/scripts/enrich_evaluation_set.py`
- Various routers that read chunk/speech properties

**Full codebase scope:**
- Backend services: ~15 Python modules in `backend/app/services/`
- Backend routers: 7 FastAPI routers in `backend/app/routers/`
- Backend scripts: 5 utility scripts in `backend/scripts/`
- Frontend: Next.js app in `frontend/src/`

## Constraints

- **Tech stack**: Python 3.11+, Neo4j 5.x, FastAPI backend — no changes
- **Backward compatibility**: Database will be rebuilt from scratch, no migration needed
- **Embeddings cache**: `embeddings_cache.db` (344MB) must be preserved — avoids re-computing all embeddings
- **API contract**: Backend API responses to frontend must not change shape
- **Data sources**: Camera XML API + CSV from dati.camera.it — no changes to download logic

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Remove start_char_raw/end_char_raw | Never used by backend, buggy implementation | — Pending |
| Keep Speech.text as preprocessed only | Raw text not needed by any consumer | — Pending |
| Rename Votazione → Vote, keep in schema | User wants future RAG integration with votes | — Pending |
| Remove Italian save path entirely | Only English schema used in production | — Pending |
| Update backend queries in same milestone | Avoid broken state between build and backend | — Pending |
| Full codebase refactoring scope | User wants expert-level cleanup of everything, not just build | — Pending |
| Analyze XML for additional extractable data | May reveal valuable metadata not currently captured | — Pending |

---
*Last updated: 2026-04-02 after scope expansion to full codebase*
