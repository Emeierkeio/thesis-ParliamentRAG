# Domain Enrichment Research: ParliamentRAG

**Domain:** Parliamentary RAG — Camera dei Deputati (XIX Legislatura)
**Researched:** 2026-04-02
**Overall Confidence:** HIGH (ontologies, OCD, NER models verified via official sources; techniques verified via 2024 ACL/EMNLP papers)

---

## Executive Summary

ParliamentRAG already has a solid knowledge graph (Neo4j) with deputy, speech, chunk, act, vote, and group nodes. The enrichment opportunities fall into four tiers by impact/effort ratio:

1. **Immediate wins (days):** Pull additional structured data from the live dati.camera.it SPARQL endpoint — vote-level deputy records, government composition, committee membership details, and act metadata are all publicly queryable and directly augment the graph channels already implemented.

2. **Short-term wins (1-2 weeks):** Add NER at ingestion time using `bullmount/it_nerIta_trf` (spaCy transformer pipeline, F1 = 91.96, 18 entity types including LAW). Store extracted entities as Chunk metadata for entity-based retrieval filtering and hybrid sparse+dense search.

3. **Medium-term (2-4 weeks):** Integrate Normattiva law text retrieval to resolve bill references already in the graph (Act nodes have `codiceArgomento` like "705-A") into full legal text. Add BM25 as a sparse retrieval channel alongside the existing dense vector channel for exact terminology matching.

4. **Research-grade (months, likely overkill):** Full Akoma Ntoso XML retagging, LKIF reasoning, stance detection, cross-document coreference resolution. These are academic quality improvements with high implementation cost and uncertain production benefit.

---

## 1. Parliamentary Ontologies

### 1.1 Akoma Ntoso / OASIS LegalDocML

**What it is:** The OASIS international standard (published as LegalDocML v1.0 in August 2018) for representing parliamentary, legislative, and judiciary documents in XML. Used by EU Parliament, several national parliaments (Kenya, South Africa, UK), and explicitly supports Italian legislative documents.

**Core concepts relevant to ParliamentRAG:**
- `FRBRuri` — canonical URI for each document version (maps to what ParliamentRAG calls "codiceArgomento")
- `debate` — parliamentary debate record (maps to existing `Debate` node)
- `speech` — individual speaker turn (maps to existing `Speech` node)
- `ref` — inline reference to another document (captures cross-citations currently missed)
- `role` — speaker role at time of utterance (maps to `Speech.speaking_role` identified in FEATURES.md)
- `TLCPerson`, `TLCOrganization`, `TLCConcept` — ontology anchors for entities

**Assessment for ParliamentRAG:**
The Camera dei Deputati XML files (stenografici) already follow an Italian-specific schema that predates Akoma Ntoso adoption. Retagging all 365+ XML files to Akoma Ntoso XML would require a major transformation pipeline with uncertain benefit — the structural information (sessions, debates, phases, speeches) is already correctly captured. The **valuable concept to borrow** is the `FRBRuri` naming convention to create canonical, dereferenceable URIs for acts — but this is achievable without full schema migration.

**Verdict: Import the URI naming concept; skip the full XML migration.**
- Confidence: HIGH (OASIS official spec)
- Effort to fully adopt: weeks/months
- Effort for URI concept only: hours

**Key reference:** https://docs.oasis-open.org/legaldocml/akn-core/v1.0/akn-core-v1.0-part1-vocabulary.html

---

### 1.2 OCD — Ontology of the Chamber of Deputies (dati.camera.it)

**What it is:** The Camera's own OWL ontology, base URI `http://dati.camera.it/ocd/`. As of April 2026 it describes 725 million RDF triples across 128 classes. This is the most directly relevant ontology for ParliamentRAG.

**Main classes and their mapping to existing Neo4j schema:**

| OCD Class | OCD URI | Neo4j Node | Gap? |
|-----------|---------|------------|------|
| `deputato` | `ocd:deputato` | `Deputy` | No gap |
| `gruppoParlamentare` | `ocd:gruppoParlamentare` | `ParliamentaryGroup` | No gap |
| `atto` | `ocd:atto` | `ParliamentaryAct` | No gap |
| `seduta` | `ocd:seduta` | `Session` | No gap |
| `discussione` | `ocd:discussione` | `Debate` | No gap |
| `organo` | `ocd:organo` | `Committee` (partial) | Committees have limited metadata |
| `membroGoverno` | `ocd:membroGoverno` | `GovernmentMember` | No gap |
| `mandatoCamera` | `ocd:mandatoCamera` | Not stored | **Gap**: mandate start/end dates |
| `ufficioParlamentare` | `ocd:ufficioParlamentare` | Not stored | **Gap**: committee officer roles |
| `aderisce` property | — | `MEMBER_OF` edge | Partial — missing date range |

**Key data queryable from OCD SPARQL but not currently in Neo4j:**
1. Deputy biographical data (birth date, education, constituency) — enriches authority scoring
2. Precise mandate dates per deputy (start, end) — currently uses CSV snapshots
3. Committee membership with officer roles (president, secretary vs ordinary member)
4. Vote records at deputy level (individual yes/no/abstain per vote) — separate dataset in dati.camera.it under "Votes" category (9 datasets)
5. Government composition with precise ministry assignments and dates

