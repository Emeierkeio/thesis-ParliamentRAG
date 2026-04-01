# Project Research Summary

**Project:** ParliamentRAG Full Codebase Refactoring
**Domain:** Parliamentary RAG — Camera dei Deputati (XIX Legislatura)
**Researched:** 2026-04-02
**Confidence:** HIGH

---

## Executive Summary

ParliamentRAG is a production RAG system built on a fixed stack (Neo4j 5.x, FastAPI, Next.js) that is functionally correct but structurally compromised: dead Italian-schema code coexists with a live English schema, business logic is embedded in router files, duplicate service singletons create hidden connection pools, and schema property names are scattered as string literals across a dozen files. The refactoring goal is not to change what the system does but to make it maintainable, testable, and extensible — with a clean English-only codebase, proper separation of concerns, and a schema free of redundant properties.

The recommended approach is a strict bottom-up refactoring in five phases ordered by the dependency graph: build pipeline first (produces a clean schema), then backend services (update all Cypher consumers simultaneously with the schema change), then backend routers (thin them by extracting logic into services), then backend scripts (fix the router-import coupling violation), and finally the frontend (English URLs, strict TypeScript). The critical constraint is that the build pipeline schema changes and the backend Cypher updates must land together — the two layers are not independently deployable.

Beyond refactoring, two categories of improvements are actionable now. First, three high-value XML fields are being discarded by the production build path: Vote nodes (parse logic already exists, just not called), Debate-to-Act links (closes the biggest graph traversal gap), and Speaker institutional roles (enriches authority scoring with zero retrieval cost). Second, BM25 sparse retrieval via Neo4j's native full-text index requires no new dependencies and consistently improves retrieval quality for exact-term queries ("decreto 231", specific deputy names). The most significant medium-term enrichment is NER at ingestion time using the Italian spaCy model, which enables entity-filtered retrieval and law-reference graph edges.

---

## Key Findings

### 1. Stack — Conventions and Patterns

The technology stack is fixed. The research focused on the right patterns within that stack.

**Core findings:**

- **Neo4j naming:** Node labels are PascalCase (already correct in production schema), relationship types SCREAMING_SNAKE_CASE (already correct), property keys camelCase (currently violated — `start_date`, `surname_name`, `complete_date` need normalization). Multi-word property names are the primary violation category.
- **Transaction pattern:** Current `session.run()` auto-commit should migrate to `session.execute_read()` / `session.execute_write()` for automatic retry on transient errors. The one exception is SSE streaming, where partial results are already sent and retry is unsafe.
- **Build pipeline performance:** Per-item Neo4j writes are the primary build bottleneck. The `UNWIND $batch` pattern with 1,000–5,000 items per transaction is the correct replacement.
- **FastAPI DI:** The global `get_services()` dict should be replaced with typed `Depends()` functions using `@lru_cache` for process-lifetime singletons. This eliminates untyped dict access and enables unit testing.
- **Async boundary:** Do NOT migrate to AsyncDriver. Keep synchronous Neo4j driver with `run_in_executor`. Async migration is a separate non-trivial effort and out of scope.
- **Chunking:** The current 1,200-char / 250-char overlap values are validated against the corpus — do not change them. The only bug to fix is the `text.find()` silent sentence-skip.
- **Python:** Python 3.11+ built-in generics (`list[float]` not `List[float]`), `X | None` not `Optional[X]`, Google-style English docstrings, TypedDict for structured dict contracts.

See `/Users/mirkotritella/Desktop/ParliamentRAG/.planning/research/STACK.md` for full details.

---

### 2. New Data Extraction Opportunities

**From FEATURES.md (XML stenographic records — direct inspection of 365 files):**

| Feature | Priority | Value | Implementation Cost |
|---------|----------|-------|-------------------|
| Debate-to-Act links (`<argomenti>/<atto>`) | MUST | Highest graph value; enables Act→Debate→Speech traversal | Low — extract at parse time, MERGE edge |
| Vote nodes in production schema | MUST | Parse logic already exists; just not called in `_save_session_english()` | Trivial |
| Speaker institutional role (`<emphasis type="italic">`) | SHOULD | Enriches Speech metadata for authority scoring | Low |
| Phase type enum from title | SHOULD | Enables filtering by procedural context | Low |
| Act number regex from debate title | DEFER | Redundant if Debate-to-Act link implemented | Low |
| Inline group code from speech text | DEFER | Supplements existing Deputy→Group edge | Low |
| Session start/end time | DEFER | Minimal retrieval value | Low |

