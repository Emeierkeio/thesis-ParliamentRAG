# ParliamentRAG — Build Pipeline Refactoring

## What This Is

A complete refactoring of the ParliamentRAG database build pipeline and backend query layer. The project transforms a working but messy ingestion system (mixed Italian/English schema, redundant properties, fragile chunking) into clean, explainable, best-practice code with a normalized English-only Neo4j schema. The end result is a single `make db-all` command that builds the entire database from scratch with fresh data.

## Core Value

A clean, correct, and explainable build pipeline that produces a well-structured Neo4j database optimized for the RAG retrieval system.

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

- [ ] Normalize entire Neo4j schema to English (node labels, properties, relationships)
- [ ] Remove `ingest_stenografici.py` Italian-schema save path (keep only parser logic)
- [ ] Consolidate build into single clean `build_and_update.py` with English-only output
- [ ] Remove redundant properties: `start_char_raw`/`end_char_raw` on Chunk, `preprocessed_text` on Speech (keep only `text`), `complete_date` on Session (use Neo4j Date)
- [ ] Clean up chunking code: remove dead alignment_map logic, simplify sentence splitting
- [ ] Keep Votazione nodes (English: `Vote`) for future RAG integration
- [ ] Update all backend Cypher queries to match new schema property names
- [ ] Single `make db-all` target that rebuilds entire DB from scratch with fresh data
- [ ] Code follows Python best practices: type hints, docstrings in English, no dead code
- [ ] Remove Italian-schema legacy code (constraints, indexes, save_to_neo4j method)

### Out of Scope

- Votazioni integration into RAG pipeline — future milestone
- Frontend changes — not affected by schema refactoring
- Changing the RAG retrieval logic itself — only updating property names in queries
- Changing embedding dimensions or model — keep OpenAI text-embedding-ada-002 @ 1536d
- Rewriting the XML parser from scratch — `parse_xml_file` logic is correct, just needs cleanup

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

**Backend files that need query updates:**
- `backend/app/services/neo4j_client.py` — vector search query
- `backend/app/services/retrieval/graph_channel.py` — graph retrieval
- `backend/scripts/compute_baseline_experts.py`
- `backend/scripts/enrich_evaluation_set.py`
- Various routers that read chunk/speech properties

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

---
*Last updated: 2026-04-02 after initialization*