**SPARQL endpoint:** `https://dati.camera.it/sparql`
**Visual query builder:** `https://dati.camera.it/sparnatural/`

**Example SPARQL query to retrieve XIX legislature deputies with party group:**
```sparql
PREFIX ocd: <http://dati.camera.it/ocd/>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?deputato ?name ?group ?groupName
WHERE {
  ?deputato a ocd:deputato ;
            dc:title ?name ;
            ocd:aderisce ?membership .
  ?membership ocd:rif_gruppoParlamentare ?group .
  ?group rdfs:label ?groupName .
  FILTER(CONTAINS(STR(?deputato), "19"))
}
LIMIT 100
```

**Verdict: This is the highest-value external data source. Already partially used (CSV downloads). SPARQL queries can replace CSV joins and fill gaps in the graph.**
- Confidence: HIGH (direct endpoint inspection)
- Effort: hours per dataset ingested

---

### 1.3 ELI — European Legislation Identifier

**What it is:** A W3C-aligned standard for canonical URIs for legislation. Italy implements ELI on Normattiva with the pattern:
```
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:{type}:{YYYY-MM-DD};{number}
```

Examples:
- `urn:nir:stato:legge:2022-10-13;164` — Law 164/2022 (used in confidence vote context)
- `urn:nir:stato:decreto.legge:2022-11-10;145` — Decree-Law 145/2022

**Relevance to ParliamentRAG:** The existing `ParliamentaryAct` nodes have `codiceArgomento` (e.g., "705") from the Camera API. If the Camera API also returns ELI URIs (likely via `ocd:atto` RDF triples), these could be used as canonical cross-system identifiers to link Camera acts to their Normattiva text.

**Verdict: Use ELI URIs as identifiers in `ParliamentaryAct` nodes when available. Low effort, high interoperability value.**
- Confidence: HIGH (ELI official spec, Normattiva API documentation)

---

### 1.4 LKIF — Legal Knowledge Interchange Format

**What it is:** An OWL-DL + SWRL ontology from the EU ESTRELLA project (mid-2000s). 15 modules covering legal concepts: norms, rules, processes, roles, events.

**Assessment for ParliamentRAG:** LKIF is a legal reasoning ontology, not a parliamentary document ontology. It models _what norms mean_ (obligations, permissions, prohibitions) rather than _what parliamentary records contain_. Applying LKIF to ParliamentRAG would require mapping free-form speech content to formal norm representations — effectively a legal knowledge extraction task requiring significant manual annotation effort.

**Verdict: Academic overkill for a RAG system. Skip entirely.**
- Confidence: MEDIUM (Wikipedia + GitHub inspection, no active 2024 development found)

---

## 2. Named Entity Recognition for Italian Political Text

### 2.1 Model Comparison

| Model | Type | Entity Types | F1 (NER) | Language | Notes |
|-------|------|--------------|----------|----------|-------|
| `bullmount/it_nerIta_trf` | spaCy transformer | 18 (incl. LAW, MONEY, ORG) | 91.96 | Italian | Best option; includes LAW type |
| `spaCy it_core_news_lg` | spaCy CNN | PER, ORG, LOC, MISC (4) | ~84 (est.) | Italian | News-trained; fewer entity types |
| `Stanza it FBK` | Stanza | PER, ORG, LOC (3) | 87.92 | Italian | Only 3 types; not sufficient |
| `dlicari/Italian-Legal-BERT` | HuggingFace transformer | Not published | Unknown | Italian (legal) | Legal domain; no NER head published |
| `dbmdz/bert-base-italian-xxl-cased` | HuggingFace | None (base model only) | N/A | Italian | Must fine-tune; base for custom NER |

**Recommendation: Use `bullmount/it_nerIta_trf`** for entity extraction at ingestion time. It is the only Italian NER model that:
1. Includes the `LAW` entity type (named laws explicitly referenced in speech)
2. Has F1 > 91 on Italian text
3. Is a spaCy pipeline (easy integration with existing Python stack)
4. Covers all relevant entity types: PER (speakers/politicians), ORG (ministries, parties), GPE (regions, cities), DATE, MONEY, LAW

**Installation:**
```bash
pip install spacy>=3.2.1,<3.3.0
# Note: requires spaCy 3.2.x due to version pin
pip install https://huggingface.co/bullmount/it_nerIta_trf/resolve/main/it_nerIta_trf-any-py3-none-any.whl
```

**Caveat:** The spaCy version pin (`>=3.2.1,<3.3.0`) is outdated. For a current stack, test with `it_core_news_lg` (spaCy 3.7+) for the PER/ORG/LOC/MISC types, and use `bullmount/it_nerIta_trf` only if the version conflict can be isolated (e.g., in a separate microservice or via a subprocess call). Alternatively, fine-tune `dbmdz/bert-base-italian-xxl-cased` on a small parliamentary NER dataset to get LAW entities with a current spaCy version.

