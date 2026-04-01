# Domain Pitfalls

**Domain:** Production Neo4j + FastAPI + Next.js RAG system refactoring
**Researched:** 2026-04-02
**Confidence:** HIGH — all findings verified from actual codebase files

---

## Critical Pitfalls

Mistakes that cause silent data corruption, broken evaluation metrics, or a non-functional pipeline.

---

### Pitfall 1: Schema Property Rename Without Updating All Cypher Consumers

**What goes wrong:** A Neo4j property is renamed in the build pipeline (e.g., `Chunk.text` stays the same but `Speech.preprocessed_text` becomes `Speech.text`), but one or more Cypher queries still reference the old name. Neo4j returns `null` for missing properties without raising an error, so the pipeline continues running with silently empty values.

**Why it happens:** The same property is accessed in at least six distinct locations across the codebase:
- `backend/app/services/retrieval/dense_channel.py` — lines 80–99: `c.text AS chunk_text`, `c.start_char_raw AS span_start`, `c.end_char_raw AS span_end`, `i.text AS text`
- `backend/app/services/retrieval/graph_channel.py` — lines 282–287: identical property set
- `backend/app/services/retrieval/engine.py` — lines 319–328: `prev.start_char_raw`, `next.start_char_raw`, `i.text AS speech_text`; also lines 497–502
- `backend/app/routers/query.py` — lines 277–295: inline Cypher for extra citation resolution
- `backend/app/routers/chat.py` — lines 420–422 and 903–905: duplicate citation-building Cypher
- `backend/app/services/authority/scorer.py` — lines 451–468: `i.text_embedding`, `d.profession_embedding`, `d.education_embedding`

If `start_char_raw`/`end_char_raw` are removed from the schema (as planned), all six locations must be updated simultaneously. Missing even one causes `span_start`/`span_end` to be `null` in SSE citation events, which silently breaks the CitationSurgeon's sentence-boundary expansion logic.

**Consequences:** Citations render with empty quote text. Authority scores that depend on `text_embedding` on Speech nodes return 0.5 (neutral fallback) for all speakers, inflating scores for low-quality speakers.

**Prevention:**
1. Before renaming any property, run a project-wide search: `grep -rn "property_name" backend/ --include="*.py"` to enumerate every query site.
2. Treat `dense_channel.py`, `graph_channel.py`, `engine.py`, `query.py`, `chat.py`, and `authority/scorer.py` as a mandatory update group for any Chunk or Speech property change.
3. After schema rebuild, run a Neo4j spot-check: `MATCH (c:Chunk) RETURN c.text LIMIT 1` to verify the renamed property exists before starting the backend.

**Phase warning:** Build pipeline phase and backend refactoring phase must be coordinated. Do not deploy a new schema against the old backend or vice versa — they will be incompatible.

---

### Pitfall 2: Embedding Cache Key Depends on Model Name — Wrong Default Invalidates 329MB Cache

**What goes wrong:** The embedding cache key is `sha256(EMBEDDING_MODEL + "\n" + normalized_text)` (see `build/embedding_service.py` lines 83–85). The `EMBEDDING_MODEL` env var defaults to `"text-embedding-3-small"` in the build scripts. The backend services also hardcode `"text-embedding-3-small"` (e.g., `backend/app/services/retrieval/engine.py` line 69, `backend/app/services/generation/sectional.py` line 285). The PROJECT.md states the model is `text-embedding-ada-002 @ 1536d`, but the actual code uses `text-embedding-3-small`. Both models produce 1536-dimensional vectors, so the vector index accepts both — silently.

**Why it happens:** Any refactoring that introduces a new reference to the embedding model as a string literal risks accidentally using the wrong model name. If build scripts ever run with the wrong `EMBEDDING_MODEL`, all 329MB of cached embeddings become orphaned (their SHA-256 keys will not match), triggering a full re-embedding run costing significant OpenAI API fees.