**Key confirmed gap:** The production `build_and_update.py` (`_save_session_english()`) never calls `parse_votazione()` — all Vote data is discarded. The parse logic exists and works; connecting it is the highest-ROI XML change.

**From ENRICHMENT.md (external data sources and NLP techniques):**

| Technique | Tier | Impact | Effort | Reprocess? |
|-----------|------|--------|--------|-----------|
| BM25 sparse channel via Neo4j full-text index | 1 | HIGH | 1-2 days | No |
| dati.camera.it SPARQL — deputy vote records | 1 | HIGH | 1-2 days | No |
| Speech→Act citation graph via regex | 1 | MEDIUM | 1 day | Yes |
| NER at ingestion (`it_nerIta_trf` or `it_core_news_lg`) | 2 | HIGH | 3-4 days | Yes |
| Entity-augmented Chunk metadata (law_refs, person_refs) | 2 | MEDIUM-HIGH | 2-3 days | Yes |
| Wikidata deputy biographical enrichment | 2 | MEDIUM | 1-2 days | No |
| dati.camera.it SPARQL — committee officer roles | 2 | MEDIUM | 1-2 days | No |

**Explicitly not worth building:** Relation extraction triples, stance/sentiment detection on formal Italian, full Akoma Ntoso XML migration, LKIF reasoning, EUR-Lex integration, cross-document coreference resolution.

See `/Users/mirkotritella/Desktop/ParliamentRAG/.planning/research/FEATURES.md` and `/Users/mirkotritella/Desktop/ParliamentRAG/.planning/research/ENRICHMENT.md`.

---

### 3. Architecture

The current architecture has the right structure in principle (routers → services → Neo4j) but is undermined by four coupling violations and a split identity problem (the build pipeline has both Italian and English schema paths in the same files).

**Major components (target state):**

| Component | Responsibility | Key Change |
|-----------|---------------|-----------|
| `build/pipeline/xml_parser.py` | Parse XML, preprocess, chunk | Extracted from `ingest_stenografici.py`; no Italian save path |
| `build/pipeline/orchestrator.py` | Full rebuild or incremental update | Renamed from `build_and_update.py` |
| `backend/app/services/experts.py` | Expert panel computation per party | NEW — extracted from `chat.py` + `query.py` (eliminates coupling violation) |
| `backend/app/services/evaluation_service.py` | All evaluation metric computation | NEW — extracted from `evaluation.py` router |
| `backend/app/routers/*` | Thin HTTP adapters only | Remove all inline business logic and Cypher queries |

**Four coupling violations to fix:**

1. `evaluation.py` router imports `_load_surveys` / `_calculate_stats` from `survey.py` router (cross-router import).
2. `seed_evaluation_topic.py` script imports `_compute_experts_for_frontend` from `chat.py` router (script→router).
3. `search.py` maintains its own `_neo4j_client` singleton instead of using `deps.py` (duplicate connection pool).
4. Pydantic models (`QueryRequest`, `CitationInfo`, `ExpertInfo`) defined inline in routers instead of `models/`.

**Schema properties being removed:** `Chunk.start_char_raw`, `Chunk.end_char_raw`, `Speech.preprocessed_text`, `Session.complete_date`. These appear in 12+ Cypher query sites across 6 backend files — all must be updated simultaneously with the schema rebuild.

See `/Users/mirkotritella/Desktop/ParliamentRAG/.planning/research/ARCHITECTURE.md`.

---

### 4. Critical Pitfalls

1. **Schema property rename without updating all Cypher consumers** (Pitfall 1) — `start_char_raw`/`end_char_raw` appear in `dense_channel.py`, `graph_channel.py`, `engine.py` (×2), `query.py`, `chat.py` (×2), and backend scripts. Neo4j returns `null` silently for missing properties. _Prevention: treat these 6 files as a mandatory atomic update group with the schema rebuild._

2. **Embedding cache key invalidation** (Pitfall 2) — The 329MB embedding cache key is `sha256(EMBEDDING_MODEL + "\n" + text)`. The model name `"text-embedding-3-small"` must be identical everywhere. Any change orphans the entire cache and triggers full re-embedding at significant API cost. _Prevention: never change the model name string or cache key format._

3. **SSE event contract breakage** (Pitfall 3) — The frontend `use-chat.ts` dispatches on `data.type` values from the query pipeline. The `experts` event is sent twice (first after retrieval, second after citation verification). Any rename of a `type` field or removal of a payload key silently breaks the frontend. _Prevention: treat SSE event types and payload field names as a frozen API contract during refactoring._

