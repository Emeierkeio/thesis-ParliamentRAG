---
phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages
plan: 02
subsystem: build-pipeline
tags: [sparql, votes, camera, aggregate, tdd, cli]
requirements: [VOT-02, VOT-03, VOT-06]

dependency_graph:
  requires: ["08-01"]
  provides: ["ingest_camera_aggregate_votes", "camera aggregate Vote nodes", "CLI --aggregate-only/--skip-aggregate"]
  affects: ["build/sparql_ingester.py", "Neo4j Vote nodes", "Session HAS_VOTE timeline counts"]

tech_stack:
  added: []
  patterns:
    - "SELECT DISTINCT per-sitting SPARQL query to avoid duplicate rows (two rdf:type triples per votazione)"
    - "Per-sitting skip logic: _get_camera_sittings_with_votes() guards against double-counting XML-sourced votes"
    - "Stable id camera_leg{N}_sed{num:03d}_v{vote:03d} prevents collision with XML-sourced ids"
    - "_to_int() module helper for None-safe SPARQL binding integer extraction"
    - "UNWIND batch Cypher with legislature param (not in batch dict) for correct scoping"

key_files:
  modified:
    - build/sparql_ingester.py
    - build/tests/test_sparql_ingester.py

decisions:
  - "SELECT DISTINCT in Camera aggregate query — avoids 2x row count from two rdf:type triples per votazione (verified live)"
  - "start_session=350 default skips XML-covered sittings without DB lookup — combined with sittings_with_votes guard for correctness"
  - "outcome derived from ocd:approvato string ('1'/'0'/absent) → ('approved'/'rejected'/'unknown')"
  - "CLI --aggregate-only / --skip-aggregate / --legislature / --start-session cover all enrich-votes Makefile target patterns"
  - "_write_camera_aggregate_votes passes legislature as separate Cypher param (not in UNWIND batch) to match _write_votes pattern"

metrics:
  duration: "5 minutes"
  completed: "2026-07-05"
  tasks_completed: 2
  files_modified: 2
---

# Phase 08 Plan 02: Camera Aggregate Vote Ingest (SPARQL) Summary

**One-liner:** Camera aggregate Vote ingest via SELECT DISTINCT per-sitting SPARQL query with skip-if-HAS_VOTE guard and stable `camera_leg{N}_sed{num:03d}_v{vote:03d}` ids.

## What Was Built

Extended `build/sparql_ingester.py` with Camera aggregate vote ingest (VOT-02/VOT-03) and CLI flags to separate aggregate and individual runs (VOT-06 resume path).

### New Methods Added to `SparqlIngester`

| Method | Purpose |
|--------|---------|
| `ingest_camera_aggregate_votes(legislature, start_session, limit_sessions)` | Public entry point — iterates sittings, skips done, writes Vote nodes |
| `_get_camera_sittings_in_db(legislature)` | Returns set of Camera session numbers in Neo4j |
| `_get_camera_sittings_with_votes(legislature)` | Returns sessions already having HAS_VOTE (skip set) |
| `_get_camera_votes_for_sitting(legislature, session_num)` | SELECT DISTINCT query via `seduta.rdf/s{leg}_{num}` URI |
| `_write_camera_aggregate_votes(batch, legislature)` | UNWIND MERGE Vote + HAS_VOTE, chunked by BATCH_SIZE |

### New Module Helper

`_to_int(row, key)` — None-safe integer extraction from SPARQL binding dict; used for all numeric vote fields (presenti, votanti, favorevoli, contrari, astenuti, maggioranza).

### CLI Extensions

Four new flags added to `__main__`:
- `--aggregate-only` — run Camera aggregate ingest only, exit
- `--skip-aggregate` — skip aggregate, run individual votes only
- `--legislature N` — legislature number (default 19)
- `--start-session N` — first session for aggregate ingest (default 350)
- `--limit-sessions N` — cap sessions processed (testing)

## Tests Added (TDD)

| Class | Assertions |
|-------|-----------|
| `TestCameraAggregateQuery` | SELECT DISTINCT present; seduta.rdf/s19_405 URI; ocd:approvato; id = camera_leg19_sed405_v013 |
| `TestCameraAggregateSkip` | Sitting 349 (in done set) not fetched; sitting below start_session not fetched |
| `TestCameraAggregateOutcome` | approvato '1' → approved; '0' → rejected; absent → unknown |

**Test count:** 50 passed (41 existing + 9 new).

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check

### Files created/modified
- [ ] `build/sparql_ingester.py` — `grep -q "def ingest_camera_aggregate_votes"` passes
- [ ] `build/tests/test_sparql_ingester.py` — 3 new test classes added

### Commits
- `62513cb` — test(08-02): add failing tests for Camera aggregate query + skip logic (RED)
- `901b15d` — feat(08-02): implement Camera aggregate vote ingest + CLI aggregate/individual split (GREEN)

## Self-Check: PASSED