**Consequences:** Full re-embedding cost (thousands of API calls), hours of build time. More dangerously: if the model name changes between build and backend, query embeddings are in a different space than document embeddings — retrieval silently degrades (high cosine similarity scores for irrelevant chunks).

**Prevention:**
1. The `EMBEDDING_MODEL` value must be identical in all locations. Currently: `"text-embedding-3-small"` everywhere — preserve this exact string.
2. Never rename or add a new embedding model reference without updating `build/embedding_service.py` and all backend callsites together.
3. Do not change the cache key format (`f"{EMBEDDING_MODEL}\n{normalized_text}"`) — any change orphans the entire cache.
4. The cache file lives at `build/embeddings_cache.db` (329MB). Never delete or move it during refactoring. Add it to `.gitignore` if not already excluded, and verify the volume mount in `docker-compose.yml` before rebuilding.

**Phase warning:** Any build pipeline phase that touches `build/embedding_service.py` must include a cache preservation check as its first step.

---

### Pitfall 3: SSE Event Shape Changes Break the Frontend Silently

**What goes wrong:** The query pipeline emits a precisely ordered sequence of SSE events with typed JSON payloads: `progress`, `experts`, `compass`, `topic_stats`, `citations`, `citation_details`, `experts` (second time), `chunk` (repeated), `complete`, `error`. The frontend `use-chat.ts` switch statement (line 230) dispatches on `data.type`. Any rename of a `type` value, addition of a required field, or removal of an existing field breaks the frontend without a compile-time error, because the SSE stream is untyped at the transport layer.

**Why it happens:** The SSE contract spans:
- `backend/app/routers/query.py` — all `yield f"data: ..."` statements (18 distinct yield sites)
- `frontend/src/app/api/chat/route.ts` — passthrough proxy (transparent, no transformation for most events)
- `frontend/src/hooks/use-chat.ts` — `switch (data.type)` consumer
- `frontend/src/types/api.ts` — type definitions for event payloads

The second `experts` SSE event (line 502 in query.py) is a post-citation update that replaces the first experts panel. If refactoring moves this emit or changes its `data.experts` field structure, the frontend will display stale experts (the first panel) permanently.

**Consequences:** Silent data display errors. The `chunk` events build the response text character-by-character — any interruption of the yield sequence causes a truncated answer with no error shown to the user.

**Prevention:**
1. Treat the SSE event sequence as a frozen contract during refactoring. The only safe changes are internal implementation changes that preserve the same JSON payloads.
2. Before refactoring any `yield` statement in `query.py`, cross-reference the corresponding `case` in `frontend/src/hooks/use-chat.ts` (line 230+) and the type in `frontend/src/types/api.ts`.
3. The `experts` event is emitted twice (lines 166 and 502). Any refactoring that consolidates these into one emit must be tested end-to-end — the frontend renders the post-citation experts panel based on the second event.
4. The `waiting` event (line 52–54) includes an Italian-language hardcoded string — do not remove this event type or the frontend's queue-position display breaks.

**Phase warning:** Backend query router refactoring must be done before or in tandem with frontend type updates. Never deploy a backend-only change to the SSE contract.

---

### Pitfall 4: Evaluation Baseline Breaks If Expert Field Names Change

**What goes wrong:** The evaluation system (`backend/app/routers/evaluation.py`) reads expert records from two sources: (a) `ChatHistory.experts` stored as JSON in Neo4j, and (b) `evaluation_set.json`'s `baseline_experts` arrays. These records are accessed by field name: `expert.get("first_name")`, `expert.get("last_name")`, `expert.get("authority_score")`, `expert.get("group")` (and the legacy alias `expert.get("party")`), `expert.get("total_score")` (legacy alias for `authority_score`).

If any of these field names change during refactoring of the expert-building code in `query.py` or `history.py`, existing `ChatHistory` nodes in Neo4j store the old field names. The evaluation dashboard will silently compute zeros for authority metrics because `_build_expert_full_lookup` (lines 161–174) and `_compute_automated_metrics` (lines 295–435) will find no matching keys.

