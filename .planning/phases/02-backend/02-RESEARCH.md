# Phase 2: Backend - Research

**Researched:** 2026-04-02
**Domain:** FastAPI backend refactoring — Cypher schema migration, service extraction, DI, SSE contract, tests
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Cypher Query Updates**
- Remove `start_char_raw`/`end_char_raw` completely from all Cypher queries, Pydantic models, and API responses. The frontend does not use `span_start`/`span_end`. Do not keep as 0 placeholder — remove the fields entirely.
- `i.text` is now preprocessed only — update silently. Remove any comments referencing "raw text" or "preprocessed_text" distinction. `sp.text` / `i.text` is simply "the speech text".
- Expose new schema data via basic endpoints: GET endpoint for votes (linked to sessions), GET endpoint for act-debate links (DISCUSSES relationships), speakingRole and phaseType available in existing query responses where Speech/Phase are returned.
- Blast radius files for Cypher updates: `neo4j_client.py`, `graph_channel.py`, `evidence.py`, `query.py`, `chat.py`, `search.py`, `scorer.py`, `evaluation.py`, `compute_baseline_experts.py`, `enrich_evaluation_set.py`

**Service Extraction**
- Two new service modules: `backend/app/services/experts.py` and `backend/app/services/evaluation_service.py`
- FastAPI DI migration: Replace `get_services()` dict in deps.py with typed `Depends()` functions using `@lru_cache` for singleton semantics.
- Thin routers: Routers contain ONLY input validation + service call + output formatting. Zero business logic in routers.
- Fix coupling violations: evaluation.py must NOT import from survey.py router; seed_evaluation_topic.py must NOT import from chat.py router; search.py must use shared Neo4j client from deps.py.

**SSE Event Contract**
- Document ALL 18 event types BEFORE touching query.py — create an SSE contract document listing every event name, payload shape, and emission order.
- Event types and payload field names are FROZEN — no renames, no restructuring.
- Keep snake_case in SSE payloads — `authority_score`, `group`, `first_name` etc. stay as-is.
- Keep dual `experts` emission — first before generation, second after citation verification.
- SSE logic can move to a service but the yield points and event structure must remain identical.

**Smoke Tests**
- Comprehensive unit tests, not just smoke tests. Full coverage of service functions, with mocked Neo4j driver for logic tests.
- Test location: `backend/tests/` (separate directory in backend root).
- Strategy: Unit tests with mock Neo4j driver + integration tests with `@pytest.mark.integration`.
- Target: ~100+ tests.
- Critical paths: expert computation, evaluation metrics, retrieval pipeline, authority scoring, SSE event emission order and payload shapes.

### Claude's Discretion
- Exact module file layout within `backend/app/services/`
- How to structure the SSE contract document
- Test fixture design and conftest.py organization
- Order of refactoring within the phase (which files first)
- Whether to use pytest-asyncio for async endpoint tests
- How much of the generation pipeline to refactor vs leave as-is

### Deferred Ideas (OUT OF SCOPE)
- BM25 sparse retrieval channel — Phase 4 (RET-01)
- SPARQL enrichment from dati.camera.it — Phase 4 (ENR-01, ENR-02)
- NER entity extraction — Phase 4 (ENR-03, ENR-04)
- Frontend refactoring — Phase 3
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SVC-01 | Update all Cypher queries across 6+ modules to match new camelCase property names | Dead property audit (section below) identifies every exact line needing change |
| SVC-02 | Extract expert computation into `services/experts.py` | Diff of the two implementations documented; unified signature specified |
| SVC-03 | Extract evaluation business logic into `services/evaluation_service.py` | evaluation.py at 959 lines is the riskiest refactor; boundary is clear (all metric computation functions) |
| SVC-04 | Replace `get_services()` dict with FastAPI `Depends()` typed DI | Current pattern documented; FastAPI lru_cache pattern specified |
| SVC-05 | Fix search.py duplicate Neo4j connection pool | search.py creates its own `_neo4j_client` global; fix is to use `get_neo4j_client()` from deps.py |
| SVC-06 | Clean naming, type hints, English docstrings across all service modules | No research blocker; code quality standard documented |
| API-01 | Refactor routers to thin wrappers around services | chat.py at 1,792 lines has ~800 lines of business logic to extract |
| API-02 | Fix cross-router import violations | Violations enumerated: evaluation.py→survey.py, seed_evaluation_topic.py→chat.py |
| API-03 | Freeze SSE event contract | Full 18-event inventory documented in SSE Contract section |
| API-04 | Clean endpoint naming, Pydantic v2 models, consistent error handling | `CitationInfo` model in query.py has `span_start`/`span_end` to remove |
| API-05 | Preserve API response shapes | Frontend contract field names documented; frozen fields identified |
| SCR-01 | Refactor utility scripts with consistent naming, docstrings, error handling | Scripts enumerated with sizes |
| SCR-02 | Fix seed_evaluation_topic.py router-import coupling | Imports `_compute_experts_for_frontend` from `app.routers.chat` — replace with service call |
| SCR-03 | Scripts use shared Neo4j client instead of creating own connections | compute_baseline_experts.py creates its own driver directly |
| QA-01 | Add smoke tests for critical paths | No existing tests; Wave 0 must create backend/tests/ directory and fixtures |
| QA-02 | Python code follows best practices | Standard; no research blocker |
| QA-03 | Consistent naming conventions | camelCase Neo4j properties, snake_case Python — already the standard |
</phase_requirements>

---

## Summary

Phase 2 refactors the entire backend to match the Phase 1 Neo4j schema and apply clean-code principles. There are three work streams: (1) Cypher schema migration — remove dead properties `start_char_raw`/`end_char_raw` from 9 files, touching 15 distinct Cypher sites; (2) service extraction — pull duplicated expert computation and evaluation business logic into dedicated service modules, replace the global `get_services()` dict with typed FastAPI `Depends()`; (3) tests — build `backend/tests/` from scratch targeting 100+ unit and integration tests.

