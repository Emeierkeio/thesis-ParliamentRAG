# Phase 7: Pipeline Optimization — Research

**Researched:** 2026-04-05
**Domain:** RAG pipeline optimization — cost, latency, quality, retrieval architecture, generation architecture, ideological compass
**Confidence:** HIGH (code-based analysis) / MEDIUM (external benchmark claims)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **Optimization philosophy**: Zero preconceptions — current 4-stage pipeline, 3-channel retrieval, and authority scoring are ALL open to replacement. Every architectural choice must justify itself with measurable quality/cost/latency data. Simplicity preferred.
- **LLM cost reduction**: Aggressive — test gpt-4o-mini or gpt-4.1-mini for ALL generation stages. Measure quality delta using evaluation_set.json automated benchmarks. Keep embedding model as text-embedding-3-small (no re-embedding).
- **Generation architecture**: 4-stage pipeline (analyst → sectional writer → integrator → citation surgeon) is open to radical simplification. Evaluate single-prompt, 2-stage, or keeping 4 stages — whatever produces best quality/cost ratio.
- **Latency optimization**: Parallelize where possible AND reduce LLM round-trips. Full end-to-end audit. Profile Neo4j queries, check index usage, optimize Cypher across all channels. Identify serialization bottlenecks.
- **Retrieval architecture**: Systematic weight sweep for RRF parameters. Tune merger scoring weights holistically. Add 4th retrieval channel: dedicated NER entity-aware channel using lawRefs/personRefs.
- **Ideological compass**: Full review of PCA + KDE clustering approach. Evaluate alternatives for political spectrum positioning. Must work with Senate data.
- **Evaluation framework**: Formal benchmark report — automated script comparing before/after on every metric. Metrics: cost per query, end-to-end latency, citation accuracy, section completeness, retrieval precision.
- **Quality baseline**: No major quality issues in current output. Primary concern: quality must not degrade when switching to cheaper models or simpler architecture.

### Claude's Discretion
- None explicitly specified — all major decisions are locked above.

### Deferred Ideas (OUT OF SCOPE)
- None — discussion stayed within phase scope.
</user_constraints>

---

## Summary

