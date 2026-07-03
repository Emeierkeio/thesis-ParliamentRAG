---
phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
plan: 03
subsystem: database
tags: [neo4j, python, build-pipeline, csv-loaders, legislature, argparse, makefile]

# Dependency graph
requires:
  - phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
    plan: 01
    provides: legislature field on QueryRequest/ChatRequest
  - phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
    plan: 02
    provides: ROMAN_MAP + legislature-filtered get_existing_session_numbers + test_db_builder.py (committed in 12-02 docs)
provides:
  - Six CSV loaders parametrized via ROMAN_MAP (legislature: int = 19 default preserves existing behavior)
  - build_and_update.py four modes (do_build, do_update, do_build_senate, do_update_senate) all accept legislature param
  - --legislature CLI flag on build_and_update.py main()
  - Parametrized globs (stenografico_leg{legislature}_*.xml, resaula_leg{legislature}_*.akn) and regex patterns
  - db-ingest-leg18 Makefile target: additive leg18 ingest using update+update-senate modes
affects:
  - Phase 12 Plan 04 (XVIII ingest execution — depends on this parametrization being live)
  - Any future legislature (e.g., XX) — loaders and build modes ready without code changes

# Tech tracking
tech-stack:
  added: []
  patterns:
    - ROMAN_MAP.get(legislature, f"leg{legislature}") fallback for unmapped legislature numbers
    - legislature: int = 19 default on all loaders preserves backward compat with no-arg call sites
    - Standalone ROMAN_MAP in db_builder.py (not imported from download_deputies_csv.py — no cross-script coupling)

key-files:
  created:
    - backend/tests/test_db_builder.py (source-inspection tests; committed in 12-02 docs commit d74957d)
  modified:
    - build/db_builder.py
    - build/build_and_update.py
    - Makefile

key-decisions:
  - "ROMAN_MAP duplicated in db_builder.py (not imported from download_deputies_csv.py) to keep build/ scripts standalone per Phase 01-04 decision"
  - "_build_gov_uri_map and load_government_members_from_path also parametrized to achieve deputati_xix.csv grep count = 0"
  - "db-ingest-leg18 uses update+update-senate modes (not build) — additive, no nuke, preserves XIX data"
  - "legislature param placed last in all do_* function signatures to avoid breaking existing keyword-only callers"

patterns-established:
  - "Build script loaders: all accept legislature: int = 19, derive roman via ROMAN_MAP.get(legislature, f'leg{legislature}')"
  - "Build modes: all four do_* functions thread legislature through to loaders + parser + dedup query"

requirements-completed: [LEG-03]

# Metrics
duration: 7min
completed: 2026-07-03
---

# Phase 12 Plan 03: Legislature Parametrization (Build Pipeline) Summary

**Build pipeline fully parametrized by legislature: six CSV loaders use ROMAN_MAP, four build modes accept --legislature CLI flag, and make db-ingest-leg18 enables additive XVIII ingest without nuking XIX data.**

## Performance

- **Duration:** 7 min
- **Started:** 2026-07-03T16:09:15Z
- **Completed:** 2026-07-03T16:16:30Z
- **Tasks:** 3 (Task 1 artifacts pre-committed in plan 12-02 docs commit)
- **Files modified:** 3

## Accomplishments

- Six CSV loaders (load_deputies, load_groups, load_committees, load_senators, load_senator_groups, load_senator_committees) plus _build_gov_uri_map and load_government_members_from_path all parametrized via ROMAN_MAP; zero "deputati_xix.csv" hardcodes remain
- Four build modes (do_build, do_update, do_build_senate, do_update_senate) parametrized with legislature param; globs and regex patterns use f-strings; get_existing_session_numbers calls pass legislature; SenateStenograficoParser instantiated with legislature=legislature
- db-ingest-leg18 Makefile target added for additive XVIII ingest with --skip-download (raw data already on disk from Plan 01 download layer)

## Task Commits