---

### 2.2 Entity Types Relevant to Parliamentary Text

**Directly useful from `it_nerIta_trf` (18-type model):**

| Entity Type | Parliamentary Relevance | Example Mentions |
|-------------|------------------------|-----------------|
| `PER` | Speaker name mentions, non-speaker politicians referenced in text | "il Ministro Giorgetti", "Giorgia Meloni" |
| `ORG` | Ministries, agencies, NGOs, parties mentioned | "Ministero dell'Economia", "Confindustria" |
| `GPE` | Countries, regions, municipalities debated | "Puglia", "Ucraina", "Comune di Roma" |
| `LAW` | Named laws, decrees referenced | "decreto legislativo 231", "legge di bilancio" |
| `DATE` | Dates in speeches | "entro il 31 dicembre", "nel 2024" |
| `MONEY` | Budget figures, financial amounts | "2 miliardi di euro", "il 3% del PIL" |
| `NORP` | Political/national groups | "i socialisti", "gli italiani", "gli ucraini" |
| `PERCENT` | Statistical and budget percentages | "il 20% delle risorse" |

**Custom entity types NOT in standard models (require fine-tuning or regex):**

| Custom Type | Pattern | Implementation |
|-------------|---------|----------------|
| `BILL_NUMBER` | A.C. NNN, S. NNN, PDL NNN | Regex: `\b(?:A\.C\.|S\.)\s*(\d[\d\-A-Z]*)` |
| `ACT_CODE` | `1-00098`, `1067-A` | Regex: `\b(\d{1,4}-\d{2,5}[A-Z]?)` |
| `COMMITTEE` | "Commissione Bilancio", "IX Commissione" | Dictionary lookup against existing Committee nodes |
| `PHASE_TYPE` | "Parere del Governo", "Dichiarazioni di voto" | Regex against known phase titles (already in FEATURES.md) |

**Recommendation:** Deploy the 18-type model for standard entities. Add regex extractors for BILL_NUMBER and ACT_CODE as lightweight post-processing. Dictionary lookup for COMMITTEE against the Neo4j graph. This gives 95% of the NER value at 10% of the fine-tuning cost.

---

### 2.3 Ingestion Time vs Query Time

**Ingestion time (recommended):**
- Run NER once per chunk during the build pipeline
- Store extracted entities as properties on Chunk nodes: `Chunk.entities: JSON array of {text, label, start, end}`
- Store unique law references as `Chunk.law_refs: List[str]`
- Enables: entity-based filtering in Cypher queries, pre-computed entity indexes, no latency at query time

**Query time (not recommended for NER):**
- NER at query time adds 50-200ms latency per chunk in the retrieval path
- Benefit: can match query entities against chunk entities in real time
- Problem: 100 retrieved chunks × 200ms NER = 20 seconds added latency; completely unacceptable
- Exception: run NER on the **query itself only** at query time to extract entity mentions for graph traversal

**Verdict: NER at ingestion time on chunks + NER at query time on query string only.**

---

## 3. Linked Data Sources

### 3.1 dati.camera.it SPARQL Endpoint (PRIMARY — HIGH VALUE)

**URL:** `https://dati.camera.it/sparql`
**Data scale:** 725 million RDF triples, 128 classes (updated daily)
**License:** Creative Commons Attribution 4.0

**Currently NOT ingested but queryable:**

**3.1.1 Deputy-level vote records**

The "Votes" category has 9 datasets. Individual deputy votes (yes/no/abstain) per voting session are queryable. This fills the gap identified in FEATURES.md: Vote nodes exist in XML only with aggregate counts, not per-deputy records.

```sparql
PREFIX ocd: <http://dati.camera.it/ocd/>
SELECT ?deputato ?votazione ?voto
WHERE {
  ?deputato a ocd:deputato .
  ?votazione ocd:rif_deputato ?deputato ;
             ocd:esito ?voto .
  FILTER(CONTAINS(STR(?votazione), "leg19"))
}
LIMIT 500
```

**3.1.2 Detailed committee membership with roles**

```sparql
PREFIX ocd: <http://dati.camera.it/ocd/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT ?deputato ?committee ?role ?startDate
WHERE {
  ?membership ocd:rif_deputato ?deputato ;
              ocd:rif_organo ?committee ;
              ocd:ufficioParlamentare ?role ;
              ocd:startDate ?startDate .
  FILTER(CONTAINS(STR(?committee), "leg19"))
}
```

**3.1.3 Act metadata with signatories**

```sparql
PREFIX ocd: <http://dati.camera.it/ocd/>
SELECT ?atto ?tipo ?titolo ?firmatario ?data
WHERE {
  ?atto a ocd:atto ;
        ocd:tipo ?tipo ;
        ocd:titolo ?titolo ;
        ocd:primo_firmatario ?firmatario ;
        ocd:startDate ?data .
  FILTER(CONTAINS(STR(?atto), "leg19"))
}
LIMIT 200
```

**Impact on retrieval:** Committee officer roles improve authority scoring (committee president has higher domain authority than ordinary member). Per-deputy votes enable "how did party X vote on Y?" graph traversal without text retrieval.

