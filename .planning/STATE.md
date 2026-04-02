---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: planning
stopped_at: Completed 01-build-pipeline-04-PLAN.md
last_updated: "2026-04-02T10:39:03.784Z"
last_activity: 2026-04-02 — Roadmap created
progress:
  total_phases: 4
  completed_phases: 0
  total_plans: 5
  completed_plans: 4
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-02)

**Core value:** A clean, correct, and explainable codebase that is easy to maintain, extend, and reason about
**Current focus:** Phase 1 — Build Pipeline

## Current Position

Phase: 1 of 4 (Build Pipeline)
Plan: 0 of TBD in current phase
Status: Ready to plan
Last activity: 2026-04-02 — Roadmap created

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: -
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**
- Last 5 plans: -
- Trend: -

*Updated after each plan completion*
| Phase 01-build-pipeline P02 | 4min | 1 tasks | 5 files |
| Phase 01-build-pipeline P01 | 13 | 2 tasks | 7 files |
| Phase 01-build-pipeline P03 | 5 | 2 tasks | 2 files |
| Phase 01-build-pipeline P04 | 12 | 2 tasks | 5 files |

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Phase 1: Extract xml_parser.py BEFORE deleting any code from ingest_stenografici.py (Pitfall 5)
- Phase 1: Embedding cache key must not change — model name string "text-embedding-3-small" is frozen
- Phase 1+2: Phase 1 schema rebuild and Phase 2 Cypher updates MUST deploy as a unit
- Phase 4: NER model version compatibility must be validated during planning (it_nerIta_trf requires spaCy 3.2.x — may need it_core_news_lg instead)
- [Phase 01-02]: Used re.split with captured delimiter instead of lookbehind regex for sentence splitting — Python re does not support variable-width lookbehinds
- [Phase 01-02]: build/ directory gitignored by Python convention; source files tracked via git add -f — gitignore needs update in Plan 01
- [Phase 01-02]: Orphan final chunk below min_speech_length merged into previous chunk rather than dropped
- [Phase 01-build-pipeline]: xml_parser.py has zero Neo4j dependency — StenograficoParser is a pure data extraction class
- [Phase 01-build-pipeline]: Votes parsed from raccoltaVotazioni at resoconto level — Session-[:HAS_VOTE]->Vote (not Debate-[:HAS_VOTE]->Vote)
- [Phase 01-build-pipeline]: preprocess_text() drops alignment_map return — no alignment_map logic in new codebase
- [Phase 01-build-pipeline]: DatabaseBuilder accepts an injected neo4j.Driver (no credentials in constructor)
- [Phase 01-build-pipeline]: load_government_members_from_path added as enriched variant alongside basic load_government_members
- [Phase 01-04]: csv_loader.py is standalone (no db_builder import) to prevent circular imports
- [Phase 01-04]: download.py requires explicit xml_dir param — no module-level default for path-agnosticism
- [Phase 01-04]: build_and_update.py calls load_government_members_from_path (CSV-enriched) for government member creation

### Pending Todos

None yet.

### Blockers/Concerns

- [Pre-Phase 1] Embedding model name discrepancy: PROJECT.md says text-embedding-ada-002 but code uses text-embedding-3-small. Fix the doc during Phase 1 — do not change code.
- [Pre-Phase 2] SSE event contract (18 yield sites in query.py) must be documented before any router refactoring. Payload field names are a frozen contract.
- [Pre-Phase 4] NER version compatibility: bullmount/it_nerIta_trf requires spaCy >=3.2.1,<3.3.0. Validate against current stack before scheduling NER work.

## Session Continuity

Last session: 2026-04-02T10:39:03.782Z
Stopped at: Completed 01-build-pipeline-04-PLAN.md
Resume file: None
