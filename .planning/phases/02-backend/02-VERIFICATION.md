---
phase: 02-backend
verified: 2026-04-02T12:00:00Z
status: gaps_found
score: 14/17 must-haves verified
re_verification: false
gaps:
  - truth: "No Cypher query in the codebase references start_char_raw or end_char_raw"
    status: failed
    reason: "Two retrieval modules were not in Plan 01's file list and were never cleaned: dense_channel.py (lines 83-84, 132) and engine.py (lines 326, 328, 500-501)"
    artifacts:
      - path: "backend/app/services/retrieval/dense_channel.py"
        issue: "Lines 83-84 return c.start_char_raw AS span_start, c.end_char_raw AS span_end; line 132 references them in a comment"
      - path: "backend/app/services/retrieval/engine.py"
        issue: "Lines 326/328 MATCH prev/next chunk windows with start_char_raw/end_char_raw; lines 500-501 return them"
    missing:
      - "Remove c.start_char_raw AS span_start and c.end_char_raw AS span_end from dense_channel.py vector_search Cypher"
      - "Remove the dead-property comment on line 132 of dense_channel.py"
      - "Remove start_char_raw/end_char_raw from the context-window MATCH query in engine.py (lines 326/328)"
      - "Remove the second span_start/span_end return site in engine.py (lines 500-501)"

  - truth: "All test_deps.py tests pass (SVC-04 and SVC-05 verified by automated tests)"
    status: partial
    reason: "4 tests FAIL and 3 ERROR due to NumPy 2.4.3 / scipy binary incompatibility when importing app.routers inside test_deps.py. The underlying implementation is correct (zero _neo4j_client globals in search.py and evidence.py), but the tests themselves cannot pass in the current environment."
    artifacts:
      - path: "backend/tests/unit/test_deps.py"
        issue: "TestNoLocalClientInRouters tests (4 FAIL) cannot import app.routers.search/evidence because app.routers.__init__ triggers scipy via query.py, which fails with NumPy 1.x vs 2.4.3 ABI mismatch. TestGetNeo4jClientSingleton / TestGetRetrievalEngineSingleton / TestGetServicesBackwardCompat (3 ERROR) fail for the same setup-time import reason."
    missing:
      - "Fix test_deps.py to import search.py / evidence.py directly (not via app.routers package) so scipy is not pulled in: use importlib.util.spec_from_file_location or inspect individual source files with pathlib instead of module imports"
      - "Alternatively: isolate app.routers.__init__ from auto-importing query_router until scipy/NumPy ABI is resolved"
human_verification: []
---

# Phase 2: Backend Verification Report