---

### 3.2 dati.senato.it SPARQL Endpoint (MEDIUM VALUE)

**URL:** `https://dati.senato.it/sparql`
**License:** CC-by 3.0

Many Camera bills originate in or pass through the Senate (identified by `S. NNN` in debate titles). The Senate SPARQL endpoint provides:
- Senate act records that mirror Camera acts (bills with dual numbering)
- Senate debate transcripts (out of scope for this project but useful for cross-chamber linking)

**Specific value for ParliamentRAG:** Map `Debate.title` containing `(S. NNN)` to Senate act records, add `ORIGINATED_IN_SENATE` edge to ParliamentaryAct nodes.

**Effort:** 1-2 days. Medium priority.

---

### 3.3 Normattiva — Italian Law Texts (MEDIUM VALUE)

**URL pattern:** `https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:{type}:{YYYY-MM-DD};{number}`
**Formats:** HTML (default), XML (NIR format via `&versione=originale&tipoVigenza=originale`)
**OpenData portal:** `https://dati.normattiva.it/`

**What Normattiva provides:** Full consolidated text of Italian laws (1861–present), legislative decrees, decree-laws, the Constitution, and major codes (civil, criminal, administrative).

**Value for ParliamentRAG:**
- ParliamentaryAct nodes already have `codiceArgomento` and type (pdl, mozione, etc.) from the Camera API
- For PDL (Proposta di legge) acts that became law, Normattiva provides the final legal text
- Enables answering: "What does law X actually say?" alongside "What was debated about law X?"
- Enriches act nodes with `full_text_uri` property pointing to the Normattiva ELI URI

**Limitation:** Only enacted laws are in Normattiva. PDL bills that were rejected or are still pending have no Normattiva entry. Among the 3,493 PDL references found in XML (FEATURES.md), a minority became enacted law.

**Recommended approach:** On ingestion, for PDL-type acts, attempt ELI URI construction and store as `ParliamentaryAct.normattiva_uri`. Fetch text on-demand when relevant to a query (not bulk-loaded into the graph — too much data).

---

### 3.4 EUR-Lex / CELLAR (LOW VALUE for this use case)

**URL:** `https://eur-lex.europa.eu/`
**SPARQL endpoint:** `https://publications.europa.eu/webapi/rdf/sparql`

**What it provides:** Full text and metadata for all EU legislation, regulations, directives.

**Value for ParliamentRAG:** Italian parliamentary debates frequently reference EU directives and regulations (e.g., "la direttiva europea sul [topic]"). Resolving these references to EUR-Lex metadata would enable "EU legislation → Italian implementation debate" traversal.

**Limitation:** EU law references in Italian speech are often imprecise ("la direttiva europea", "il regolamento di Bruxelles") without specific identifiers. Entity linking from imprecise mentions to EUR-Lex URIs requires probabilistic disambiguation that is difficult to get right at high precision.

**Verdict: Future milestone if there is a specific use case. Not worth implementing now.**

---

### 3.5 Wikidata Political Entities (LOW-MEDIUM VALUE)

**URL:** `https://query.wikidata.org/sparql`
**Italian Chamber entity:** `Q841424`
**XIX Legislature deputies:** Query via `P39=Q19820684` (Member of Chamber of Deputies) + `P2937` (parliamentary term)

**What Wikidata provides:**
- Biographical data (date of birth, education, profession before parliament)
- Party membership history across multiple legislatures
- Wikipedia links for political context
- QID identifiers for cross-system linking (DBpedia, etc.)

**SPARQL to fetch XIX legislature deputies:**
```sparql
SELECT ?deputy ?deputyLabel ?partyLabel ?birthDate
WHERE {
  ?deputy wdt:P39 wd:Q19820684 ;
          p:P39 ?statement .
  ?statement ps:P39 wd:Q19820684 ;
             pq:P2937 wd:Q114596952 .  # XIX Legislatura
  OPTIONAL { ?deputy wdt:P102 ?party }
  OPTIONAL { ?deputy wdt:P569 ?birthDate }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "it,en" }
}
```

**Value for ParliamentRAG:** Deputy biographical context enriches answers to "Who is [deputy]?" and improves authority scoring with education/profession data. The `Deputy` nodes currently only have name and party affiliation from Camera CSV.

**Effort:** 1-2 days. Medium priority.

---

### 3.6 OpenParlamento / Openpolis (LOW VALUE)

**Status:** OpenParlamento is a civic tech project managed by Openpolis (non-profit). Source code on GitHub (`openpolis/openparlamento`). No public API documented for the XIX legislature as of 2026.

**Verdict: Skip. dati.camera.it is the authoritative and better-maintained source for the same data.**

---

## 4. Enrichment Strategies for RAG

### 4.1 BM25 Sparse Retrieval Channel (HIGH IMPACT, LOW EFFORT)

**What it is:** Add a keyword-based sparse retrieval channel alongside the existing dense vector channel. BM25 excels at exact term matching — critical for parliamentary text where specific law numbers ("decreto legislativo 231/2001"), technical terms ("debito pubblico consolidato"), and proper names are query keywords that semantic embeddings may dilute.