The riskiest element is `evaluation.py` (959 lines, multiple historical bug fixes documented in MEMORY.md). It must be refactored with surgical care: the metric computation functions move to `evaluation_service.py` while the router keeps only HTTP orchestration. The SSE contract in both `chat.py` and `query.py` must be documented before any changes — field names and emission order are a frozen frontend contract.

The two `_compute_experts_*` functions are nearly identical in algorithm (both select top-authority speaker per party, fetch details in parallel) but differ in one important detail: `_compute_experts_for_frontend` in `chat.py` uses a combined score `0.70 * authority + 0.30 * best_chunk_similarity` for ranking while `_compute_experts` in `query.py` ranks by `authority_score` alone. The unified service must use the chat.py formula (more accurate) as the single implementation.

**Primary recommendation:** Tackle SVC-01 (Cypher migration) in Wave 1 since it is mechanical and makes the codebase deployable against the Phase 1 DB. Then extract services (SVC-02, SVC-03) in Wave 2 since they depend on knowing the existing code deeply. Fix DI and coupling violations (SVC-04, SVC-05, API-02) in Wave 3. Tests (QA-01) should be written alongside each wave.

---

## Standard Stack

### Core (already in use — do not change)
| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| FastAPI | current in venv | HTTP framework, SSE streaming | Keep `StreamingResponse` pattern |
| Pydantic v2 | current in venv | Request/response models | Remove `span_start`/`span_end` fields |
| neo4j Python driver | current in venv | Cypher execution | Keep `execute_read`/`execute_write` pattern from Phase 1 |
| pytest | current in venv | Test runner | No existing test suite — build from scratch |
| pytest-asyncio | needs install check | Async test support | For router tests via TestClient |
| httpx | needs install check | FastAPI TestClient dep | For router integration tests |

### Test-Specific Additions
| Library | Purpose | When to Use |
|---------|---------|-------------|
| pytest-asyncio | Async test functions | Router tests calling `async def` endpoints |
| unittest.mock | Neo4j driver mocking | Unit tests for service logic (no live DB) |
| httpx | ASGI test transport | FastAPI `TestClient` already includes this |

**Installation check:**
```bash
pip show pytest-asyncio httpx
```

---

## Architecture Patterns

### Recommended Project Structure (after refactoring)
```
backend/
├── app/
│   ├── routers/
│   │   ├── chat.py          # Thin: delegates to services
│   │   ├── query.py         # Thin: delegates to services
│   │   ├── evaluation.py    # Thin: delegates to evaluation_service
│   │   ├── evidence.py      # Thin: delegates to neo4j_client
│   │   ├── search.py        # Uses shared client from deps.py
│   │   └── ...
│   ├── services/
│   │   ├── deps.py          # Typed Depends() functions (replaces get_services dict)
│   │   ├── experts.py       # NEW: unified expert computation
│   │   ├── evaluation_service.py  # NEW: evaluation business logic
│   │   ├── neo4j_client.py  # Keep class, update Cypher strings
│   │   ├── authority/       # Keep, update property names
│   │   └── retrieval/       # Keep, update Cypher strings
│   └── models/
│       ├── evidence.py      # Remove span_start/span_end
│       └── ...
├── tests/
│   ├── conftest.py          # Shared fixtures (mock driver, test app)
│   ├── unit/
│   │   ├── test_experts.py
│   │   ├── test_evaluation_service.py
│   │   ├── test_authority_scorer.py
│   │   └── ...
│   └── integration/         # Marked @pytest.mark.integration
│       └── test_retrieval.py
└── scripts/                 # Use shared client, not own connections
```

### Pattern 1: FastAPI Typed Dependency Injection
**What:** Replace `get_services() -> dict` with individual typed `Depends()` functions.
**When to use:** All routers that currently call `get_services()`.

```python
# backend/app/services/deps.py — NEW pattern
from functools import lru_cache
from fastapi import Depends
from .neo4j_client import Neo4jClient
from .retrieval import RetrievalEngine
from .authority import AuthorityScorer
from ..config import get_settings, Settings

@lru_cache()
def get_neo4j_client() -> Neo4jClient:
    settings = get_settings()
    return Neo4jClient(
        uri=settings.neo4j_uri,
        user=settings.neo4j_user,
        password=settings.neo4j_password,
    )

@lru_cache()
def get_retrieval_engine() -> RetrievalEngine:
    return RetrievalEngine(get_neo4j_client())

@lru_cache()
def get_authority_scorer() -> AuthorityScorer:
    return AuthorityScorer(get_neo4j_client())

# In router:
@router.post("/endpoint")
async def endpoint(
    neo4j: Neo4jClient = Depends(get_neo4j_client),
    retrieval: RetrievalEngine = Depends(get_retrieval_engine),
):
    ...
```

**Critical note:** `@lru_cache()` on module-level functions gives singleton semantics. This replaces the `global _neo4j_client` pattern currently in `deps.py`, `search.py`, and `evidence.py`.

### Pattern 2: Thin Router
**What:** Router function does only: parse input → call service → format output → return response.
**When to use:** All router functions that currently contain business logic.

```python
# Before (business logic in router):
@router.get("/experts")
async def get_experts(query: str):
    services = get_services()
    evidence_list = services["retrieval"].retrieve(...)
    authority_scores = {}
    for sid in speaker_ids:
        authority_scores[sid] = services["authority"].compute(...)
    experts = _build_experts_list(evidence_list, authority_scores, ...)
    return {"experts": experts}

# After (thin router):
@router.get("/experts")
async def get_experts(
    query: str,
    expert_service: ExpertService = Depends(get_expert_service),
):
    experts = await expert_service.compute_for_query(query)
    return {"experts": experts}
```

### Pattern 3: Unified Expert Service Signature
The unified `services/experts.py` must support both the pre-generation and post-generation expert computation paths. Based on the diff between the two implementations:

```python
# backend/app/services/experts.py
async def compute_experts(
    evidence_list: List[UnifiedEvidence],
    authority_scores: Dict[str, float],
    authority_details: Dict[str, Dict[str, Any]],
    neo4j_client: Neo4jClient,
    ranking_formula: str = "combined",  # "combined" (chat) or "authority_only" (query)
) -> List[Dict[str, Any]]:
    """
    Compute one expert per party from evidence list.

    ranking_formula="combined": 0.70 * authority + 0.30 * best_chunk_similarity
    ranking_formula="authority_only": authority_score only

    Use "combined" for the standard pipeline (matches chat.py behavior).
    """
    ...
```