**Why it happens:**
- `query.py` builds expert dicts with keys: `id`, `first_name`, `last_name`, `group`, `authority_score`, `score_breakdown`, `photo`, `camera_profile_url`, `profession`, `education`, `committee`, `institutional_role`, `coalition`, `relevant_speeches_count` (lines 645–680).
- `evaluation.py` reads these keys back from stored JSON. ChatHistory nodes are never migrated — they persist as written.
- `evaluation_set.json` (329 lines, multiple topics) hard-codes `baseline_experts` with field `"group"` for party name and `"authority_score"` for score. These are not auto-migrated.

**Consequences:** All historical chat evaluations show zero authority scores on the dashboard. The A/B comparison becomes meaningless. The survey evaluation modal's group-authority panel goes blank.

**Prevention:**
1. The fields `first_name`, `last_name`, `group`, `authority_score`, `score_breakdown` in expert dicts are a frozen API contract — treat them as such.
2. If any field must be renamed, add a read-time alias in `_build_expert_full_lookup` and `_compute_automated_metrics` (the `expert.get("authority_score") or expert.get("total_score")` pattern already handles one such alias — extend it rather than remove old aliases).
3. `evaluation_set.json` must be updated manually if the `baseline_experts` field schema changes. There is no migration tooling.
4. The frontend `Expert` type in `frontend/src/types/chat.ts` (lines 29–57) must stay in sync — it expects `group`, `authority_score`, `score_breakdown.speeches/acts/committee/profession/education/role`.

**Phase warning:** Backend services refactoring phase must explicitly list expert dict field names as a no-change constraint.

---

### Pitfall 5: `ingest_stenografici.py` Parser Is Coupled to `build_and_update.py` — Cannot Be Deleted Without Breaking the Build

**What goes wrong:** `build_and_update.py` imports `StenograficoIngester` from `ingest_stenografici` (line 40) and calls `StenograficoIngester.__new__(StenograficoIngester)` without `__init__` to use only the XML parsing methods (lines 556–594). The Italian `save_to_neo4j()` method is dead code, but the XML parsing logic (`parse_xml`, `_chunk_speech`, `_merge_continuation_speeches`, etc.) is live. Deleting `ingest_stenografici.py` or moving the parser class without updating `build_and_update.py` breaks the entire build pipeline.

**Consequences:** `build_and_update.py` fails on import, making it impossible to rebuild the database.

**Prevention:**
1. When refactoring the build pipeline, extract the parser class from `ingest_stenografici.py` into a new `parser.py` module before deleting the old file.
2. Update the import in `build_and_update.py` at the same time.
3. The Italian schema initialization code in `ingest_stenografici.py` (constraints for `Seduta`, `Dibattito`, `Fase`, `Intervento`) must not be called — it is only safe to use the parser methods (`StenograficoIngester.__new__` pattern). Do not accidentally call `__init__`, which triggers schema creation for the Italian labels.

**Phase warning:** Build pipeline refactoring phase must not split "remove Italian save path" and "extract parser class" into separate commits without a working intermediate state.

---

## Moderate Pitfalls

### Pitfall 6: Vector Index Name Hardcoded in Multiple Files — Rename Breaks Retrieval

**What goes wrong:** The vector index `chunk_embedding_index` is hardcoded as a string literal in:
- `backend/app/services/retrieval/dense_channel.py` — line 67 (via config key, default `"chunk_embedding_index"`)
- `backend/app/routers/search.py` — line 274 (hardcoded string literal)
- `backend/app/main.py` — line 179 (warmup query, hardcoded)
- `backend/app/routers/search.py` — line 371 (`act_description_embedding_index`, second index)
- `build/build_and_update.py` — line 912 (`chunk_embedding_index`)
- `build/create_vector_index.py` — lines 24 and 28

