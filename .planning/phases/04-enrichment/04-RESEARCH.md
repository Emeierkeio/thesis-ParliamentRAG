# Phase 4: Enrichment - Research

**Researched:** 2026-04-02
**Domain:** Neo4j full-text indexing, SPARQL ingestion, spaCy NER, RRF merging
**Confidence:** HIGH (Neo4j API, SPARQL endpoint verified live), MEDIUM (NER design, RRF params)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **BM25**: Full-text index on `Chunk.text` only. Neo4j native Lucene index. Zero new deps.
- **Index creation**: `CREATE FULLTEXT INDEX chunk_fulltext IF NOT EXISTS FOR (n:Chunk) ON EACH [n.text]`
- **New module**: `backend/app/services/retrieval/sparse_channel.py` — mirrors `dense_channel.py` interface
- **RRF merger**: Update existing `merger.py`. Formula: `score = sum(1 / (k + rank_i))` for each channel
- **RRF weights**: Configurable in `backend/config/default.yaml`
- **Fulltext index**: Also created in `db_builder.py` alongside vector index
- **SPARQL ENR-01**: `Deputy-[:VOTED]->IndividualVote-[:ON_VOTE]->Vote`. `IndividualVote.outcome` = "favor"/"against"/"abstain"/"absent"
- **SPARQL ENR-02**: Enrich existing `MEMBER_OF_COMMITTEE` relationships with officer roles
- **Makefile target**: `make enrich-sparql` — separate from `db-all`
- **New module**: `build/sparql_ingester.py`
- **NER model**: `it_core_news_lg` (spaCy standard Italian)
- **NER entity types**: LAW and PERSON references (Chunk properties `lawRefs`, `personRefs` string arrays)
- **NER integration**: Run during chunk creation in `db_builder.py`, before Neo4j write
- **NER requires full rebuild** (`make db-all`)
- **Entity-filtered retrieval**: Update `graph_channel.py` to optionally filter by `lawRefs`/`personRefs`

### Claude's Discretion
- RRF k parameter value and channel weight ratios
- SPARQL query structure for dati.camera.it
- How to handle SPARQL endpoint timeouts/errors
- NER confidence threshold for entity inclusion
- Whether to normalize entity strings
- Exact Makefile target names and structure
- Test structure for new modules

### Deferred Ideas (OUT OF SCOPE)
- Wikidata biographical enrichment (ENR-V2-01)
- Normattiva.it ELI URI linking (ENR-V2-02)
- Entity linking via ReLiK (ENR-V2-03)
- Citation graph Speech→Act (ENR-V2-04)
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| RET-01 | Add BM25 sparse retrieval channel via Neo4j full-text index | Neo4j full-text index syntax verified. `SparseChannel` design established. |
| RET-02 | Implement RRF merger for hybrid dense+sparse+graph retrieval | RRF formula confirmed, k=60 industry standard. `merger.py` extension design established. |
| ENR-01 | SPARQL ingestion from dati.camera.it — per-deputy individual vote records | SPARQL endpoint live at `https://dati.camera.it/sparql`. OCD vocabulary confirmed. Vote matching strategy defined. |
| ENR-02 | SPARQL ingestion from dati.camera.it — committee officer roles with dates | `ufficioParlamentare` pattern confirmed. `carica` predicate gives role. `rif_organo` + `rdfs:label` give committee name. |
| ENR-03 | NER at ingestion time on chunks — extract LAW and PERSON entity references | `it_core_news_lg` has no LAW label (confirmed). Strategy: PER from spaCy + regex for law references. |
| ENR-04 | Store NER results as Chunk properties (lawRefs, personRefs) for entity-filtered retrieval | `_create_chunks` Cypher must be extended. `graph_channel.py` entity filter design established. |
</phase_requirements>

---

## Summary

Phase 4 adds BM25 sparse retrieval with RRF merging, ingests per-deputy voting records and committee officer roles from the dati.camera.it SPARQL endpoint, and runs NER at build time to store law and person entity references on Chunk nodes.

The highest-impact item is BM25+RRF (RET-01/RET-02): Neo4j already uses Lucene under the hood, so a full-text index is a zero-dependency addition. The `SparseChannel` mirrors `DenseChannel` exactly, returning the same `List[Dict[str, Any]]` evidence format. The merger extends the current `ChannelMerger` to support three channels via RRF, replacing the current custom scoring formula for the pre-merge step.

SPARQL ingestion (ENR-01/ENR-02) is viable — the endpoint at `https://dati.camera.it/sparql` is live and was queried during research. There are 34,237 votazioni in XIX leg; individual `voto` records number in the millions and cannot be counted in a single query without timeout. The module must page via LIMIT/OFFSET. Critical: the Deputy URI format in SPARQL (`deputato.rdf/d{id}_19`) differs from Neo4j's `Deputy.id` format (`persona.rdf/p{id}`); matching is done by extracting the numeric person ID from both.