The locked decision is to use the chat.py formula (`combined`) as the standard — it is more accurate. The `query.py` variant should be migrated to also use `combined`.

### Anti-Patterns to Avoid
- **Router importing from router:** `evaluation.py` currently imports `_load_surveys`, `_calculate_stats` from `survey.py`. These are private functions of a sibling router. They must move to a service.
- **Script importing from router:** `seed_evaluation_topic.py` imports `_compute_experts_for_frontend` from `app.routers.chat`. Scripts may only import from `app.services.*`.
- **Module-level Neo4j client globals:** `search.py` and `evidence.py` both maintain `_neo4j_client: Optional[Neo4jClient] = None` globals with their own init logic. Replace with `Depends(get_neo4j_client)`.
- **Business logic in router functions:** `chat.py` has 1,792 lines; the actual HTTP handlers are a small fraction. The rest is authority scoring orchestration, citation resolution, expert computation — all of which belong in services.

---

## Dead Property Audit: Complete Cypher Hit Map

The Phase 1 schema (confirmed from `build/db_builder.py`) removes these Chunk properties:
- `start_char_raw` — removed (BUILD-04)
- `end_char_raw` — removed (BUILD-04)
- `preprocessed_text` on Speech — removed (BUILD-05); `sp.text` is now the only text field
- `complete_date` on Session — removed (BUILD-06); `s.date` is the only date field

The new properties added:
- `Speech.speakingRole` — string enum (e.g., "president", "deputy", "government")
- `Phase.phaseType` — string enum (e.g., "government_opinion", "vote_declaration", "general_discussion")
- `Vote` nodes with `Session-[:HAS_VOTE]->Vote`
- `Debate-[:DISCUSSES]->ParliamentaryAct` edges

### File-by-File Dead Property Hits

**`backend/app/services/neo4j_client.py`** (358 lines)
| Line | Dead Property | Context | Fix |
|------|--------------|---------|-----|
| 159 | `c.start_char_raw AS span_start` | `vector_search()` RETURN clause | Remove both lines; remove from returned dict |
| 160 | `c.end_char_raw AS span_end` | `vector_search()` RETURN clause | Remove both lines |

**`backend/app/services/retrieval/graph_channel.py`** (407 lines)
| Line | Dead Property | Context | Fix |
|------|--------------|---------|-----|
| 285 | `c.start_char_raw AS span_start` | `_get_chunks_from_signatories()` RETURN | Remove |
| 286 | `c.end_char_raw AS span_end` | `_get_chunks_from_signatories()` RETURN | Remove |
| 341 | Comment referencing `start_char_raw/end_char_raw` | `_process_results()` | Delete comment |
| 336 | `span_start = row.get("span_start", 0)` | Processing dead return values | Remove; also remove span_start/span_end from processed dict on lines 396-397 |

**`backend/app/routers/query.py`** (1,061 lines)
| Line | Dead Property | Context | Fix |
|------|--------------|---------|-----|
| 88-89 | `span_start: int` / `span_end: int` | `CitationInfo` Pydantic model | Remove both fields |
| 284 | `c.start_char_raw AS span_start` | Extra citation DB lookup query | Remove |
| 285 | `c.end_char_raw AS span_end` | Extra citation DB lookup query | Remove |
| 339-340 | `"span_start": row.get("span_start", 0)` / `"span_end"` | Building extra_evidence_map | Remove both |
| 394-397 | `"span_start": ev.get("span_start", 0)` | Recovering citations from text | Remove |

**`backend/app/routers/chat.py`** (1,792 lines)
| Line | Dead Property | Context | Fix |
|------|--------------|---------|-----|
| 427 | `c.start_char_raw AS span_start` | Extra citation DB lookup (process_chat_background) | Remove |
| 428 | `c.end_char_raw AS span_end` | Extra citation DB lookup (process_chat_background) | Remove |
| 475-476 | `"span_start": row.get("span_start", 0)` | Building extra_evidence_map (background) | Remove |
| 910 | `c.start_char_raw AS span_start` | Extra citation DB lookup (process_chat_streaming) | Remove |
| 911 | `c.end_char_raw AS span_end` | Extra citation DB lookup (process_chat_streaming) | Remove |
| 958-959 | `"span_start": row.get("span_start", 0)` | Building extra_evidence_map (streaming) | Remove |
| 1028-1030 | `"span_start": ev.get("span_start", 0)` | Citation recovery from text | Remove |

**`backend/app/routers/evidence.py`** (226 lines)
| Line | Dead Property | Context | Fix |
|------|--------------|---------|-----|
| 97 | `c.start_char_raw AS span_start` | `get_evidence()` main Cypher | Remove |
| 98 | `c.end_char_raw AS span_end` | `get_evidence()` main Cypher | Remove |
| 187 | `c.start_char_raw AS span_start` | `verify_evidence()` Cypher | Remove |
| 188 | `c.end_char_raw AS span_end` | `verify_evidence()` Cypher | Remove |

**`backend/app/routers/evaluation.py`** (959 lines)
No direct `start_char_raw`/`end_char_raw` hits found in grep. The evaluation router uses stored `ChatHistory` data from Neo4j which uses field names `first_name`, `last_name`, `group`, `authority_score` — these are frozen stored-data field names, not schema properties, so they are NOT changed.

**`backend/app/services/authority/scorer.py`** (705 lines)
No direct `start_char_raw`/`end_char_raw` hits. The scorer queries `interventions` and `acts` data — no Chunk properties involved. No Cypher changes required here.

**`backend/scripts/compute_baseline_experts.py`** (777 lines)
The script creates its own Neo4j driver. Check for Chunk queries — none found in the first 80 lines. The main risk is the script querying `i.text` which is now correct (preprocessed only). Need to audit remaining 700 lines for any span/char references.

**`backend/scripts/enrich_evaluation_set.py`** (376 lines)
Not read in full — needs audit for span_start/span_end references.