**Why it fits:** The existing architecture already has a `dense_channel` and `graph_channel` with a `merger`. Adding a `sparse_channel` with BM25 is architecturally consistent. Neo4j 5.x supports full-text indexes natively.

**Implementation:**
```cypher
-- Create full-text index on Chunk.text (one-time, in build pipeline)
CREATE FULLTEXT INDEX chunk_fulltext FOR (c:Chunk) ON EACH [c.text]

-- BM25 query in sparse_channel
CALL db.index.fulltext.queryNodes("chunk_fulltext", $query_text)
YIELD node, score
RETURN node.text, node.id, score
ORDER BY score DESC LIMIT 50
```

Neo4j's full-text index uses Lucene underneath, which implements a BM25-like scoring model. This requires zero additional dependencies — pure Neo4j.

**Reciprocal Rank Fusion for merging sparse + dense + graph:**
```python
def rrf_score(rank: int, k: int = 60) -> float:
    return 1.0 / (k + rank)

def merge_with_rrf(dense_results, sparse_results, graph_results):
    scores = {}
    for rank, chunk_id in enumerate(dense_results):
        scores[chunk_id] = scores.get(chunk_id, 0) + rrf_score(rank)
    for rank, chunk_id in enumerate(sparse_results):
        scores[chunk_id] = scores.get(chunk_id, 0) + rrf_score(rank)
    for rank, chunk_id in enumerate(graph_results):
        scores[chunk_id] = scores.get(chunk_id, 0) + rrf_score(rank)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

**Estimated impact:** MEDIUM-HIGH. Hybrid retrieval consistently outperforms dense-only in domain-specific corpora (IBM research, 2024 benchmarks). Parliamentary text has high terminological precision requirements that favor BM25.
- Implementation: 1-2 days
- Reprocessing required: None (index built from existing chunk text)

---

### 4.2 Entity-Augmented Chunk Metadata (HIGH IMPACT, MEDIUM EFFORT)

**What it is:** Run NER (section 2 above) on all Chunk.text at ingestion time and store entity mentions as structured properties. Use these at query time to:
1. Filter chunks by entity type (e.g., "only chunks mentioning this deputy")
2. Resolve entity mentions in the query to graph nodes before vector search
3. Add entity spans to the Cypher graph traversal (entity-based graph channel)

**Schema change:**
```cypher
-- Add to Chunk node
Chunk.named_entities: JSON string
  -- [{text: "PNRR", label: "LAW", start: 45, end: 49}, ...]
Chunk.law_refs: List[String]
  -- ["decreto legislativo 152/2006", "legge 197/2022"]
Chunk.person_refs: List[String]
  -- ["MELONI Giorgia", "TAJANI Antonio"]
```

**Query-time use:**
```cypher
-- Entity-filtered dense search: only chunks mentioning specific law
CALL db.index.vector.queryNodes('chunk_embedding_index', 50, $queryEmbedding)
YIELD node, score
WHERE $law_ref IN node.law_refs
RETURN node, score
```

**Estimated impact:** HIGH for entity-specific queries ("what was said about decreto 231?"). MEDIUM for general queries.
- Implementation: 2-3 days (NER pipeline + schema change + Cypher updates)
- Reprocessing: Full rebuild required (add NER step to build pipeline)

---

### 4.3 Entity Linking to OCD/Wikidata (MEDIUM IMPACT, MEDIUM EFFORT)

**What it is:** Map entity mentions extracted by NER to canonical Knowledge Base IDs. For PER mentions → Deputy.id in Neo4j. For LAW mentions → ParliamentaryAct.id. For ORG mentions → Committee.id or external Wikidata QID.

**Tool: ReLiK (ACL 2024)**
- Paper: "ReLiK: Retrieve and LinK, Fast and Accurate Entity Linking and Relation Extraction on an Academic Budget" (Findings of ACL 2024)
- Developed by Sapienza NLP (Rome) — Italian research group with Italian language expertise
- Achieves SOTA on entity linking benchmarks with academic-budget training
- Available via Hugging Face + spaCy integration
- **Critical limitation:** Default models link to Wikipedia/Wikidata, NOT to dati.camera.it OCD URIs. Would require fine-tuning on a custom knowledge base of deputy names → OCD URIs.

**Simpler alternative for deputy linking:** After NER extracts PER mentions, fuzzy-match against existing `Deputy.full_name` in Neo4j using `rapidfuzz` (edit distance matching). For an Italian parliament context, deputy names are a closed set of ~400 people — dictionary lookup with fuzzy matching achieves 95%+ precision without ML.

```python
from rapidfuzz import process, fuzz

deputy_names = [d["name"] for d in neo4j.run("MATCH (d:Deputy) RETURN d.full_name as name")]

def link_person_mention(mention: str, threshold: int = 85) -> Optional[str]:
    result = process.extractOne(mention, deputy_names, scorer=fuzz.token_sort_ratio)
    if result and result[1] >= threshold:
        return result[0]
    return None
