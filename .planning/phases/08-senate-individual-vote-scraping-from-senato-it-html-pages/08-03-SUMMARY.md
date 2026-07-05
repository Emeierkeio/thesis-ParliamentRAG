---
phase: 08-senate-individual-vote-scraping-from-senato-it-html-pages
plan: "03"
subsystem: build-pipeline
tags: [senate, sparql, votes, neo4j, tdd]
dependency-graph:
  requires: []
  provides: [senate_sparql_ingester.SenateVoteIngester, build/senate_sparql_ingester.py]
  affects: [Session HAS_VOTE Vote graph, timeline voteCount for senato chamber]
tech-stack:
  added: []
  patterns:
    - GET-based SPARQL with browser User-Agent for dati.senato.it
    - Per-seduta SPARQL scoping (no global osr:Votazione queries — avoids 5xx)
    - UNWIND batch writes with chamber+legislature-scoped Session MATCH
    - Outcome derivation from favorevoli/maggioranza (no approvato flag)
key-files:
  created:
    - build/senate_sparql_ingester.py
    - build/tests/test_senate_sparql_ingester.py
  modified: []
decisions:
  - GET-only for dati.senato.it because POST returns 403 (WAF blocks non-browser clients)
  - Per-seduta SPARQL scoping to avoid 5xx from global osr:Votazione aggregate queries
  - Vote ids prefixed with senato_ to avoid collision with Camera and XML-sourced Vote nodes
  - Outcome derived from favorevoli >= maggioranza (Senate has no approvato flag)
  - --individual-only stub left in CLI (Plan 04 fills senator-level IndividualVote ingest)
metrics:
  duration: "3 minutes"
  completed: "2026-07-05"
  tasks: 2
  files: 2
---

# Phase 08 Plan 03: Senate SPARQL Aggregate Vote Ingester Summary

**One-liner:** GET-based Senate SPARQL ingester with per-seduta scoping, outcome derivation from favorevoli/maggioranza, and senato-prefixed Vote ids matching the existing Vote node shape.

## What Was Built

Two new files implementing VOT-04 (Senate aggregate votes from SPARQL):

**`build/senate_sparql_ingester.py`** — New module with:
- `_senato_sparql_get()` — GET-based SPARQL helper with browser User-Agent (dati.senato.it blocks POST with 403); query encoded as URL query string; same retry/backoff pattern as Camera's `_sparql_get`
- `parse_senate_votazione_uri()` — extracts `(leg, seduta, vote)` from URI `http://dati.senato.it/votazione/{leg}-{seduta}-{vote}`
- `_derive_senate_outcome()` — derives "approved"/"rejected"/"unknown" from `favorevoli >= maggioranza` (no `approvato` flag on Senate votazioni)
- `SenateVoteIngester` class with `ingest_aggregate_votes(legislature, limit_sessions)`:
  - Reads session numbers from DB for `chamber='senato'`
  - Skips sittings that already have HAS_VOTE (idempotency/resume)
  - Per-sitting: resolves sedutaassemblea URI via SPARQL lookup, fetches all votazioni, writes Vote nodes
  - Vote id format: `senato_leg{N}_sed{MMM}_v{KKK}` (3-digit zero-padded, no collision with Camera/XML nodes)
  - Cypher scoped to `chamber='senato'` and `legislature` for Session MATCH
- CLI argparse: `--aggregate-only`, `--individual-only` (stub), `--legislature`, `--limit-sessions`

**`build/tests/test_senate_sparql_ingester.py`** — 21 unit tests across 4 classes:
- `TestSenateUriParsing` — URI parsing (5 cases)
- `TestSenateOutcomeDerivation` — outcome derivation including None/None edge cases (6 cases)
- `TestSenateVoteId` — Vote id format with 3-digit zero-padding (4 cases)
- `TestSenateHttpMethod` — GET method, no data body, Mozilla+Chrome User-Agent, query in URL (6 cases)

## TDD Execution

| Phase | Files | Status |
|-------|-------|--------|
| RED | test_senate_sparql_ingester.py | ModuleNotFoundError confirmed |
| GREEN | senate_sparql_ingester.py | 21/21 tests pass |

## Verification

- `cd build && python -m pytest tests/test_senate_sparql_ingester.py -q` — 21 passed
- `python build/senate_sparql_ingester.py --help` — lists `--aggregate-only`, `--legislature`, `--limit-sessions`
- No `data=` body parameter anywhere in `senate_sparql_ingester.py` (GET-only confirmed)
- `ast.parse()` exits 0 (no syntax errors)

## Deviations from Plan

None — plan executed exactly as written.

The `_write_senate_votes` method adds `v.onMission = row.onMission` in addition to the plan's Cypher template (maps `osr:congedoMissione`). This is a minor additive improvement matching the Vote node shape used for Camera votes; not a deviation from intent.

## Self-Check: PASSED

Files exist:
- FOUND: build/senate_sparql_ingester.py
- FOUND: build/tests/test_senate_sparql_ingester.py

Commits exist:
- FOUND: 5c87ea1 (test RED)
- FOUND: 5779000 (feat GREEN)