**`backend/app/models/evidence.py`** (272 lines)
`UnifiedEvidence` Pydantic model has `span_start: int` and `span_end: int` fields. Also `EvidenceResponse` in `evidence.py` router has `span_start: int` / `span_end: int`. These must be removed from all Pydantic models.

### Properties NOT Changing (confirmed correct in Phase 1 schema)
- `c.text` — Chunk text (camelCase already; was `preprocessed_text` but now just `text`)
- `sp.text` / `i.text` — Speech text (correct in new schema)
- `s.date` — Session date as Neo4j Date type (correct)
- `s.number` — Session number (correct)
- `d.title` — Debate title (correct)
- `d.first_name`, `d.last_name` — Deputy name properties (snake_case, intentional legacy)
- `c.index` — Chunk index within speech (correct)
- `c.embedding` — Chunk embedding vector (correct)

---

## SSE Event Contract (FROZEN)

### chat.py (`process_chat_background` and `process_chat_streaming`)
Both functions emit the same event sequence. `process_chat_background` stores to TaskStore; `process_chat_streaming` yields directly.

| # | Event Type | Payload Fields | When Emitted |
|---|-----------|----------------|--------------|
| 1 | `waiting` | `queue_position`, `ahead_count`, `active_count`, `elapsed_seconds` | Only if semaphore locked |
| 2 | `progress` | `step: 1`, `total: 8`, `message: "Analisi query"` | Immediately on start |
| 3 | `progress` | `step: 2`, `total: 8`, `message: "Commissioni"` | Before commission matching |
| 4 | `commissioni` | `commissioni: List[{...}]` | After commission matching |
| 5 | `progress` | `step: 3`, `total: 8`, `message: "Esperti"` | Before retrieval+authority |
| 6 | `experts` | `experts: List[ExpertDict]` | After first compute_experts (pre-generation) |
| 7 | `progress` | `step: 4`, `total: 8`, `message: "Interventi"` | Before citations build |
| 8 | `citations` | `citations: List[CitationDict]` | After building initial citation list |
| 9 | `progress` | `step: 5`, `total: 8`, `message: "Statistiche"` | Before balance computation |
| 10 | `balance` | `maggioranza_percentage`, `opposizione_percentage`, `bias_score`, ... | After balance metrics |
| 11 | `progress` | `step: 6`, `total: 8`, `message: "Bussola Ideologica"` | Before compass |
| 12 | `compass` | `meta`, `axes`, `groups`, `scatter_sample` | After compass computation |
| 13 | `progress` | `step: 7`, `total: 8`, `message: "Generazione"` | Before generation |
| 14 | `topic_stats` | `intervention_count`, `speaker_count`, `first_date`, `last_date`, `speakers_detail`, `interventions_detail`, `sessions_detail` | After generation, if topic_statistics present |
| 15 | `chunk` | `content: str` (50 chars at a time) | During text streaming |
| 16 | `citation_details` | `citations: List[VerifiedCitationDict]` | After text streaming complete |
| 17 | `experts` | `experts: List[ExpertDict]` | Second emission after _patch_experts_for_cited_speakers (only if patched) |
| 18 | `hq_variants` | `variants: [{text, score, is_best}]` | Only in high_quality mode |
| 19 | `complete` | `metadata: {timing, dense_channel_count, graph_channel_count, ...}` | Final event |
| 20 | `error` | `message: str` | On exception |

### query.py (`process_query_streaming`)
Different pipeline — shorter, no background task pattern.

| # | Event Type | Payload Fields | When Emitted |
|---|-----------|----------------|--------------|
| 1 | `waiting` | `message: str` | If semaphore at capacity |
| 2 | `progress` | `step: 1`, `message: "Avvio retrieval..."` | Start |
| 3 | `progress` | `step: 2`, `message: f"Trovate {n} evidenze"` | After retrieval |
| 4 | `progress` | `step: 3`, `message: "Calcolo authority scores..."` | Before authority |
| 5 | `experts` | `data: List[ExpertDict]` | After _compute_experts (pre-generation) |
| 6 | `progress` | `step: 4`, `message: "Analisi compass ideologico..."` | Before compass |
| 7 | `compass` | `data: {meta, axes, groups, scatter_sample}` | After compass |
| 8 | `progress` | `step: 5`, `message: "Generazione risposta multi-view..."` | Before generation |
| 9 | `topic_stats` | same as chat.py | If topic_statistics present |
| 10 | `citations` | `data: List[CitationDict]` | After initial citation build |
| 11 | `citation_details` | `citations: List[VerifiedCitationDict]` | After verification |
| 12 | `experts` | `data: List[ExpertDict]` | Second emission after citation-aligned update |
| 13 | `chunk` | `data: str` (100 chars at a time) | Text streaming |
| 14 | `complete` | `metadata: dict` | Final event |
| 15 | `error` | `message: str` | On exception |

**Key difference from chat.py:** `chunk` payload uses `data:` key in query.py vs `content:` key in chat.py. Both are frozen. The frontend likely handles both.

**Expert dict shape (frozen):**
```json
{
  "id": "speaker_id_string",
  "first_name": "Mario",
  "last_name": "Rossi",
  "group": "Partito Democratico - ...",
  "coalition": "opposizione",
  "authority_score": 0.72,
  "relevant_speeches_count": 5,
  "camera_profile_url": "https://...",
  "photo": "https://...",
  "profession": "Avvocato",
  "education": "Laurea in Giurisprudenza",
  "committee": "Commissione Giustizia",
  "institutional_role": null,
  "score_breakdown": {
    "speeches": 0.81,
    "acts": 0.63,
    "committee": 0.55,
    "profession": 0.40,
    "education": 0.35,
    "role": 0.00
  }
}
```

---

## Expert Computation Duplication: Diff Analysis

### `chat.py::_compute_experts_for_frontend` (line 1355)
- Ranking formula: `0.70 * authority_score + 0.30 * best_chunk_similarity`
- Tracks `best_chunk_similarity` per speaker across evidence pieces
- Handles `party_changed` → uses `current_party` if speaker switched groups
- Has `_patch_experts_for_cited_speakers` post-generation step (separate function, not inlined)