**Phase Goal:** All backend layers (services, routers, scripts) are clean, correctly typed, and deploy atomically with the new schema — all Cypher consumers updated, business logic extracted into services, dependency injection fixed, cross-layer coupling violations resolved
**Verified:** 2026-04-02T12:00:00Z
**Status:** gaps_found
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|---------|
| 1 | No Cypher query in the codebase references start_char_raw or end_char_raw | FAILED | dense_channel.py lines 83-84, 132; engine.py lines 326, 328, 500-501 still contain dead properties |
| 2 | No Pydantic model contains span_start or span_end fields | VERIFIED | grep on models/ and routers/query.py returns zero matches |
| 3 | SSE contract document lists all event types with payload shapes | VERIFIED | backend/docs/SSE_CONTRACT.md exists, marked FROZEN, contains both pipeline event tables |
| 4 | Test infrastructure exists and pytest collects tests | VERIFIED | conftest.py with mock_neo4j fixture; 136 tests collected and passing (excluding test_deps) |
| 5 | All service instances obtained via typed Depends() functions | VERIFIED | deps.py has 5 @lru_cache functions; get_services() backward-compat wrapper present |
| 6 | search.py and evidence.py use shared Neo4j client from deps.py | VERIFIED | Both files import get_neo4j_client from deps.py; zero module-level _neo4j_client globals (grep confirms 0 matches on ^_neo4j_client pattern) |
| 7 | Expert computation exists in exactly one place: services/experts.py | VERIFIED | experts.py exists (200+ lines), contains async def compute_experts with 0.70*auth + 0.30*sim formula and party_changed handling; zero _compute_experts_for_frontend or _compute_experts definitions in routers |
| 8 | chat.py and query.py both delegate expert computation to the service | VERIFIED | Both routers import from ..services.experts; no local expert functions remain |
| 9 | seed_evaluation_topic.py imports from services, not from routers | VERIFIED | imports get_services from app.services.deps; zero "from app.routers" in seed script |
| 10 | evaluation.py does not import from survey.py | VERIFIED | survey_helpers.py extracted; evaluation.py imports from app.services.evaluation_service; zero "from app.routers.survey" in evaluation.py |
| 11 | Evaluation business logic lives in services/evaluation_service.py | VERIFIED | evaluation_service.py is 517 lines; contains _compute_automated_metrics, _compute_baseline_authority_from_precomputed, party_top_expert (all 3 historical bug fixes present) |
| 12 | evaluation.py router is a thin wrapper (< 600 lines) | VERIFIED | evaluation.py is 509 lines (down from 959); imports from app.services.evaluation_service; no def _compute_ in router |
| 13 | New GET endpoints for session votes and debate acts exist | VERIFIED | data.py contains get_session_votes and get_debate_acts; registered in main.py via data_router |
| 14 | speakingRole and phaseType included in vector_search responses | VERIFIED | neo4j_client.py lines 162-163 return i.speakingRole AS speaking_role, f.phaseType AS phase_type |
| 15 | Scripts use shared Neo4j client from deps.py | VERIFIED | All 3 scripts import get_neo4j_client from app.services.deps; zero GraphDatabase.driver in scripts |
| 16 | All service modules have English docstrings and type hints | VERIFIED | experts.py and evaluation_service.py have module-level docstrings; no Italian comments found in app/ |
| 17 | Comprehensive test suite with 100+ tests | VERIFIED | 136 tests pass (excluding 7 test_deps tests that fail due to environment incompatibility unrelated to the refactoring) |