If the index is renamed during schema normalization, some files will still reference the old name. Neo4j does not error on a missing index — it returns zero results.

**Prevention:** Define the index name as a constant in one place and import it. If kept as strings, run `grep -rn "chunk_embedding_index"` across the entire repo before and after any rename to verify all sites are updated.

---

### Pitfall 7: `deputy_first_name`/`deputy_last_name` Citation Fields Have Deep Frontend Coupling

**What goes wrong:** Citation objects use `deputy_first_name` and `deputy_last_name` as field names. These appear in:
- `frontend/src/types/chat.ts` lines 12–13
- `frontend/src/components/chat/CitationCard.tsx` (4 render sites)
- `frontend/src/components/chat/MessageBubble.tsx` lines 72, 308
- `frontend/src/components/survey/CitationReviewStep.tsx` line 343
- `frontend/src/components/survey/SurveyModal.tsx` line 858
- `frontend/src/hooks/use-chat.ts` line 351
- `backend/app/routers/evaluation.py` lines 382–383 (used for citation-expert matching)
- `backend/app/routers/chat.py` (3 citation-building sites, lines 1515–1516, 1669–1670)
- `backend/app/routers/query.py` lines 835–836, 920–921

Renaming to `speaker_first_name`/`speaker_last_name` requires simultaneous updates across all backend routers, the evaluation citation-matching logic, and all frontend components. The evaluation router uses these fields to match citations to authority scores — a mismatch causes authority-by-group to compute as zero for all cited parties.

**Prevention:** If renaming, use a read-time alias in the evaluation router first (add `cit.get("speaker_first_name") or cit.get("deputy_first_name")`), then propagate the rename through all other files.

---

### Pitfall 8: `populate_ruoli.py` References Italian Label `Intervento` — Will Fail After Schema Migration

**What goes wrong:** `build/populate_ruoli.py` line 168 queries `MATCH (i:Intervento)` — an Italian-schema label that does not exist in the English schema (`Speech` is the equivalent). This script assigns institutional roles and is part of the build pipeline. If it runs after the Italian schema is removed, it silently processes zero nodes and returns without error.

**Prevention:** Update `populate_ruoli.py` to use `MATCH (i:Speech)` before removing the Italian schema. Verify by running it on the rebuilt database and checking that institutional roles are populated.

---

### Pitfall 9: `evaluation_set.json` Path Is Relative — Breaks If File Is Moved

**What goes wrong:** `backend/app/routers/survey.py` lines 33–34 compute the path as:
```python
_EVAL_SET_PATH = os.path.join(os.path.dirname(__file__), "..", "..", "evaluation_set.json")
```
This resolves to `backend/evaluation_set.json`. If the file is moved (e.g., to a `data/` directory) during refactoring, both `survey.py` and `evaluation.py` (which calls `_load_evaluation_set_raw`) will silently return an empty dict. The evaluation dashboard will show zero baseline chats and all baseline metrics will be null.

**Prevention:** Do not move `evaluation_set.json`. If the directory structure changes, update the path constant in `survey.py` and add a startup assertion that the file exists.

---

### Pitfall 10: No Test Suite — Regressions Are Only Caught in Production

**What goes wrong:** There are zero project-owned test files (all test files found are inside `backend/venv/lib/python3.13/site-packages/thinc/tests/`, which are library tests). Any refactoring change that breaks the pipeline, changes a Cypher property name, or alters an SSE event will only be detected by manually running the full system.

**Why it matters here:** The authority scoring system has already had three significant bugs fixed (documented in MEMORY.md). The fixes involved subtle interactions between `evaluation.py`, `query.py`, and `SurveyModal.tsx`. These bugs are very likely to recur during refactoring without regression tests.

