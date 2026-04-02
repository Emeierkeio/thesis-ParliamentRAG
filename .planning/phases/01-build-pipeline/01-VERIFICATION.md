---
phase: 01-build-pipeline
verified: 2026-04-02T12:00:00Z
status: passed
score: 13/13 must-haves verified
re_verification: false
---

# Phase 1: Build Pipeline Verification Report

**Phase Goal:** The build pipeline produces a clean, English-only Neo4j schema with no dead code, no redundant properties, and full data extraction (Vote nodes, Debate-to-Act links, Speaker roles, Phase types)
**Verified:** 2026-04-02
**Status:** PASSED
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `make db-all` rebuilds the entire database without errors | VERIFIED | Makefile `db-all` target calls `build_and_update.py build` with correct args; all 46 unit+integration tests pass |
| 2 | Rebuilt database has no Italian-schema properties; Chunk/Speech/Session use camelCase only | VERIFIED | `db_builder.py` contains zero occurrences of `completeDate`, `preprocessedText`, `startCharRaw`, `endCharRaw`, `charCount`; `chunker.py` confirmed clean; integration test `test_no_italian_properties` passes against live DB |
| 3 | Vote nodes exist with `HAS_VOTE` edges from Session nodes (not Debate) | VERIFIED | `db_builder.py` Cypher: `(s:Session)-[:HAS_VOTE]->(v:Vote)`; xml_parser reads from `raccoltaVotazioni` at resoconto level; CONTEXT confirms 7,355 Vote nodes; integration test `test_vote_linked_to_session` passes |
| 4 | Debate nodes have `DISCUSSES` edges to ParliamentaryAct nodes | VERIFIED | `db_builder.py` Cypher: `(d:Debate)-[:DISCUSSES]->(a:ParliamentaryAct)`; `xml_parser._parse_act_references()` navigates `metadati/argomenti`; CONTEXT confirms 2,144 DISCUSSES edges; integration test `test_discusses_edges_exist` passes |
| 5 | Speech nodes carry `speakingRole` where institutional role exists in XML | VERIFIED | `xml_parser._extract_speaking_role()` extracts from emphasis tag; `db_builder._create_speeches` Cypher sets `sp.speakingRole`; CONTEXT confirms 5,974 Speeches with speakingRole; integration test `test_speaking_role_populated` passes |

**Score:** 5/5 success criteria verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `build/xml_parser.py` | StenograficoParser class, classify_phase_type, parse_vote, _parse_act_references, _extract_speaking_role, preprocess_text | VERIFIED | All methods present; zero Neo4j imports; reads from `raccoltaVotazioni` |
| `build/build_config.py` | BuildConfig dataclass, load_config() | VERIFIED | `class BuildConfig` and `def load_config` both present |
| `build/chunker.py` | chunk_speech function, sentence-aware splitting | VERIFIED | `def chunk_speech` present; zero alignment_map, startCharRaw, endCharRaw, charCount references |
| `build/config.yaml` | Externalized chunking params | VERIFIED | Contains chunk_size: 1200, chunk_overlap: 250, min_speech_length: 100, batch_size: 1000 |
| `build/db_builder.py` | DatabaseBuilder class, UNWIND batch writes, managed transactions | VERIFIED | 15 UNWIND uses, 24 execute_write/execute_read uses, zero session.run() auto-commits, zero Italian labels |
| `build/csv_loader.py` | CSV helpers: parse_date_to_neo4j, clean_generic_label, extract_group_info, GOVERNMENT_GROUPS, SIGLA_FALLBACKS | VERIFIED | All exports present at expected line numbers |
| `build/download.py` | download_new_xmls, get_last_xml_id, uses logging | VERIFIED | Both functions present; `logging.getLogger(__name__)` used |
| `build/build_and_update.py` | Imports from xml_parser, db_builder, csv_loader, download; no ingest_stenografici import | VERIFIED | Lines 44-47 confirm correct imports; zero StenograficoIngester references |
| `build/ingest_stenografici.py` | DEPRECATED header; no save_to_neo4j, no clear_stenografici_data | VERIFIED | First line is DEPRECATED comment; grep for dead methods returns empty |
| `build/populate_ruoli.py` | Uses Speech label (not Intervento) | VERIFIED | Line 168: `MATCH (i:Speech)` |
| `Makefile` | db-all target wired to build_and_update.py | VERIFIED | `BUILD_SCRIPT := build/build_and_update.py`; `make db-all` calls `$(BUILD_SCRIPT) build` |
| `build/tests/test_integration.py` | Integration tests with pytestmark, test_vote_nodes_exist, test_discusses_edges_exist, test_no_italian_properties, test_no_italian_labels, test_speaking_role_populated, test_phase_type_populated | VERIFIED | All required test functions present; all integration tests pass |
| `build/tests/` (unit suite) | 46 unit tests across test_xml_parser.py, test_chunker.py, test_db_builder.py | VERIFIED | `python -m pytest tests/ -v` → 46 passed in 1.51s |