**Score: 14/17 truths verified** (2 truths failed, 1 partially failed due to environment)

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `backend/docs/SSE_CONTRACT.md` | Frozen SSE event contract | VERIFIED | FROZEN header, both pipeline tables, dual-experts note |
| `backend/tests/conftest.py` | Shared fixtures with mock_neo4j | VERIFIED | mock_neo4j fixture present on line 15 |
| `backend/tests/unit/test_cypher_queries.py` | Dead-property tests | VERIFIED | Contains test_neo4j_client_vector_search_no_dead_properties |
| `backend/app/services/deps.py` | Typed DI with @lru_cache | VERIFIED | 5 @lru_cache functions + backward-compat get_services() |
| `backend/tests/unit/test_deps.py` | DI singleton tests | PARTIAL | File exists; 7 tests fail/error due to NumPy/scipy ABI mismatch in environment |
| `backend/app/services/experts.py` | Unified expert computation | VERIFIED | 200+ lines, combined formula (0.70/0.30), party_changed, patch function |
| `backend/tests/unit/test_experts.py` | Expert tests | VERIFIED | 10 test functions |
| `backend/tests/unit/test_import_violations.py` | Cross-layer import tests | VERIFIED | test_no_router_imports_from_router present |
| `backend/app/services/evaluation_service.py` | Evaluation metrics service | VERIFIED | 517 lines, all 3 historical bug fixes present |
| `backend/tests/unit/test_evaluation_service.py` | Evaluation service tests | VERIFIED | 9 test functions |
| `backend/app/routers/data.py` | New vote/act endpoints | VERIFIED | get_session_votes and get_debate_acts defined, uses Depends(get_neo4j_client) |
| `backend/tests/unit/test_new_endpoints.py` | New endpoint tests | VERIFIED | File exists |
| `backend/tests/unit/test_scripts.py` | Script DI tests | VERIFIED | File exists |
| `backend/tests/unit/test_sse_contract.py` | SSE contract tests | VERIFIED | 12 test functions |
| `backend/tests/unit/test_response_shapes.py` | Response shape tests | VERIFIED | 31 test functions |
| `backend/tests/unit/test_routers.py` | Router thinness tests | VERIFIED | 37 test functions |
| `backend/app/services/survey_helpers.py` | Extracted survey helpers | VERIFIED | File exists (resolves evaluation.py cross-router import) |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `backend/app/services/neo4j_client.py` | Neo4j database | Cypher without dead properties (vector_search) | VERIFIED | speakingRole/phaseType added; start_char_raw/end_char_raw removed from this file |
| `backend/app/services/retrieval/dense_channel.py` | Neo4j database | Cypher without dead properties | NOT WIRED | Lines 83-84 still return start_char_raw/end_char_raw |
| `backend/app/services/retrieval/engine.py` | Neo4j database | Cypher without dead properties | NOT WIRED | Lines 326, 328, 500-501 still reference start_char_raw/end_char_raw |
| `backend/app/services/deps.py` | `backend/app/services/neo4j_client.py` | @lru_cache get_neo4j_client() | VERIFIED | lru_cache decorator on get_neo4j_client confirmed |
| `backend/app/routers/search.py` | `backend/app/services/deps.py` | Depends(get_neo4j_client) | VERIFIED | Line 44 and 423/519/570/606/664 use get_neo4j_client |
| `backend/app/routers/chat.py` | `backend/app/services/experts.py` | import compute_experts | VERIFIED | Line 24: from ..services.experts import compute_experts, patch_experts_for_cited_speakers |
| `backend/app/routers/query.py` | `backend/app/services/experts.py` | import compute_experts | VERIFIED | Line 21: from ..services.experts import compute_experts, patch_experts_for_cited_speakers |
| `backend/scripts/seed_evaluation_topic.py` | `backend/app/services/deps.py` | import get_services | VERIFIED | Line 33: from app.services.deps import get_services |
| `backend/app/routers/evaluation.py` | `backend/app/services/evaluation_service.py` | import from service | VERIFIED | Line 30: from app.services.evaluation_service import (...) |
| `backend/app/routers/data.py` | `backend/app/services/deps.py` | Depends(get_neo4j_client) | VERIFIED | Endpoints use Depends(get_neo4j_client) |
| `backend/tests/unit/test_sse_contract.py` | `backend/docs/SSE_CONTRACT.md` | Tests encode frozen contract | VERIFIED | File has 12 test functions covering event types and payload keys |

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|---------|
| SVC-01 | 02-01 | Update all Cypher queries to camelCase | PARTIAL | Target files cleaned; dense_channel.py and engine.py missed |
| SVC-02 | 02-03 | Extract expert computation into services/experts.py | SATISFIED | experts.py exists with unified implementation; both routers delegate |
| SVC-03 | 02-04 | Extract evaluation business logic into evaluation_service.py | SATISFIED | evaluation_service.py is 517 lines; evaluation.py is 509 (thin) |
| SVC-04 | 02-02 | Replace get_services() dict with FastAPI Depends() | SATISFIED | 5 typed @lru_cache functions in deps.py; backward compat preserved |
| SVC-05 | 02-02 | Fix search.py duplicate Neo4j connection pool | SATISFIED | Zero _neo4j_client globals in search.py or evidence.py |
| SVC-06 | 02-06 | Clean naming, type hints, English docstrings | SATISFIED | Service modules have English docstrings; no Italian comments in app/ |
| API-01 | 02-04, 02-06 | Thin routers around services | SATISFIED | evaluation.py 509 lines (from 959); no _compute_ functions in routers |
| API-02 | 02-03 | Fix cross-router import violations | SATISFIED | evaluation.py no longer imports from survey.py; survey_helpers.py extracted |
| API-03 | 02-01 | Freeze SSE event contract | SATISFIED | SSE_CONTRACT.md exists, FROZEN, covers both pipelines |
| API-04 | 02-05 | Clean endpoint naming, Pydantic v2, consistent error handling | SATISFIED | New data.py endpoints use proper patterns |
| API-05 | 02-05, 02-06 | Preserve API response shapes | SATISFIED | 31 response-shape tests pass; no field changes to live endpoints |
| SCR-01 | 02-05 | Refactor scripts with consistent naming/docstrings | SATISFIED | Scripts have docstrings; no dead properties |
| SCR-02 | 02-03 | Fix seed_evaluation_topic.py router import coupling | SATISFIED | Now imports from app.services.deps, not from router |
| SCR-03 | 02-05 | Scripts use shared Neo4j client | SATISFIED | All 3 scripts import get_neo4j_client from app.services.deps |
| QA-01 | 02-01, 02-06 | Add smoke tests for critical paths | SATISFIED | 136 tests across 10 test files |
| QA-02 | 02-06 | Best practices: type hints, English docstrings | SATISFIED | All service modules have module docstrings; no Italian comments |
| QA-03 | 02-06 | Consistent naming conventions | SATISFIED | snake_case Python, camelCase Neo4j Cypher properties confirmed |

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `backend/app/services/retrieval/dense_channel.py` | 83-84 | `c.start_char_raw AS span_start, c.end_char_raw AS span_end` in Cypher RETURN | BLOCKER | Queries return null for these properties against Phase 1 schema — data silently absent |
| `backend/app/services/retrieval/dense_channel.py` | 132 | Comment references `start_char_raw/end_char_raw` as if still present | WARNING | Misleading documentation |
| `backend/app/services/retrieval/engine.py` | 326, 328 | `prev.start_char_raw AS prev_start, prev.end_char_raw AS prev_end` and next variant | BLOCKER | Context-window enrichment query returns null for offset values against Phase 1 schema |
| `backend/app/services/retrieval/engine.py` | 500-501 | `c.start_char_raw AS span_start, c.end_char_raw AS span_end` in secondary query | BLOCKER | Secondary retrieval path also broken |
| `backend/tests/unit/test_deps.py` | all | NumPy/scipy ABI incompatibility prevents any import of app.routers in test context | WARNING | 7 tests in test_deps.py cannot pass in current environment; test coverage for DI singleton behavior is absent |