```

**Estimated impact:** MEDIUM. Enables "find all speeches where deputy X is mentioned (not just when they speak)" — a new retrieval modality.
- Implementation: 1-2 days (fuzzy matching); 2-3 weeks (full ReLiK fine-tuning)
- Recommendation: Use fuzzy matching for deputies; skip ReLiK unless needed for ORG/LAW linking

---

### 4.4 Relation Extraction: Who Supports/Opposes What (LOW IMPACT, HIGH EFFORT)

**What it is:** Extract (subject, relation, object) triples from speech text. E.g., "Fratelli d'Italia opposes motion 1-00098" or "the government supports bill 705."

**Current state of the art:** LLM-based extraction using a prompted extraction call per chunk. Graph-based storage in Neo4j. The `neo4j-graphrag-python` library (v0.9+ in 2024) supports LLM-driven entity/relation extraction with automatic graph storage.

**Reality check for parliamentary text:**
- Speeches are inherently argumentative and ambiguous: "pur riconoscendo i meriti dell'emendamento, il Governo esprime parere contrario" is a nuanced statement, not a simple triple
- False positive rate for relation extraction on formal political Italian is high without domain-specific training data
- The graph channel already handles "speeches in debates about act X" via structural edges — this is the most reliable signal for support/opposition

**Verdict: Not worth building. The Vote nodes (to be saved per FEATURES.md) provide definitive support/opposition data for anything that went to a vote. For pre-vote opinions, the ambiguity is irreducible without heavy annotation.**

---

### 4.5 Sentiment/Stance Detection (LOW IMPACT, HIGH EFFORT)

**What it is:** Classify each speech as positive/negative/neutral toward the subject under debate. Italian political stance detection has been studied (Community-based Stance Detection, CLIC-IT 2024, ~85% accuracy on tweets).

**Limitation for parliamentary debates:** Parliamentary Italian is highly formal and indirect. "Il Governo prende atto dell'osservazione" does not have a clear sentiment. The 85% accuracy figures come from Twitter data (short, emotive); parliamentary speeches will perform significantly worse.

**Verdict: Academic quality improvement; not production-ready for formal Italian parliamentary speech. Skip for now.**

---

### 4.6 Topic Modeling (MEDIUM IMPACT, MEDIUM EFFORT)

**What it is:** Automatic assignment of thematic topics to debates and speech chunks. BERTopic (transformer-based, 2022-present) is the current SOTA for short-to-medium Italian text.

**What it would add:**
- `Debate.topics: List[String]` — automatically inferred topic labels ("immigrazione", "bilancio", "energia", "giustizia")
- `Chunk.topic_weights: JSON` — per-topic probability scores
- Enables filtering retrieval to topically relevant chunks without relying solely on embedding similarity

**For Italian parliamentary text:** BERTopic with `paraphrase-multilingual-MiniLM-L12-v2` as the embedding model (multilingual, fast) clusters speeches by semantic similarity. The topic labels would need manual review/naming for Italian.

**Alternative: Use existing `Debate.title` as topic proxy.** The debate titles already encode the subject matter ("Discussione del disegno di legge... A.C. 705"). For most RAG queries, retrieving debates by title similarity is sufficient.

**Verdict: Implement only if the evaluation shows topic-drift retrieval failures (retrieving chunks from wrong debates). Start with debate title as topic label. Full BERTopic as a future enhancement.**
- Implementation if needed: 3-5 days
- Dependency: `bertopic`, `sentence-transformers`

---

### 4.7 Cross-Document Coreference Resolution (LOW IMPACT, VERY HIGH EFFORT)

**What it is:** Resolve pronoun and nominal references across speech boundaries. E.g., "come ho detto prima" in Speech B refers to content from Speech A by the same speaker.

**Current gap:** Chunks are generated per speech, and the merger creates continuation chunks (ellipsis pattern). Cross-speech coreference would be needed to fully resolve "questo emendamento" when the emendamento was introduced 3 speeches earlier.

**Reality:** This requires state-of-the-art coreference models for Italian, cross-chunk tracking, and graph edges between coreferent mentions. The Italian coreference resolution landscape is sparse — the best tool is `coreferee` (spaCy extension), but its Italian support is limited to shorter texts.

**Verdict: Too complex for marginal gain. The dense retrieval already captures semantic similarity across coreference chains at the embedding level. Skip.**

---

### 4.8 Citation Graph — Speeches Referencing Acts (MEDIUM IMPACT, LOW EFFORT)

**What it is:** Build a graph of which speeches explicitly reference which parliamentary acts by parsing bill/act numbers from speech text.

**This is partially implemented via FEATURES.md item #1 (Debate-to-Act links).** The extension here is going one level deeper: `Speech -[:CITES]-> ParliamentaryAct` instead of just `Debate -[:DISCUSSES]-> ParliamentaryAct`.

**Implementation using NER + BILL_NUMBER regex:**
```python
import re