4. **Evaluation baseline breaks on expert field rename** (Pitfall 4) — `ChatHistory.experts` JSON in Neo4j and `evaluation_set.json` `baseline_experts` arrays are accessed by field name: `first_name`, `last_name`, `group`, `authority_score`. These are a frozen contract — any rename must use read-time aliases in evaluation.py and update the JSON file manually. _Prevention: document as no-change constraint in all phases touching expert computation._

5. **`ingest_stenografici.py` parser coupled to `build_and_update.py`** (Pitfall 5) — Cannot delete the file without first extracting `StenograficoIngester` to a new `xml_parser.py`. Calling `StenograficoIngester.__init__` (rather than `__new__`) triggers Italian schema creation. _Prevention: extract parser class before deleting anything; never split into separate commits without a working intermediate state._

**Additional moderate risks:** vector index name hardcoded in 6 files (Pitfall 6), `deputy_first_name`/`deputy_last_name` citation fields in 10+ frontend and backend locations (Pitfall 7), `populate_ruoli.py` references Italian label `Intervento` (Pitfall 8), zero test suite means all regressions are production-only (Pitfall 10).

See `/Users/mirkotritella/Desktop/ParliamentRAG/.planning/research/PITFALLS.md`.

---

## Implications for Roadmap

### Phase 1: Build Pipeline Refactoring

**Rationale:** The build pipeline is independent of the backend. Refactoring it first produces a clean English-only schema with correct properties (`span_start`/`span_end` instead of `start_char_raw`/`end_char_raw`; `text` only on Speech; `date` only on Session). All subsequent backend work targets the clean schema — otherwise Cypher queries must be updated twice.

**Delivers:** Clean `build/pipeline/` package with English-only code, no dead Italian paths, batched Neo4j writes, correct chunking (no silent sentence drops). Schema rebuilt with Vote nodes, Debate-to-Act links, and Speaker roles included. Dead code (`migrate_foti.py`, alignment map, `preprocessed_text`, `complete_date`) removed.

**Critical constraint:** Phase 1 and Phase 2 (Cypher updates) must be deployed as a unit. Do not deploy a new schema against the old backend.

**Pitfalls to avoid:** Pitfall 5 (extract parser before deleting), Pitfall 8 (update `populate_ruoli.py` Italian label), Pitfall 2 (preserve embedding cache).

**Research flag:** Standard patterns — no additional research needed.

---

### Phase 2: Backend Services Layer

**Rationale:** Services are the foundation for routers and scripts. Updating all Cypher consumers for the new schema properties, extracting business logic into services, and fixing dependency injection must happen before routers can be thinned.

**Delivers:** All Cypher queries updated to new property names. New `services/experts.py` (unified expert computation from `chat.py` and `query.py`). New `services/evaluation_service.py` (metric logic from `evaluation.py` router). `search.py` migrated to use `deps.py` singleton. Pydantic models consolidated in `models/`. FastAPI `Depends()` DI replacing global service dict. Full type hints and English docstrings throughout services.

**Critical constraint:** Two expert-computation implementations (`chat.py` `_compute_experts_for_frontend` and `query.py` `_compute_experts`) have subtly different behavior (the second SSE `experts` event). Read both fully before writing the unified service.

**Pitfalls to avoid:** Pitfall 1 (update all Cypher consumers simultaneously), Pitfall 4 (expert field names as frozen contract).

**Research flag:** Standard patterns for FastAPI DI. The expert service unification requires careful behavioral comparison of the two existing implementations — no external research needed, but thorough code reading is required.

---

### Phase 3: Backend API Routers

**Rationale:** After Phase 2, services exist for all business logic. Routers can be thinned to pure HTTP adapters without touching logic.

**Delivers:** All routers are thin (input validation → service call → response formatting). Cross-router imports eliminated. Italian error messages and SSE strings translated to English. `baseline-experts` endpoint moved from `history.py` to `evaluation.py` where it belongs. Dead endpoints removed.

**Critical constraint:** SSE event `type` values and payload field names in `query.py` yield statements must not change. The `waiting` event with its Italian string must be preserved (frontend depends on the event type).

**Pitfalls to avoid:** Pitfall 3 (SSE contract), Pitfall 7 (citation field names).

**Research flag:** Standard patterns — no additional research needed.

---

