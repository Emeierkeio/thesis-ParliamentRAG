---
phase: 09-parliamentary-timeline-with-daily-debates-recap-and-per-debate-speaker-summaries
plan: "01"
subsystem: build-pipeline
tags: [ai-summaries, neo4j, openai, makefile, async, timeline]
dependency_graph:
  requires: []
  provides:
    - build/generate_summaries.py
    - make generate-summaries target
    - make db-full target
    - Session.recapIt / Session.recapEn properties (written at build time)
    - Debate.recapIt / Debate.recapEn properties (written at build time)
    - SpeakerDebateSummary nodes with HAS_DEBATE_SUMMARY + FOR_DEBATE relationships
  affects:
    - Makefile (db-full dependency chain)
    - build/requirements-build.txt (added openai + tqdm)
tech_stack:
  added:
    - openai AsyncOpenAI (gpt-4.1-mini)
    - tqdm (progress bar)
    - asyncio (async generation with Semaphore rate limiting)
  patterns:
    - Resumable build script (WHERE recapIt IS NULL)
    - MERGE idempotency for SpeakerDebateSummary nodes
    - coalesce(Deputy, GovernmentMember) pattern for speaker resolution
    - Async pair generation (IT + EN in parallel per summary)
key_files:
  created:
    - build/generate_summaries.py
  modified:
    - Makefile
    - build/requirements-build.txt
decisions:
  - gpt-4.1-mini selected (consistent with Phase 7 pipeline optimization decision)
  - asyncio.Semaphore(concurrency) caps parallel calls; default concurrency=10
  - Short debates (<3 speeches) skipped to avoid trivial AI output
  - Speaker texts truncated to 4000 chars to stay within context limits
  - Resumability at session level (recapIt IS NULL) and debate level (debate.recapIt check)
  - openai and tqdm added to requirements-build.txt (were missing — prerequisite for script)
metrics:
  duration: "8min"
  completed: "2026-04-08"
  tasks_completed: 2
  files_created: 1
  files_modified: 2
---

# Phase 9 Plan 01: AI Summary Generation Build Script Summary

**One-liner:** Async gpt-4.1-mini pipeline that pre-computes session/debate/speaker summaries at build time and stores them in Neo4j, with dry-run cost estimation and full resumability.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create generate_summaries.py build script | 34fae04 | build/generate_summaries.py, build/requirements-build.txt |
| 2 | Add Makefile targets for summary generation | f510e6c | Makefile |

## What Was Built

### build/generate_summaries.py (613 lines)

Standalone async Python script following the existing `sparql_ingester.py` CLI pattern. Key components:

- **SummaryGenerator class** — reads from Neo4j, generates AI summaries via `AsyncOpenAI`, writes results back
- **Resumability** — queries sessions `WHERE s.recapIt IS NULL`; debates skip if `recapIt` already set
- **Dry-run mode** — counts pending sessions/debates/speaker pairs, estimates tokens (`len(text)/4`) and cost at gpt-4.1-mini rates ($0.40/1M input + $1.60/1M output)
- **Rate limiting** — `asyncio.Semaphore(concurrency)` caps parallel OpenAI calls (default: 10)
- **Speaker handling** — `coalesce(d:Deputy, g:GovernmentMember)` pattern to resolve speaker node type
- **SpeakerDebateSummary nodes** — MERGE with `id = f"{debate_id}_{speaker_id}"`, linked via `HAS_DEBATE_SUMMARY` and `FOR_DEBATE`
- **Short content guards** — debates with <3 speeches skipped; speakers with <100 chars total text skipped
- **Progress** — tqdm progress bar for sessions; logger.info per session with debate/speaker counts

### Makefile targets

- `generate-summaries` — starts Neo4j, waits for bolt port 7689, runs script; `DRY_RUN=1` passes `--dry-run`
- `db-full` — composite `db-all generate-summaries` for complete one-shot build including AI summaries

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added openai and tqdm to requirements-build.txt**
- **Found during:** Task 1 planning
- **Issue:** Plan stated "No new dependencies required (openai, tqdm already in requirements-build.txt)" but neither was present. Script would fail to import at runtime.
- **Fix:** Added `openai>=1.10.0` and `tqdm>=4.64.0` to `build/requirements-build.txt`. openai was already in backend/requirements.txt; tqdm was not in any requirements file.
- **Files modified:** build/requirements-build.txt
- **Commit:** 34fae04 (included in Task 1 commit)

## Verification Results

- `python -c "import ast; ast.parse(...)"` — PASS (valid Python syntax)
- `grep "generate-summaries" Makefile` — 3 occurrences (target def + .PHONY + db-full dep)
- `grep "db-full" Makefile` — 2 occurrences (.PHONY + target def with dependency chain)
- `grep "SpeakerDebateSummary" build/generate_summaries.py` — present
- `grep "recapIt IS NULL" build/generate_summaries.py` — present

## Self-Check: PASSED

Files:
- FOUND: build/generate_summaries.py (613 lines, >200 minimum)
- FOUND: Makefile updated with generate-summaries and db-full

Commits:
- FOUND: 34fae04 — feat(09-01): create generate_summaries.py AI summary generation script
- FOUND: f510e6c — feat(09-01): add generate-summaries and db-full Makefile targets
