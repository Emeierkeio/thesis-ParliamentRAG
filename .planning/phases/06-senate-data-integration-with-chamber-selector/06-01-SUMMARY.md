---
phase: 06-senate-data-integration-with-chamber-selector
plan: "01"
subsystem: build-pipeline
tags: [senate, akn-xml, parser, download, makefile, tdd]
dependency_graph:
  requires: []
  provides: [senate_parser.SenateStenograficoParser, download_senate.download_senate_xmls, do_build_senate, db-senate-makefile-target]
  affects: [build/build_and_update.py, Makefile]
tech_stack:
  added: [akoma-ntoso-3.0-akn-xml]
  patterns: [same-contract-parser, additive-build-mode, tdd-red-green]
key_files:
  created:
    - build/senate_parser.py
    - build/download_senate.py
    - build/fixtures/sample_session.akn
    - build/tests/test_senate_parser.py
  modified:
    - build/build_and_update.py
    - Makefile
decisions:
  - "SenateStenograficoParser reuses StenograficoParser.preprocess_text() via composition (not inheritance) to avoid coupling"
  - "InizioSeduta/FineSeduta/Presidenza sections skipped at parse time — no runtime filtering needed"
  - "Flat AKN structure: each debateSection maps to exactly 1 debate + 1 phase (no sub-phases)"
  - "PRESIDENTE filtering uses two mechanisms: TLCRole lookup (primary) + <from> text check (fallback) for robustness"
  - "download_senate.py targets listing page (29 most recent) — matches RESEARCH.md recommended strategy"
  - "Session IDs: sen_leg19_sed{N}; deputatoId: sen_{numeric_id} — both avoid Camera namespace collision"
metrics:
  duration_minutes: 4
  completed_date: "2026-04-04"
  tasks_completed: 2
  files_created: 4
  files_modified: 2
---

# Phase 6 Plan 1: Senate AKN Parser + Build Pipeline Summary

**One-liner:** AKN XML parser for Senate stenografici (Akoma Ntoso 3.0) with same dict contract as Camera parser, download script with User-Agent, additive `do_build_senate()`, and `make db-senate` / `make db-all` targets.

## What Was Built

### Task 1: Senate AKN parser + download script + unit tests (TDD)

**`build/senate_parser.py`** — `SenateStenograficoParser` class:
- Parses `an:akomaNtoso/an:debate` AKN XML (namespace `http://docs.oasis-open.org/legaldocml/ns/akn/3.0/CSD03`)
- Extracts session metadata from `FRBRWork`: number, date → `session_id = f"sen_leg19_sed{N}"`
- Builds `person_lookup` (person id → showAs) and `presidente_roles` set from `TLCPerson`/`TLCRole` entries
- Skips `InizioSeduta`, `FineSeduta`, `Presidenza`, `Comunicazioni` sections entirely
- Flat structure: each remaining `debateSection` → 1 debate node + 1 phase node
- PRESIDENTE filtering: primary check via `speech.as` attribute against `presidente_roles`; fallback check via `<an:from>` text
- `deputatoId = f"sen_{numeric_id}"` (e.g. `sen_32600`)
- Reuses `StenograficoParser.preprocess_text()` via composition
- Returns `votes=[], act_references={}` (Senate votes in separate documents)

**`build/download_senate.py`** — `download_senate_xmls(xml_dir)`:
- Fetches listing page with `User-Agent: Mozilla/5.0 (compatible; ParliamentRAG/1.0)` (required — 403 otherwise)
- Extracts BGT IDs via `re.findall(r'show-doc\?leg=19&tipodoc=Resaula&id=(\d+)', html)`
- Downloads each `.akn` file, parses `FRBRnumber` to derive session number
- Saves as `resaula_leg19_{N:04d}.akn`; skips already-present files
- 1-second rate limit between downloads; graceful error handling per file

**`build/fixtures/sample_session.akn`** — Minimal valid AKN fixture:
- FRBRnumber=5, FRBRdate=2022-11-15 → session id `sen_leg19_sed5`
- TLCPerson: p32600 CASTELLONE, p12345 ROSSI
- TLCRole: rolePresidente
- 2 debateSections: InizioSeduta (skipped), DiscussioneArgomento (included)
- 3 speeches: 1 PRESIDENTE (filtered), 2 regular senators

**`build/tests/test_senate_parser.py`** — 6 unit tests, all passing:
- `test_parse_returns_correct_keys`
- `test_session_metadata`
- `test_presidente_filtered`
- `test_speeches_structure`
- `test_debates_and_phases`
- `test_votes_and_acts_empty`

### Task 2: build_and_update.py integration + Makefile targets

**`build/build_and_update.py`** changes:
- Added `from senate_parser import SenateStenograficoParser`
- Added `from download_senate import download_senate_xmls`
- Added `SENATE_XML_DIR = os.path.join(DATA_DIR, "senate_xml")`
- Added `do_build_senate()` — additive (no `nuke_database()` call), downloads AKN files, ingests all `resaula_leg19_*.akn`, runs embeddings
- CLI `choices` extended to `["build", "update", "build-senate"]`
- `main()` dispatches `build-senate` → `do_build_senate()`

**`Makefile`** changes:
- `db-senate` target: starts Neo4j, waits for readiness, calls `build-senate` mode
- `db-all` updated: calls Camera `build` then Senate `build-senate` (additive)
- `db-senate` added to `.PHONY` declaration

## Verification

```
cd build && python -m pytest tests/test_senate_parser.py -x -v
# 6 passed in 0.07s

python build/build_and_update.py --help
# usage: ... {build,update,build-senate}

grep -c "db-senate" Makefile
# 2

grep "nuke" build/build_and_update.py | grep -v "do_build_senate"
# confirms nuke_database() only in do_build(), not do_build_senate()
```

## Commits

| Hash | Description |
|------|-------------|
| 699e6fe | test(06-01): add failing tests for SenateStenograficoParser (RED) |
| e1fe0f6 | feat(06-01): implement SenateStenograficoParser + download_senate.py (GREEN) |
| 8cd3bcc | feat(06-01): integrate Senate build into build_and_update.py and Makefile |

## Deviations from Plan

None - plan executed exactly as written.

## Self-Check: PASSED

- build/senate_parser.py: FOUND
- build/download_senate.py: FOUND
- build/fixtures/sample_session.akn: FOUND
- build/tests/test_senate_parser.py: FOUND
- build_and_update.py has do_build_senate: FOUND
- Makefile has db-senate target: FOUND
- All 6 tests pass: CONFIRMED
