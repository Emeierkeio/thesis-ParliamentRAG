---
phase: 04-enrichment
verified: 2026-04-03T00:00:00Z
status: gaps_found
score: 9/12 must-haves verified
re_verification: false
gaps:
  - truth: "RetrievalEngine runs dense + sparse + graph channels in parallel"
    status: failed
    reason: "engine.py uses re.findall() at line 118 without importing re — NameError at runtime on every query"
    artifacts:
      - path: "backend/app/services/retrieval/engine.py"
        issue: "Missing 'import re' at module top; re.findall used at lines 118-120"
      - path: "backend/app/services/retrieval/engine.py"
        issue: "Undefined variables 'appended' and 'early_speech' in _expand_neighbors() log line 385 — NameError whenever _expand_neighbors runs"
    missing:
      - "Add 'import re' to the import block of engine.py"
      - "Fix _expand_neighbors() log line 385: remove 'appended' and 'early_speech' references (they were never defined in the current implementation)"

  - truth: "Graph channel can filter results by entity references when query mentions specific laws or persons"
    status: failed
    reason: "GraphChannel._get_chunks_by_entity() is called at graph_channel.py line 163 but the method does not exist — AttributeError at runtime whenever entity_filter is non-empty"
    artifacts:
      - path: "backend/app/services/retrieval/graph_channel.py"
        issue: "_get_chunks_by_entity() is called but never defined; only _find_relevant_acts, _semantic_rerank, _get_chunks_from_signatories, _process_results exist"
    missing:
      - "Implement _get_chunks_by_entity(entity_filter, query_embedding, date_start, date_end) in graph_channel.py with Cypher: WHERE ANY(ref IN c.lawRefs WHERE toLower(ref) CONTAINS toLower($law_term))"

  - truth: "ENR-03 and ENR-04 marked complete in REQUIREMENTS.md"
    status: failed
    reason: "REQUIREMENTS.md still has ENR-03 and ENR-04 as unchecked ([ ]) despite Plan 04-03 implementing the NER pipeline"
    artifacts:
      - path: ".planning/REQUIREMENTS.md"
        issue: "Lines 65-66 show [ ] for ENR-03 and ENR-04 — not updated after implementation"
    missing:
      - "Update REQUIREMENTS.md: change [ ] to [x] for ENR-03 and ENR-04"
---

# Phase 4: Enrichment Verification Report

**Phase Goal:** The system retrieves more relevant results via BM25 sparse channel merged with RRF, and the graph contains per-deputy vote records, committee officer roles, and NER-extracted law/person references on chunks
**Verified:** 2026-04-03
**Status:** gaps_found — 3 gaps found, 2 are runtime-breaking bugs
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | SparseChannel.retrieve() returns evidence dicts from Neo4j full-text index | VERIFIED | sparse_channel.py: class SparseChannel exists, retrieve() returns List[Dict] with similarity=0.5, bm25_score, retrieval_channel="sparse" |
| 2 | RRF merger ranks results using rank-based fusion from 3 channels | VERIFIED | merger.py: merge(dense, sparse, graph) signature confirmed; _compute_rrf() implements sum(weight/(k+rank)); 10 RRF unit tests green |
| 3 | RetrievalEngine runs dense + sparse + graph channels in parallel | FAILED | engine.py: SparseChannel instantiated, max_workers=3, all channel calls present — but missing `import re` causes NameError at line 118 on every retrieve_sync() call |
| 4 | Full-text index with Italian analyzer is created during db-all build | VERIFIED | db_builder.py line 979: create_fulltext_index() with CREATE FULLTEXT INDEX chunk_fulltext IF NOT EXISTS + Italian analyzer fallback; build_and_update.py steps 8b and 5b call it |
| 5 | sparql_ingester.py fetches vote records and writes IndividualVote nodes to Neo4j | VERIFIED | sparql_ingester.py: ingest_votes() with MERGE (iv:IndividualVote), VOTED and ON_VOTE relationships; correct persona.rdf ID mapping |
| 6 | sparql_ingester.py fetches committee officer roles and enriches MEMBER_OF_COMMITTEE | VERIFIED | sparql_ingester.py: ingest_committee_roles() with SET r.officerRole; toLower CONTAINS fuzzy committee matching |
| 7 | make enrich-sparql runs the SPARQL ingestion pipeline | VERIFIED | Makefile lines 338-352: enrich-sparql and enrich-sparql-test targets both call sparql_ingester.py with correct args |
| 8 | Deputy ID matching correctly maps SPARQL deputato.rdf URIs to Neo4j persona.rdf IDs | VERIFIED | sparql_dep_uri_to_neo4j_id() extracts d{id}_19 -> persona.rdf/p{id}; 33 unit tests pass |
| 9 | NER extraction produces lawRefs and personRefs arrays from Italian parliamentary chunk text | VERIFIED | ner.py: extract_law_refs() (regex LAW_PATTERNS), extract_person_refs() (spaCy PER), enrich_chunks_with_ner() — all functions present and correct; 29 unit tests pass |
| 10 | Chunk nodes carry lawRefs and personRefs list properties after full rebuild | VERIFIED | db_builder.py: _create_chunks Cypher has SET c.lawRefs = row.lawRefs, SET c.personRefs = row.personRefs; enrich_chunks_with_ner called in ingest_session() before _batch_write |
| 11 | Graph channel can filter results by entity references when query mentions specific laws or persons | FAILED | graph_channel.py calls self._get_chunks_by_entity() at line 163 but method is not defined — AttributeError at runtime when entity_filter is non-empty |
| 12 | Law references are extracted via regex (not spaCy NER) | VERIFIED | ner.py docstring explicitly states it; LAW_PATTERNS list of compiled regex patterns used; no spaCy label="LAW" anywhere |