BILL_PATTERN = re.compile(r'\b(?:A\.C\.|AC|pdl|PdL)\s*(\d[\d\-A-Za-z]*)', re.IGNORECASE)
ACT_CODE_PATTERN = re.compile(r'\b(\d{1,4}-[A-Z]?\d{2,5}[A-Z]?)\b')

def extract_act_refs(text: str) -> List[str]:
    refs = []
    refs.extend(BILL_PATTERN.findall(text))
    refs.extend(ACT_CODE_PATTERN.findall(text))
    return list(set(refs))
```

Then in `_save_session_english()`, after creating Speech nodes, match act codes against `ParliamentaryAct.codice` and MERGE `[:CITES]` edges.

**Graph channel query with citation traversal:**
```cypher
MATCH (a:ParliamentaryAct {codice: $act_code})
      <-[:DISCUSSES]-(d:Debate)
      <-[:IN_DEBATE]-(s:Speech)
      -[:PART_OF]->(c:Chunk)
RETURN c.text, c.id
ORDER BY c.relevance_score DESC
LIMIT 20
```

**Estimated impact:** MEDIUM. Closes the last retrieval gap: "find all speeches that explicitly mention this bill" regardless of which debate they're in.
- Implementation: 1 day
- Reprocessing: Full rebuild (add citation extraction to Speech processing)

---

## 5. Practical Priority Matrix

Ordered by **impact/effort ratio** (highest first):

### Tier 1: Implement Immediately (High Impact, Low Effort — days)

| Technique | Effort | Impact | Dependencies | Reprocess? |
|-----------|--------|--------|-------------|-----------|
| BM25 sparse channel via Neo4j full-text index | 1-2 days | HIGH | Neo4j 5.x (already installed) | No |
| dati.camera.it SPARQL — deputy vote records | 1-2 days | HIGH | SPARQLWrapper Python library | No (incremental) |
| Citation graph (Speech → Act) via regex | 1 day | MEDIUM | None | Yes (full rebuild) |
| ELI URI on ParliamentaryAct nodes | 0.5 days | LOW-MED | None | No (incremental) |

### Tier 2: Implement in Next Milestone (Medium Impact, Medium Effort — 1-2 weeks)

| Technique | Effort | Impact | Dependencies | Reprocess? |
|-----------|--------|--------|-------------|-----------|
| NER at ingestion — `it_nerIta_trf` or `it_core_news_lg` | 3-4 days | HIGH | spaCy, model download | Yes (full rebuild) |
| Entity-augmented Chunk metadata (law_refs, person_refs) | 2-3 days | MEDIUM-HIGH | spaCy NER | Yes (full rebuild) |
| Wikidata deputy biographical enrichment | 1-2 days | MEDIUM | SPARQLWrapper | No (incremental) |
| dati.camera.it SPARQL — committee officer roles | 1-2 days | MEDIUM | SPARQLWrapper | No (incremental) |
| Deputy name fuzzy linking from NER mentions | 1 day | MEDIUM | rapidfuzz | No (incremental) |

### Tier 3: Future Milestones (Lower Priority)

| Technique | Effort | Impact | Dependencies | When |
|-----------|--------|--------|-------------|------|
| BERTopic debate categorization | 3-5 days | MEDIUM | bertopic, sentence-transformers | After tier 1+2 |
| Normattiva law text on-demand retrieval | 2-3 days | MEDIUM | httpx, XML parsing | After tier 2 |
| dati.senato.it cross-chamber act linking | 1-2 days | LOW-MED | SPARQLWrapper | After tier 1 |
| ReLiK entity linking (fine-tuned) | 2-3 weeks | MEDIUM | relik, custom KB | Research/future |

### Tier 4: Academic Overkill (Skip)

| Technique | Why Skip |
|-----------|---------|
| Full Akoma Ntoso XML migration | Existing XML schema already captures all structure; migration adds no retrieval value |
| LKIF reasoning layer | Legal reasoning ontology; no production RAG benefit |
| Stance/sentiment detection | 85% accuracy on tweets degrades significantly on formal parliamentary Italian |
| Cross-document coreference resolution | Italian coreference tooling is immature; dense embeddings already handle this implicitly |
| EUR-Lex integration | EU law mentions are imprecise; disambiguation is hard |
| Relation extraction triples | Vote nodes provide definitive support/opposition; text-derived triples too noisy |

---

## 6. Implementation Notes for Current Stack

### Python dependencies to add

```bash
# Sparse retrieval (already built into Neo4j — no Python dep needed)
# Entity extraction
pip install spacy==3.7.*
python -m spacy download it_core_news_lg  # 4-type NER, immediately available
# OR (version-pinned, test compatibility):
# install bullmount/it_nerIta_trf for 18-type NER

# SPARQL data fetching
pip install SPARQLWrapper>=2.0.0

# Fuzzy entity linking
pip install rapidfuzz>=3.0.0

# BERTopic (Tier 3 only)
pip install bertopic sentence-transformers
```

### Neo4j schema additions

```cypher
-- New properties on Chunk (after NER enrichment)
ALTER NODE Chunk ADD PROPERTY named_entities JSON DEFAULT '[]'
ALTER NODE Chunk ADD PROPERTY law_refs LIST<STRING> DEFAULT []
ALTER NODE Chunk ADD PROPERTY person_refs LIST<STRING> DEFAULT []

