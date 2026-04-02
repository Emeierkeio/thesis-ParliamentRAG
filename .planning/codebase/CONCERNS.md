# Codebase Concerns

**Analysis Date:** 2026-04-01

## Tech Debt

**Complex SSE streaming state management:**
- Issue: Multiple overlapping streaming mechanisms (chat.py and query.py) with complex queue/semaphore coordination for pipeline concurrency limiting
- Files: `backend/app/routers/chat.py` (1792 lines), `backend/app/routers/query.py` (1061 lines)
- Impact: Hard to reason about race conditions; global state (`_pipeline_active`, `_waiting_queue`, `_pipeline_semaphore`) managed via locks; semaphore + queue-position tracking increases complexity
- Fix approach: Extract pipeline coordination into dedicated TaskCoordinator service; consolidate queue-based backpressure into a single authoritative state machine

**Authority scoring computation duplicated across multiple contexts:**
- Issue: Authority scores computed in `_compute_experts` (query.py), `_compute_baseline_authority_from_precomputed` (evaluation.py), `_compute_automated_metrics` (evaluation.py); multiple fallback chains (`expert_full_lookup` → `party_top_expert`)
- Files: `backend/app/routers/query.py:_compute_experts`, `backend/app/routers/evaluation.py:_compute_baseline_authority_*`, `backend/app/routers/history.py:_compute_experts_from_matched`
- Impact: Score computation logic scattered; hard to ensure consistency across endpoints; fallback chains fragile (if cached data missing, scores diverge)
- Fix approach: Centralize authority scoring into single AuthorityComputer service with explicit cache management and no fallback chains

**Expert correction patch runs post-generation (non-critical failure path):**
- Issue: `_patch_experts_for_cited_speakers` in chat.py line 1050 is wrapped in try-except that silently logs warnings; if patch fails, survey panel may show mismatched experts
- Files: `backend/app/routers/chat.py:1066-1067`
- Impact: Inconsistency between displayed experts and actual citations; not enforced or alerted; silent degradation
- Fix approach: Make expert matching a pre-generation step; fail early if cited speakers cannot be resolved

**Evaluation metrics calculation depends on fragile text matching:**
- Issue: `_compute_baseline_authority_for_text` uses regex with 4-character surname threshold and manual name matching (line 271 in evaluation.py)
- Files: `backend/app/routers/evaluation.py:252-275`
- Impact: Surname-only matches in Italian parliamentary text (e.g., "onorevole Rossi") may miss short surnames or match false positives; metrics inflated/deflated
- Fix approach: Use entity extraction (spaCy/transformer) instead of regex; build surname index at evaluation startup

**Large monolithic components in frontend:**
- Issue: SurveyModal.tsx (2325 lines) and evaluation page (1371 lines) handle multiple concerns: form state, data fetching, UI rendering, citation review
- Files: `frontend/src/components/survey/SurveyModal.tsx`, `frontend/src/app/valutazione/page.tsx`
- Impact: Hard to test; refactoring risky; state management scattered across useState hooks; citation step tightly coupled to survey modal
- Fix approach: Extract citation review into independent component; split SurveyModal into context/form/presenter layers

## Known Bugs

**Expert authority scores may be stale across query boundaries:**
- Symptoms: Baseline experts shown with different authority scores than system experts for same deputy
- Files: `backend/app/routers/evaluation.py:252-275`, `backend/app/routers/history.py:439-474`
- Trigger: Pre-computed baseline experts in evaluation_set.json use a fixed query embedding; system experts use current-query embedding; authority scores are embedding-specific
- Workaround: Baseline experts authority should be query-specific OR authority scoring should be embedding-agnostic

**Citation ID resolution may lose extra citations during generation:**
- Symptoms: Generated text references citations (e.g., [1]) that are not in the citations array sent to frontend
- Files: `backend/app/routers/query.py:263-347` (extra_citation_ids resolution), `backend/app/services/generation/surgeon.py` (citation stripping)
- Trigger: Surgeon strips unresolvedIDs after citation insertion; if DB lookup fails for extra_citation_ids, those citations remain broken
- Workaround: Check generated text for orphaned citation references in frontend; render a warning