### `query.py::_compute_experts` (line 594)
- Ranking formula: `authority_score` only (no similarity component)
- Does not track `best_similarity`
- Does not handle `party_changed` / `current_party`
- Has inline post-generation patch (lines 422-502) that replaces pre-generation experts

### Key Differences Table
| Feature | chat.py version | query.py version |
|---------|-----------------|------------------|
| Ranking formula | 0.70 auth + 0.30 sim | authority only |
| Handles party_changed | Yes | No |
| Post-gen patch | Separate `_patch_experts_for_cited_speakers` | Inlined in `process_query_streaming` |
| Speaker name parsing | `speaker_name.split(" ", 1)` | `speaker_name.split(" ", 1)` — identical |
| Expert dict fields | Identical | Identical |

### Unification Decision (from CONTEXT.md)
Use the chat.py formula as the unified implementation. The `query.py` version was an older, simpler variant. The unified service must:
1. Accept optional `ranking_formula` param defaulting to `"combined"` (chat.py behavior)
2. Handle `party_changed` / `current_party` (chat.py behavior)
3. Return the same expert dict shape both callers expect

---

## Cross-Router Coupling Violations

### Violation 1: `evaluation.py` imports from `survey.py`
**File:** `backend/app/routers/evaluation.py`, line 29
```python
from app.routers.survey import _load_surveys, _calculate_stats
```
**Problem:** `_load_surveys` and `_calculate_stats` are private helpers of the survey router. Evaluation depends on survey's internals.
**Fix:** Move `_load_surveys` and `_calculate_stats` to a new `services/survey_service.py` (or `services/evaluation_service.py`). Both routers import from the service.

### Violation 2: `seed_evaluation_topic.py` imports from `chat.py`
**File:** `backend/scripts/seed_evaluation_topic.py`, line 34
```python
from app.routers.chat import _compute_experts_for_frontend
```
**Problem:** A script may not import from a router. Routers are HTTP layer; scripts should use services.
**Fix:** After `services/experts.py` is created, `seed_evaluation_topic.py` imports `compute_experts` from the service.

### Violation 3: `search.py` duplicate Neo4j connection pool
**File:** `backend/app/routers/search.py`, lines 18-33
```python
_neo4j_client: Optional[Neo4jClient] = None

def get_client() -> Neo4jClient:
    global _neo4j_client
    if _neo4j_client is None:
        settings = get_settings()
        _neo4j_client = Neo4jClient(...)
    return _neo4j_client
```
**Problem:** Creates a second Neo4j connection pool. At peak load this doubles connections.
**Fix:** Replace `get_client()` with `Depends(get_neo4j_client)` from deps.py. Pass `neo4j_client` as parameter to all `_search_*` helper functions.

### Violation 4: `evidence.py` local Neo4j client
**File:** `backend/app/routers/evidence.py`, lines 53-66
```python
_neo4j_client: Optional[Neo4jClient] = None

def get_neo4j():
    global _neo4j_client
    ...
```
**Same fix:** Replace with `Depends(get_neo4j_client)`.

---

## Current DI Pattern vs. Target Pattern

### Current (`deps.py`)
```python
_neo4j_client: Optional[Neo4jClient] = None
# ... 4 more globals

def get_services() -> dict:
    global _neo4j_client, ...
    if _neo4j_client is None:
        # init all at once
    return {"neo4j": ..., "retrieval": ..., ...}
```
**Problems:**
- Returns untyped dict — `services["neo4j"]` has no type checking
- All services initialized together even if only one is needed
- Cannot be overridden in tests without monkey-patching globals

### Target (FastAPI `Depends` + `lru_cache`)
```python
from functools import lru_cache
from fastapi import Depends

@lru_cache()
def get_neo4j_client() -> Neo4jClient:
    settings = get_settings()
    return Neo4jClient(uri=..., user=..., password=...)

@lru_cache()
def get_retrieval_engine() -> RetrievalEngine:
    return RetrievalEngine(get_neo4j_client())
```
**Benefits:**
- Fully typed (`Neo4jClient` not `dict["neo4j"]`)
- Lazy — each service initialized only when first requested
- Testable — override with `app.dependency_overrides[get_neo4j_client] = lambda: mock_client`

**Test override pattern:**
```python
def test_endpoint(client: TestClient):
    mock_neo4j = MagicMock(spec=Neo4jClient)
    app.dependency_overrides[get_neo4j_client] = lambda: mock_neo4j
    response = client.get("/api/search/results?q=test")
    app.dependency_overrides.clear()
```

---

## New Schema Properties to Expose (API-04)

Phase 1 added these properties that are not yet exposed in any API:

### `Speech.speakingRole`
Stored as string enum: `"president"`, `"deputy"`, `"government"`, or `None`.
Available in any query that returns `Speech` nodes. Should be included in:
- `neo4j_client.vector_search()` RETURN clause
- `graph_channel._get_chunks_from_signatories()` RETURN clause
- Any response where speech metadata is returned

### `Phase.phaseType`
Stored as string enum: `"government_opinion"`, `"vote_declaration"`, `"general_discussion"`, `None`.
Available in any query that traverses `Phase` nodes.

### `Vote` nodes
**New endpoint:** `GET /api/sessions/{session_id}/votes`
Schema (from `db_builder._create_votes`):
```
Vote {
  id: str,
  number: int,
  type: str,
  subject: str,
  present: int,
  voters: int,
  abstained: int,
  majority: int,
  inFavor: int,
  against: int,
  onMission: int,
  outcome: str
}
Session -[:HAS_VOTE]-> Vote
```

### `DISCUSSES` relationships
**New endpoint:** `GET /api/debates/{debate_id}/acts`
Traversal: `Debate -[:DISCUSSES]-> ParliamentaryAct`
Note: Placeholder acts from XML may have `isPlaceholder: true` — include this in response.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Singleton service instances | Manual global + init flag | `@lru_cache()` on Depends functions | Thread-safe, testable, standard FastAPI |
| Async Neo4j queries | Custom async wrapper | `run_in_executor` (already in use) | Neo4j Python driver is sync; this is the correct pattern |
| SSE streaming | Custom chunked response | `StreamingResponse` + async generator (already in use) | FastAPI's built-in handles headers, cleanup |
| Test mocking | Manual monkey-patching | `app.dependency_overrides` | FastAPI-native, cleaner test isolation |
| Authority scoring parallelism | Custom thread management | `ThreadPoolExecutor` + `asyncio.gather` (already in use) | Correct pattern, keep it |

