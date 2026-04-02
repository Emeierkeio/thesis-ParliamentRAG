---
phase: 01-build-pipeline
plan: 02
subsystem: build
tags: [python, chunking, nlp, yaml, tdd]

# Dependency graph
requires:
  - phase: 01-build-pipeline
    plan: 01
    provides: "BuildConfig dataclass (load_config function already present in build_config.py)"
provides:
  - "chunk_speech(text, speech_id, config) function in build/chunker.py"
  - "build/config.yaml with externalized chunking parameters"
  - "13 passing unit tests for chunk_speech in build/tests/test_chunker.py"
affects: [01-03-db-builder, 01-04-build-script, 01-05-make]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "TDD red-green: test file committed before implementation"
    - "Minimal chunk dicts: id/text/index only — no legacy offset fields"
    - "Sentence-aware split with re.split on captured delimiter, abbreviation check via regex"
    - "Greedy overlap: step back from chunk boundary until chunk_overlap chars accumulated"

key-files:
  created:
    - build/chunker.py
    - build/config.yaml
    - build/tests/test_chunker.py
  modified:
    - build/build_config.py (tracked in git for the first time via git add -f)
    - build/tests/__init__.py (tracked in git for the first time via git add -f)

key-decisions:
  - "Used re.split with captured delimiter instead of lookbehind regex — Python re module does not support variable-width lookbehinds"
  - "build/ directory is gitignored (Python convention); used git add -f to force-track source files"
  - "Orphan final chunk below min_speech_length is merged into previous chunk rather than dropped"

patterns-established:
  - "chunk_speech: returns list[dict] with exactly {id, text, index} — no other keys ever"
  - "Sentence split: _SENTENCE_SPLIT captures delimiter; _ABBREV_PATTERN guards abbreviation boundaries"

requirements-completed: [BUILD-04]

# Metrics
duration: 4min
completed: 2026-04-02
---

# Phase 1 Plan 02: Chunker Module Summary

**Sentence-aware speech chunker extracting id/text/index-only chunk dicts from ingest_stenografici.py create_chunks, with YAML-externalized parameters and 13 green TDD tests**

## Performance

- **Duration:** 4 min
- **Started:** 2026-04-02T10:03:08Z
- **Completed:** 2026-04-02T10:06:59Z
- **Tasks:** 1 (TDD: RED + GREEN + ancillary commits)
- **Files created:** 5

## Accomplishments

- `chunk_speech(text, speech_id, config)` standalone function with zero Neo4j dependency
- Sentence-aware splitting ports abbreviation-handling logic from `ingest_stenografici.py` create_chunks
- Overlap logic preserved: steps back from chunk boundary accumulating up to `chunk_overlap` chars
- Short speech guard (`min_speech_length`) and orphan-chunk merge both implemented
- `build/config.yaml` externalizes all four build parameters (chunk_size, chunk_overlap, min_speech_length, batch_size)
- 13 unit tests covering contract, edge cases, overlap, and abbreviation handling — all green

## Task Commits

TDD task committed in three atomic steps:

1. **RED — failing tests** - `980319b` (test)
2. **GREEN — chunker.py + config.yaml** - `4c90a78` (feat)
3. **Ancillary — track build_config.py + tests/__init__.py** - `3f7f2fa` (chore)

## Files Created/Modified

- `build/chunker.py` — `chunk_speech` function with sentence-aware splitting and overlap
- `build/config.yaml` — Externalized chunking and ingestion parameters
- `build/tests/test_chunker.py` — 13 unit tests for chunk_speech contract
- `build/build_config.py` — BuildConfig dataclass (pre-existing; now tracked in git)
- `build/tests/__init__.py` — Package marker (pre-existing; now tracked in git)

## Decisions Made

- **Regex approach:** Used `re.split` with a captured delimiter pattern rather than lookbehind assertions. Python's `re` module does not support variable-width lookbehinds; the plan's suggested pattern `(?<!\b(?:on|On|...))` would raise `re.error: look-behind requires fixed-width pattern`. The fix uses split-then-check: split on `[.!?]\s+` keeping the delimiter, then inspect the preceding token against `_ABBREV_PATTERN`.
- **git add -f for build/:** The `build/` directory is gitignored (standard Python `.gitignore` template excludes it as a build output dir). Plan 02 requires committing source files there; used `git add -f` to force-track them. This is a project convention issue for Plan 01 to address by updating `.gitignore`.
- **Orphan chunk merging:** When the final chunk is below `min_speech_length`, it is merged into the previous chunk rather than dropped, to avoid losing content from the tail of a speech.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed variable-width lookbehind in sentence-split regex**
- **Found during:** Task 1 GREEN phase (first test run)
- **Issue:** The plan's suggested regex used `(?<!\b(?:on|On|art|...))` — Python `re` does not support variable-width lookbehinds, raising `re.error` at compile time
- **Fix:** Replaced with `re.split` on captured `[.!?]\s+` delimiter, plus post-split abbreviation check using `_ABBREV_PATTERN.search()` on the preceding token
- **Files modified:** `build/chunker.py`
- **Verification:** `python -m pytest tests/test_chunker.py -v` — 13/13 passed, including `test_chunk_abbreviation_handling`
- **Committed in:** `4c90a78` (GREEN phase commit)

---

**Total deviations:** 1 auto-fixed (Rule 1 — bug in suggested regex pattern)
**Impact on plan:** Single regex fix required to make the implementation work on Python 3.12. No scope creep; all plan acceptance criteria met.

## Issues Encountered

- `build/` gitignore required `git add -f` for all build source files. This is a pre-existing project configuration issue (`.gitignore` used a generic Python template that excludes `build/` as a build-output dir, but this project uses `build/` as a source directory).

## Next Phase Readiness

- `chunk_speech` is ready for import by `db_builder.py` (Plan 03)
- `config.yaml` is ready for `load_config()` usage in the pipeline entry point (Plan 04)
- `build_config.py` is now tracked in git and importable
- Test infrastructure in `build/tests/` is ready for Plan 03 test stubs

---
*Phase: 01-build-pipeline*
*Completed: 2026-04-02*

## Self-Check: PASSED

- build/chunker.py: FOUND
- build/config.yaml: FOUND
- build/tests/test_chunker.py: FOUND
- build/build_config.py: FOUND
- Commit 980319b (RED): FOUND
- Commit 4c90a78 (GREEN): FOUND
- Commit 3f7f2fa (chore): FOUND
