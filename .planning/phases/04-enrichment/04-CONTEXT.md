# Phase 4: Enrichment - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Add BM25 sparse retrieval channel with Reciprocal Rank Fusion (RRF) merging, ingest per-deputy individual vote records and committee officer roles from dati.camera.it SPARQL endpoint, and run NER at ingestion time to extract LAW and PERSON entity references on chunks.

</domain>

<decisions>
## Implementation Decisions

### BM25 + RRF Hybrid Retrieval

- **Full-text index on Chunk.text only** — the retrieval unit already used by dense search. No Speech.text.
- **Neo4j native full-text index** — uses Lucene BM25 scoring, zero new dependencies
- **Create index:** `CREATE FULLTEXT INDEX chunk_fulltext IF NOT EXISTS FOR (n:Chunk) ON EACH [n.text]`
- **RRF weights configurable** in `backend/config/default.yaml` — Claude decides initial values based on parliamentary text characteristics
- **New module:** `backend/app/services/retrieval/sparse_channel.py` — mirrors dense_channel.py interface
- **RRF merger:** Update existing `merger.py` to implement RRF formula: `score = sum(1 / (k + rank_i))` for each channel
- **Create full-text index in build pipeline** — add to `db_builder.py` alongside vector index creation

### SPARQL Data Ingestion

- **Per-deputy individual vote records:** `Deputy-[:VOTED]->IndividualVote-[:ON_VOTE]->Vote`
  - `IndividualVote` node with properties: `outcome` ("favor"/"against"/"abstain"/"absent")
  - Links to existing Vote nodes (created in Phase 1 from XML)
  - Source: dati.camera.it SPARQL endpoint
- **Committee officer roles with dates:** enrich existing `MEMBER_OF_COMMITTEE` relationships
- **Makefile target:** `make enrich-sparql` — separate from `db-all` (incremental, slow, network-dependent)
- **New module:** `build/sparql_ingester.py` — SPARQL queries + Neo4j writes

### NER at Ingestion Time

- **Model:** `it_core_news_lg` (spaCy standard Italian) — compatible with current stack, already has `it_core_news_sm` installed
- **Entity types:** LAW and PERSON references only
- **Storage:** Chunk properties `lawRefs` (string array) and `personRefs` (string array)
- **Integration:** Run NER during chunk creation in `db_builder.py` — after chunking, before Neo4j write
- **Requires full rebuild** (`make db-all`) to populate NER fields on all chunks
- **Entity-filtered retrieval:** Update `graph_channel.py` to optionally filter by `lawRefs`/`personRefs` when the query mentions specific laws or persons

### Claude's Discretion
- RRF k parameter value and channel weight ratios
- SPARQL query structure for dati.camera.it
- How to handle SPARQL endpoint timeouts/errors
- NER confidence threshold for entity inclusion
- Whether to normalize entity strings (e.g., "D.L. 231" vs "decreto legislativo 231")
- Exact Makefile target names and structure
- Test structure for new modules

</decisions>

<canonical_refs>
## Canonical References

### Retrieval layer (being extended)
- `backend/app/services/retrieval/engine.py` — Retrieval orchestrator (add sparse channel)
- `backend/app/services/retrieval/dense_channel.py` — Pattern to follow for sparse_channel.py
- `backend/app/services/retrieval/merger.py` — Currently merges dense+graph, extend with RRF
- `backend/config/default.yaml` — Configuration weights

### Build pipeline (being extended)
- `build/db_builder.py` — Add full-text index creation + NER integration
- `build/chunker.py` — Chunk creation (NER runs after this)
- `build/config.yaml` — Build configuration

### Neo4j schema (from Phase 1)
- Vote nodes: `Session-[:HAS_VOTE]->Vote` (Phase 1)
- New: `Deputy-[:VOTED]->IndividualVote-[:ON_VOTE]->Vote`
- Chunk properties to add: `lawRefs`, `personRefs`

### Research
- `.planning/research/ENRICHMENT.md` — BM25, SPARQL, NER detailed analysis
- `.planning/research/STACK.md` — Neo4j full-text index patterns

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `dense_channel.py` — Pattern for sparse_channel.py (same interface: query → list of evidence dicts)
- `merger.py` — Already merges 2 channels, extend to 3 with RRF
- `db_builder.py` — Has `create_vector_index()`, add `create_fulltext_index()`
- `download.py` — Pattern for SPARQL download module
- `build/requirements-build.txt` — Add spacy model if needed

### Established Patterns
- Retrieval channels return `List[Dict[str, Any]]` with standard evidence keys
- Merger deduplicates by `evidence_id` and combines scores
- Build pipeline uses UNWIND batch writes with batch_size=1000
- Makefile targets follow `db-*` naming convention

### Integration Points
- `engine.py` orchestrates channels — add sparse_channel.search() call
- `deps.py` provides Neo4j client via Depends() — SPARQL ingester uses same client
- `precalculate_embeddings.py` runs after build — NER must run before it

</code_context>

<specifics>
## Specific Ideas

- BM25 is the highest-impact, lowest-effort improvement — implement first
- SPARQL ingestion is network-dependent and slow — separate Makefile target
- NER requires full rebuild — integrate into db_builder.py chunk creation path
- The `it_core_news_lg` model may not have a `LAW` entity type — research during planning to confirm exact entity labels available

</specifics>

<deferred>
## Deferred Ideas

- Wikidata biographical enrichment (ENR-V2-01)
- Normattiva.it ELI URI linking (ENR-V2-02)
- Entity linking via ReLiK (ENR-V2-03)
- Citation graph Speech→Act (ENR-V2-04)

</deferred>

---

*Phase: 04-enrichment*
*Context gathered: 2026-04-02*