**Prevention:**
1. Before refactoring each module, write a minimal smoke test that exercises the critical path (even a simple `pytest` function that instantiates the class and calls the main method with fixture data).
2. Priority test targets: `AuthorityScorer.compute_all_authority`, `_compute_automated_metrics` in evaluation.py, and the citation-building functions in `query.py`.
3. For SSE, write an integration test that fires a request to `/api/query` and asserts that the event sequence contains `experts`, `citations`, `citation_details`, and `complete` in order.

---

## Minor Pitfalls

### Pitfall 11: `SurveyEvaluation` Node Property `ab_assignment` Stored Per-Evaluator

**What goes wrong:** `ab_assignment` (which response is "A" vs "B" for each evaluator) is stored on `SurveyEvaluation` nodes, not on `ChatHistory`. Legacy code in `survey.py` line 299 includes a fallback that reads from `ChatHistory` for old records. If this fallback is removed during cleanup, historical survey records that used the old storage location will lose their A/B assignment and all their votes become unblindable.

**Prevention:** Keep the `ChatHistory.ab_assignment` fallback read in `survey.py` permanently. Do not remove it even if it appears unused — it protects historical records.

---

### Pitfall 12: Authority Score `total_score` Alias Must Be Preserved

**What goes wrong:** Expert dicts have historically used both `authority_score` and `total_score` for the same value. The evaluation router reads `expert.get("authority_score") or expert.get("total_score") or 0` (evaluation.py lines 152, 169, 350, 370). Removing `total_score` from expert dicts is safe as long as `authority_score` is always present. But if `authority_score` is renamed without maintaining the alias in all read paths, all historical ChatHistory records with `total_score` will return 0.

**Prevention:** Keep `authority_score` as the canonical key and preserve `total_score` aliases in evaluation read paths permanently.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Build pipeline: remove Italian schema | Pitfall 5 (parser coupling), Pitfall 8 (populate_ruoli Italian label) | Extract parser class first; update populate_ruoli before removing Italian labels |
| Build pipeline: remove `start_char_raw`/`end_char_raw` | Pitfall 1 (property rename across 6 files) | Grep all sites; update dense_channel, graph_channel, engine, query, chat in same commit |
| Build pipeline: Speech.preprocessed_text → Speech.text | Pitfall 1 (property rename), Pitfall 4 (authority scorer reads `i.text_embedding`) | Rename in build and authority scorer.py simultaneously |
| Backend services refactoring | Pitfall 4 (expert field names), Pitfall 7 (citation field names) | Treat these as frozen API contracts; document any renames as dual-layer changes |
| Backend API refactoring | Pitfall 3 (SSE event shape) | No changes to `type` values or payload field names in query.py yield statements |
| Embedding/cache changes | Pitfall 2 (cache key invalidation) | Never change `EMBEDDING_MODEL` string or cache key format |
| Frontend refactoring | Pitfall 3 (SSE contract), Pitfall 7 (citation fields) | Update types in `frontend/src/types/` first; verify against backend before deploying |
| Any phase | Pitfall 10 (no test suite) | Add smoke tests before modifying any critical service |

## Sources

All findings from direct codebase inspection (no external sources needed — pitfalls are specific to this codebase):

- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/services/retrieval/dense_channel.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/services/retrieval/graph_channel.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/services/retrieval/engine.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/services/authority/scorer.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/services/authority/components.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/routers/query.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/routers/evaluation.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/routers/chat.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/routers/survey.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/app/routers/history.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/build/embedding_service.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/build/build_and_update.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/build/ingest_stenografici.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/build/populate_ruoli.py`
- `/Users/mirkotritella/Desktop/ParliamentRAG/frontend/src/types/chat.ts`
- `/Users/mirkotritella/Desktop/ParliamentRAG/frontend/src/hooks/use-chat.ts`
- `/Users/mirkotritella/Desktop/ParliamentRAG/frontend/src/app/api/chat/route.ts`
- `/Users/mirkotritella/Desktop/ParliamentRAG/backend/evaluation_set.json`
- `/Users/mirkotritella/Desktop/ParliamentRAG/.planning/PROJECT.md`
