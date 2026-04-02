---
phase: 01-build-pipeline
plan: 01
subsystem: testing
tags: [xml-parsing, python, pytest, xml.etree.ElementTree, regex, dataclass]

requires: []
provides:
  - StenograficoParser class in build/xml_parser.py — pure XML extraction with zero Neo4j dependency
  - classify_phase_type() module-level function mapping Italian phase titles to English enum values
  - BuildConfig dataclass in build/build_config.py with chunk_size, chunk_overlap, min_speech_length, batch_size
  - load_config(path) function loading from YAML or returning defaults
  - pytest test infrastructure in build/tests/ (conftest.py, test_xml_parser.py, test_chunker.py, test_db_builder.py)
  - Vote bug fix — votes now parsed from raccoltaVotazioni at resoconto level (not dibattito)
  - Act references parser — navigates metadati/argomenti for debate-to-act mapping
  - Speaker role extraction — institutional role from emphasis tag after nominativo
affects: [01-02, 01-03, 01-04, 01-05]

tech-stack:
  added: [regex (recursive patterns for parenthesis removal)]
  patterns:
    - StenograficoParser returns plain Python dicts — no ORM or Neo4j objects
    - TDD RED/GREEN cycle — test stubs written first, implementation second
    - Force git-add for build/ directory files (build/ is in .gitignore but files are intentionally tracked)

key-files:
  created:
    - build/xml_parser.py
    - build/build_config.py
    - build/tests/__init__.py
    - build/tests/conftest.py
    - build/tests/test_xml_parser.py
    - build/tests/test_chunker.py
    - build/tests/test_db_builder.py
  modified: []

key-decisions:
  - "xml_parser.py has zero Neo4j dependency — StenograficoParser is a pure data extraction class"
  - "Votes parsed from raccoltaVotazioni at resoconto level (fixes zero-votes bug in original code)"
  - "preprocess_text() simplified — no alignment_map return (alignment_map removed per schema decision)"
  - "Phase types classified to English enum values at parse time (phaseType property on phase dict)"
  - "build/ directory is gitignored but source files tracked via git add -f"

patterns-established:
  - "Parser returns camelCase property names matching target Neo4j node schema"
  - "Speeches with len(text) < config.min_speech_length are discarded at parse time"
  - "PRESIDENTE speeches filtered at parse time (not at DB write time)"

requirements-completed: [BUILD-03, DATA-01, DATA-02, DATA-03, DATA-04]

duration: 13min
completed: 2026-04-02
---

# Phase 1 Plan 01: XML Parser Extraction Summary

**StenograficoParser extracted into standalone xml_parser.py with vote bug fixed, act references parsed from metadati/argomenti, speaker roles from emphasis tags, and phase type classification — all 16 tests green.**

## Performance

- **Duration:** 13 min
- **Started:** 2026-04-02T10:03:09Z
- **Completed:** 2026-04-02T10:16:30Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments

- Created `build/xml_parser.py` with `StenograficoParser` class — zero Neo4j dependency, pure data extraction
- Fixed critical vote parsing bug: original code searched inside `dibattito` (found 0 votes); new code correctly reads from `raccoltaVotazioni` at resoconto level
- Added three new data extractions not present in original: act references from `metadati/argomenti`, speaker institutional roles from `<emphasis>` tags, and phase type classification
- Created pytest test infrastructure with shared fixtures (conftest.py), 16 xml_parser tests, 13 chunker stubs, 6 db_builder stubs (all skipped for Plan 03)
- All 16 tests in `test_xml_parser.py` pass; `test_db_builder.py` stubs collect without failing

## Task Commits

1. **Task 1: Create test infrastructure and build_config.py** - `8def510` (feat)
2. **Task 2: Extract StenograficoParser into xml_parser.py** - `4810ac5` (feat)

## Files Created/Modified

- `build/xml_parser.py` — StenograficoParser class, classify_phase_type function
- `build/build_config.py` — BuildConfig dataclass, load_config() (committed in prior session as part of 01-02 execution attempt)
- `build/tests/__init__.py` — test package marker (committed in prior session)
- `build/tests/conftest.py` — shared fixtures: sample_session_xml, sample_vote_xml, speech role fixtures, tmp_xml_file
- `build/tests/test_xml_parser.py` — 16 tests for StenograficoParser and classify_phase_type
- `build/tests/test_chunker.py` — 13 stubs for chunk_speech contract (Plan 02)
- `build/tests/test_db_builder.py` — 6 skipped stubs for DatabaseBuilder (Plan 03)

## Decisions Made

- `xml_parser.py` must never import neo4j — enforced by test assertion and acceptance criteria grep
- Votes are session-level in the XML (`Session-[:HAS_VOTE]->Vote` relationship is correct, not `Debate-[:HAS_VOTE]->Vote` as CONTEXT.md originally stated)
- `preprocess_text()` drops the alignment_map return entirely — no alignment_map logic in the new codebase

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Raw string syntax error in speaker_pattern regex**
- **Found during:** Task 2 (first test run)
- **Issue:** `r'...\''` in raw string caused `SyntaxError: unmatched ]` — backslash-single-quote in raw string doesn't escape the closing quote
- **Fix:** Changed to double-quoted string `r"...'..."` for the speaker pattern
- **Files modified:** build/xml_parser.py
- **Verification:** Module imports successfully; all 16 tests pass
- **Committed in:** 4810ac5 (Task 2 commit)

**2. [Rule 3 - Blocking] build/ directory gitignored — files needed force-add**
- **Found during:** Task 1 commit
- **Issue:** `.gitignore` excludes `build/` (Python build artifacts pattern); `git add` silently refused
- **Fix:** Used `git add -f` for all new build/ source files; confirmed existing tracked files (build_config.py, __init__.py, chunker.py) were already force-added in prior session
- **Files modified:** none (git operation only)
- **Verification:** `git ls-files build/` confirms all files tracked
- **Committed in:** 8def510 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (1 bug, 1 blocking)
**Impact on plan:** Both auto-fixes necessary for correctness and git tracking. No scope creep.

## Issues Encountered

- `build_config.py` and `build/tests/__init__.py` were already committed in a prior aborted 01-02 session. The files matched expected content, so no re-work was needed.
- `test_chunker.py` also existed from the prior session with richer content than the plan specified — the richer version was kept (more test coverage is better).

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- `build/xml_parser.py` is ready for use by Plan 02 (chunker) and Plan 03 (db_builder)
- `build/tests/test_chunker.py` stubs define the exact contract for `chunk_speech()` in Plan 02
- `build/tests/test_db_builder.py` stubs define the contract for `DatabaseBuilder` in Plan 03
- No blockers

---
*Phase: 01-build-pipeline*
*Completed: 2026-04-02*