---

## Common Pitfalls

### Pitfall 1: Removing span fields breaks evidence.py's compute_quote_text
**What goes wrong:** `evidence.py` currently uses `span_start`/`span_end` offsets to call `compute_quote_text(text, span_start, span_end)`. If the fields are removed from Neo4j queries, this function receives 0/0 and falls back to `chunk_text`.
**Why it happens:** The offsets no longer exist in the schema (Phase 1 removed them).
**How to avoid:** After removing the fields, `compute_quote_text` must be updated to treat `span_start=0, span_end=0` as "no offsets, use chunk_text directly". The fallback path already exists (line 136: `quote_text = data.get("chunk_text", "")[:200]`). Remove the 200 char limit — use full chunk_text as quote_text.
**Warning signs:** `citation_verified: false` for all evidence after migration.

### Pitfall 2: evaluation.py has historical bug fixes baked into metric logic
**What goes wrong:** evaluation.py has 400+ lines of metric computation with corrections documented in MEMORY.md (authority_by_group inconsistency fix, baseline_authority inflation fix, expert panel non-cited parties fix). Extracting to `evaluation_service.py` must preserve all these fixes exactly.
**Why it happens:** The fixes are subtle (e.g., using `party_top_expert` instead of `expert_full_lookup` as fallback) and easy to lose during refactoring.
**How to avoid:** Extract function bodies verbatim first. Add unit tests that encode the known correct behavior BEFORE refactoring logic. Only after tests pass, clean up code style.
**Warning signs:** Evaluation dashboard shows authority_by_group scores inconsistent with A/B survey panel.

### Pitfall 3: `@lru_cache` on Depends functions won't work in tests without clearing
**What goes wrong:** `@lru_cache()` on `get_neo4j_client` caches the real client. `app.dependency_overrides` is ignored because `lru_cache` returns the cached real instance.
**Why it happens:** `@lru_cache` runs at import time; `dependency_overrides` runs at request time. They operate at different scopes.
**How to avoid:** For tests, clear the cache explicitly: `get_neo4j_client.cache_clear()`. Or use `@lru_cache` only on the underlying factory, and wrap with a Depends function that is NOT cached (the Depends wrapper itself gets overridden).
**Warning signs:** Mocked client is not being used in tests; real DB calls happening in unit tests.

### Pitfall 4: SSE chunk size mismatch between chat.py and query.py
**What goes wrong:** `chat.py` streams `chunk_size=50` chars with `content:` key; `query.py` streams `chunk_size=100` chars with `data:` key. These are different. If the frontend has code to handle both, changing either breaks it.
**Why it happens:** The two pipelines were developed independently.
**How to avoid:** Leave both as-is. Do not unify the chunk event payload key. Document the difference in the SSE contract.

### Pitfall 5: `search.py` uses `session.run()` directly (not `client.query()`)
**What goes wrong:** Several queries in `search.py` call `with client.session() as session: result = session.run(...)` directly, bypassing `Neo4jClient.query()`. After migrating to shared client, these will work but are inconsistent.
**Why it happens:** `search.py` was written with a direct driver style.
**How to avoid:** Wrap the raw `session.run()` calls in `Neo4jClient.query()` calls during the SVC-05 fix. Consistent access through the client simplifies future changes.

### Pitfall 6: `lru_cache` makes services singletons — lifecycle on app shutdown
**What goes wrong:** `@lru_cache` services are never closed. `Neo4jClient.close()` is never called.
**Why it happens:** `lru_cache` has no lifecycle hooks.
**How to avoid:** Register a FastAPI `lifespan` event handler that calls `get_neo4j_client().close()` on shutdown. The existing `main.py` likely already has startup/shutdown hooks.

---

## Backend File Inventory (for task sizing)

| File | Lines | What It Does | Phase 2 Work |
|------|-------|-------------|-------------|
| `routers/chat.py` | 1,792 | Primary pipeline: SSE streaming, experts, citations, generation orchestration | Extract ~800 lines to services; fix Cypher (3 sites) |
| `routers/query.py` | 1,061 | Secondary pipeline: SSE streaming (used by /api/query endpoint) | Extract to services; fix Cypher (3 sites) |
| `services/generation/pipeline.py` | 1,053 | 4-stage generation pipeline | SVC-06 cleanup only |
| `routers/evaluation.py` | 959 | Evaluation dashboard metrics, automated scoring | Extract to evaluation_service.py; fix import violation |
| `services/generation/sectional.py` | 899 | Sectional response generation | SVC-06 cleanup only |
| `routers/survey.py` | 795 | A/B survey collection and stats | Move `_load_surveys`, `_calculate_stats` to service |
| `scripts/compute_baseline_experts.py` | 777 | Compute baseline_experts for evaluation_set.json | Fix SCR-03 (own driver); audit for dead Cypher |
| `services/authority/components.py` | 745 | Authority score components ([0,1] normalized) | SVC-06 cleanup only |
| `routers/search.py` | 735 | Text + semantic search across speeches and acts | Fix SVC-05 (own client); minor Cypher cleanup |
| `services/authority/scorer.py` | 705 | Authority score orchestration | SVC-06 cleanup only |
| `services/citation/sentence_extractor.py` | 685 | Sentence boundary extraction for citations | SVC-06 cleanup only |
| `services/compass/pipeline.py` | 683 | Ideological compass 2D positioning | SVC-06 cleanup only |
| `services/generation/surgeon.py` | 654 | Citation surgeon: quote extraction | SVC-06 cleanup only |
| `routers/history.py` | 597 | Chat history retrieval | Minor cleanup |
| `services/generation/integrator.py` | 551 | Generation integrator | SVC-06 cleanup only |
| `services/retrieval/engine.py` | 549 | Retrieval orchestration (dense + graph) | SVC-06 cleanup only |
| `services/compass/scorer.py` | 483 | Compass scoring | SVC-06 cleanup only |
| `services/generation/coherence_validator.py` | 476 | Coherence validation | SVC-06 cleanup only |
| `services/retrieval/graph_channel.py` | 407 | Graph retrieval channel | Fix Cypher (3 sites); remove span comments |
| `routers/config.py` | 405 | Config read/write endpoints | Minor cleanup |
| `scripts/enrich_evaluation_set.py` | 376 | Enrich evaluation_set.json with metrics | SCR-01, SCR-03 |
| `services/neo4j_client.py` | 358 | Neo4j client with connection pooling | Fix Cypher (2 sites) |
| `models/survey.py` | 322 | Survey models | No change |
| `services/compass/reference_axes.py` | 321 | Compass reference axes | SVC-06 only |
| `app/config.py` | 320 | App configuration | No change |
| `scripts/seed_evaluation_topic.py` | 159 | Seed evaluation_set.json | Fix SCR-02 (router import) |
| `services/deps.py` | 46 | DI (current: get_services dict) | Full rewrite for SVC-04 |
| `models/evidence.py` | 272 | UnifiedEvidence model | Remove span_start/span_end fields |

