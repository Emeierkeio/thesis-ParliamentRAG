---
phase: 01-build-pipeline
plan: 04
subsystem: database
tags: [python, neo4j, csv, xml, pipeline, refactoring]

requires:
  - phase: 01-build-pipeline plan 01
    provides: xml_parser.py with StenograficoParser
  - phase: 01-build-pipeline plan 02
    provides: chunker.py with chunk_speech
  - phase: 01-build-pipeline plan 03
    provides: db_builder.py with DatabaseBuilder

provides:
  - csv_loader.py: standalone CSV helpers (GOVERNMENT_GROUPS, parse_date_to_neo4j, clean_generic_label, extract_group_info, SIGLA_FALLBACKS, format_date_ddmmyyyy)
  - download.py: XML download logic (get_last_xml_id, download_new_xmls) with logging not print
  - build_and_update.py: refactored CLI entry point using xml_parser, db_builder, csv_loader, download — no ingest_stenografici import
  - ingest_stenografici.py: dead Italian save code removed; deprecated header added; pure parser retained for backward compat
  - populate_ruoli.py: updated to English schema labels (Speech, SPOKEN_BY, Deputy, GovernmentMember)

affects:
  - phase 02 (backend): build pipeline is now the canonical way to populate the DB
  - Makefile: db-all target is compatible with refactored build_and_update.py

tech-stack:
  added: []
  patterns:
    - csv_loader.py has zero Neo4j dependency — pure data transformation helpers
    - download.py uses logging.getLogger(__name__) instead of print()
    - build_and_update.py injects GraphDatabase.driver into DatabaseBuilder (no credentials in class)
    - do_build and do_update use load_government_members_from_path for CSV enrichment

key-files:
  created:
    - build/csv_loader.py
    - build/download.py
  modified:
    - build/build_and_update.py
    - build/ingest_stenografici.py
    - build/populate_ruoli.py

key-decisions:
  - "csv_loader.py is a standalone module (no Neo4j import, no db_builder dependency) to avoid circular imports"
  - "build_and_update.py calls load_government_members_from_path (CSV-enriched) not load_government_members (stub)"
  - "download.py requires explicit xml_dir param — no module-level path default for path-agnosticism"
  - "ingest_stenografici.py __init__ signature changed to zero-arg (driver removed) to keep backward compat while preventing accidental Neo4j use"

patterns-established:
  - "All build modules use logging.getLogger(__name__) — no print() in module code"
  - "DB credentials flow through GraphDatabase.driver in build_and_update.py only — never inside DatabaseBuilder"

requirements-completed: [BUILD-02, BUILD-09, DATA-01, DATA-02, DATA-03, DATA-04]

duration: 12min
completed: 2026-04-02
---

# Phase 01 Plan 04: Integration and Entry Point Summary

**Refactored build_and_update.py wires xml_parser + db_builder + csv_loader + download into a clean, logging-based CLI; Italian dead code removed from ingest_stenografici.py**

## Performance

- **Duration:** 12 min
- **Started:** 2026-04-02T10:31:58Z
- **Completed:** 2026-04-02T10:43:00Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- Extracted `csv_loader.py` (7 helpers: GOVERNMENT_GROUPS, SIGLA_FALLBACKS, parse_date_to_neo4j, format_date_ddmmyyyy, clean_generic_label, extract_group_info) with type hints, English docstrings, and zero Neo4j dependency
- Extracted `download.py` (get_last_xml_id, download_new_xmls) replacing all `print()` calls with structured logging and requiring explicit `xml_dir` param
- Rewrote `build_and_update.py` importing from the four new modules; removed `StenograficoIngester` import; replaced all `print()` with `logger.info/warning/error`; uses `load_government_members_from_path` for CSV-enriched government member creation
- Deleted Italian save path dead code from `ingest_stenografici.py` (save_to_neo4j, clear_stenografici_data, create_constraints, create_indexes, __init__ Neo4j driver, close, main); added DEPRECATED header
- Updated `populate_ruoli.py` MATCH clause from `Intervento`/`PRONUNCIATO_DA` to `Speech`/`SPOKEN_BY` (English schema)

## Task Commits

1. **Task 1: Extract csv_loader.py and download.py** - `19041db` (feat)
2. **Task 2: Rewire build_and_update.py and clean up dead code** - `91045e7` (feat)

## Files Created/Modified

- `build/csv_loader.py` - Standalone CSV helper functions; no Neo4j dep; `__all__` exports
- `build/download.py` - XML download from Camera API; logging-based; explicit xml_dir param
- `build/build_and_update.py` - Refactored CLI; imports xml_parser, db_builder, csv_loader, download; logging throughout
- `build/ingest_stenografici.py` - Dead save code removed; DEPRECATED header; class retained as pure parser
- `build/populate_ruoli.py` - English label update: Intervento->Speech, PRONUNCIATO_DA->SPOKEN_BY

## Decisions Made

- `csv_loader.py` is standalone (not importing from `db_builder.py`) to prevent circular imports — `db_builder.py` already contains these helpers as module-level utilities from Plan 03; the two copies coexist until a future cleanup pass
- `download.py` requires explicit `xml_dir` (no module-level default) to keep the module path-agnostic
- `build_and_update.py` calls `load_government_members_from_path` (CSV-enriched variant) not `load_government_members` (stub without CSV)
- `ingest_stenografici.py` `__init__` changed to zero-arg — driver removed; Neo4j no longer needed in the class

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None. All 37 build/tests/ pass (37 tests: 13 chunker + 8 db_builder + 16 xml_parser).

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Full build pipeline is wired end-to-end: `make db-all` calls `build_and_update.py build` which invokes all four modules in sequence
- All build/ tests pass — xml_parser, chunker, db_builder verified
- Phase 2 (backend refactoring) can proceed; the DB schema is stable and the pipeline is executable

---
*Phase: 01-build-pipeline*
*Completed: 2026-04-02*