---

### Human Verification Required

None. All concerns are verifiable programmatically.

---

### Gaps Summary

Two root-cause gaps block the SVC-01 requirement being fully satisfied:

**Gap 1 — Two retrieval modules have dead Cypher properties (SVC-01 incomplete):**
Plan 02-01 specified an 8-file hit list for dead-property removal. `dense_channel.py` and `engine.py` were not on that list, were not cleaned, and still contain `start_char_raw`/`end_char_raw` in live Cypher queries. Against the Phase 1 Neo4j schema these properties do not exist, so queries return `null` for those fields silently. This is a data correctness issue affecting the dense retrieval path (dense_channel.py) and the context-window enrichment path (engine.py). The fix is mechanical: remove the dead RETURN columns and any downstream dict construction that consumes them.

**Gap 2 — test_deps.py tests fail due to environment incompatibility (QA-01 partial):**
The 7 failing tests in `test_deps.py` are not a code defect — the underlying implementation passes all direct grep checks (zero `_neo4j_client` module globals, proper Depends usage). The test failures stem from `app.routers.__init__.py` auto-importing `query_router`, which transitively loads `scipy` via the compass pipeline, which fails under NumPy 2.4.3. The fix is to rewrite the 4 `TestNoLocalClientInRouters` tests to use `pathlib.Path.read_text()` or `importlib.util.spec_from_file_location` to read the source directly without triggering the package-level import. The 3 ERROR tests need the fixture-level NumPy issue addressed first (or mock the settings import to avoid touching scipy entirely).

**Group note:** Both gaps are independent. Gap 1 is the higher priority because it affects runtime behavior on the live schema.

---

*Verified: 2026-04-02T12:00:00Z*
*Verifier: Claude (gsd-verifier)*