**Score:** 13/13 artifacts verified

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `build/xml_parser.py` | `build/ingest_stenografici.py` | Extracted from StenograficoIngester | WIRED | StenograficoParser class provides all parser methods extracted from original |
| `build/tests/test_xml_parser.py` | `build/xml_parser.py` | `from xml_parser import StenograficoParser` | WIRED | 16 tests pass against implementation |
| `build/chunker.py` | `build/build_config.py` | `from build_config import BuildConfig` | WIRED | Import present; BuildConfig used as default config |
| `build/db_builder.py` | `build/xml_parser.py` | Consumes parse_xml_file output dict | WIRED | ingest_session() unpacks `session, debates, phases, speeches, votes, act_references` |
| `build/db_builder.py` | `build/chunker.py` | `from chunker import chunk_speech` | WIRED | Called inside ingest_session for each speech |
| `build/db_builder.py` | `neo4j` | `execute_write` with UNWIND | WIRED | 15 UNWIND, 24 execute_write/execute_read; zero session.run() auto-commits |
| `build/build_and_update.py` | `build/xml_parser.py` | `from xml_parser import StenograficoParser` | WIRED | Line 44 |
| `build/build_and_update.py` | `build/db_builder.py` | `from db_builder import DatabaseBuilder` | WIRED | Line 45 |
| `Makefile` | `build/build_and_update.py` | `make db-all` calls `$(BUILD_SCRIPT) build` | WIRED | BUILD_SCRIPT variable set; `db-all` target confirmed |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| BUILD-01 | 01-03 | English-only Neo4j schema (camelCase, PascalCase, SCREAMING_SNAKE_CASE) | SATISFIED | db_builder.py creates Session, Debate, Phase, Speech, Chunk, Vote, ParliamentaryAct labels; zero Italian labels; HAS_VOTE, DISCUSSES, HAS_DEBATE, HAS_PHASE, CONTAINS_SPEECH, HAS_CHUNK, SPOKEN_BY relationships |
| BUILD-02 | 01-04 | Remove Italian-schema dead code from ingest_stenografici.py | SATISFIED | DEPRECATED header on line 1; save_to_neo4j, clear_stenografici_data, create_constraints, create_indexes, main() all removed |
| BUILD-03 | 01-01 | Extract XML parser into standalone module | SATISFIED | build/xml_parser.py with StenograficoParser; zero Neo4j imports confirmed |
| BUILD-04 | 01-02 | Remove redundant Chunk properties (startCharRaw, endCharRaw) | SATISFIED | chunker.py output is {id, text, index} only; CONTEXT confirms 0 Chunks with start_char_raw |
| BUILD-05 | 01-03 | Remove redundant Speech.preprocessedText property | SATISFIED | db_builder._create_speeches Cypher sets only text and speakingRole; CONTEXT confirms 0 Speeches with preprocessed_text |
| BUILD-06 | 01-03 | Remove redundant Session.completeDate property | SATISFIED | db_builder._create_session Cypher has no completeDate; CONTEXT confirms Session keys = [id, month, day, chamber, date, number, legislature, year] — no completeDate |
| BUILD-07 | 01-03 | UNWIND batch writes for bulk ingestion | SATISFIED | 15 UNWIND occurrences in db_builder.py; _batch_write helper generic method present |
| BUILD-08 | 01-03 | Managed transactions (execute_read/execute_write, no session.run) | SATISFIED | 24 managed transaction calls; 0 session.run() auto-commits in db_builder.py |
| BUILD-09 | 01-04, 01-05 | Single `make db-all` target that rebuilds entire DB from scratch | SATISFIED | Makefile db-all target confirmed; all integration tests pass |
| DATA-01 | 01-01, 01-03, 01-04 | Persist Vote nodes with HAS_VOTE from Session | SATISFIED | CONTEXT: 7,355 Vote nodes; Session-[:HAS_VOTE]->Vote Cypher confirmed; xml_parser reads from raccoltaVotazioni |
| DATA-02 | 01-01, 01-03, 01-04 | Debate-[:DISCUSSES]->ParliamentaryAct edges from argomenti | SATISFIED | CONTEXT: 2,144 DISCUSSES edges; _parse_act_references navigates metadati/argomenti |
| DATA-03 | 01-01, 01-03, 01-04 | Speech.speakingRole from emphasis tag | SATISFIED | CONTEXT: 5,974 Speeches with speakingRole; _extract_speaking_role present in xml_parser |
| DATA-04 | 01-01, 01-03 | Phase.phaseType enum from Italian titles | SATISFIED | classify_phase_type maps 10+ patterns to English enum values; CONTEXT confirms phase types: ballot, discussion, reply, vote_declaration, vote, resolution_announcement, government_opinion, general_discussion, other, article_examination |

**All 13 requirements: SATISFIED**

### Anti-Patterns Found

None of significance. No TODO/FIXME blockers, no empty implementations, no stubs in production code paths.

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| `build/ingest_stenografici.py` | DEPRECATED file retained for backward compatibility | INFO | Intentional — plan explicitly required keeping parser methods, only deleting write methods |

### Human Verification Required

The following items were provided as confirmed facts in the verification context and do not require additional human testing:

1. **Database state verified by user** — 7,355 Vote nodes, 2,144 DISCUSSES edges, 5,974 Speeches with speakingRole, 0 Chunks with start_char_raw, 0 Speeches with preprocessed_text, Session keys confirmed (no completeDate), Phase types confirmed (10 distinct values).

2. **make db-all end-to-end** — The full pipeline has been run and verified. Integration tests pass against the live Neo4j instance.

### Gaps Summary

No gaps. All phase goal components verified:

- Clean English-only schema: VERIFIED (zero Italian labels in db_builder, constraints use PascalCase labels, relationships use SCREAMING_SNAKE_CASE)
- No dead code: VERIFIED (Italian save methods removed from ingest_stenografici.py, DEPRECATED header, populate_ruoli.py uses Speech not Intervento)
- No redundant properties: VERIFIED (startCharRaw, endCharRaw, completeDate, preprocessedText absent from both code and live database)
- Full data extraction: VERIFIED (Vote nodes, DISCUSSES edges, speakingRole, phaseType all present with correct counts in live DB)
- Test coverage: VERIFIED (46 unit tests + 9 integration tests, all passing)

---

_Verified: 2026-04-02_
_Verifier: Claude (gsd-verifier)_