NER (ENR-03/ENR-04): `it_core_news_lg` does NOT have a LAW entity type — it only supports `PER`, `ORG`, `LOC`, `MISC`. Law references must be extracted via regex. Person entities use the spaCy `PER` label. Processing ~155K chunks with `it_core_news_lg` is estimated at 1-3 hours on CPU (the model is large; `sm` is much faster but lower quality for person names).

**Primary recommendation:** Implement BM25+RRF first (lowest risk, highest impact), then NER (full rebuild required anyway), then SPARQL (network-dependent, separate target).

---

## Standard Stack

### Core (all already in requirements.txt or zero new deps)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| neo4j (driver) | >=5.0 | Full-text index creation + query | Native Lucene BM25, zero new deps |
| spacy | >=3.5.0 | Italian NER for person extraction | Already in `backend/requirements.txt`; `it_core_news_sm` referenced in Makefile |
| requests / urllib | stdlib | SPARQL HTTP calls | `sparql_ingester.py` — no SPARQL client library needed |

### New to Install
| Library | Version | Purpose | Install |
|---------|---------|---------|---------|
| it_core_news_lg | latest (3.8.x) | Large Italian NER model | `python -m spacy download it_core_news_lg` |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| regex for law refs | fine-tuned NER | Custom NER requires labeled data we don't have |
| `it_core_news_lg` | `it_core_news_sm` | sm is 3-5x faster but lower NER F1; lg preferred per CONTEXT decision |
| urllib for SPARQL | SPARQLWrapper | Extra dep; urllib sufficient for simple GET queries |

**Installation:**
```bash
# In backend venv (for NER at query-time if needed later):
pip install spacy
python -m spacy download it_core_news_lg

# In build venv (for NER at ingestion time):
pip install spacy
python -m spacy download it_core_news_lg
```

---

## Architecture Patterns

### Recommended Project Structure (new files)
```
backend/app/services/retrieval/
├── dense_channel.py       # Existing (pattern to follow)
├── sparse_channel.py      # NEW: BM25 via Neo4j full-text index
├── graph_channel.py       # Existing (add entity filter)
├── merger.py              # Existing (add RRF logic)
└── engine.py              # Existing (add sparse_channel call)

build/
├── sparql_ingester.py     # NEW: SPARQL queries + Neo4j writes
├── db_builder.py          # EXTEND: create_fulltext_index(), NER in _create_chunks
└── build_and_update.py    # EXTEND: call create_fulltext_index() in do_build()

backend/config/default.yaml  # EXTEND: sparse_channel + rrf sections
Makefile                      # EXTEND: enrich-sparql target
```

### Pattern 1: SparseChannel — BM25 via Neo4j Full-Text Index

**What:** A new retrieval channel that queries Neo4j's Lucene full-text index on `Chunk.text`, then traverses to speech/speaker metadata, returning the same evidence dict format as `DenseChannel`.

**When to use:** Every query (run in parallel with dense channel via the existing `ThreadPoolExecutor`).

**Key syntax (HIGH confidence — verified against official Neo4j docs):**
```python
# Source: https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/full-text-indexes/

# Index creation (in db_builder.py create_fulltext_index):
"""
CREATE FULLTEXT INDEX chunk_fulltext IF NOT EXISTS
FOR (n:Chunk) ON EACH [n.text]
OPTIONS {indexConfig: {`fulltext.analyzer`: 'italian'}}
"""

# Query (in sparse_channel.py):
"""
CALL db.index.fulltext.queryNodes('chunk_fulltext', $query_text, {limit: $top_k})
YIELD node AS c, score
MATCH (c)<-[:HAS_CHUNK]-(i:Speech)-[:SPOKEN_BY]->(speaker)
MATCH (i)<-[:CONTAINS_SPEECH]-(f:Phase)<-[:HAS_PHASE]-(d:Debate)<-[:HAS_DEBATE]-(s:Session)
OPTIONAL MATCH (speaker)-[mg:MEMBER_OF_GROUP]->(g:ParliamentaryGroup)
WHERE mg.start_date <= s.date AND (mg.end_date IS NULL OR mg.end_date >= date())
OPTIONAL MATCH (speaker)-[mg_now:MEMBER_OF_GROUP]->(g_now:ParliamentaryGroup)
WHERE mg_now.end_date IS NULL
RETURN c.id AS chunk_id, c.text AS chunk_text, score AS similarity, ...
"""
```

**Critical implementation notes:**
- The `score` from `queryNodes` is a Lucene BM25 score (unnormalized, range roughly 0–10+). It must be normalized before RRF (e.g., sigmoid or min-max across results) or used directly as rank-based input to RRF (rank only, not score value).
- The `fulltext.analyzer: 'italian'` option enables Italian-specific stemming and stop word removal, improving BM25 quality on parliamentary text.
- CALL syntax: `db.index.fulltext.queryNodes` takes `(indexName, queryString, {limit: N})`. The `limit` option inside the call is not standard in all versions; use Cypher `LIMIT` in the outer query as fallback.
- NO `MATCH` clause before the CALL (same restriction as vector index).
- `queryNodes` supports Lucene boolean syntax: `"governo" AND "immigrazione"`, `"decreto legge~"` (fuzzy), `"riforma pensioni"~2` (proximity).

