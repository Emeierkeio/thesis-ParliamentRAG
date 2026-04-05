---
phase: 07-pipeline-optimization
plan: 03
subsystem: retrieval
tags: [ner, entity-retrieval, rrf, cypher-profiling, compass]
dependency_graph:
  requires: [07-01]
  provides: [NER entity channel, 4-channel RRF merger, RRF sweep script, Cypher timing]
  affects: [retrieval pipeline, merger, engine]
tech_stack:
  added: []
  patterns:
    - NER entity retrieval gated on query entity detection
    - Cypher execution timing via time.perf_counter()
    - 4-channel Reciprocal Rank Fusion with configurable ner_weight
    - RRF weight grid search against evaluation_set.json precision metric
key_files:
  created:
    - backend/app/services/retrieval/ner_channel.py
    - backend/scripts/rrf_sweep.py
    - backend/tests/unit/test_ner_channel.py
  modified:
    - backend/app/services/retrieval/engine.py
    - backend/app/services/retrieval/merger.py
    - backend/config/default.yaml
decisions:
  - NERChannel gated on entity_filter so it never fires for generic queries — avoids latency penalty when not needed
  - similarity=1.0 for NER results — RRF uses rank position not raw score, exact match gets neutral sentinel
  - max_workers=4 only when NER is active — 3 workers for non-entity queries
  - ner_weight=0.9 (second highest after dense=1.0) — entity matches are precision signals
  - rrf_sweep.py overrides config in-memory, never writes to disk — safe for concurrent use
  - 18 source-file inspection tests follow established project pattern (avoids scipy/NumPy 2.x in anaconda)
metrics:
  duration: 10min
  completed: 2026-04-05
  tasks: 2
  files: 6
---

# Phase 7 Plan 3: NER Entity Channel and RRF Sweep Summary

4th NER entity retrieval channel added (lawRefs/personRefs), integrated into 4-channel parallel engine with Cypher timing and RRF sweep grid search.

## What Was Built

### Task 1: NER Entity Channel + Engine + Merger Integration

**`backend/app/services/retrieval/ner_channel.py`** (new, 170 lines)

- `NERChannel` class with `retrieve(entity_filter, top_k, chambers)` method
- Cypher query matches `c.lawRefs` and `c.personRefs` arrays via `toLower(ref) CONTAINS toLower($keyword)` pattern (safe degradation if properties not yet populated)
- Returns result dicts in same format as `sparse_channel` (evidence_id, chunk_text, speaker_id, party, coalition, date, similarity, embedding, etc.)
- `similarity=1.0` sentinel for entity matches (rank position drives RRF, not raw score)
- `time.perf_counter()` timing: logs `"NER channel Cypher: %.1fms, %d results"` for profiling

**`backend/app/services/retrieval/engine.py`** (modified)

- Imported `NERChannel`, instantiated as `self.ner_channel`
- Added person name detection: uppercase words 3+ chars excluding Italian stopwords
- NER channel gated: only submitted to ThreadPoolExecutor when `entity_filter` is non-empty
- `max_workers=4` when NER active, 3 otherwise
- `run_dense/run_sparse/run_graph/run_ner` return `(result, elapsed_ms)` tuples for per-channel timing
- `logger.debug("Retrieval timing breakdown: dense=%.1fms sparse=%.1fms graph=%.1fms ner=%.1fms total=%.1fms", ...)`
- Passes `ner_results=ner_results if has_ner else None` to `merger.merge()`

**`backend/app/services/retrieval/merger.py`** (modified)

- `merge()` accepts `ner_results: Optional[List[Dict[str, Any]]] = None`
- `_compute_rrf()` accepts `ner_results: Optional[List[Dict[str, Any]]] = None`
- Reads `ner_weight` from `config.retrieval.rrf.get("ner_weight", 0.9)`
- Processes NER as 4th channel (priority: dense > sparse > graph > ner for metadata deduplication)

**`backend/config/default.yaml`** (modified)

- Added `ner_weight: 0.9` under `retrieval.rrf`
- `compass.clustering.min_fragments_for_kde: 3` already present at line 117 — no change needed

### Task 2: RRF Sweep Script and Tests

**`backend/scripts/rrf_sweep.py`** (new, 230 lines)

7-point RRF weight grid:
| Config | k | dense | sparse | graph | ner |
|---|---|---|---|---|---|
| baseline (no NER) | 60 | 1.0 | 0.8 | 0.5 | 0.0 |
| with NER | 60 | 1.0 | 0.8 | 0.5 | 0.9 |
| graph-boosted | 60 | 1.0 | 0.5 | 0.8 | 0.9 |
| equal sparse | 60 | 1.0 | 1.0 | 0.5 | 0.9 |
| lower k | 30 | 1.0 | 0.8 | 0.5 | 0.9 |
| higher k | 100 | 1.0 | 0.8 | 0.5 | 0.9 |
| high NER weight | 60 | 1.0 | 0.8 | 0.5 | 1.2 |

- `compute_retrieval_precision(evidence_list, baseline_experts)` — fraction of baseline experts found in retrieved speakers
- Overrides config in-memory (no file writes) via `_apply_grid_point_to_config()`
- Prints ranked results table sorted by avg_precision descending
- Saves to `backend/benchmark_results/{timestamp}_rrf_sweep.json`

**`backend/tests/unit/test_ner_channel.py`** (new, 18 tests)

Tests: NER channel existence, class definition, lawRefs/personRefs Cypher, perf_counter timing, engine NERChannel import, entity_filter gating, timing breakdown logging, merger ner_results param, ner_weight config, default.yaml ner_weight, compass clustering config (min_fragments_for_kde for Senate Senate group support), RRF sweep existence, syntax, RRF_GRID presence, compute_retrieval_precision function, 7-point grid count, ner weight in grid.

## Deviations from Plan

None - plan executed exactly as written.

## Self-Check

### Files created/modified

- [x] `backend/app/services/retrieval/ner_channel.py` exists
- [x] `backend/app/services/retrieval/engine.py` contains `NERChannel`
- [x] `backend/app/services/retrieval/merger.py` contains `ner_results`
- [x] `backend/config/default.yaml` contains `ner_weight`
- [x] `backend/scripts/rrf_sweep.py` exists
- [x] `backend/tests/unit/test_ner_channel.py` exists (18 tests, all passing)

### Commits

- [x] 82418ae — feat(07-03): NER entity channel with Cypher timing, 4-channel RRF merger
- [x] 48cfe61 — feat(07-03): RRF sweep script and NER channel unit tests

## Self-Check: PASSED