**Neo4j connection pool may be exhausted under high concurrency:**
- Symptoms: Timeout errors or "connection pool exhausted" exceptions during peak load
- Files: `backend/app/services/neo4j_client.py:44` (pool size = 50), `backend/app/routers/chat.py` (concurrent ThreadPoolExecutor calls)
- Trigger: Multiple pipelines each spawn ThreadPoolExecutor(10) with many parallel Neo4j queries; with 5 concurrent pipelines = 50+ concurrent queries
- Workaround: Monitor pool usage; set `MAX_CONCURRENT_PIPELINES` environment variable; consider query batching

**Compass analysis failure is non-fatal but degrades UX:**
- Symptoms: Compass visualization missing from response; client sees empty axis data
- Files: `backend/app/routers/query.py:193-194` (compass exception caught, pipeline continues)
- Trigger: Ideology scorer computation fails (insufficient evidence, embedding issues); entire compass skipped
- Workaround: Frontend should display "compass unavailable" message instead of empty plot

## Security Considerations

**File operations on evaluation_set.json without validation:**
- Risk: evaluation_set.json is read/written without schema validation; malformed JSON could crash precalc endpoint
- Files: `backend/app/routers/history.py:495-552`
- Current mitigation: FileNotFoundError and generic Exception caught; invalid JSON logged but not validated
- Recommendations: (1) Validate JSON schema before write; (2) Use atomic file operations (write to temp, rename); (3) Add JSON schema validation on read

**No input validation on Cypher query endpoint:**
- Risk: `/api/graph/query` accepts raw Cypher; injection possible if client sends malicious queries
- Files: `backend/app/routers/graph.py:190` (cypher = request.cypher.strip())
- Current mitigation: Only .strip() called; no query validation
- Recommendations: (1) Use allowlist of safe Cypher patterns; (2) Parse/validate query AST; (3) Enforce read-only mode if for exploration

**Sensitive data in Neo4j (deputy names, institutions) exposed via REST API:**
- Risk: All endpoints expose full deputy/minister details (name, institution, profession, education); no access control
- Files: All routers expose deputy data via JSON endpoints
- Current mitigation: Public Italian parliamentary data; no authentication needed
- Recommendations: (1) Document that all data is public; (2) Add rate limiting if scraping becomes a concern; (3) Consider deprecating any sensitive fields from public APIs

## Performance Bottlenecks

**Authority computation for hundreds of deputies during baseline expert precalc:**
- Problem: `_compute_experts_from_matched` runs Neo4j authority scorer for each matched deputy; for 50-100 deputies per topic, 10+ topics = 500+ Neo4j queries
- Files: `backend/app/routers/history.py:474-545`
- Cause: Authority scoring requires embedding similarity to query + Neo4j statistics; no batching, sequential execution
- Improvement path: (1) Batch Neo4j queries (UNWIND multiple IDs); (2) Cache embedding computations; (3) Pre-compute per-deputy global authority scores at data load time

**Evaluation dashboard loads all chat history into memory:**
- Problem: `get_dashboard` in evaluation.py loads all chats from Neo4j, then iterates to compute automated metrics
- Files: `backend/app/routers/evaluation.py` (dashboard aggregation logic)
- Cause: No pagination or streaming; if 1000+ chats exist, memory spike and slow response
- Improvement path: (1) Add pagination with lazy loading; (2) Compute metrics in Neo4j (Cypher aggregations); (3) Cache dashboard results with TTL

**Extra citation resolution queries DB for each unresolved ID:**
- Problem: `process_query_streaming` resolves extra_citation_ids via a single UNWIND query; if 50 extra IDs exist, result set large
- Files: `backend/app/routers/query.py:272-300`
- Cause: No filtering of IDs that don't exist; full chunk/speech/speaker join for each ID
- Improvement path: (1) Pre-filter IDs that exist in retrieval pool; (2) Use Neo4j OPTIONAL MATCH with fallback; (3) Index chunk IDs

