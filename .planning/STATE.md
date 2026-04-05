---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: planning
stopped_at: Completed 07-02-PLAN.md
last_updated: "2026-04-05T20:40:50.906Z"
last_activity: 2026-04-02 — Roadmap created
progress:
  total_phases: 8
  completed_phases: 6
  total_plans: 26
  completed_plans: 24
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-02)

**Core value:** A clean, correct, and explainable codebase that is easy to maintain, extend, and reason about
**Current focus:** Phase 1 — Build Pipeline

## Current Position

Phase: 1 of 4 (Build Pipeline)
Plan: 0 of TBD in current phase
Status: Ready to plan
Last activity: 2026-04-02 — Roadmap created

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: -
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**
- Last 5 plans: -
- Trend: -

*Updated after each plan completion*
| Phase 01-build-pipeline P02 | 4min | 1 tasks | 5 files |
| Phase 01-build-pipeline P01 | 13 | 2 tasks | 7 files |
| Phase 01-build-pipeline P03 | 5 | 2 tasks | 2 files |
| Phase 01-build-pipeline P04 | 12 | 2 tasks | 5 files |
| Phase 02-backend P02 | 5 | 2 tasks | 6 files |
| Phase 02-backend P01 | 8min | 2 tasks | 12 files |
| Phase 02-backend P03 | 25min | 2 tasks | 8 files |
| Phase 02-backend P04 | 6min | 2 tasks | 3 files |
| Phase 02-backend P05 | 5min | 2 tasks | 6 files |
| Phase 02-backend P06 | 15min | 2 tasks | 3 files |
| Phase 03-frontend P01 | 525623min | 2 tasks | 17 files |
| Phase 03-frontend P02 | 10min | 2 tasks | 20 files |
| Phase 04-enrichment P02 | 2min | 2 tasks | 3 files |
| Phase 04-enrichment P01 | 4min | 2 tasks | 8 files |
| Phase 05-multi-language-support P01 | 18min | 2 tasks | 9 files |
| Phase 05-multi-language-support P02 | 30min | 2 tasks | 14 files |
| Phase 06-senate-data-integration-with-chamber-selector P02 | 4min | 2 tasks | 6 files |
| Phase 06-senate-data-integration-with-chamber-selector P01 | 4 | 2 tasks | 6 files |
| Phase 06-senate-data-integration-with-chamber-selector P03 | 6 | 2 tasks | 6 files |
| Phase 05-multi-language-support P03 | 5 | 2 tasks | 7 files |
| Phase 07-pipeline-optimization P01 | 18min | 2 tasks | 4 files |
| Phase 07-pipeline-optimization P02 | 8min | 2 tasks | 4 files |

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Phase 1: Extract xml_parser.py BEFORE deleting any code from ingest_stenografici.py (Pitfall 5)
- Phase 1: Embedding cache key must not change — model name string "text-embedding-3-small" is frozen
- Phase 1+2: Phase 1 schema rebuild and Phase 2 Cypher updates MUST deploy as a unit
- Phase 4: NER model version compatibility must be validated during planning (it_nerIta_trf requires spaCy 3.2.x — may need it_core_news_lg instead)
- [Phase 01-02]: Used re.split with captured delimiter instead of lookbehind regex for sentence splitting — Python re does not support variable-width lookbehinds
- [Phase 01-02]: build/ directory gitignored by Python convention; source files tracked via git add -f — gitignore needs update in Plan 01
- [Phase 01-02]: Orphan final chunk below min_speech_length merged into previous chunk rather than dropped
- [Phase 01-build-pipeline]: xml_parser.py has zero Neo4j dependency — StenograficoParser is a pure data extraction class
- [Phase 01-build-pipeline]: Votes parsed from raccoltaVotazioni at resoconto level — Session-[:HAS_VOTE]->Vote (not Debate-[:HAS_VOTE]->Vote)
- [Phase 01-build-pipeline]: preprocess_text() drops alignment_map return — no alignment_map logic in new codebase
- [Phase 01-build-pipeline]: DatabaseBuilder accepts an injected neo4j.Driver (no credentials in constructor)
- [Phase 01-build-pipeline]: load_government_members_from_path added as enriched variant alongside basic load_government_members
- [Phase 01-04]: csv_loader.py is standalone (no db_builder import) to prevent circular imports
- [Phase 01-04]: download.py requires explicit xml_dir param — no module-level default for path-agnosticism
- [Phase 01-04]: build_and_update.py calls load_government_members_from_path (CSV-enriched) for government member creation
- [Phase 02-backend]: lru_cache provides singleton semantics for FastAPI Depends without extra DI container
- [Phase 02-backend]: Kept get_services() backward-compat wrapper so unmigrated routers continue to work during incremental migration
- [Phase 02-backend]: conftest.py client fixture patches get_neo4j_client function (not removed _neo4j_client global) after deps.py rewrite
- [Phase 02-backend]: Pydantic model tests use source-file inspection (not live import) to avoid scipy/numpy NumPy 2.x incompatibility in the Python 3.12 anaconda environment
- [Phase 02-backend]: chunk_text replaces span-based extraction as citation source throughout evidence.py router (start_char_raw/end_char_raw removed from Phase 1 schema)
- [Phase 02-backend]: services/experts.py uses combined ranking formula 0.70*authority+0.30*similarity as canonical; authority_only available via param
- [Phase 02-backend]: Cross-router imports broken by extracting shared helpers to services/survey_helpers.py (both evaluation.py and survey.py can import from it)
- [Phase 02-backend]: Extracted evaluation metric computation verbatim to evaluation_service.py; router retains Neo4j I/O; all three MEMORY.md bug fixes preserved
- [Phase 02-backend]: TestClient functional tests use sys.modules stubs to bypass scipy/NumPy 2.x import chain in anaconda environment
- [Phase 02-backend]: data.py router registered directly in main.py (not via routers/__init__.py) to avoid changing package init
- [Phase 02-backend]: Source-file inspection tests avoid scipy/NumPy 2.x incompatibility in anaconda Python 3.12
- [Phase 02-backend]: SSE contract tests use multi-pattern matching (emit, emit_fn, sse_event, inline JSON) to detect all event emission styles
- [Phase 03-frontend]: StepResult.details typed as Record<string,unknown> — collapsed from union for simplicity
- [Phase 03-frontend]: GraphRecord = Record<string,unknown> with strProp() helper for type-safe string extraction
- [Phase 03-frontend]: Wire-format SSE values preserved (maggioranza_percentage, opposizione_percentage, commissioni wire field); only local TypeScript identifiers translated to English
- [Phase 03-frontend]: CoalitionFilter uses English majority/opposition values; filter predicate maps to wire maggioranza/opposizione
- [Phase 04-enrichment]: stdlib urllib used for SPARQL HTTP (no requests/httpx dep); IndividualVote.id format iv_{person_id}_{session}_{vote} for MERGE idempotency
- [Phase 04-enrichment]: RRF replaces weighted scoring in merger: rank-based fusion is parameter-free and cross-channel comparable
- [Phase 04-enrichment]: similarity=0.5 sentinel for sparse results: BM25 raw scores not comparable to cosine; rank position drives fusion
- [Phase 04-enrichment]: Italian analyzer with standard fallback in create_fulltext_index: Neo4j Community may not have italian analyzer
- [Phase 05-multi-language-support]: Cookie-based locale (NEXT_LOCALE) selected over [locale] URL routing to avoid mass route migration
- [Phase 05-multi-language-support]: translate_citation_batch returns original citation unchanged (no translated_* keys) on failure — cleaner fallback than partial dict
- [Phase 05-multi-language-support]: TRANSLATION_PROMPT instructs model to not translate proper nouns (speaker names, party names, dates, session numbers)
- [Phase 05-02]: Remove label/description/whyDescription from config.progressSteps; keep only id and icon — locale files are the single source of truth for UI text
- [Phase 05-02]: Pass tPi as prop to StepResultDetails (sub-function) to avoid hooks-in-non-component violation
- [Phase 05-02]: Use `as Parameters<typeof tPs>[0]` cast to allow dynamic key construction like `step${id}.label`
- [Phase 06-02]: chambers passed as list[str] not enum — each layer defaults to ['camera','senato'] so missing callers still get both chambers with no silent filtering
- [Phase 06-02]: graph_channel uses session_conditions list (not f-string date_clause append) so chamber condition is always first and no 1=1 workaround needed
- [Phase 06-01]: SenateStenograficoParser uses composition (not inheritance) to reuse StenograficoParser.preprocess_text()
- [Phase 06-01]: PRESIDENTE filtering uses TLCRole lookup + <from> text fallback for robustness in AKN XML
- [Phase 06-01]: Flat AKN structure: each debateSection maps to 1 debate + 1 phase (no sub-phases in Senate XML)
- [Phase 06-03]: ChamberSelector placed in sticky header below ChatInput row, right-aligned — visible on every message state
- [Phase 06-03]: chamber/onChamberChange props added to ChatArea (not managed internally) — follows existing prop-down pattern
- [Phase 05-03]: ChatRequest.locale injected by chat_endpoint from http_request.headers (not sent by client in body) — clean separation between transport and business logic
- [Phase 05-03]: TranslationBanner starts dismissed=true to avoid hydration flash, reads localStorage in useEffect
- [Phase 07-01]: retrieve_sync returns query_embedding in result dict — non-breaking, enables Plan 02 to skip duplicate embed_query call
- [Phase 07-01]: baseline_before_opt.json written as placeholder in NumPy 2.x local env; real benchmark run via Docker
- [Phase 07-02]: gpt-4.1-mini selected for all three generation stages — ~12x cost reduction vs gpt-4o
- [Phase 07-02]: query_embedding reused from retrieval_result dict — eliminates ~300ms embed_query call per request
- [Phase 07-02]: asyncio.gather for authority+compass — both CPU-bound with no mutual dependency, safe to parallelize

### Roadmap Evolution

- Phase 5 added: Multi-language support with translated citations and original-text hover disclaimer
- Phase 6 added: Senate data integration with chamber selector
- Phase 7 added: Pipeline optimization — analyze and optimize retrieval-to-generation pipeline for cost, latency, and quality
- Phase 8 added: Senate individual vote scraping from senato.it HTML pages

### Pending Todos

None yet.

### Blockers/Concerns

- [Pre-Phase 1] Embedding model name discrepancy: PROJECT.md says text-embedding-ada-002 but code uses text-embedding-3-small. Fix the doc during Phase 1 — do not change code.
- [Pre-Phase 2] SSE event contract (18 yield sites in query.py) must be documented before any router refactoring. Payload field names are a frozen contract.
- [Pre-Phase 4] NER version compatibility: bullmount/it_nerIta_trf requires spaCy >=3.2.1,<3.3.0. Validate against current stack before scheduling NER work.

## Session Continuity

Last session: 2026-04-05T20:40:50.903Z
Stopped at: Completed 07-02-PLAN.md
Resume file: None