### Pattern 2: RRF Merger Extension

**What:** Replace the current custom scoring formula in `merger.py` with RRF for the pre-merge ranking step, while preserving the diversity/coverage logic in `_select_diverse`.

**RRF Formula (HIGH confidence — standard academic formula):**
```python
# Source: Cormack, Clarke, Buettcher (2009), "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods"

def rrf_score(ranks: list[tuple[int, float]], k: int = 60) -> float:
    """
    ranks: list of (rank_position, channel_weight) tuples
    rank_position: 1-based position in the channel's result list
    Returns: weighted RRF score
    """
    return sum(weight / (k + rank) for rank, weight in ranks)

# In merger.py — build combined score:
# For each evidence_id, collect (rank, weight) from each channel it appeared in
# dense rank 1-based from top_k=200 dense results
# sparse rank 1-based from top_k=100 sparse results
# graph rank 1-based from top_k=200 graph results
```

**Recommended initial weights (Claude's discretion — parliamentary text rationale):**
```yaml
# backend/config/default.yaml — add sparse_channel + rrf sections:
retrieval:
  sparse_channel:
    top_k: 100          # Fewer than dense (BM25 is precise, less recall)
    index_name: "chunk_fulltext"

  rrf:
    k: 60               # Industry standard; lowers dominance of rank-1 items
    dense_weight: 1.0   # Semantic similarity — primary signal
    sparse_weight: 0.8  # BM25 keyword match — strong for law names, proper nouns
    graph_weight: 0.5   # Structure traversal — supplementary signal
```

**Rationale for weights:** Parliamentary text contains many proper nouns (law names, deputy names, decree numbers) that BM25 captures better than dense embeddings. Sparse weight 0.8 (vs dense 1.0) reflects BM25's high precision for keyword queries while respecting the primacy of semantic meaning. Graph weight 0.5 reflects its supplementary role.

**Integration in `engine.py`:** Add `SparseChannel` alongside `DenseChannel` and `GraphChannel`. Run all three in `ThreadPoolExecutor(max_workers=3)`. Pass all three result lists to updated `merger.merge(dense, sparse, graph, ...)`.

### Pattern 3: SPARQL Ingestion

**What:** `build/sparql_ingester.py` fetches individual vote records and committee officer roles from `https://dati.camera.it/sparql` and writes them to Neo4j.

**Confirmed SPARQL vocabulary (HIGH confidence — verified with live endpoint):**

```
# Individual vote record (ocd:voto):
# URI pattern: http://dati.camera.it/ocd/voto.rdf/vd19_{session}_{vote}_{personId}
# Predicates:
#   ocd:rif_deputato   -> deputato.rdf/d{personId}_19
#   ocd:rif_votazione  -> votazione.rdf/vs19_{session}_{vote}
#   ocd:type           -> "Favorevole" | "Contrario" | "Astenuto" | "Non ha votato" | "In missione"

# Votazione (ocd:votazione):
# URI pattern: votazione.rdf/vs19_{session}_{vote}
# identifier literal: e.g. "029089" = session 029, vote 089
# Vote.number in Neo4j = the numeric part of vote portion = 89
# Session.number in Neo4j = 29
# Match: WHERE v.number = toInteger(SPLIT(identifier, '')[3:6]) AND s.number = session

# Committee officer role (ocd:ufficioParlamentare):
# Predicates:
#   ocd:carica      -> "PRESIDENTE" | "VICEPRESIDENTE" | "SEGRETARIO" | "CAPOGRUPPO" | "QUESTORE" | ...
#   ocd:rif_organo  -> organo.rdf/o19_{id}  (rdfs:label = committee name, e.g. "GIUNTA DELLE ELEZIONI")
#   ocd:rif_deputato -> deputato.rdf/d{personId}_19
#   ocd:startDate   -> "YYYYMMDD"
#   ocd:endDate     -> "YYYYMMDD" (may be absent for current roles)
```

**Deputy ID matching (CRITICAL):**
```python
# SPARQL gives: "http://dati.camera.it/ocd/deputato.rdf/d308908_19"
# Neo4j Deputy.id: "http://dati.camera.it/ocd/persona.rdf/p308908"
# Both contain the same numeric person ID: 308908

import re

def sparql_dep_uri_to_person_id(uri: str) -> str:
    """Extract numeric person ID from deputato.rdf URI."""
    # e.g. ".../d308908_19" -> "308908"
    m = re.search(r'/d(\d+)_\d+$', uri)
    return m.group(1) if m else None

def person_id_to_neo4j_id(person_id: str) -> str:
    """Reconstruct Neo4j Deputy.id from person ID."""
    return f"http://dati.camera.it/ocd/persona.rdf/p{person_id}"
```

**Vote matching (CRITICAL):**
```python
# SPARQL votazione URI: ".../vs19_029_089"
# → session_number=29, vote_number=89

def parse_votazione_uri(uri: str) -> tuple[int, int]:
    """Extract (session_number, vote_number) from votazione URI."""
    m = re.search(r'/vs\d+_(\d+)_(\d+)$', uri)
    if m:
        return int(m.group(1)), int(m.group(2))
    return None, None

# Cypher match:
# MATCH (s:Session {number: $session_num})-[:HAS_VOTE]->(v:Vote {number: $vote_num})
```

**SPARQL query for individual votes (per deputy, paginated):**
```sparql
# Source: verified with live endpoint at https://dati.camera.it/sparql

PREFIX ocd: <http://dati.camera.it/ocd/>

SELECT ?voto ?type ?votazione WHERE {
  ?voto a ocd:voto ;
        ocd:rif_deputato <http://dati.camera.it/ocd/deputato.rdf/d{person_id}_19> ;
        ocd:type ?type ;
        ocd:rif_votazione ?votazione .
  FILTER(CONTAINS(STR(?votazione), 'vs19_'))
}
LIMIT 1000 OFFSET 0
```

**SPARQL query for committee officer roles (per deputy):**
```sparql
PREFIX ocd: <http://dati.camera.it/ocd/>

SELECT ?carica ?organoLabel ?startDate ?endDate WHERE {
  ?up a ocd:ufficioParlamentare ;
      ocd:rif_deputato <http://dati.camera.it/ocd/deputato.rdf/d{person_id}_19> ;
      ocd:rif_leg <http://dati.camera.it/ocd/legislatura.rdf/repubblica_19> ;
      ocd:carica ?carica ;
      ocd:rif_organo ?organo ;
      ocd:startDate ?startDate .
  ?organo rdfs:label ?organoLabel .
  OPTIONAL { ?up ocd:endDate ?endDate }
}
```

**Neo4j write for ENR-01 (IndividualVote):**
```cypher
# Source: design based on confirmed SPARQL structure

UNWIND $batch AS row
MATCH (d:Deputy {id: row.deputyId})
MATCH (s:Session {number: row.sessionNumber})-[:HAS_VOTE]->(v:Vote {number: row.voteNumber})
MERGE (iv:IndividualVote {id: row.id})
SET iv.outcome = row.outcome
MERGE (d)-[:VOTED]->(iv)
MERGE (iv)-[:ON_VOTE]->(v)
```

**Outcome mapping:**
```python
OUTCOME_MAP = {
    "Favorevole": "favor",
    "Contrario": "against",
    "Astenuto": "abstain",
    "Non ha votato": "absent",
    "In missione": "on_mission",
}
```

**Neo4j write for ENR-02 (committee officer roles):**

The existing `MEMBER_OF_COMMITTEE` relationship lacks officer role info. Two options:
1. Add a `role` property to the existing relationship (requires MATCH+SET, not MERGE)
2. Add a separate `officerRole` property or a new relationship `OFFICER_OF_COMMITTEE`

Decision (Claude's discretion): Add `officerRole` string property to existing `MEMBER_OF_COMMITTEE` relationship where the deputy held an officer position. This avoids schema proliferation.

```cypher
UNWIND $batch AS row
MATCH (d:Deputy {id: row.deputyId})-[r:MEMBER_OF_COMMITTEE]->(c:Committee {name: row.committeeName})
WHERE r.start_date = date(row.startDate)
   OR r.start_date IS NULL
SET r.officerRole = row.role,
    r.officerRoleStart = date(row.startDate),
    r.officerRoleEnd = CASE WHEN row.endDate IS NOT NULL THEN date(row.endDate) ELSE NULL END
```

**Fallback if no matching MEMBER_OF_COMMITTEE exists:** Create a new `OFFICER_OF_COMMITTEE` relationship directly.

### Pattern 4: NER Integration at Ingestion Time

**What:** Run spaCy NER on each chunk text during `_create_chunks` to extract law references (regex) and person mentions (`PER` label). Store as `Chunk.lawRefs` and `Chunk.personRefs` string arrays.

**Critical finding: `it_core_news_lg` has NO LAW entity type.** The model supports only: `PER`, `ORG`, `LOC`, `MISC`. Law references MUST use regex.

**Italian parliamentary law reference patterns:**
```python
import re
import spacy

# Source: Italian legislative naming conventions

LAW_PATTERNS = [
    # Decreto legislativo / legge
    re.compile(r'\b(?:D\.?L(?:gs)?\.?|decreto(?:\s+legislativo)?|legge)\s+(?:n\.\s*)?\d+(?:/\d{4})?', re.I),
    # Specific decree formats: D.L. 231/2001, D.lgs. 50/2016
    re.compile(r'\bD\.?\s*(?:L(?:gs)?|lgs)\.?\s*\d+(?:[/-]\d{2,4})?', re.I),
    # Law article references: art. 10 comma 2
    re.compile(r'\bart(?:icolo)?\.?\s*\d+(?:\s+(?:comma|c\.)\s*\d+)?', re.I),
    # Constitution: art. 18 Cost.
    re.compile(r'\bCostituzione\b|\bCost\.\b', re.I),
]

def extract_law_refs(text: str) -> list[str]:
    refs = []
    for pat in LAW_PATTERNS:
        refs.extend(m.group(0).strip() for m in pat.finditer(text))
    # Deduplicate preserving order
    seen = set()
    return [r for r in refs if not (r in seen or seen.add(r))]

def extract_person_refs(doc) -> list[str]:
    """Extract PER entities from spaCy doc, confidence threshold via ent.kb_id_ or score."""
    return list({ent.text.strip() for ent in doc.ents if ent.label_ == "PER" and len(ent.text.strip()) > 2})
```

**Integration point in `db_builder.py`:**
```python
# In ingest_session(), before _batch_write(neo_session, _create_chunks, all_chunks):
# Load model once at DatabaseBuilder.__init__ (not per call):
#   self._nlp = spacy.load("it_core_news_lg")

# In the chunk loop:
for chunk in chunks:
    doc = self._nlp(chunk["text"])
    chunk["lawRefs"] = extract_law_refs(chunk["text"])
    chunk["personRefs"] = extract_person_refs(doc)
    chunk["speechId"] = speech["id"]
```

**Extended `_create_chunks` Cypher:**
```cypher
UNWIND $batch AS row
MERGE (c:Chunk {id: row.id})
SET c.text = row.text,
    c.index = row.index,
    c.lawRefs = row.lawRefs,
    c.personRefs = row.personRefs
WITH c, row
MATCH (sp:Speech {id: row.speechId})
MERGE (sp)-[:HAS_CHUNK]->(c)
```

**Performance note:** Loading `it_core_news_lg` for each `DatabaseBuilder` instance adds ~2-3 seconds startup. The NLP processing for 155K chunks at ~1000 tokens/sec (rough CPU estimate for large model) is approximately 2-3 hours. Consider using `nlp.pipe()` for batch processing with `disable=["parser", "tagger"]` to speed up NER-only extraction.

```python
# Faster batch NER:
texts = [chunk["text"] for chunk in all_chunks_for_session]
for doc, chunk in zip(nlp.pipe(texts, disable=["parser", "tagger"]), all_chunks_for_session):
    chunk["lawRefs"] = extract_law_refs(chunk["text"])
    chunk["personRefs"] = extract_person_refs(doc)
```

### Pattern 5: Entity-Filtered Retrieval in GraphChannel

**What:** When a query explicitly mentions a law (via regex) or a person name, filter graph channel results to only chunks containing matching `lawRefs`/`personRefs`.

**Integration point in `graph_channel.py`:**
```python
def retrieve(self, query: str, query_embedding: ..., entity_filter: dict = None, ...) -> ...:
    # entity_filter = {"laws": ["D.L. 231"], "persons": ["Mario Draghi"]}
    # Add WHERE clause to _get_chunks_from_signatories if filter present
    if entity_filter and entity_filter.get("laws"):
        # WHERE ANY(ref IN c.lawRefs WHERE ref CONTAINS $law_term)
        pass
```

This is a lightweight extension — only used when query-time NER detects explicit references.

### Anti-Patterns to Avoid
- **Using `session.run()` auto-commit for SPARQL writes:** All Neo4j writes MUST use `execute_write` managed transactions. The existing `_batch_write` pattern is correct.
- **Loading spaCy model per chunk:** Load once in `DatabaseBuilder.__init__`, reuse across all `ingest_session()` calls.
- **Requesting all XIX leg voti in one SPARQL query:** The endpoint times out on large result sets. Always paginate with LIMIT/OFFSET per deputy.
- **Hardcoding the `k=60` RRF constant without config:** Put in `default.yaml` under `retrieval.rrf.k` for tunability.
- **Mixing BM25 raw scores with cosine similarity scores before RRF:** Use rank position only, not raw score values, when computing RRF. Raw BM25 scores are unbounded and incomparable to cosine [0,1].

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Italian text search | Custom inverted index | Neo4j full-text index (Lucene) | Built-in BM25, zero deps, same DB |
| Italian NER | Custom regex-only NER | spaCy `it_core_news_lg` for PER | Model trained on WikiNER Italian |
| SPARQL HTTP client | Custom HTTP wrapper | `urllib.request` with url-encoding | Sufficient for GET/POST SPARQL; no auth needed |
| RRF implementation | Complex custom ranking | Simple rank-based formula | 3 lines of Python; don't over-engineer |

**Key insight:** Neo4j's full-text index is the correct tool here — it avoids bringing in Elasticsearch, Typesense, or any external search engine. The trade-off is less tuning flexibility, but for 155K chunks this is entirely sufficient.

---

## Common Pitfalls

### Pitfall 1: SPARQL Endpoint Timeouts on COUNT or Large Range Queries
**What goes wrong:** Queries without FILTER or with COUNT(*) on `ocd:voto` time out (>25 seconds) on the endpoint. This was observed during research — `COUNT(*)` on all voti failed; targeted per-deputy queries with LIMIT succeeded.
**Why it happens:** The SPARQL store is a Virtuoso instance; full table scans on large graphs time out.
**How to avoid:** Always add a specific subject filter (per deputy URI). Paginate with LIMIT/OFFSET. Set HTTP timeout to 30 seconds per request; retry once on timeout.
**Warning signs:** `urllib.error.URLError: timed out` or 504 Gateway Timeout.

### Pitfall 2: SPARQL endpoint returns duplicate triples
**What goes wrong:** The `vs19_029_089` votazione query showed duplicate `rdf:type` and other triples. Neo4j SPARQL ingestion must use MERGE, not CREATE, to avoid duplicate `IndividualVote` nodes.
**How to avoid:** Always use `MERGE (iv:IndividualVote {id: row.id})` — never `CREATE`.

### Pitfall 3: BM25 score is unnormalized — do NOT use it as a similarity value
**What goes wrong:** `db.index.fulltext.queryNodes` returns a `score` that is an unnormalized Lucene BM25 score (e.g., 8.4, 12.1, 0.3). If this is passed as `similarity` to the merger (which expects [0,1]), the scoring breaks.
**How to avoid:** In `SparseChannel._process_results()`, store the raw score in `bm25_score` and set `similarity = 0.5` (neutral placeholder). Use only rank position for RRF computation. Do NOT normalize BM25 scores across queries (they are query-dependent).

### Pitfall 4: Deputy ID mismatch between SPARQL and Neo4j
**What goes wrong:** SPARQL `ocd:rif_deputato` gives `deputato.rdf/d308908_19`; Neo4j `Deputy.id` stores `persona.rdf/p308908`. A direct string match will find no deputies.
**How to avoid:** Always extract the numeric person ID from the SPARQL URI (`d(\d+)_\d+`) and reconstruct the Neo4j URI (`persona.rdf/p{id}`). This logic belongs in `sparql_ingester.py`.

### Pitfall 5: spaCy model not available in build venv
**What goes wrong:** `build/requirements-build.txt` only has `pandas` and `regex`. Adding NER requires `spacy>=3.5.0` and downloading `it_core_news_lg` in the build environment.
**How to avoid:** Add `spacy>=3.5.0` to `build/requirements-build.txt`. Add `python -m spacy download it_core_news_lg` to `make db-install`.

### Pitfall 6: Neo4j full-text index `IF NOT EXISTS` does not update analyzer config
**What goes wrong:** If `chunk_fulltext` was created with the default analyzer (standard), adding `IF NOT EXISTS` in a subsequent run will silently skip the creation without applying `italian` analyzer.
**How to avoid:** During `nuke_database()`, the existing code already drops all non-LOOKUP indexes. The full rebuild path (`make db-all`) will always recreate the index fresh with the correct analyzer.

### Pitfall 7: Vote matching fails when Vote.number is None
**What goes wrong:** Some XML `<votazione>` elements may lack a `<numero>` element, causing `Vote.number = NULL`. The SPARQL match `WHERE v.number = $vote_num` will fail silently.
**How to avoid:** Add a fallback in `sparql_ingester.py`: if no Vote is matched, log a warning and skip (do not fail). Track match rate in logs.

---

## Code Examples

### Full-Text Index Creation (db_builder.py)
```python
# Source: https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/full-text-indexes/
def create_fulltext_index(self) -> None:
    """Create full-text index on Chunk.text for BM25 sparse retrieval."""
    with self._driver.session() as neo_session:
        neo_session.execute_write(lambda tx: tx.run("""
            CREATE FULLTEXT INDEX chunk_fulltext IF NOT EXISTS
            FOR (n:Chunk) ON EACH [n.text]
            OPTIONS {indexConfig: {`fulltext.analyzer`: 'italian'}}
        """))
    print("Full-text index created.")
```

### SparseChannel.retrieve() skeleton
```python
# Source: mirrors DenseChannel interface; Neo4j full-text index docs
def retrieve(self, query_text: str, top_k: int = 100) -> List[Dict[str, Any]]:
    sparse_config = self.config.retrieval.get("sparse_channel", {})
    top_k = top_k or sparse_config.get("top_k", 100)
    index_name = sparse_config.get("index_name", "chunk_fulltext")

    cypher = """
    CALL db.index.fulltext.queryNodes($index_name, $query_text)
    YIELD node AS c, score
    MATCH (c)<-[:HAS_CHUNK]-(i:Speech)-[:SPOKEN_BY]->(speaker)
    MATCH (i)<-[:CONTAINS_SPEECH]-(f:Phase)<-[:HAS_PHASE]-(d:Debate)<-[:HAS_DEBATE]-(s:Session)
    OPTIONAL MATCH (speaker)-[mg:MEMBER_OF_GROUP]->(g:ParliamentaryGroup)
    WHERE mg.start_date <= s.date AND (mg.end_date IS NULL OR mg.end_date >= date())
    OPTIONAL MATCH (speaker)-[mg_now:MEMBER_OF_GROUP]->(g_now:ParliamentaryGroup)
    WHERE mg_now.end_date IS NULL
    RETURN c.id AS chunk_id, c.text AS chunk_text, score AS bm25_score,
           i.id AS speech_id, i.text AS text,
           speaker.id AS speaker_id, speaker.first_name AS speaker_first_name,
           speaker.last_name AS speaker_last_name,
           CASE WHEN 'GovernmentMember' IN labels(speaker) THEN 'GovernmentMember' ELSE 'Deputy' END AS speaker_type,
           g.name AS party, g_now.name AS current_party,
           s.id AS session_id, s.date AS session_date, s.number AS session_number,
           d.title AS debate_title
    LIMIT $top_k
    """
    results = self.client.query(cypher, {"index_name": index_name, "query_text": query_text, "top_k": top_k})
    return self._process_results(results)
```

### RRF Merger (merger.py extension)
```python
# Source: Cormack et al. (2009) RRF paper; k=60 industry standard

def merge(self, dense_results, sparse_results, graph_results, authority_scores=None, top_k=100):
    """RRF-based merge of three channels."""
    rrf_config = self.config.retrieval.get("rrf", {})
    k = rrf_config.get("k", 60)
    w_dense = rrf_config.get("dense_weight", 1.0)
    w_sparse = rrf_config.get("sparse_weight", 0.8)
    w_graph = rrf_config.get("graph_weight", 0.5)

    # Build rank maps per channel
    channels = [
        (dense_results, w_dense),
        (sparse_results, w_sparse),
        (graph_results, w_graph),
    ]

    rrf_scores: Dict[str, float] = defaultdict(float)
    all_results: Dict[str, Dict] = {}

    for results, weight in channels:
        for rank, result in enumerate(results, start=1):
            eid = result.get("evidence_id", "")
            if not eid:
                continue
            rrf_scores[eid] += weight / (k + rank)
            if eid not in all_results:
                all_results[eid] = result

    # Attach RRF score as similarity for downstream use
    for eid, result in all_results.items():
        result["similarity"] = rrf_scores[eid]
        result["rrf_score"] = rrf_scores[eid]

    # Sort by RRF score
    sorted_results = sorted(all_results.values(), key=lambda x: x["rrf_score"], reverse=True)

    # Apply diversity selection (existing _select_diverse logic)
    return self._select_diverse(sorted_results, top_k)
```

### SPARQL Ingester Skeleton (sparql_ingester.py)
```python
# Source: verified live SPARQL endpoint https://dati.camera.it/sparql

SPARQL_ENDPOINT = "https://dati.camera.it/sparql"

def _sparql_get(query: str, timeout: int = 30) -> list[dict]:
    """Execute SPARQL SELECT query; return list of binding dicts."""
    import urllib.request, urllib.parse, json
    url = SPARQL_ENDPOINT + "?" + urllib.parse.urlencode({
        "query": query, "format": "application/sparql-results+json"
    })
    try:
        with urllib.request.urlopen(url, timeout=timeout) as resp:
            data = json.load(resp)
            return data["results"]["bindings"]
    except Exception as e:
        logger.warning(f"SPARQL query failed: {e}")
        return []

def get_deputy_votes(dep_sparql_uri: str, offset: int = 0) -> list[dict]:
    """Fetch one page of individual votes for a deputy."""
    query = f"""
    PREFIX ocd: <http://dati.camera.it/ocd/>
    SELECT ?voto ?type ?votazione WHERE {{
      ?voto a ocd:voto ;
            ocd:rif_deputato <{dep_sparql_uri}> ;
            ocd:type ?type ;
            ocd:rif_votazione ?votazione .
      FILTER(CONTAINS(STR(?votazione), 'vs19_'))
    }}
    LIMIT 1000 OFFSET {offset}
    """
    return _sparql_get(query)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Dense-only retrieval | Dense + Graph dual channel | Phase 1-2 | Existing |
| Custom weighted scoring in merger | RRF rank fusion | Phase 4 | Simpler, principled hybrid ranking |
| No entity annotations on chunks | `lawRefs`/`personRefs` on Chunk | Phase 4 | Enables entity-filtered graph retrieval |
| No individual vote data | `Deputy-[:VOTED]->IndividualVote-[:ON_VOTE]->Vote` | Phase 4 | Enables vote-based authority enrichment |

**Deprecated / Not Applicable:**
- `it_nerIta_trf` (bullmount): Requires `spaCy >=3.2.1,<3.3.0` — INCOMPATIBLE with `spacy>=3.5.0` in requirements.txt. Do not use.
- BM25 via Elasticsearch/Typesense: Out of scope; Neo4j native is the chosen approach.

---

## Open Questions

1. **SPARQL vote data volume per deputy**
   - What we know: 34,237 total votazioni in XIX leg; individual voto records number in the millions (count query timed out).
   - What's unclear: Average votes per deputy. If a deputy has 20,000+ votes, pagination at LIMIT 1000 means 20+ HTTP requests per deputy, multiplied by ~600 deputies = potentially 12,000+ requests total.
   - Recommendation: Implement per-deputy pagination with exponential backoff; measure actual rate in integration test before full run. Consider adding `--limit-deputies N` flag for testing.

2. **Neo4j full-text index `italian` analyzer availability**
   - What we know: Neo4j 5.x ships with multiple analyzers including `italian` (Lucene Italian analyzer with stemming).
   - What's unclear: Whether the Docker image version in `docker-compose.yml` supports `italian`. If not, fall back to `standard` (no stemming).
   - Recommendation: In `create_fulltext_index()`, catch exception and retry with `standard` analyzer as fallback.

3. **Merger refactor: preserve existing scoring or replace with RRF**
   - What we know: Current merger has relevance/diversity/coverage/authority/salience weights. RRF replaces the pre-merge ranking, but diversity/coverage in `_select_diverse` should be preserved.
   - What's unclear: Whether to replace `_compute_scores` entirely or layer RRF on top of existing scores.
   - Recommendation: Use RRF score as the `similarity` field (replacing raw cosine similarity), then keep `_select_diverse` as-is. This is the minimal-risk change.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest (existing in backend/tests/) + build/tests/ |
| Config file | backend/pytest.ini or discovered automatically |
| Quick run command | `cd backend && python -m pytest tests/unit/ -x -q` |
| Full suite command | `cd backend && python -m pytest -v --tb=short` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RET-01 | SparseChannel returns evidence list from full-text index | unit | `python -m pytest tests/unit/test_sparse_channel.py -x` | ❌ Wave 0 |
| RET-02 | RRF merger produces correct ranked scores for 3 channels | unit | `python -m pytest tests/unit/test_rrf_merger.py -x` | ❌ Wave 0 |
| ENR-01 | IndividualVote nodes created with correct outcome mapping | unit | `python -m pytest build/tests/test_sparql_ingester.py::test_vote_outcome_mapping -x` | ❌ Wave 0 |
| ENR-02 | Committee officer roles set on MEMBER_OF_COMMITTEE | unit | `python -m pytest build/tests/test_sparql_ingester.py::test_committee_officer_role -x` | ❌ Wave 0 |
| ENR-03 | extract_law_refs regex matches D.L., legge, art. patterns | unit | `python -m pytest build/tests/test_ner.py::test_law_ref_extraction -x` | ❌ Wave 0 |
| ENR-04 | Chunk nodes have lawRefs/personRefs list properties | unit | `python -m pytest build/tests/test_db_builder.py::test_chunk_ner_properties -x` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `cd backend && python -m pytest tests/unit/ -x -q --tb=short`
- **Per wave merge:** `cd backend && python -m pytest -v --tb=short && cd ../build && python -m pytest tests/ -v --tb=short`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `backend/tests/unit/test_sparse_channel.py` — covers RET-01
- [ ] `backend/tests/unit/test_rrf_merger.py` — covers RET-02
- [ ] `build/tests/test_sparql_ingester.py` — covers ENR-01, ENR-02 (uses mocked HTTP responses)
- [ ] `build/tests/test_ner.py` — covers ENR-03 (pure function, no DB)
- [ ] Extend `build/tests/test_db_builder.py` — covers ENR-04

---

## Sources

### Primary (HIGH confidence)
- Neo4j Cypher Manual — Full-Text Indexes: https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/full-text-indexes/ — createindex syntax, queryNodes procedure, `fulltext.analyzer` option, score description
- dati.camera.it SPARQL endpoint: https://dati.camera.it/sparql — verified live during research; queried `ocd:voto`, `ocd:votazione`, `ocd:ufficioParlamentare`, `ocd:incarico` predicates
- spaCy it_core_news_lg model card: https://huggingface.co/spacy/it_core_news_lg — confirmed entity labels: PER, ORG, LOC, MISC only (NO LAW)
- `backend/app/services/retrieval/dense_channel.py` — interface to mirror for SparseChannel
- `backend/app/services/retrieval/merger.py` — structure to extend with RRF
- `backend/app/services/retrieval/engine.py` — orchestration to extend with third channel
- `build/db_builder.py` — integration points for fulltext index + NER

### Secondary (MEDIUM confidence)
- RRF paper (Cormack, Clarke, Buettcher 2009) — k=60 standard, weighted RRF formula; cross-confirmed by Azure AI Search docs and OpenSearch blog
- Italian parliamentary law reference patterns — derived from corpus knowledge; regex patterns are hypotheses to be validated against actual chunk text

### Tertiary (LOW confidence)
- spaCy Italian model CPU throughput estimates (~1000 tokens/sec for lg model) — estimate from community benchmarks; actual speed depends on hardware
- SPARQL per-deputy vote count estimate — could not measure due to timeout; "millions" is an inference from 34,237 votazioni × ~600 deputies

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries in existing requirements.txt; SPARQL endpoint verified live
- Architecture (BM25+RRF): HIGH — Neo4j API verified, RRF formula is standard
- Architecture (SPARQL): HIGH for vocabulary/URIs (verified live), MEDIUM for volume/timing
- Architecture (NER): HIGH for entity labels (confirmed no LAW), MEDIUM for regex patterns
- Pitfalls: HIGH — most derived from live research observations

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (SPARQL vocabulary stable; spaCy model releases infrequently)
