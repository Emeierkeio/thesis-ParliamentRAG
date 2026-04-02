---
phase: 01-build-pipeline
plan: 03
subsystem: build-pipeline
tags: [neo4j, batch-writes, schema, db-builder, python]
key-decisions:
  - "DatabaseBuilder accepts an injected neo4j.Driver (no credentials in constructor)"
  - "load_roles reconciliation uses Speech.speakingRole not Speech.surname_name (schema cleanup)"
  - "load_government_members_from_path added as enriched variant alongside basic load_government_members"
requirements: [BUILD-01, BUILD-05, BUILD-06, BUILD-07, BUILD-08]
dependency-graph:
  requires: [01-01, 01-02]
  provides: [db_builder.DatabaseBuilder]
  affects: [build_and_update.py replacement in Phase 1]
tech-stack:
  added: []
  patterns:
    - UNWIND $batch pattern for all Neo4j bulk writes
    - execute_write/execute_read managed transactions throughout
    - Dependency injection for neo4j.Driver
metrics:
  duration: 5 minutes
  completed: 2026-04-02
  tasks-completed: 2
  files-created: 1
  files-modified: 1
key-files:
  created:
    - build/db_builder.py
  modified:
    - build/tests/test_db_builder.py
---

# Phase 1 Plan 3: DatabaseBuilder — Clean Schema and UNWIND Batch Writes Summary

**One-liner:** DatabaseBuilder with English-only Neo4j schema, UNWIND batch writes, and managed transactions replacing monolithic session.run() pattern.

## What Was Built

`build/db_builder.py` — a `DatabaseBuilder` class that accepts a `neo4j.Driver` via dependency injection and writes all parliamentary data to Neo4j using the clean English-only schema.

Key design choices:
- Constructor takes `driver` + optional `BuildConfig` — no credentials, no URI, no ownership of the driver lifecycle
- Every write uses `execute_write`, every read uses `execute_read` — zero `session.run()` auto-commit calls
- All bulk operations use `UNWIND $batch AS row` with batch splitting via `_batch_write()` helper
- English-only schema enforced: PascalCase labels, camelCase properties, SCREAMING_SNAKE_CASE relationships

## Schema Enforced

**Constraints created** (11 nodes): Session, Debate, Phase, Speech, Chunk, Deputy, ParliamentaryGroup, Committee, GovernmentMember, Vote, ParliamentaryAct

**No Italian labels** — Seduta, Dibattito, Fase, Intervento, Votazione are absent

**Dead properties removed:**
- Session: no `completeDate` — only `date` (Neo4j Date type via `date($date)`)
- Speech: no `preprocessedText` — only `text` and `speakingRole`
- Chunk: no `startCharRaw`, `endCharRaw`, `charCount` — only `text` and `index`

## Session Ingestion Flow

`ingest_session(parsed_data)` handles one XML file in order:
1. Session node (single MERGE)
2. Debates (UNWIND, HAS_DEBATE)
3. Phases (UNWIND, HAS_PHASE, phaseType set)
4. Speeches + chunk_speech() per speech (UNWIND, CONTAINS_SPEECH, HAS_CHUNK)
5. SPOKEN_BY links — Deputy match by URI, GovernmentMember fallback by name
6. Votes (UNWIND, Session-[:HAS_VOTE]->Vote — session-level, not debate-level)
7. Act references (UNWIND, Debate-[:DISCUSSES]->ParliamentaryAct with isPlaceholder flag)

## CSV Loaders

All ported with UNWIND batch writes:
- `load_deputies` — reads deputati_xix.csv, excludes GovernmentMembers by name match
- `load_groups` — reads deputati_xix_gruppi.csv, handles end_date/no end_date variants
- `load_committees` — reads deputati_xix_commissioni.csv
- `load_government_members` — creates GovernmentMember nodes from GOVERNMENT_GROUPS constant
- `load_government_members_from_path` — enriched variant with CSV photo/gender data

## Utility Methods

- `load_roles` — assigns institutional_role, role_type, IS_PRESIDENT/IS_VICE_PRESIDENT/IS_SECRETARY/GOVERNMENT_REFERENCE from app_config.py dicts
- `create_vector_index` — chunk_embedding_index on Chunk.embedding (1536 dims, cosine)
- `get_existing_session_numbers` — returns `set[int]` via execute_read

## Tests

`build/tests/test_db_builder.py` — 8 tests, all green, no skips:

| Test | What It Verifies |
|------|-----------------|
| test_create_constraints_uses_english_labels | English labels present, Italian labels absent |
| test_batch_write_splits_at_batch_size | batch_size=2, 5 items → 3 execute_write calls |
| test_create_votes_cypher_uses_session | Session + HAS_VOTE in vote Cypher, not Debate |
| test_create_speeches_no_preprocessed_text | preprocessedText absent from speech Cypher |
| test_create_chunks_no_dead_properties | startCharRaw/endCharRaw/charCount absent |
| test_session_no_complete_date | completeDate absent from session Cypher |
| test_load_deputies_uses_unwind | UNWIND present in deputy write Cypher |
| test_get_existing_session_numbers_returns_set | Returns Python set of ints |

## Verification Metrics

- `grep -c "session\.run(" db_builder.py` → 0
- `grep -c "UNWIND" db_builder.py` → 15
- `grep -c "execute_write\|execute_read" db_builder.py` → 24

## Deviations from Plan

### Implementation Consolidation

**Tasks 1 and 2 in single commit**
- **Found during:** Task 2 execution
- **Issue:** Both tasks modify the same file (`db_builder.py`). Writing the CSV loaders and utility methods simultaneously with the schema code was more coherent than splitting across commits.
- **Impact:** Single commit covers all acceptance criteria. All 8 tests pass, including Task 2 tests.

### Utility Method Addition

**[Rule 2 - Missing Functionality] Added `load_government_members_from_path`**
- **Found during:** Task 2
- **Issue:** `load_government_members()` in the original code required `DATA_DIR` global path. New DB builder uses dependency injection — no global paths. Added `load_government_members_from_path(data_path)` as enriched variant alongside the basic version.
- **Fix:** Two variants — basic (no CSV enrichment) and from_path (with photo/gender data).
- **Files modified:** build/db_builder.py

## Self-Check: PASSED
