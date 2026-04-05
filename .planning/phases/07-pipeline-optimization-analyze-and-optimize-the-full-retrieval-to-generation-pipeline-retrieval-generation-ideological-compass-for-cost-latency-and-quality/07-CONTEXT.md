# Phase 7: Pipeline Optimization - Context

**Gathered:** 2026-04-05
**Status:** Ready for planning

<domain>
## Phase Boundary

Full architectural review and optimization of the entire retrieval-to-generation pipeline, including the ideological compass. This is NOT incremental tuning — the current architecture is open to radical change if analysis shows better alternatives exist. Every component must justify its existence with measurable value.

Scope includes integrating new capabilities already built (multi-language, Senate data, NER entities) into the pipeline in the optimal way.

</domain>

<decisions>
## Implementation Decisions

### Optimization philosophy
- Zero preconceptions: the current 4-stage generation pipeline, 3-channel retrieval, and authority scoring architecture are ALL open to replacement
- Every architectural choice must justify itself with measurable quality/cost/latency data
- Simplicity preferred: if a single well-engineered prompt matches the quality of 4 stages at lower cost, switch
- New capabilities (multi-lingua, Senate, NER) must be integrated optimally, not bolted on

### LLM cost reduction
- Aggressive approach: test gpt-4o-mini or gpt-4.1-mini for ALL generation stages (analyst, writer, integrator)
- Measure quality delta using evaluation_set.json automated benchmarks
- Keep embedding model as text-embedding-3-small (no re-embedding)
- Also audit and trim prompt sizes — reduce token usage across all system prompts

### Generation architecture
- The 4-stage pipeline (analyst → sectional writer → integrator → citation surgeon) is open to radical simplification
- Evaluate: single-prompt approach, 2-stage approach, or keeping 4 stages — whatever produces best quality/cost ratio
- Each stage must prove measurable value-add vs simpler alternatives
- Current pipeline: ~4.9k lines across 12 modules — complexity must justify itself

### Latency optimization
- Parallelize where possible AND reduce LLM round-trips
- Full end-to-end latency audit: retrieval + generation + authority scoring + compass
- Profile Neo4j queries, check index usage, optimize Cypher across all 3 retrieval channels
- Identify serialization bottlenecks (e.g., sectional writer processes parties sequentially)

### Retrieval architecture
- Systematic weight sweep: test multiple RRF weight combinations (dense/sparse/graph) against evaluation_set
- Tune merger scoring weights (diversity, coverage, authority, relevance, salience) holistically with RRF weights
- Add 4th retrieval channel: dedicated NER entity-aware channel using lawRefs/personRefs for entity-specific queries
- Evaluate if current 3-channel RRF is optimal or if alternative fusion strategies exist

### Ideological compass
- Full review of PCA + KDE clustering approach
- Evaluate if the weighted PCA / z-score / pole labeling pipeline is the best method
- Consider alternative approaches for political spectrum positioning
- Must work correctly with Senate data (new groups, different dynamics)

### Evaluation framework
- Formal benchmark report: automated script comparing before/after on every metric
- Metrics: cost per query, end-to-end latency, citation accuracy, section completeness, retrieval precision
- Use evaluation_set.json as ground truth for regression testing
- Every architectural change must have a benchmark comparison

### Quality baseline
- No major quality issues identified in current output
- Primary concern: ensuring quality doesn't degrade when switching to cheaper models or simpler architecture
- Automated benchmarks are the safety net for all changes

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Pipeline configuration
- `backend/config/default.yaml` — All pipeline weights, thresholds, model selections, RRF parameters, authority weights, compass parameters

### Generation pipeline
- `backend/app/services/generation/pipeline.py` — 4-stage orchestrator (1053 lines)
- `backend/app/services/generation/analyst.py` — Stage 1: claim decomposition (183 lines)
- `backend/app/services/generation/sectional.py` — Stage 2: per-party section writing (899 lines)
- `backend/app/services/generation/integrator.py` — Stage 3: narrative integration (551 lines)
- `backend/app/services/generation/surgeon.py` — Stage 4: citation extraction (654 lines)
- `backend/app/services/generation/synthesis.py` — Convergence/divergence analysis (122 lines)
- `backend/app/services/generation/coherence_validator.py` — Citation semantic validation (476 lines)
- `backend/app/services/generation/citation_registry.py` — Citation tracking through pipeline (306 lines)

### Retrieval pipeline
- `backend/app/services/retrieval/engine.py` — Main retrieval orchestrator (486 lines)
- `backend/app/services/retrieval/dense_channel.py` — Vector similarity search (207 lines)
- `backend/app/services/retrieval/sparse_channel.py` — BM25 full-text search (234 lines)
- `backend/app/services/retrieval/graph_channel.py` — Graph-based retrieval (510 lines)
- `backend/app/services/retrieval/merger.py` — RRF channel fusion + scoring (202 lines)
- `backend/app/services/retrieval/query_rewriter.py` — Query expansion via gpt-4o-mini (80 lines)

### Authority scoring
- `backend/app/services/authority/scorer.py` — Main authority scorer (705 lines)
- `backend/app/services/authority/components.py` — 6 scoring components (745 lines)
- `backend/app/services/authority/coalition_logic.py` — Coalition-based logic (279 lines)

### Query endpoints
- `backend/app/routers/query.py` — SSE streaming endpoint, pipeline orchestration
- `backend/app/routers/chat.py` — Chat endpoint with history

### Evaluation
- `backend/evaluation_set.json` — Ground truth topics with baseline answers and experts
- `backend/app/routers/evaluation.py` — Dashboard metrics endpoint

</canonical_refs>

<code_context>
## Existing Code Insights

### Pipeline size and structure
- Generation: 12 modules, ~4.9k lines — 4-stage pipeline with citation integrity system
- Retrieval: 7 modules, ~1.7k lines — 3-channel (dense, sparse, graph) + RRF merger
- Authority: 3 modules, ~1.7k lines — 6-component scoring with time decay
- Total pipeline: ~8.6k lines

### LLM usage
- 3x gpt-4o calls per query: analyst, sectional writer, integrator
- 1x gpt-4o-mini call: query rewriting (short queries only)
- Embedding: text-embedding-3-small for query embedding + coherence validation
- All LLM clients use `key_pool.make_client()` / `make_async_client()`

### Configuration
- All weights and thresholds in `backend/config/default.yaml` — single source of truth
- Models configurable per stage via `generation.models.*`
- RRF weights configurable via `retrieval.rrf.*`
- Merger weights configurable via `retrieval.merger.*`

### Integration points
- NER entities (lawRefs, personRefs) stored on Chunk nodes — currently used only in graph_channel
- Senate data has `chamber` property on all nodes — retrieval channels already filter by chamber
- Translation service exists for citations — called post-generation in query.py and chat.py
- Evaluation set provides ground truth for automated benchmarking

### Established patterns
- SSE streaming for progressive response delivery
- ConfigManager for YAML-based configuration
- FastAPI dependency injection for services
- Async generation pipeline with sync retrieval

</code_context>

<specifics>
## Specific Ideas

- "Non voglio preconcetti — l'architettura attuale potrebbe essere errata"
- Evaluate whether a single well-crafted prompt can replace the entire 4-stage pipeline
- NER entities should have their own dedicated retrieval channel, not just be a graph_channel boost
- All new capabilities (multi-lingua, Senate, NER) must be integrated into the pipeline natively, not as afterthoughts
- Benchmark everything: no change without data proving it's better

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 07-pipeline-optimization*
*Context gathered: 2026-04-05*