The ParliamentRAG pipeline currently uses gpt-4o for 3 of its 4 generation stages at ~$2.50/$10.00 per million tokens input/output. The GPT-4.1-mini model (released April 2025) offers dramatically better price-performance than gpt-4o-mini for this use case: $0.20/$0.80 per million tokens, 1M token context window, and benchmark results showing it outperforms gpt-4o on instruction following (the primary requirement for the sectional writer's complex rule system). This makes gpt-4.1-mini the primary model candidate for cost reduction experiments.

The generation pipeline's 4 stages serve distinct purposes that are not trivially collapsible: the Analyst stage produces structured JSON (claims + party assignments), the Sectional Writer uses aggressive per-section parallelization (asyncio.gather over all parties simultaneously — already fully parallel), the Integrator reorganizes sections into a coherent narrative with government/majority/opposition structure, and the Citation Surgeon is deterministic (no LLM). The key insight is that the Analyst stage's output (claim decomposition) may add less value than its gpt-4o cost suggests — it can be tested as a gpt-4.1-mini or even removed in favor of passing raw evidence directly to the sectional writer.

The retrieval pipeline is already well-optimized for parallelism (all 3 channels run via ThreadPoolExecutor). The primary retrieval improvements are: (1) adding a dedicated NER entity channel using Chunk.lawRefs/Chunk.personRefs stored properties, (2) systematic RRF weight sweep against evaluation_set.json, and (3) Neo4j Cypher query profiling for index utilization. The compass PCA+KDE approach is sound for unsupervised semantic positioning but has known sensitivity to data sparsity; Senate data integration requires testing whether the new parties cluster meaningfully.

**Primary recommendation:** Run a model cost experiment first (gpt-4.1-mini across all 3 LLM stages), measuring against evaluation_set.json ground truth. If quality holds, the cost reduction is ~12x on generation tokens. Architecture simplification (collapsing stages) should be a second experiment, not a simultaneous change.

---

## Standard Stack

### Core (already in use — no changes required)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| openai | 1.x | LLM calls, embeddings | Already in use; supports all model variants transparently |
| neo4j | 5.x | Graph DB queries | Already in use; all retrieval channels |
| numpy | 1.x/2.x | PCA, vector ops | Already in use for compass + semantic dedup |
| scipy | 1.x | KDE, stats | Already in use for compass |

### LLM Model Candidates (NEW — research finding)
| Model | Input $/1M | Output $/1M | Context | Key Capability |
|-------|-----------|------------|---------|----------------|
| gpt-4o (current) | $2.50 | $10.00 | 128K | Current baseline |
| gpt-4o-mini | $0.15 | $0.60 | 128K | Cheapest; lower instruction-following |
| gpt-4.1-mini | $0.20 | $0.80 | 1M | Outperforms gpt-4o on IFEval (87.4% vs 81%); MultiChallenge (35.8%); best candidate |
| gpt-4.1 | $2.00 | $8.00 | 1M | Near gpt-4o cost; larger context only benefit |
| gpt-4.1-nano | $0.05 | $0.20 | 1M | Too cheap; likely insufficient for complex parliamentary structure |

**Verification:** Pricing confirmed from pricepertoken.com/pricing-page/provider/openai (per-provider authoritative aggregator). GPT-4.1-mini outperforms gpt-4o on instruction following benchmarks — confirmed from OpenAI's own release post and DataCamp analysis. This is MEDIUM confidence because parliamentary-specific quality has not been tested.

### Evaluation Libraries (new for benchmark script)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| pytest | 8.x | Test framework | Benchmark script structure |
| time / asyncio | stdlib | Latency profiling | End-to-end timing |
| json | stdlib | Loading evaluation_set.json | Ground truth comparison |

**Installation for new evaluation tooling:**
```bash
# No new dependencies needed — evaluation script uses stdlib + existing openai client
# If adopting RAGAS for automated quality evaluation:
pip install ragas
```

---

## Architecture Patterns

### Current Pipeline Flow (audit baseline)

```
User Query
  │
  ├── [PARALLEL] QueryRewriter (gpt-4o-mini, short queries only)
  │
  ├── embed_query (text-embedding-3-small)
  │
  ├── [PARALLEL via ThreadPoolExecutor] Retrieval
  │     ├── DenseChannel (Neo4j vector index, top_k*2=200)
  │     ├── SparseChannel (Neo4j BM25 fulltext, top_k=100)
  │     └── GraphChannel (EuroVoc matching + graph traversal)
  │
  ├── ChannelMerger (RRF: k=60, dense=1.0, sparse=0.8, graph=0.5)
  │     └── _select_diverse (greedy: max 10/speaker, max 33/party)
  │
  ├── _coverage_fill (vector search for missing parties)
  ├── _expand_neighbors (salience-based chunk expansion)
  │
  ├── [SERIAL] AuthorityScorer (2 DB queries + parallel CPU scoring)
  │
  ├── compute_experts (top expert per party for frontend)
  │
  ├── CompassPipeline (IC-1..IC-6: weighted PCA + KDE + TF-IDF)
  │
  └── [SERIAL → PARALLEL] GenerationPipeline
        ├── Stage 1: ClaimAnalyst (gpt-4o, ~2000 tokens out, JSON)
        ├── Stage 2: SectionalWriter (gpt-4o × N_parties IN PARALLEL via asyncio.gather)
        ├── Stage 3: NarrativeIntegrator (gpt-4o, SERIAL — waits for all sections)
        └── Stage 4: CitationSurgeon (deterministic, no LLM)
```

### Key Parallelism Already in Place

- Retrieval: 3 channels via ThreadPoolExecutor (fully parallel)
- Sectional writing: all party sections via asyncio.gather (fully parallel)
- Authority scoring: 2 DB queries then parallel CPU scoring

### Serialization Bottlenecks (confirmed from code)

1. **Analyst → Sectional Writer → Integrator** is sequential. Analyst must complete before Sectional starts; Integrator must wait for ALL sections. This is inherent to the design — Analyst produces claims that SectionalWriter consumes. **Reducing LLM round-trips here means eliminating the Analyst stage**, not parallelizing it.

2. **Query embedding computed twice**: once in `embed_query()` for retrieval, once again in `query.py` for authority scoring (`query_embedding = await run_in_executor(None, lambda: services["retrieval"].embed_query(request.query))`). The retrieval embedding is not reused. **Fix: cache/reuse the first embedding.**

3. **Authority scoring is serial with respect to compass**: compass runs after authority completes. These are logically independent — compass uses evidence embeddings, authority uses speaker metadata. **Opportunity: parallelize authority + compass.**

4. **Coverage fill + neighbor expansion** run serially inside retrieve_sync() after RRF. These add Neo4j round-trips. Coverage fill makes a separate vector query per missing party.

### Pattern 1: Model A/B Benchmark Script

The benchmark script must compare models against the same evaluation_set.json topics.

```python
# Source: project-internal pattern, confirmed from evaluation_set.json structure
import json, time, asyncio

TOPICS = json.load(open("backend/evaluation_set.json"))

async def benchmark_topic(topic_name: str, query: str, model_config: dict):
    """Run pipeline with given model config, return metrics."""
    start = time.perf_counter()
    # Override generation.models.* in config
    result = await pipeline.generate(query, evidence_list, model_config=model_config)
    latency = time.perf_counter() - start

    return {
        "topic": topic_name,
        "latency_s": latency,
        "cost_estimate_usd": estimate_cost(result["token_usage"], model_config),
        "citation_count": len(result["citations"]),
        "parties_covered": count_parties_with_citations(result["text"]),
        "section_completeness": check_all_parties_present(result["text"]),
    }
```

### Pattern 2: RRF Weight Sweep

```python
# Sweep k and channel weights, score against evaluation_set ground truth
RRF_GRID = [
    {"k": 60, "dense": 1.0, "sparse": 0.8, "graph": 0.5},   # current
    {"k": 60, "dense": 1.0, "sparse": 0.5, "graph": 0.8},   # graph-boosted
    {"k": 60, "dense": 1.0, "sparse": 1.0, "graph": 0.5},   # equal sparse
    {"k": 30, "dense": 1.0, "sparse": 0.8, "graph": 0.5},   # lower k
    {"k": 100, "dense": 1.0, "sparse": 0.8, "graph": 0.5},  # higher k
    {"k": 60, "dense": 1.0, "sparse": 0.8, "graph": 0.5, "ner": 0.9},  # with NER channel
]
# Metric: retrieval precision = fraction of baseline_experts found in top retrieved speakers
```

### Pattern 3: NER Entity Channel

The 4th channel is a targeted BM25/exact-match lookup on Chunk.lawRefs and Chunk.personRefs.

```cypher
// NER entity channel: find chunks mentioning specific law references
CALL db.index.fulltext.queryNodes("chunk_fulltext", $query_escaped)
YIELD node AS c, score
WHERE ANY(ref IN c.lawRefs WHERE ref CONTAINS $law_keyword)
   OR ANY(ref IN c.personRefs WHERE ref CONTAINS $person_keyword)
MATCH (c)<-[:HAS_CHUNK]-(i:Speech)-[:SPOKEN_BY]->(speaker)
// ... rest of standard metadata joins
RETURN c.id AS chunk_id, score AS bm25_score, ...
```

The lawRefs/personRefs are stored as arrays on Chunk nodes (from Phase 4 NER enrichment). A dedicated NER channel pre-filters candidates by entity match before scoring, giving precision that neither dense nor sparse capture for entity-specific queries (e.g., "Decreto Flussi", "Meloni").

### Pattern 4: Compass with Senate Groups

Senate parliamentary groups differ from Camera groups. The compass PCA uses `group_id` as the clustering unit. Senate groups must be mapped correctly in the `Fragment.group_id` field. The anchor configuration in `default.yaml` (compass.anchors) currently lists Camera groups only — Senate groups need separate anchor entries or the anchors feature must remain disabled.

The PCA+KDE approach works for any set of groups as long as there are sufficient fragments per group (min_fragments_for_kde=3). Senate groups with few speeches may fall back to mean positioning, which is acceptable.

### Anti-Patterns to Avoid

- **Simultaneous model + architecture change**: Don't collapse stages AND swap models in the same experiment. Isolate variables.
- **Removing the Citation Surgeon**: Stage 4 is deterministic and adds citation verification at zero LLM cost. Keep it.
- **Removing the Integrator without testing**: The Integrator enforces the government/majority/opposition structure. Removing it requires the sectional writer to output in final format directly — doable but quality risk.
- **Raising RRF k beyond 100**: k=60 is empirically well-validated across datasets. Sweeping to 100 is reasonable; beyond that, consensus effects diminish.
- **Benchmarking only on the evaluation_set topics**: The 7-topic evaluation_set is narrow. Use it for regression, not as sole quality truth.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Token cost estimation | Custom token counter | `tiktoken` or OpenAI response `.usage` fields | Already returned in API response; no extra calls |
| Citation quality scoring | Custom overlap metric | Substring check already in CitationSurgeon (verified verbatim match) | Pipeline already has definitive citation verification |
| Model capability testing | Long manual evals | A/B run on evaluation_set.json with automated citation_count + section_completeness metrics | Ground truth already exists |
| Political spectrum labeling | Train a custom classifier | TF-IDF pole labeling (IC-6, already implemented) + CHES framework reference | CHES provides the left-right / GAL-TAN framework used in political science; no training required |
| RRF implementation | Custom fusion algorithm | Existing `ChannelMerger._compute_rrf()` is correct and parameterizable | Already implemented; just tune `k` and channel weights via `default.yaml` |

**Key insight:** The evaluation infrastructure (evaluation_set.json + evaluation_service.py) already exists. The benchmark script for this phase should instrument the existing pipeline, not create a parallel evaluation system.

---

## Common Pitfalls

### Pitfall 1: Query Embedding Double-Computation
**What goes wrong:** `query.py` calls `embed_query()` twice — once inside `retrieval.retrieve()` and once explicitly for authority scoring. This wastes ~300ms and ~$0.0002 per query.
**Why it happens:** The retrieval result does not return the computed embedding; authority scoring needs it separately.
**How to avoid:** Return `query_embedding` from `retrieval.retrieve()` in the result dict. Use it for authority scoring instead of re-computing.
**Warning signs:** `embed_query` appears more than once in the call trace for a single request.

### Pitfall 2: gpt-4.1-mini "Literal" Interpretation
**What goes wrong:** GPT-4.1 models are documented as "more literal" than gpt-4o. The sectional writer's SYSTEM_PROMPT has 30+ rules with examples. If gpt-4.1-mini interprets constraints too literally (e.g., "ESATTAMENTE" for verbatim quotes), it may refuse valid partial matches.
**Why it happens:** gpt-4.1-mini scores 87.4% on IFEval (instruction following) vs 81% for gpt-4o — it follows instructions more strictly, which can cut both ways.
**How to avoid:** Run benchmark on evaluation_set.json before deploying. Pay attention to citation extraction rate and section completeness metrics.
**Warning signs:** Sections with `has_evidence: False` increase; citation_count drops vs baseline.

### Pitfall 3: Integrator Citation Loss
**What goes wrong:** The Integrator (Stage 3) already has a guard (`integrate_with_guard`) and a party-paragraph injection fallback because the LLM drops citations. If the Integrator is replaced with a cheaper model, citation loss rate may increase.
**Why it happens:** The Integrator must preserve `[CIT:id]` markers through a complex restructuring operation. This is the highest-risk stage for model downgrades.
**How to avoid:** The citation_registry tracks all expected citations. Monitor `citations_repaired` in pipeline_metadata after model change. Set threshold: if repair rate > 10%, revert model.
**Warning signs:** `citations_repaired` counter increases in pipeline metadata.

### Pitfall 4: NER Channel Double-Counting
**What goes wrong:** If a chunk matches both the BM25 sparse channel AND the NER entity channel, and both are fed to RRF, the chunk gets double-counted unfairly over non-NER chunks.
**Why it happens:** RRF accumulates scores per evidence_id across channels. Same chunk in 4 channels = higher RRF score.
**How to avoid:** Design the NER channel to be additive: only include it when the query contains entity patterns (law references or person names). The entity detection already exists in `engine.py` (`law_matches = re.findall(...)`). Gate the NER channel on this detection.

### Pitfall 5: Compass Instability with Senate Groups
**What goes wrong:** Senate groups may not have enough fragments (speeches) to produce stable KDE peaks. Groups with <3 fragments fall back to mean positioning, which may cluster unrelated parties together.
**Why it happens:** Senate debates have different structure — some groups may have few representatives in the retrieved top-100 evidence pool.
**How to avoid:** Test compass on queries that include Senate chamber. Check `is_stable` flag and `n_fragments` per group in compass output. Consider raising `scatter_sample_size` for Senate-heavy queries.

### Pitfall 6: RRF Weight Sweep Without Held-Out Evaluation Set
**What goes wrong:** Overfitting RRF weights to the 7 evaluation_set.json topics, then discovering weights generalize poorly to real queries.
**Why it happens:** 7 topics is a very small sample for tuning retrieval parameters.
**How to avoid:** Use the evaluation set only for regression (does it degrade?), not optimization. Manual spot-checks on diverse real queries are required alongside automated metrics.

---

## Code Examples

### Current Generation LLM Call Pattern (analyst.py)
```python
# Source: backend/app/services/generation/analyst.py
response = self.client.chat.completions.create(
    model=self.model,  # configurable via generation.models.analyst in default.yaml
    messages=[
        {"role": "system", "content": self.SYSTEM_PROMPT},
        {"role": "user", "content": user_prompt}
    ],
    temperature=0.3,
    max_tokens=2000,
    response_format={"type": "json_object"}
)
```

Model swap is purely a config change in `default.yaml`:
```yaml
generation:
  models:
    analyst: "gpt-4.1-mini"   # was "gpt-4o"
    writer: "gpt-4.1-mini"    # was "gpt-4o"
    integrator: "gpt-4.1-mini" # was "gpt-4o"
```

### Current RRF Configuration (default.yaml)
```yaml
retrieval:
  rrf:
    k: 60              # Standard default; sweep 30/60/100
    dense_weight: 1.0  # Primary signal
    sparse_weight: 0.8
    graph_weight: 0.5  # Lowest — structural signal
```

### Query Embedding Reuse Fix Pattern
```python
# Source: backend/app/routers/query.py (current inefficient pattern)
# BEFORE:
retrieval_result = await services["retrieval"].retrieve(query=request.query, ...)
# Later:
query_embedding = await run_in_executor(None, lambda: services["retrieval"].embed_query(request.query))  # SECOND call

# AFTER: Return embedding from retrieval
retrieval_result = await services["retrieval"].retrieve(query=request.query, ...)
query_embedding = retrieval_result["query_embedding"]  # Reuse from first call
```

### Parallelizing Authority + Compass
```python
# Source: pattern derived from existing asyncio.gather usage in sectional.py
# Currently sequential — compass runs after authority completes
# PROPOSED: parallel execution
authority_task = asyncio.get_running_loop().run_in_executor(
    None, lambda: services["authority"].compute_all_authority(speaker_ids, query_embedding)
)
compass_task = asyncio.get_running_loop().run_in_executor(
    None, services["ideology"].compute_2d_text_positions, evidence_dicts
)
authority_all, compass_result = await asyncio.gather(authority_task, compass_task)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Weighted similarity fusion | RRF (rank-based) | Phase 4 | Cross-channel comparable scores; no scale normalization needed |
| Sequential party section writing | asyncio.gather parallel | Pre-existing | 10+ party sections written simultaneously |
| Single embedding model choice | text-embedding-3-small frozen | Phase 1 decision | No re-embedding; stable cache keys |
| gpt-4o for all LLM stages | Still gpt-4o (optimization target) | Phase 7 goal | ~12x cost reduction if gpt-4.1-mini matches quality |

**Deprecated/outdated patterns in scope:**
- **Weighted similarity fusion in merger**: Already replaced by RRF in Phase 4. The `authority_weight`, `relevance_weight`, etc. in `merger` config section of `default.yaml` appear as dead config — `_compute_rrf()` reads only `rrf.*` keys. These dead config keys should be removed or the comment clarified.
- **GPT-4o for generation**: Superseded by gpt-4.1-mini for instruction-following tasks at 8x lower cost.

---

## Open Questions

1. **Can the Analyst stage be eliminated?**
   - What we know: Analyst decomposes query into atomic claims with party assignments. SectionalWriter receives these claims and uses them to select relevant evidence. However, the prompt already includes the full evidence grouped by party, and the LLM could select relevant evidence directly without pre-claimed decomposition.
   - What's unclear: Whether the quality of party sections degrades without claim guidance. The analyst adds ~1 gpt-4o call (~500 input + ~2000 output tokens).
   - Recommendation: A/B test: run SectionalWriter with and without analyst claims on evaluation_set.json topics. Measure citation accuracy and section completeness.

2. **Can the Integrator be merged with the SectionalWriter?**
   - What we know: The Integrator restructures 10+ party sections into government/majority/opposition structure and adds an introduction. This is substantial formatting logic.
   - What's unclear: Whether a single combined prompt could produce correctly structured output without a separate integration pass. The current Integrator system prompt is 130+ lines with strict rules.
   - Recommendation: Attempt a 2-stage design (combined analyst+sectional → integrator) only after model experiments confirm quality holds with cheaper models.

3. **Will gpt-4.1-mini reliably extract verbatim Italian citations?**
   - What we know: GPT-4.1-mini outperforms gpt-4o on instruction following (IFEval 87.4% vs 81%). Italian verbatim citation is the most critical rule: "La frase tra «» DEVE apparire esattamente nel TESTO DISPONIBILE, parola per parola."
   - What's unclear: Whether the "more literal" interpretation of gpt-4.1-mini helps (follows verbatim rule more strictly) or hurts (rejects valid partial matches).
   - Recommendation: Monitor CitationSurgeon failure rate in benchmark. If verbatim verification failures increase, the model is paraphrasing.

4. **What is the actual end-to-end latency breakdown?**
   - What we know: Retrieval runs in parallel (ThreadPoolExecutor). Sections run in parallel (asyncio.gather). Authority and compass are sequential.
   - What's unclear: Actual millisecond breakdown of each stage. embed_query, authority DB queries, and compass PCA all happen in sequence before generation.
   - Recommendation: Add time.perf_counter() instrumentation at every major boundary in query.py and pipeline.py in Wave 0 of this phase. Profile before optimizing.

5. **Do Senate parliamentary groups cluster meaningfully in the compass?**
   - What we know: Senate groups have `chamber: "senato"` on all nodes. The compass uses chunk embeddings aggregated by `group_id`. Senate groups are different from Camera groups.
   - What's unclear: Whether Senate debate text is semantically diverse enough to separate groups in PCA space, or whether all Senate parties on a given topic occupy the same semantic region.
   - Recommendation: Run compass on a Chamber=senato query and inspect `n_fragments` per group and `is_stable` flag.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest (already in use from Phase 2) |
| Config file | `backend/tests/` directory |
| Quick run command | `pytest backend/tests/ -x -q` |
| Full suite command | `pytest backend/tests/ -v` |

### Phase Requirements → Test Map

This phase has no formal requirement IDs (TBD per scope). Behavioral requirements derived from CONTEXT.md decisions:

| Behavior | Test Type | Automated Command | Notes |
|----------|-----------|-------------------|-------|
| Benchmark script runs without error | smoke | `pytest backend/tests/test_benchmark.py -x` | Wave 0 gap |
| gpt-4.1-mini produces valid section JSON | unit | `pytest backend/tests/test_generation.py::test_analyst_mini -x` | Wave 0 gap |
| Citation count with new model >= baseline - threshold | integration | `pytest backend/tests/test_benchmark.py::test_citation_regression -x` | Wave 0 gap |
| RRF weight sweep produces ranked metric table | smoke | `pytest backend/tests/test_retrieval_sweep.py -x` | Wave 0 gap |
| NER channel returns results for entity queries | unit | `pytest backend/tests/test_ner_channel.py -x` | Wave 0 gap |
| Query embedding reused (called once per request) | unit | `pytest backend/tests/test_query_embedding_reuse.py -x` | Wave 0 gap |
| Compass works with Senate-only evidence | unit | `pytest backend/tests/test_compass_senate.py -x` | Wave 0 gap |

### Wave 0 Gaps
- [ ] `backend/tests/test_benchmark.py` — benchmark harness using evaluation_set.json
- [ ] `backend/tests/test_generation.py` — model swap smoke test (analyst, writer, integrator)
- [ ] `backend/tests/test_retrieval_sweep.py` — RRF parameter grid search
- [ ] `backend/tests/test_ner_channel.py` — NER entity channel unit tests
- [ ] `backend/tests/test_query_embedding_reuse.py` — verify embed_query called once

---

## Sources

### Primary (HIGH confidence — code inspection)
- `/backend/app/services/generation/pipeline.py` — 4-stage orchestrator, citation integrity system
- `/backend/app/services/generation/analyst.py` — Stage 1 prompts, LLM call pattern
- `/backend/app/services/generation/sectional.py` — Stage 2 parallel execution (asyncio.gather confirmed)
- `/backend/app/services/generation/integrator.py` — Stage 3 system prompt, citation guard
- `/backend/app/services/retrieval/engine.py` — ThreadPoolExecutor parallelization, double embed_query confirmed
- `/backend/app/services/retrieval/merger.py` — RRF implementation, dead config keys
- `/backend/app/services/compass/pipeline.py` — IC-1..IC-6 full implementation
- `/backend/app/routers/query.py` — Full pipeline orchestration, serialization bottlenecks
- `/backend/config/default.yaml` — All configurable parameters
- `/backend/evaluation_set.json` — 7 ground truth topics with baseline_experts

### Secondary (MEDIUM confidence — verified with official/authoritative sources)
- [pricepertoken.com — OpenAI pricing](https://pricepertoken.com/pricing-page/provider/openai): gpt-4o $2.50/$10.00, gpt-4.1-mini $0.20/$0.80, gpt-4o-mini $0.15/$0.60 per million tokens (cross-referenced with OpenRouter)
- [OpenAI GPT-4.1 announcement](https://openai.com/index/gpt-4-1/): gpt-4.1-mini outperforms gpt-4o on IFEval (87.4% vs 81%), MultiChallenge (35.8% vs 27.8%)
- [DataCamp GPT-4.1 comparison](https://www.datacamp.com/blog/gpt-4-1): "All models of the GPT-4.1 family outperform GPT-4o across the board, with major gains in coding and instruction following"
- [RRF k=60 empirical baseline](https://opensearch.org/blog/introducing-reciprocal-rank-fusion-hybrid-search/): k=60 is the standard empirically-validated default; confirmed by Elastic Labs

### Tertiary (LOW confidence — single source, not verified)
- RAGAS citation accuracy claim "65-70% without explicit attribution training" — from a WebSearch summary, not directly verified against source paper. Flagged for validation.
- gpt-4.1-mini "more literal" interpretation — from community reports, not OpenAI official docs. Treat as hypothesis to verify in benchmarks.

---

## Metadata

**Confidence breakdown:**
- Standard stack (model pricing): MEDIUM — pricing from aggregators, not OpenAI API page directly (403)
- Architecture analysis: HIGH — direct code inspection of all canonical files
- Parallelism opportunities: HIGH — confirmed from code (double embed_query, sequential authority+compass)
- Model capability claims (gpt-4.1-mini vs gpt-4o): MEDIUM — from official announcement + reputable analysis, but not tested on this specific domain
- Compass Senate concerns: MEDIUM — architectural analysis; actual clustering quality requires empirical testing
- Pitfalls: HIGH — derived from code reading + known failure modes already in the codebase (citation repair, coherence validation)

**Research date:** 2026-04-05
**Valid until:** 2026-07-05 (90 days — model pricing is stable quarter-to-quarter; recheck if new OpenAI models release)