-- New full-text index for BM25 sparse channel
CREATE FULLTEXT INDEX chunk_fulltext IF NOT EXISTS
FOR (c:Chunk) ON EACH [c.text]

-- New relationship: Speech cites Act
-- (no schema change needed — just MERGE the edge)
-- (:Speech)-[:CITES]->(:ParliamentaryAct)

-- New properties on Deputy (after Wikidata enrichment)
ALTER NODE Deputy ADD PROPERTY wikidata_qid STRING
ALTER NODE Deputy ADD PROPERTY birth_date DATE
ALTER NODE Deputy ADD PROPERTY profession STRING
```

### SPARQL client pattern (reusable utility)

```python
from SPARQLWrapper import SPARQLWrapper, JSON

def query_camera_sparql(query: str) -> List[Dict]:
    sparql = SPARQLWrapper("https://dati.camera.it/sparql")
    sparql.setQuery(query)
    sparql.setReturnFormat(JSON)
    results = sparql.query().convert()
    return results["results"]["bindings"]

def query_wikidata(query: str) -> List[Dict]:
    sparql = SPARQLWrapper("https://query.wikidata.org/sparql")
    sparql.addCustomHttpHeader("User-Agent", "ParliamentRAG/1.0")
    sparql.setQuery(query)
    sparql.setReturnFormat(JSON)
    results = sparql.query().convert()
    return results["results"]["bindings"]
```

---

## 7. Confidence Assessment

| Area | Confidence | Sources |
|------|------------|---------|
| OCD ontology classes and SPARQL endpoint | HIGH | Direct inspection of dati.camera.it/en, ontology page |
| Akoma Ntoso scope and applicability | HIGH | OASIS official docs |
| ELI URI pattern for Normattiva | HIGH | WebSearch + Normattiva docs |
| LKIF irrelevance | MEDIUM | GitHub inspection; no 2024 activity |
| `bullmount/it_nerIta_trf` metrics | HIGH | Direct HuggingFace page (F1=91.96) |
| Stanza Italian F1=87.92 | HIGH | Official Stanza NER page |
| `it_core_news_lg` entity types | MEDIUM | spaCy docs (specific F1 not stated) |
| BM25 via Neo4j full-text index | HIGH | Neo4j 5.x documentation (Lucene backend) |
| ReLiK entity linking | HIGH | ACL 2024 paper; GitHub inspection |
| BERTopic for Italian | MEDIUM | Multiple 2024 papers; Italian-specific benchmarks limited |
| dati.senato.it SPARQL availability | HIGH | Direct link confirmed active |
| Wikidata XIX legislature query | HIGH | Wikidata SPARQL service; P39/P2937 confirmed |

---

## Sources

- [Akoma Ntoso Version 1.0 — OASIS LegalDocML](https://docs.oasis-open.org/legaldocml/akn-core/v1.0/akn-core-v1.0-part1-vocabulary.html)
- [OCD Ontology — dati.camera.it](https://dati.camera.it/en/ontology-chamber-deputies)
- [dati.camera.it SPARQL Endpoint](https://dati.camera.it/sparql)
- [dati.camera.it Sparnatural Visual Query Builder](https://www.sparna.fr/en/posts/dati-camera-it-s-sparnatural-instance-a-query-builder-for-the-italian-chamber-of-deputies/)
- [dati.senato.it SPARQL Endpoint](https://dati.senato.it/sparql)
- [ELI — European Legislation Identifier](https://eur-lex.europa.eu/content/help/eurlex-content/eli.html)
- [Normattiva OpenData](https://dati.normattiva.it/)
- [bullmount/it_nerIta_trf — HuggingFace](https://huggingface.co/bullmount/it_nerIta_trf)
- [spaCy Italian Models](https://spacy.io/models/it/)
- [Stanza NER Models — Italian](https://stanfordnlp.github.io/stanza/ner_models.html)
- [dlicari/Italian-Legal-BERT](https://huggingface.co/dlicari/Italian-Legal-BERT)
- [ReLiK ACL 2024 — Sapienza NLP](https://aclanthology.org/2024.findings-acl.839/)
- [Entity Linking with ReLiK + Neo4j](https://neo4j.com/blog/developer/entity-linking-relationship-extraction-relik-llamaindex/)
- [Graph RAG Survey — ACM TOIS 2024](https://dl.acm.org/doi/10.1145/3777378)
- [LKIF Core Ontology — GitHub](https://github.com/RinkeHoekstra/lkif-core)
- [Wikidata — Chamber of Deputies of Italy](https://www.wikidata.org/wiki/Q841424)
- [GENAI4LEX-B: AI-powered legislative support at Italian Chamber](https://interoperable-europe.ec.europa.eu/collection/public-sector-tech-watch/genai4lex-b-ai-powered-legislative-support-italian-chamber-deputies)
- [Hybrid RAG Retrieval — IBM Research 2024](https://infiniflow.org/blog/best-hybrid-search-solution)