**ThreadPoolExecutor created multiple times in request pipeline:**
- Problem: `process_chat_streaming` spawns multiple ThreadPoolExecutor instances (line 263, 732, 1265, 1412)
- Files: `backend/app/routers/chat.py` (multiple imports of ThreadPoolExecutor)
- Cause: No thread pool singleton; each executor has its own threads
- Improvement path: (1) Create module-level ThreadPool singleton with fixed max_workers; (2) Reuse across all requests; (3) Monitor queue depth

## Fragile Areas

**Expert matching relies on name parsing and party assignment:**
- Files: `backend/app/routers/chat.py:1420-1456`, `backend/app/routers/history.py:238-278`
- Why fragile: Name splitting on first space (line 1424 in chat.py) fails for multi-part surnames (e.g., "De Luca" → ["De", "Luca"]); party assignment from Neo4j group may not match current party if deputy changed group
- Safe modification: (1) Store first_name/last_name separately in all APIs, not split on space; (2) Use historical party at speech date, not current party; (3) Add validation tests for all observed name formats

**Baseline expert authority scores baked into evaluation_set.json:**
- Files: `backend/app/routers/history.py:545-548`, `backend/app/routers/evaluation.py:145-175`
- Why fragile: Once computed and stored in JSON, scores won't update if Neo4j data changes; if new deputies added or scored differently, baseline experts outdated
- Safe modification: (1) Compute baseline experts on-the-fly during evaluation (cache results); (2) Version evaluation_set.json; (3) Add metadata: {baseline_answer, baseline_experts, computed_date, embedding_version}

**Citation surgeon pattern matching for insertion:**
- Files: `backend/app/services/generation/surgeon.py` (citation insertion logic)
- Why fragile: Text-based pattern matching for quoted text in narrative; if LLM reformats text slightly, patterns won't match; citations fail to insert
- Safe modification: (1) Use byte-level span tracking from model tokenization; (2) Add fuzzy match fallback with similarity threshold; (3) Log all failed insertions for debugging

**Compass analysis depends on sufficient evidence volume:**
- Files: `backend/app/services/compass/pipeline.py`, `backend/app/routers/query.py:176-194`
- Why fragile: PCA requires minimum samples (typically 10+); if query returns <5 evidence pieces, dimensionality reduction fails
- Safe modification: (1) Check evidence count before compass; skip if <5; (2) Handle degenerate cases (all same party); (3) Return fallback 2D projection (centroid)

## Scaling Limits

**Neo4j connection pool (size=50) saturates at 10+ concurrent pipelines:**
- Current capacity: 50 connections; with 5 concurrent pipelines × 10 threads each = 50 queries
- Limit: If 2+ pipelines run in parallel, pool may exhaust; threads wait or timeout
- Scaling path: (1) Increase pool size to 100+ (requires Neo4j RAM); (2) Reduce max_workers in ThreadPoolExecutor; (3) Add query queuing/prioritization; (4) Use async Neo4j driver instead of sync

**Frontend SurveyModal not paginated; all chats loaded into DOM:**
- Current capacity: ~100 chats before UI becomes sluggish
- Limit: 1000+ chats → browser memory exhaustion
- Scaling path: (1) Add chat pagination (show 20 per page); (2) Virtual scrolling for large lists; (3) Implement chat search/filter to reduce dataset

**Evaluation dashboard aggregation is O(n_chats):**
- Current capacity: ~200 chats aggregate in <5s
- Limit: 1000+ chats → 30+ second load time
- Scaling path: (1) Pre-compute dashboard metrics in Neo4j; (2) Cache results with 1-hour TTL; (3) Implement incremental updates (only recompute new chats since last cache)

**Authority component percentile normalization recalculates stats per query:**
- Current capacity: ~500 deputies per topic
- Limit: If all 630 deputies in parliament queried, percentile calculation becomes expensive
- Scaling path: (1) Pre-compute global percentile stats at data load; (2) Store per-component min/max; (3) Use lookup tables instead of recalculation

## Dependencies at Risk