**Score:** 9/12 truths verified

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `backend/app/services/retrieval/sparse_channel.py` | BM25 retrieval channel | VERIFIED | SparseChannel class, retrieve(), _process_results(), _escape_lucene() — substantive, 231 lines |
| `backend/app/services/retrieval/merger.py` | RRF-based merger for 3 channels | VERIFIED | ChannelMerger.merge(dense, sparse, graph), _compute_rrf(), _select_diverse() — fully implemented |
| `backend/tests/unit/test_sparse_channel.py` | SparseChannel unit tests | VERIFIED | 6 tests, all passing |
| `backend/tests/unit/test_rrf_merger.py` | RRF merger unit tests | VERIFIED | 10 tests, all passing |
| `build/sparql_ingester.py` | SPARQL data ingestion from dati.camera.it | VERIFIED | SparqlIngester class, ingest_votes(), ingest_committee_roles(), URI helpers, CLI — 490 lines |
| `build/tests/test_sparql_ingester.py` | Unit tests with mocked HTTP responses | VERIFIED | 33 tests, all passing |
| `Makefile` | enrich-sparql target | VERIFIED | enrich-sparql and enrich-sparql-test targets present |
| `build/ner.py` | NER extraction functions | VERIFIED | extract_law_refs(), extract_person_refs(), enrich_chunks_with_ner(), load_ner_model() — all present |
| `build/tests/test_ner.py` | Unit tests for NER | VERIFIED | 29 tests pass (4 skipped as slow) |
| `build/db_builder.py` | Extended _create_chunks with lawRefs and personRefs | VERIFIED | lawRefs + personRefs in Cypher SET; enrich_chunks_with_ner called; create_fulltext_index present |
| `backend/app/services/retrieval/graph_channel.py` | Optional entity filter in graph retrieval | STUB | entity_filter parameter added, _get_chunks_by_entity called — but method body is missing |
| `backend/app/services/retrieval/engine.py` | 3-channel parallel orchestration | STUB | SparseChannel instantiated, max_workers=3, merger call updated — but `import re` missing and two undefined variables in _expand_neighbors |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| engine.py | sparse_channel.py | SparseChannel import + retrieve() in ThreadPoolExecutor | WIRED | Import at line 20, instantiation at line 51, run_sparse() at lines 134-137 |
| engine.py | merger.py | merger.merge(dense, sparse, graph) | WIRED | Call at lines 165-171 with all three channel results |
| db_builder.py | Neo4j full-text index | create_fulltext_index() | WIRED | Method defined, called in do_build() step 8b and do_update() step 5b |
| sparql_ingester.py | Neo4j IndividualVote nodes | MERGE (iv:IndividualVote) with VOTED and ON_VOTE relationships | WIRED | _write_votes() Cypher confirmed |
| sparql_ingester.py | Neo4j MEMBER_OF_COMMITTEE | SET r.officerRole on existing relationship | WIRED | _write_committee_roles() Cypher confirmed |
| sparql_ingester.py | dati.camera.it/sparql | HTTP GET with SPARQL SELECT queries | WIRED | _sparql_get() confirmed; SPARQL_ENDPOINT = "https://dati.camera.it/sparql" |
| build/ner.py | build/db_builder.py | enrich_chunks_with_ner() called before _batch_write chunks | WIRED | db_builder.py line 27: import; line 336: call in ingest_session() |
| build/db_builder.py | Neo4j Chunk nodes | SET c.lawRefs = row.lawRefs, c.personRefs = row.personRefs | WIRED | _create_chunks Cypher lines 447-448 |
| graph_channel.py | Chunk.lawRefs / Chunk.personRefs | WHERE ANY(ref IN c.lawRefs WHERE ref CONTAINS $law_term) | NOT_WIRED | _get_chunks_by_entity() called but never defined — AttributeError at runtime |
| engine.py | graph_channel.py entity_filter | entity_filter passed to run_graph() | PARTIAL | entity_filter dict built via re.findall (FAILS: no import re) then passed correctly |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|---------|
| RET-01 | 04-01 | BM25 sparse retrieval channel via Neo4j full-text index | SATISFIED | sparse_channel.py substantive, create_fulltext_index in db_builder.py, 6 unit tests green |
| RET-02 | 04-01 | RRF merger for hybrid dense+sparse+graph retrieval | SATISFIED | merger.py implements RRF, 3-channel signature, 10 unit tests green |
| ENR-01 | 04-02 | SPARQL ingestion — per-deputy individual vote records | SATISFIED | ingest_votes() with IndividualVote MERGE, 33 tests green |
| ENR-02 | 04-02 | SPARQL ingestion — committee officer roles with dates | SATISFIED | ingest_committee_roles() with officerRole SET, 33 tests green |
| ENR-03 | 04-03 | NER at ingestion time on chunks — LAW and PERSON entity references | SATISFIED (code) / UNCHECKED (tracker) | ner.py + db_builder.py integration complete — but REQUIREMENTS.md still shows [ ] |
| ENR-04 | 04-03 | Store NER results as Chunk properties (lawRefs, personRefs) for entity-filtered retrieval | PARTIAL | lawRefs/personRefs stored in Chunk nodes (build side complete); entity-filtered retrieval broken (_get_chunks_by_entity missing) |

