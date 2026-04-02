---
phase: 04-enrichment
plan: "01"
subsystem: retrieval
tags: [bm25, sparse-retrieval, rrf, merger, neo4j-fulltext]
dependency_graph:
  requires: []
  provides: [SparseChannel, RRF-merger, fulltext-index]
  affects: [engine.py, merger.py, db_builder.py, build_and_update.py]
tech_stack:
  added: [Neo4j fulltext index (Lucene), Reciprocal Rank Fusion]
  patterns: [TDD red-green, parallel ThreadPoolExecutor, graceful degradation]
key_files:
  created:
    - backend/app/services/retrieval/sparse_channel.py
    - backend/tests/unit/test_sparse_channel.py
    - backend/tests/unit/test_rrf_merger.py
  modified:
    - backend/app/services/retrieval/merger.py
    - backend/app/services/retrieval/engine.py
    - backend/config/default.yaml
    - build/db_builder.py
    - build/build_and_update.py
decisions:
  - "RRF replaces weighted scoring in merger: rank-based fusion is parameter-free and cross-channel comparable"
  - "similarity=0.5 sentinel for sparse results: BM25 raw scores are not comparable to cosine similarity; rank position drives fusion"
  - "_select_diverse preserved unchanged: it operates on final_score which is now rrf_score"
  - "Italian analyzer with standard fallback in create_fulltext_index: Neo4j Community may not have italian analyzer"
metrics:
  duration: "4 min"
  completed_date: "2026-04-02"
  tasks: 2
  files_changed: 8
---

# Phase 4 Plan 1: BM25 Sparse Channel + RRF Merger Summary

BM25 sparse retrieval via Neo4j Lucene full-text index with Reciprocal Rank Fusion merging dense + sparse + graph channels into a unified 3-channel retrieval engine.

## What Was Built

### SparseChannel (`backend/app/services/retrieval/sparse_channel.py`)

New retrieval channel that queries Neo4j's `chunk_fulltext` full-text index (Lucene with Italian analyzer). Key design:
- `retrieve(query_text, top_k)` returns evidence dicts identical in structure to DenseChannel
- `similarity` is set to `0.5` (neutral sentinel) for all results — raw BM25 scores are NOT used for ranking
- `bm25_score` field preserves the original Lucene score for diagnostics
- Graceful degradation: wraps the query in try/except; if index doesn't exist yet returns `[]`
- `retrieval_channel = "sparse"` tag on every result

### RRF Merger (`backend/app/services/retrieval/merger.py`)

Replaced the old weighted-score merger with Reciprocal Rank Fusion:

```
rrf_score = sum(weight / (k + rank))  for each channel the item appears in
```

- Default: k=60, dense_weight=1.0, sparse_weight=0.8, graph_weight=0.5
- `merge()` signature extended: `merge(dense, sparse, graph, authority_scores, top_k)`
- Deduplication: same `evidence_id` from multiple channels gets their contributions summed
- `_select_diverse()` preserved unchanged — operates on `final_score` (now = `rrf_score`)
- Empty channels handled gracefully (no crash)

### Config (`backend/config/default.yaml`)

Added two new sections under `retrieval:`:
```yaml
sparse_channel:
  top_k: 100
  index_name: "chunk_fulltext"

rrf:
  k: 60
  dense_weight: 1.0
  sparse_weight: 0.8
  graph_weight: 0.5
```

### RetrievalEngine (`backend/app/services/retrieval/engine.py`)

- Imports and instantiates `SparseChannel`
- `retrieve_sync()` now runs 3 channels in parallel: `max_workers=3`
- Passes `sparse_results` to `merger.merge()`
- Updated logging from "dual-channel" to "triple-channel"

### Build Pipeline (`build/db_builder.py`, `build/build_and_update.py`)

- `DatabaseBuilder.create_fulltext_index()`: Creates `chunk_fulltext` index with Italian analyzer, falls back to standard analyzer on failure
- `do_build()`: calls `create_fulltext_index()` as step 8b (after vector index)
- `do_update()`: calls `create_fulltext_index()` as step 5b (idempotent — `IF NOT EXISTS`)

### Unit Tests (16 tests, all passing)

- `test_sparse_channel.py`: 6 tests — evidence structure, similarity=0.5, bm25_score, channel tag, empty result, graceful fallback
- `test_rrf_merger.py`: 10 tests — 3-channel signature, RRF formula correctness, deduplication, empty channels, diversity selection

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Test used identical speaker_id for rank comparison test**
- **Found during:** Task 1 GREEN phase
- **Issue:** `test_rrf_merger_higher_rank_yields_higher_score` used `speaker_id="spk1"` for both results — diversity selection (max_per_speaker=1 at top_k=10) filtered the second result, making `low_rank` not appear in output
- **Fix:** Changed to distinct speaker IDs (`spk_a`, `spk_b`) and increased top_k=100 to avoid diversity cutoff
- **Files modified:** `backend/tests/unit/test_rrf_merger.py`
- **Commit:** 7deb4fd

## Self-Check: PASSED