1. **Task 1: Wave 0 test + ROMAN_MAP + legislature-filtered dedup** - `d74957d` (committed in 12-02 docs commit — pre-existing when plan 12-03 started)
2. **Task 2: Parametrize six CSV loaders** - `493c61f` (feat)
3. **Task 3: Parametrize four build modes + CLI flag + db-ingest-leg18 target** - `90bad90` (feat)

## Files Created/Modified

- `build/db_builder.py` - ROMAN_MAP constant; legislature param on all six loaders + _build_gov_uri_map + load_government_members_from_path + get_existing_session_numbers
- `build/build_and_update.py` - legislature: int = 19 on do_build/do_update/do_build_senate/do_update_senate; parametrized globs/regex; --legislature CLI flag; legislature threaded to all dispatch calls
- `Makefile` - db-ingest-leg18 target + db-ingest-leg18 in .PHONY
- `backend/tests/test_db_builder.py` - source-inspection tests (pre-committed in 12-02 docs)

## Decisions Made

- ROMAN_MAP duplicated in db_builder.py rather than imported from download_deputies_csv.py — keeps build/ scripts standalone (Phase 01-04 architectural decision).
- _build_gov_uri_map and load_government_members_from_path parametrized (beyond the six explicit loaders in the plan) to satisfy acceptance criterion `grep -c "deputati_xix.csv" == 0`.
- db-ingest-leg18 uses `update` + `update-senate` modes with `--skip-download` — additive, no nuke, XVIII raw data already on disk.
- legislature param placed last in all do_* function signatures to preserve backward compatibility with any no-arg callers.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Parametrized _build_gov_uri_map and load_government_members_from_path**
- **Found during:** Task 2 (parametrize six CSV loaders)
- **Issue:** Plan listed only six loaders, but _build_gov_uri_map (called by load_groups and load_committees) and load_government_members_from_path both also read "deputati_xix.csv". Leaving them would fail the acceptance criterion `grep -c "deputati_xix.csv" == 0`.
- **Fix:** Added `legislature: int = 19` param and ROMAN_MAP.get() filename derivation to both helpers.
- **Files modified:** build/db_builder.py
- **Verification:** grep count = 0; tests pass.
- **Committed in:** 493c61f (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (missing critical — scope slightly wider than plan's six-loader list)
**Impact on plan:** Necessary for acceptance criterion. No behavior change with legislature=19 default.

## Known Gaps (documented, out of Phase 12 scope)

Per plan's interfaces block — explicitly NOT implemented:

1. **MEMBER_OF_GROUP / MEMBER_OF_COMMITTEE legislature property**: relationship-level legislature tagging deferred; authority-per-legislature is a Phase 12 RESEARCH Pattern 4 open question.
2. **Authority scorer legislature filter** (scorer.py lines 452/508/634): authority scores are not yet filtered by legislature; this is out of scope per Phase 12 RESEARCH.
3. **XVIII government roles in build/app_config.py** (Conte I/II, Draghi): GovernmentMember nodes for XVIII are not configured in GOVERNMENT_GROUPS; their speeches will stay orphaned after leg18 ingest. Document, do not block.

## Issues Encountered

- Task 1 artifacts (ROMAN_MAP, legislature-filtered get_existing_session_numbers, test_db_builder.py) were already committed as part of the plan 12-02 docs commit (`d74957d`). Tasks 2 and 3 were executed normally from a clean baseline.
- The build/ and .planning/ directories are gitignored: used `git add -f` for build/ files.

## User Setup Required

None — this plan produces parametrized code only. No live ingest runs. Run `make db-download-leg18` (Plan 01) before `make db-ingest-leg18` (Plan 04 trigger).

## Next Phase Readiness

- Plan 04 (XVIII ingest execution) can now run `make db-ingest-leg18`; the parametrized build pipeline will correctly ingest all 741 Camera + 459 Senate XVIII sittings into the existing XIX database without conflict.
- Legislature filter on the backend API (Plan 02) and this build pipeline parametrization (Plan 03) are both live — prerequisite chain for Plan 04 is complete.

---
*Phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend*
*Completed: 2026-07-03*