**Neo4j Cypher query compatibility across versions:**
- Risk: Cypher 4 vs 5 compatibility; some queries use OPTIONAL MATCH chains that may be optimized differently
- Impact: If Neo4j upgraded to 5.x, query performance may degrade or syntax may break
- Migration plan: (1) Test all Cypher queries against Neo4j 5.x; (2) Use parameters (not string concat); (3) Add query execution plan monitoring

**LLM generation pipeline depends on OpenAI API stability:**
- Risk: If OpenAI rate limits or outage, all answer generation fails; no fallback to retrieval-only
- Impact: User cannot get responses during API outage
- Migration plan: (1) Add retrieval-only fallback endpoint (extract key passages without LLM); (2) Implement retry with exponential backoff; (3) Cache generated responses by query hash

**spaCy language models loaded at startup:**
- Risk: Model version pinned in requirements.txt; if model breaks or is deleted from hub, startup fails
- Impact: Application won't start
- Migration plan: (1) Pin spaCy model version in model download script; (2) Cache model locally; (3) Add fallback to simpler NLP library if model unavailable

## Missing Critical Features

**No audit logging for evaluation data changes:**
- Problem: Survey responses, ratings, and baseline answers are modified without logging who/when
- Blocks: Cannot trace evaluation data integrity; cannot reproduce results if data changed
- Recommendation: Add audit table with evaluator_id, timestamp, change_type, old_value, new_value

**No data validation on survey submission:**
- Problem: Frontend submits survey responses without schema validation; backend accepts any JSON
- Blocks: Malformed data can be stored; metrics computed on inconsistent data
- Recommendation: (1) Add Pydantic validators for SurveyResponse; (2) Enforce enum ranges (e.g., ratings 1-5); (3) Require evaluator_id

**No support for multi-evaluator consensus or inter-rater reliability:**
- Problem: System accepts surveys from multiple evaluators but doesn't track agreement
- Blocks: Cannot assess evaluation quality; cannot detect outlier evaluators
- Recommendation: (1) Add Kappa/Fleiss Kappa computation; (2) Show evaluator agreement scores; (3) Flag low-agreement topics

**No mechanism to update baseline answers or regenerate baseline experts:**
- Problem: evaluation_set.json is static; if baseline_answer needs correction, must edit JSON manually
- Blocks: Cannot refine evaluation set without manual intervention
- Recommendation: Add UI endpoint to update baseline answer + trigger expert recomputation

## Test Coverage Gaps

**Untested edge case: empty evidence list in generation:**
- What's not tested: Generator.generate() with evidence_list = []
- Files: `backend/app/services/generation/pipeline.py:69-250`
- Risk: May crash or return malformed JSON; no fallback
- Priority: High

**Untested concurrency: multiple simultaneous pipeline cancellations:**
- What's not tested: Client disconnect → task cancellation; concurrent cancellations on same task_id
- Files: `backend/app/routers/chat.py:593-596` (TaskCancelledError handling)
- Risk: Race condition in _waiting_queue; task state inconsistency
- Priority: High

**Untested: baseline expert precalc with missing deputies in Neo4j:**
- What's not tested: If a deputy mentioned in evaluation_set.json baseline_answer doesn't exist in Neo4j
- Files: `backend/app/routers/history.py:542-548`
- Risk: Empty matched list → empty experts array → metrics biased
- Priority: Medium

**Untested: compass analysis with <5 evidence pieces:**
- What's not tested: Query returns 3-4 pieces of evidence; compass PCA fails
- Files: `backend/app/services/compass/pipeline.py` (PCA implementation)
- Risk: Exception caught silently; user sees empty compass
- Priority: Medium

**Untested: citation resolution with duplicate chunk_ids:**
- What's not tested: Generated text cites same chunk twice; extra_citation_ids contains duplicates
- Files: `backend/app/routers/query.py:269-347`
- Risk: Deduplication may fail; citations rendered twice or missed
- Priority: Low

**Untested frontend: survey form submission with network failure:**
- What's not tested: User submits survey, network fails midway
- Files: `frontend/src/components/survey/SurveyModal.tsx` (createSurvey call)
- Risk: Form state unclear; user doesn't know if submission succeeded
- Priority: Medium

---

*Concerns audit: 2026-04-01*