### Phase 4: Backend Scripts

**Rationale:** Scripts import from services and routers. After Phases 2–3 the correct imports exist. The primary fix is the coupling violation in `seed_evaluation_topic.py`.

**Delivers:** `seed_evaluation_topic.py` imports from `services/experts.py` (not `routers/chat`). All script Cypher queries updated for new schema properties. Docstrings, type hints, consistent error handling across all scripts.

**Research flag:** Standard patterns — no additional research needed.

---

### Phase 5: Frontend

**Rationale:** Frontend depends on API shape, not internal backend structure. Can run in parallel with Phase 4 or after.

**Delivers:** `/valutazione` → `/evaluation` URL rename (with redirect). Italian comments translated in TypeScript files. `[key: string]: any` removed from `Citation` interface. `any` types eliminated under `strict: true`. SSE event types formalized as a discriminated union in `types/api.ts`.

**Pitfalls to avoid:** Pitfall 3 (SSE type definitions must match backend yield statements), Pitfall 7 (citation field names must stay in sync with backend).

**Research flag:** Standard patterns — no additional research needed.

---

### Phase 6: Retrieval Enrichment (Post-Refactoring)

**Rationale:** BM25 sparse retrieval via Neo4j's native full-text index requires no new dependencies and no schema rebuild. It is the highest-impact, lowest-effort retrieval improvement and should follow the refactoring phases.

**Delivers:** `sparse_channel.py` added to `services/retrieval/`. `merger.py` updated to use Reciprocal Rank Fusion across dense + sparse + graph channels. Full-text index `chunk_fulltext` created on `Chunk.text`. Measurable improvement on exact-term queries.

**Research flag:** Standard patterns. Neo4j full-text index (Lucene-backed) is well-documented.

---

### Phase 7: Graph Enrichment (SPARQL + NER)

**Rationale:** SPARQL data from dati.camera.it (deputy vote records, committee officer roles) is incremental (no rebuild). NER at ingestion time requires a full rebuild — schedule after refactoring is stable.

**Delivers:** `SPARQLWrapper` utility for `dati.camera.it` and Wikidata queries. Deputy vote records linked in graph. Committee officer roles on `MEMBER_OF` edges. NER pipeline in build (using `it_core_news_lg` or `it_nerIta_trf`). `Chunk.law_refs` and `Chunk.person_refs` properties. Entity-filtered retrieval in `graph_channel.py`.

**Research flag:** NER version compatibility (`bullmount/it_nerIta_trf` requires spaCy 3.2.x — test against current stack; may need to use `it_core_news_lg` instead). Validate SPARQL queries against live endpoint before scheduling.

---

### Phase Ordering Rationale

- **Build-before-backend** because the schema change is the source of truth for all downstream Cypher. Reversing the order means all Cypher updates must be done twice.
- **Services-before-routers** because extracting `experts.py` and `evaluation_service.py` requires touching the same code that routers will later delegate to. One touch, not two.
- **Scripts after services+routers** because the coupling violation fix (`seed_evaluation_topic.py`) depends on `services/experts.py` existing.
- **Frontend last** because it is the only layer not blocked by the schema change. It can be done in parallel with Phase 4 if resources allow.
- **Enrichment after refactoring** because adding NER, SPARQL ingestion, and a sparse retrieval channel on top of a structurally compromised codebase would compound complexity. Clean foundation first.

---

## Priority Matrix

Impact vs effort for all recommendations (independent of phase):

