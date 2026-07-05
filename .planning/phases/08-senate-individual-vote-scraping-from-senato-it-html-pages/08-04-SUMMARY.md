---
phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages
plan: 04
subsystem: build-pipeline
tags: [senate, sparql, individual-votes, tdd, neo4j]
dependency_graph:
  requires: ["08-03"]
  provides: ["ingest_individual_votes", "senate IndividualVote nodes"]
  affects: ["enrich-votes-individual Makefile target"]
tech_stack:
  added: []
  patterns: ["TDD red-green", "per-sitting resume", "SPARQL GET senator links", "UNWIND batch MERGE"]
key_files:
  created: []
  modified:
    - build/senate_sparql_ingester.py
    - build/tests/test_senate_sparql_ingester.py
decisions:
  - "_senator_id_from_uri is a module-level helper (not a method) — pure function with no dependencies, easy to test in isolation"
  - "_get_senate_senator_links issues three separate SPARQL queries per Votazione (one per outcome) matching the verified Pattern 5 step 3 from RESEARCH.md"
  - "--aggregate-only and --individual-only are mutually exclusive argparse group — prevents accidental double-run, documented in --help"
  - "Per-sitting resume uses DISTINCT s.number via Deputy{chamber:'senato'}-[:VOTED]->IndividualVote-[:ON_VOTE]->Vote<-[:HAS_VOTE]-Session path"
metrics:
  duration: 3min
  completed: "2026-07-05"
  tasks_completed: 2
  files_modified: 2
---

# Phase 08 Plan 04: Senate Individual Vote Ingest Summary

**One-liner:** Senate per-senator IndividualVote nodes via three SPARQL outcome queries per Votazione, with per-sitting resume using the VOTED→IndividualVote→ON_VOTE→Vote path.

## Tasks Completed

| # | Name | Commit | Type | Files |
|---|------|--------|------|-------|
| 1 | Write failing tests for senate individual vote id + resume (RED) | c8e9d1a | test | build/tests/test_senate_sparql_ingester.py |
| 2 | Implement Senate individual vote ingest (GREEN) | 41996b9 | feat | build/senate_sparql_ingester.py |

## What Was Built

**Module helper** (`build/senate_sparql_ingester.py`):
- `_SENATORE_URI_RE`: regex to extract senator id from `http://dati.senato.it/senatore/{id}` URIs
- `_senator_id_from_uri(uri)`: returns trailing digit string or None

**New SenateVoteIngester methods:**
- `_get_senate_senator_links(votazione_uri)`: three SPARQL queries (favorevole/contrario/astenuto) → dict `{"favor": [...], "against": [...], "abstain": [...]}`
- `_get_senate_sittings_with_individual_votes(legislature)`: Cypher resume query via `Deputy{chamber:'senato'}-[:VOTED]->IndividualVote-[:ON_VOTE]->Vote<-[:HAS_VOTE]-Session`
- `ingest_individual_votes(legislature, limit_sessions)`: per-sitting loop, skips done sittings, builds batch of `{id, senatorUri, voteId, outcome}`, delegates to `_write_senate_individual_votes`
- `_write_senate_individual_votes(batch)`: `MATCH (d:Deputy {id: row.senatorUri})` + `MATCH (v:Vote {id: row.voteId})` + `MERGE IndividualVote` + `MERGE VOTED/ON_VOTE`, chunked by BATCH_SIZE

**IndividualVote id format:** `iv_senato_{senator_id}_{seduta}_{vote}` — e.g. `iv_senato_17542_167_42` — never collides with Camera ids.

**CLI:** `--aggregate-only` and `--individual-only` are now mutually exclusive via argparse group. `--individual-only` calls `ingest_individual_votes`.

## Test Coverage

13 new tests added (all green after GREEN phase):
- `TestSenateIndividualVoteId` (5 tests): id format, `_senator_id_from_uri` extraction, None for invalid URIs
- `TestSenateIndividualOutcomeMap` (4 tests): keys favor/against/abstain, correct URI routing per outcome
- `TestSenateIndividualResume` (4 tests): skip done sittings, call count for `_get_senate_seduta_uri`, stats dict shape

Total suite: 34 passed.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

- `grep -q "def ingest_individual_votes" build/senate_sparql_ingester.py` → found
- `grep -q "def _get_senate_sittings_with_individual_votes" build/senate_sparql_ingester.py` → found
- `grep -q "iv_senato_" build/senate_sparql_ingester.py` → found
- `grep -q "MATCH (d:Deputy {id: row.senatorUri})" build/senate_sparql_ingester.py` → found
- `cd build && python -m pytest tests/test_senate_sparql_ingester.py -q` → 34 passed
- Task 1 commit c8e9d1a exists
- Task 2 commit 41996b9 exists