**Highest-risk files (most logic to preserve):**
1. `routers/evaluation.py` — historical bug fixes; test first, refactor second
2. `routers/chat.py` — 1,792 lines; extract service layer carefully
3. `scripts/compute_baseline_experts.py` — baseline data is permanent; validate output before/after

---

## Code Examples

### Correct New DI Pattern
```python
# backend/app/services/deps.py
from functools import lru_cache
from .neo4j_client import Neo4jClient
from .retrieval import RetrievalEngine
from .authority import AuthorityScorer
from .compass import IdeologyScorer
from .generation import GenerationPipeline
from ..config import get_settings

@lru_cache()
def get_neo4j_client() -> Neo4jClient:
    settings = get_settings()
    return Neo4jClient(
        uri=settings.neo4j_uri,
        user=settings.neo4j_user,
        password=settings.neo4j_password,
    )

@lru_cache()
def get_retrieval_engine() -> RetrievalEngine:
    return RetrievalEngine(get_neo4j_client())

@lru_cache()
def get_authority_scorer() -> AuthorityScorer:
    return AuthorityScorer(get_neo4j_client())

@lru_cache()
def get_ideology_scorer() -> IdeologyScorer:
    return IdeologyScorer(get_neo4j_client())

@lru_cache()
def get_generation_pipeline() -> GenerationPipeline:
    return GenerationPipeline()
```

### Corrected vector_search Cypher (after removing dead properties)
```python
# backend/app/services/neo4j_client.py — vector_search() method
cypher = """
CALL db.index.vector.queryNodes($index_name, $top_k, $query_embedding)
YIELD node AS c, score
MATCH (c)<-[:HAS_CHUNK]-(i:Speech)-[:SPOKEN_BY]->(speaker)
MATCH (i)<-[:CONTAINS_SPEECH]-(f:Phase)<-[:HAS_PHASE]-(d:Debate)<-[:HAS_DEBATE]-(s:Session)
RETURN c.id AS chunk_id,
       c.text AS chunk_text,
       c.index AS chunk_index,
       i.id AS speech_id,
       i.text AS text,
       i.speakingRole AS speaking_role,
       speaker.id AS speaker_id,
       speaker.first_name AS speaker_first_name,
       speaker.last_name AS speaker_last_name,
       labels(speaker)[0] AS speaker_type,
       s.id AS session_id,
       s.date AS session_date,
       s.number AS session_number,
       d.title AS debate_title,
       f.phaseType AS phase_type,
       score
ORDER BY score DESC
"""
```

### Vote endpoint (new, GET /api/sessions/{session_id}/votes)
```python
@router.get("/sessions/{session_id}/votes")
async def get_session_votes(
    session_id: str,
    neo4j: Neo4jClient = Depends(get_neo4j_client),
) -> List[Dict[str, Any]]:
    cypher = """
    MATCH (s:Session {id: $session_id})-[:HAS_VOTE]->(v:Vote)
    RETURN v.id AS id,
           v.number AS number,
           v.type AS type,
           v.subject AS subject,
           v.inFavor AS in_favor,
           v.against AS against,
           v.abstained AS abstained,
           v.outcome AS outcome,
           v.present AS present,
           v.voters AS voters
    ORDER BY v.number
    """
    return neo4j.query(cypher, {"session_id": session_id})
```