| Recommendation | Impact | Effort | Phase | Priority |
|---------------|--------|--------|-------|----------|
| Fix schema property rename atomically (start_char_raw → span_start/end) | CRITICAL | Medium | 1+2 | P0 |
| Extract xml_parser.py (kill Italian dead code) | HIGH | Low | 1 | P0 |
| Add Vote nodes to production schema | HIGH | Trivial | 1 | P0 |
| Add Debate-to-Act links | HIGH | Low | 1 | P0 |
| Update `populate_ruoli.py` Italian label | HIGH | Trivial | 1 | P0 |
| Extract `services/experts.py` | HIGH | Medium | 2 | P1 |
| Extract `services/evaluation_service.py` | MEDIUM | Medium | 2 | P1 |
| FastAPI Depends() DI | MEDIUM | Medium | 2 | P1 |
| Fix `search.py` duplicate Neo4j singleton | MEDIUM | Low | 2 | P1 |
| Thin routers, translate Italian strings | MEDIUM | High | 3 | P1 |
| Fix `seed_evaluation_topic.py` coupling | MEDIUM | Trivial | 4 | P2 |
| Add Speaker institutional role extraction | MEDIUM | Low | 1 | P1 |
| Add Phase type enum | MEDIUM | Low | 1 | P1 |
| BM25 sparse retrieval channel | HIGH | Low | 6 | P1 |
| dati.camera.it SPARQL — vote records + committee roles | HIGH | Medium | 7 | P2 |
| NER at ingestion (law_refs, person_refs) | HIGH | Medium | 7 | P2 |
| Wikidata deputy biographical enrichment | MEDIUM | Low | 7 | P3 |
| Rename `/valutazione` → `/evaluation` | LOW | Trivial | 5 | P2 |
| Strict TypeScript / no `any` | MEDIUM | Medium | 5 | P2 |
| Full BERTopic topic modeling | MEDIUM | High | — | Defer |
| ReLiK entity linking (fine-tuned) | MEDIUM | Very High | — | Defer |
| Normattiva law text on-demand | MEDIUM | Medium | — | Defer |
| Relation extraction triples | LOW | Very High | — | Skip |
| Stance/sentiment detection | LOW | High | — | Skip |
| Full Akoma Ntoso XML migration | LOW | Very High | — | Skip |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All findings from official Neo4j, FastAPI, and Python docs. Async driver recommendation verified. |
| Features | HIGH | Based on direct inspection of 3 actual XML files + full-text search across all 365 files. No speculation. |
| Architecture | HIGH | Based on complete codebase file listing and import analysis. All coupling violations verified by grep. |
| Pitfalls | HIGH | All pitfalls verified from actual codebase files; specific line numbers confirmed. |
| Enrichment — OCD/SPARQL | HIGH | Direct endpoint inspection; ontology classes verified. |
| Enrichment — NER models | HIGH | HuggingFace model cards and spaCy docs verified. Version pin caveat confirmed. |
| Enrichment — BM25 | HIGH | Neo4j 5.x full-text index documentation; Lucene backend confirmed. |
| Enrichment — BERTopic | MEDIUM | 2024 papers; Italian-specific benchmarks limited. |
| Chunking recommendations | MEDIUM | General RAG research applied to parliamentary domain; values already validated against corpus. |

**Overall confidence:** HIGH

### Gaps to Address

- **NER version compatibility:** `bullmount/it_nerIta_trf` requires spaCy `>=3.2.1,<3.3.0`, which conflicts with the current Python stack. Validate during Phase 7 planning whether to use `it_core_news_lg` (fewer entity types, compatible) or isolate the old version in a subprocess. Decision does not affect earlier phases.
- **Embedding model name discrepancy:** PROJECT.md states `text-embedding-ada-002 @ 1536d` but actual code uses `text-embedding-3-small`. Both produce 1536d vectors — no functional problem, but the documentation should be corrected to match code during Phase 1 or 2.
- **`evidence.py` backend references `start_char_raw`:** The `evidence.py` router currently uses character offsets for sentence expansion. After the schema change to `span_start`/`span_end`, validate that the CitationSurgeon's sentence-boundary logic still produces correct output with the new offset values.

---

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection — all findings in ARCHITECTURE.md and PITFALLS.md verified with file paths and line numbers
- Official Neo4j Cypher Naming Rules and Style Guide — STACK.md §1
- Official Neo4j Python Driver 5.x Manual — STACK.md §2
- FastAPI official documentation — STACK.md §3
- OASIS LegalDocML / Akoma Ntoso v1.0 — ENRICHMENT.md §1.1
- dati.camera.it OCD ontology (direct endpoint inspection) — ENRICHMENT.md §1.2
- `bullmount/it_nerIta_trf` HuggingFace model card (F1=91.96) — ENRICHMENT.md §2.1
- Neo4j 5.x full-text index documentation (Lucene backend) — ENRICHMENT.md §4.1
- Direct inspection of 3 XML stenographic files (Sed. 1, 29, 100) — FEATURES.md

### Secondary (MEDIUM confidence)
- zhanymkanov/fastapi-best-practices community guide — STACK.md §3
- Weaviate / Firecrawl chunking strategy guides (2025) — STACK.md §5
- NAACL 2025 findings on fixed-size chunking — STACK.md §5
- BERTopic 2024 papers (Italian-specific benchmarks limited) — ENRICHMENT.md §4.6

---

*Research completed: 2026-04-02*
*Ready for roadmap: yes*