**Note:** REQUIREMENTS.md was not updated after Plan 04-03 executed — ENR-03 and ENR-04 remain unchecked despite code implementation.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `backend/app/services/retrieval/engine.py` | 118 | `re.findall(...)` with no `import re` | BLOCKER | NameError on every retrieve_sync() call — entire retrieval pipeline crashes |
| `backend/app/services/retrieval/engine.py` | 385 | `appended` and `early_speech` undefined variables in log string | BLOCKER | NameError in _expand_neighbors() whenever the low-salience path executes (candidates list non-empty) |
| `backend/app/services/retrieval/graph_channel.py` | 163 | `self._get_chunks_by_entity()` called but method not defined | BLOCKER | AttributeError when any query contains a law reference pattern (e.g. "decreto 231") |
| `.planning/REQUIREMENTS.md` | 65-66 | ENR-03 and ENR-04 marked unchecked despite implementation | WARNING | Requirements tracker inconsistency — not a code bug |

---

## Human Verification Required

None identified — all gaps are programmatically verifiable.

---

## Gaps Summary

**3 gaps block goal achievement:**

**Gap 1 — Critical Runtime Bug: Missing `import re` in engine.py (blocks RET-01, RET-02 in practice)**

The entity-filter detection code in `retrieve_sync()` uses `re.findall()` at line 118 without importing the `re` module. Since this code runs unconditionally on every query (not in a try/except), every call to `retrieve_sync()` will raise `NameError: name 're' is not defined`. The 3-channel pipeline cannot execute. Additionally, the `_expand_neighbors()` method has an undefined variable bug in its log string (line 385: `appended` and `early_speech` are referenced but never assigned in the current implementation), which will also crash the method.

Fix: Add `import re` to engine.py imports; fix line 385 to remove the undefined variable references.

**Gap 2 — Missing Method: `_get_chunks_by_entity` in graph_channel.py (blocks ENR-04)**

`GraphChannel.retrieve()` calls `self._get_chunks_by_entity(entity_filter, ...)` at line 163 when `entity_filter` is non-empty. This method is not defined anywhere in the class — only `_find_relevant_acts`, `_semantic_rerank`, `_get_chunks_from_signatories`, and `_process_results` exist. Any query that triggers the law-pattern detection in engine.py (after fixing Gap 1) will then hit an `AttributeError`. ENR-04's entity-filtered retrieval feature is wired from engine.py but terminates with a missing method body.

Fix: Implement `_get_chunks_by_entity()` with the Cypher filter on `c.lawRefs` / `c.personRefs` as specified in Plan 04-03.

**Gap 3 — Requirements Tracker Not Updated (ENR-03, ENR-04)**

The NER pipeline code is implemented and the unit tests pass, but `REQUIREMENTS.md` still shows `[ ]` for ENR-03 and ENR-04. This is a documentation gap, not a code gap. ENR-04 code is also partially broken (see Gap 2).

Fix: Update REQUIREMENTS.md to mark ENR-03 as `[x]` and ENR-04 as `[x]` once Gap 2 is resolved.

---

**Root cause grouping:** Gaps 1 and 2 share a common root — Plan 04-03 Task 2 was noted as "partially completed by agent before rate limit; engine.py wiring and commit done by orchestrator." The wiring of the entity_filter in engine.py and graph_channel.py was added without the supporting `import re` and without implementing the `_get_chunks_by_entity` method body. These two files need targeted fixes.

---

_Verified: 2026-04-03_
_Verifier: Claude (gsd-verifier)_