### pytest conftest.py skeleton
```python
# backend/tests/conftest.py
import pytest
from unittest.mock import MagicMock, patch
from fastapi.testclient import TestClient
from app.main import app
from app.services.deps import get_neo4j_client

@pytest.fixture
def mock_neo4j():
    mock = MagicMock()
    mock.query.return_value = []
    mock.query_single.return_value = None
    mock.vector_search.return_value = []
    return mock

@pytest.fixture
def client(mock_neo4j):
    app.dependency_overrides[get_neo4j_client] = lambda: mock_neo4j
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

@pytest.fixture
def sample_evidence_list():
    """Minimal UnifiedEvidence list for unit tests."""
    from app.models.evidence import UnifiedEvidence
    # ... build minimal fixtures
    return [...]
```

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest (already in venv) |
| Config file | `backend/tests/pytest.ini` — Wave 0 gap |
| Quick run command | `cd backend && python -m pytest tests/unit/ -x -q` |
| Full suite command | `cd backend && python -m pytest tests/ -q` |
| Integration tests | `cd backend && python -m pytest tests/ -m integration -q` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SVC-01 | Cypher queries don't reference start_char_raw/end_char_raw | unit | `pytest tests/unit/test_cypher_queries.py -x` | ❌ Wave 0 |
| SVC-02 | Unified experts service returns correct expert per party | unit | `pytest tests/unit/test_experts.py -x` | ❌ Wave 0 |
| SVC-03 | Evaluation metrics match known expected values | unit | `pytest tests/unit/test_evaluation_service.py -x` | ❌ Wave 0 |
| SVC-04 | Depends() functions return singletons | unit | `pytest tests/unit/test_deps.py -x` | ❌ Wave 0 |
| SVC-05 | search.py uses shared client (no duplicate pool) | unit | `pytest tests/unit/test_search.py -x` | ❌ Wave 0 |
| API-01 | Router functions contain no business logic | manual review | — | manual |
| API-02 | No cross-router imports exist | unit (import scan) | `pytest tests/unit/test_import_violations.py -x` | ❌ Wave 0 |
| API-03 | SSE events emitted in correct order with correct shapes | unit | `pytest tests/unit/test_sse_contract.py -x` | ❌ Wave 0 |
| API-05 | API response shapes match frontend contract | unit | `pytest tests/unit/test_response_shapes.py -x` | ❌ Wave 0 |
| SCR-02 | seed_evaluation_topic.py doesn't import from routers | unit (import scan) | `pytest tests/unit/test_import_violations.py -x` | ❌ Wave 0 |
| QA-01 | ~100+ tests passing | full suite | `pytest tests/ -q` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `cd backend && python -m pytest tests/unit/ -x -q`
- **Per wave merge:** `cd backend && python -m pytest tests/ -q`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `backend/tests/` directory — does not exist
- [ ] `backend/tests/conftest.py` — shared fixtures with mock Neo4j
- [ ] `backend/tests/unit/` — unit test directory
- [ ] `backend/tests/integration/` — integration test directory (skipped by default)
- [ ] `backend/tests/pytest.ini` — with `markers = integration: marks tests needing live DB`
- [ ] Framework check: `pip show pytest pytest-asyncio httpx` — verify installed

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `get_services() -> dict` | `Depends(get_foo) -> TypedClass` | Phase 2 | Type safety, testability |
| Manual global singleton | `@lru_cache` | Phase 2 | Thread-safe, standard |
| `start_char_raw` / `end_char_raw` offsets | Remove entirely; `chunk_text` is the quote source | Phase 1 schema | Simpler citation model |
| `preprocessed_text` (Speech) | `text` only | Phase 1 schema | Single text field |
| `complete_date` (Session) | `date` (Neo4j Date type) | Phase 1 schema | Native date comparisons |
| Business logic in routers | Services layer | Phase 2 | Testable, reusable |

**Deprecated in this phase:**
- `get_services()` dict in `deps.py`: replaced by typed `Depends()` functions
- `span_start`/`span_end` fields in `UnifiedEvidence`, `CitationInfo`, `EvidenceResponse` Pydantic models
- Local `_neo4j_client` globals in `search.py` and `evidence.py`
- Private router functions used by other routers (`_load_surveys`, `_compute_experts_for_frontend`)

---

## Open Questions

1. **`compute_quote_text` behavior after span removal**
   - What we know: `evidence.py` calls `compute_quote_text(text, 0, 0)` after migration; current fallback gives `chunk_text[:200]`
   - What's unclear: Should the 200-char limit be removed? Is `chunk_text` always the full intended citation or a preview?
   - Recommendation: Remove the 200-char limit. `chunk_text` is the canonical chunk content; there is no reason to truncate it for the citation.

2. **`enrich_evaluation_set.py` dead property audit**
   - What we know: 376 lines, touches evaluation_set.json baseline metrics
   - What's unclear: Whether it queries `start_char_raw`/`end_char_raw` directly
   - Recommendation: Read lines 80-376 during Wave 1 Cypher migration task before writing any code.

3. **`process_chat_background` vs `process_chat_streaming` in chat.py**
   - What we know: Both functions exist in chat.py (1,792 lines), both do the full pipeline. The background version stores to TaskStore; the streaming version yields directly.
   - What's unclear: Can they be unified into one implementation that both paths call?
   - Recommendation: Extract the pipeline body into a service that accepts an emit callback. Both functions then call the service with different emit implementations. This is the cleanest extraction but adds complexity — Claude's discretion.

---

## Sources

### Primary (HIGH confidence)
- Direct code reading: `backend/app/routers/chat.py` (1,792 lines, read lines 1-200, 150-350, 350-550, 550-750, 750-950, 950-1110, 1149-1460)
- Direct code reading: `backend/app/routers/query.py` (1,061 lines, read lines 1-350, 350-520)
- Direct code reading: `backend/app/services/neo4j_client.py` (complete, 358 lines)
- Direct code reading: `backend/app/services/retrieval/graph_channel.py` (complete, 407 lines)
- Direct code reading: `backend/app/routers/evaluation.py` (read lines 1-200)
- Direct code reading: `backend/app/routers/search.py` (complete, 735 lines)
- Direct code reading: `backend/app/routers/evidence.py` (complete, 226 lines)
- Direct code reading: `backend/app/services/deps.py` (complete, 46 lines)
- Direct code reading: `backend/app/services/authority/scorer.py` (read lines 1-300)
- Direct code reading: `build/db_builder.py` (read lines 1-650)
- Direct code reading: `backend/app/models/evidence.py` (read lines 1-100)
- Direct code reading: `backend/scripts/seed_evaluation_topic.py` (read lines 1-60)
- Direct code reading: `backend/scripts/compute_baseline_experts.py` (read lines 1-80)
- Grep: All `start_char_raw`/`end_char_raw` occurrences across blast radius files
- File inventory: line counts for all backend Python files

### Secondary (MEDIUM confidence)
- `.planning/phases/02-backend/02-CONTEXT.md` — locked decisions from discussion session
- `.planning/REQUIREMENTS.md` — requirement IDs and descriptions
- `MEMORY.md` — historical bug fixes and their fixes (evaluation consistency bugs)

---

## Metadata

**Confidence breakdown:**
- Dead property audit (SVC-01): HIGH — direct grep + code reading
- SSE contract (API-03): HIGH — read all emit/yield sites in both pipelines
- Expert computation diff (SVC-02): HIGH — read both implementations
- Cross-router coupling (API-02): HIGH — confirmed by direct code reading
- DI pattern (SVC-04): HIGH — full deps.py read + FastAPI Depends pattern is well-documented standard
- Test architecture (QA-01): HIGH — no existing tests confirmed; framework choices are standard

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable codebase; schema is frozen after Phase 1)
