# Phase 1: Build Pipeline - Research

**Researched:** 2026-04-02
**Domain:** Python build pipeline refactoring — XML parsing, Neo4j batch ingestion, schema normalization
**Confidence:** HIGH — all findings from direct codebase inspection

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Schema Design — Minimal properties per node**

- Speech node: keep only `text` (preprocessed content). Remove `preprocessedText` duplicate and `surnameNam`. Speaker reconciliation must be complete at build time. Add `speakingRole` property from `<emphasis>` XML tag.
- Session node: remove `completeDate` string — use only Neo4j Date `date` property. Keep: `id`, `legislature`, `number`, `year`, `month`, `day`, `chamber`, `date`.
- Chunk node: minimal: `id`, `text`, `index`, `embedding`. Remove `charCount`, `startCharRaw`, `endCharRaw`. Remove all alignment_map logic from chunking code.
- All nodes: camelCase property names, PascalCase labels, SCREAMING_SNAKE_CASE relationship types.

**Vote Extraction**

- Node label: `Vote`
- Properties from XML: `number`, `type`, `subject`, `present`, `voters`, `abstained`, `majority`, `inFavor`, `against`, `onMission`, `outcome`
- Relationship: `Session-[:HAS_VOTE]->Vote` (see Critical Finding below — votes are session-level in XML)
- `parse_votazione()` already complete — wire into new `_save_session_english()`

**Debate→Act Linking**

- Parse `<argomenti>` XML element (inside `<metadati>`, NOT inside `<dibattito>`)
- Each `<argomento idDibattito="tit00050">` links to a debate; contains `<atti><atto tipologiaAtto="pdl" codiceArgomento="5-A"/></atti>`
- Relationship: `Debate-[:DISCUSSES]->ParliamentaryAct`
- Match strategy: match by `codiceArgomento` against `ParliamentaryAct.number`; create placeholder if not found
- 5 act types: `pdl` (5590), `interrogazioneRispostaOrale` (1614), `mozione` (533), `doc` (343), `interpellanza` (328)
- Total act references in corpus: 8,408 across 7,317 argomento entries

**Build Script Architecture**

- Modular: `xml_parser.py`, `chunker.py`, `db_builder.py`, `csv_loader.py`, `download.py`, `build_and_update.py`
- YAML/JSON config for chunking parameters separate from code
- UNWIND with batch size 1000
- `execute_read`/`execute_write` managed transactions

**Phase Type Enum**

- Parse Phase title patterns into `phaseType` property
- Pattern matching on Italian title string (see Architecture Patterns section for complete mapping)

### Claude's Discretion

- Exact module file layout within `build/` directory
- YAML config file name and structure
- How to handle XML files with malformed structure (error recovery)
- Exact UNWIND Cypher query structure
- Phase type enum values (derive from actual title patterns in XML corpus)

### Deferred Ideas (OUT OF SCOPE)

- Individual vote records per deputy (SPARQL from dati.camera.it) — Phase 4 ENR-01
- NER entity extraction on chunks — Phase 4 ENR-03/04
- BM25 sparse retrieval channel — Phase 4 RET-01
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| BUILD-01 | English-only Neo4j schema (camelCase, PascalCase, SCREAMING_SNAKE_CASE) | Property rename table in Architecture Patterns; all current names documented |
| BUILD-02 | Remove Italian-schema dead code from `ingest_stenografici.py` | Dead code identified: `save_to_neo4j()`, Italian constraints (`Seduta`, `Dibattito`, `Fase`, `Intervento`, `Votazione`), Italian indexes |
| BUILD-03 | Extract XML parser class into standalone module | Extraction plan: `StenograficoIngester` parser methods → `xml_parser.py`; coupling documented |
| BUILD-04 | Remove redundant Chunk properties and dead alignment_map code | `startCharRaw`, `endCharRaw` confirmed unused in backend; alignment_map only produced inside chunks struct |
| BUILD-05 | Remove redundant `preprocessedText` Speech property | `preprocessed_text` confirmed set but never queried; backend queries only `sp.text` |
| BUILD-06 | Remove redundant `completeDate` Session property | `complete_date` confirmed set but never queried by backend |
| BUILD-07 | UNWIND batch writes for bulk ingestion | Pattern documented from STACK.md; batch size 1000 |
| BUILD-08 | Managed transactions (`execute_read`/`execute_write`) | Pattern documented; current code uses auto-commit `session.run()` throughout |
| BUILD-09 | Single `make db-all` target | `db-all` target already exists; needs update to call new `build_and_update.py` entry point |
| DATA-01 | Persist Vote nodes with HAS_VOTE from Session | `parse_votazione()` already complete; votes are in `raccoltaVotazioni` at session level (NOT debate-level). Relationship must be `Session-[:HAS_VOTE]->Vote` |
| DATA-02 | `<argomenti>` metadata → Debate-[:DISCUSSES]->ParliamentaryAct | `<argomenti>` is inside `<metadati>` (sibling to `<resoconto>`), NOT inside `<dibattito>`. `argomento.idDibattito` links to debate. `atto.codiceArgomento` matches `ParliamentaryAct.number` |
| DATA-03 | Speaker role from `<emphasis>` tag → `Speech.speakingRole` | `<emphasis>` is a direct child of `<testoXHTML>` immediately after `<nominativo>`. Contains role strings like `"Presidente del Consiglio dei Ministri"`, `"Ministro dell'Interno"`, `"Relatore"`. Only present when speaker has institutional role. |
| DATA-04 | Phase title patterns → `Phase.phaseType` enum | 6199 unique phase titles across all files; ~8 canonical types derivable from first word/phrase patterns. Complete mapping in Architecture Patterns. |
</phase_requirements>

---

## Summary

The build pipeline consists of a 1078-line monolithic `build_and_update.py` that imports an XML parser from `ingest_stenografici.py` (which has a dead Italian-schema save path). The refactoring goal is to split this into focused modules, normalize property naming to camelCase, remove dead properties, and add three new data extractions (Vote nodes, Debate-to-Act links, Speaker roles).

The most important finding is a **mismatch between the CONTEXT.md decision and the actual XML structure** for Vote nodes: `raccoltaVotazioni` is a sibling of `<dibattito>` at resoconto level (not nested inside debates). The relationship must be `Session-[:HAS_VOTE]->Vote`, not `Debate-[:HAS_VOTE]->Vote`. The 9,478 vote elements across the corpus confirm this. Additionally, the current `parse_xml_file()` code uses `dib_elem.findall('.//votazioni')` which finds **zero** votes because it searches inside `dibattito` elements where votes do not exist. This bug must be fixed in the new `xml_parser.py`.

The second critical finding is that `<argomenti>` lives in `<metadati>` (not in `<resoconto>`), and each `<argomento>` element has an `idDibattito` attribute linking it to a specific debate. This is separate from and cross-referenced to the debate tree. The XML parser must traverse `root.find('metadati').findall('.//argomento')` and build a debate-to-acts map.

**Primary recommendation:** Extract `xml_parser.py` from `ingest_stenografici.py` as the first task of Wave 1, before any deletion — this is the critical sequencing constraint from STATE.md.

---

## Standard Stack

### Core (all versions from existing requirements files)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `neo4j` | 5.x (pinned in requirements.txt) | Neo4j Python driver; managed transactions | Project's existing driver |
| `pandas` | current | CSV loading for deputy/group/committee data | Already used; `clean_generic_label` / `extract_group_info` helpers depend on it |
| `regex` | current | Recursive parentheses removal in `preprocess_text_with_alignment` | Required for `(?R)` recursive patterns not supported by stdlib `re` |
| `requests` | current | XML and CSV download from Camera APIs | Already used in download functions |
| `pyyaml` | current (already in requirements.txt check needed) | YAML config file for chunking parameters | Clean separation of config from code |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `xml.etree.ElementTree` | stdlib | XML parsing | Already used; sufficient for stenografici structure |
| `hashlib` | stdlib | Embedding cache keys | Already used in `embedding_service.py` |
| `argparse` | stdlib | CLI entry point | Already used in `build_and_update.py` main |
| `logging` | stdlib | Structured logging replacing `print()` statements | Replace all `print()` in build scripts |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `xml.etree.ElementTree` | `lxml` | lxml is faster for large XML but stdlib is sufficient for the ~365 files at <1MB each; no new dependency |
| `pyyaml` for config | plain Python dataclass | YAML is more ergonomic for users who adjust chunk sizes; either works |

**Installation:**
```bash
pip install pyyaml  # if not already present
```

**Version verification:** All other packages are already installed in the project virtualenv.

---

## Architecture Patterns

### Recommended Module Structure

```
build/
├── xml_parser.py          # StenograficoParser class — pure XML parsing, no Neo4j
├── chunker.py             # chunk_speech() — sentence splitting, overlap, no alignment_map
├── db_builder.py          # DatabaseBuilder class — constraints, indexes, UNWIND batch writes
├── csv_loader.py          # load_deputies(), load_groups(), load_committees()
├── download.py            # download_xmls(), download_csv() — HTTP download logic
├── build_config.py        # BuildConfig dataclass — CHUNK_SIZE, CHUNK_OVERLAP, MIN_LENGTH
├── build_and_update.py    # CLI entry point: do_build(), do_update()
├── app_config.py          # UNCHANGED — GOVERNMENT_ROLES, PARLIAMENT_ROLES, etc.
├── neo4j_helper.py        # UNCHANGED — shared Neo4j client
├── embedding_service.py   # FROZEN — do not modify
├── precalculate_embeddings.py  # Update only property name reads
└── config.yaml            # Chunking parameters (CHUNK_SIZE=1200, OVERLAP=250, MIN_LENGTH=100)
```

### Pattern 1: UNWIND Batch Write

**What:** Collect records into a list, write as a single UNWIND transaction instead of per-item `session.run()`.

**When to use:** All multi-node write operations: sessions, debates, phases, speeches, chunks, votes, deputy loading.

**Example:**
```python
# Batch size 1000
def _ingest_chunks(self, tx, batch: list[dict]) -> None:
    tx.run(
        """
        UNWIND $batch AS row
        MERGE (c:Chunk {id: row.id})
        SET c.text = row.text,
            c.index = row.index
        """,
        batch=batch
    )

# Collect then flush
chunk_batch = []
for chunk in chunks:
    chunk_batch.append({"id": chunk["id"], "text": chunk["text"], "index": chunk["index"]})
    if len(chunk_batch) >= 1000:
        session.execute_write(self._ingest_chunks, chunk_batch)
        chunk_batch.clear()
if chunk_batch:
    session.execute_write(self._ingest_chunks, chunk_batch)
```

### Pattern 2: Managed Transactions

**What:** Replace `session.run()` with `session.execute_read()` / `session.execute_write()` for automatic retry.

**When to use:** All Neo4j read and write operations in the new `db_builder.py`.

```python
# Reads (retried on transient error):
def get_existing_sessions(self) -> set[int]:
    def _query(tx):
        return {r["number"] for r in tx.run("MATCH (s:Session) RETURN s.number AS number")}
    with self._driver.session() as session:
        return session.execute_read(_query)

# Writes (retried on transient error):
def _create_sessions(self, tx, batch: list[dict]) -> None:
    tx.run("""
        UNWIND $batch AS row
        MERGE (s:Session {id: row.id})
        SET s.legislature = row.legislature,
            s.number = row.number,
            s.year = row.year,
            s.month = row.month,
            s.day = row.day,
            s.chamber = row.chamber,
            s.date = row.date
    """, batch=batch)
```

### Pattern 3: XML Parser Extraction (Critical Sequencing)

**What:** Extract parser methods from `StenograficoIngester` into a standalone `StenograficoParser` class in `xml_parser.py`. The new class has no Neo4j dependency.

**When to use:** This is the mandatory first step. Do not delete any code from `ingest_stenografici.py` before `xml_parser.py` exists and is imported by `build_and_update.py`.

```python
# xml_parser.py — only these methods, no __init__ Neo4j driver
class StenograficoParser:
    def __init__(self, chunk_size: int = 1200, chunk_overlap: int = 250, min_length: int = 100):
        self.ns = {'xhtml': 'http://www.w3.org/1999/xhtml'}
        # No driver, no Neo4j

    def parse_xml_file(self, filepath: str) -> dict:
        """Returns dict with: session, debates, phases, speeches, votes, act_references."""
        ...

    # Internal helpers: parse_intervention, parse_vote, get_nominativo_info,
    # extract_text_from_element, merge_continuation_interventions,
    # preprocess_text_with_alignment
```

### Pattern 4: Vote Linking — Session Level (NOT Debate Level)

**What:** Votes in the XML (`raccoltaVotazioni`) are session-level siblings to debates. There is no direct XML link from vote to debate.

**XML structure:**
```
root
  metadati
    argomenti        <- act references (debate-linked via idDibattito)
  resoconto
    dibattito[id=tit00010]
    dibattito[id=tit00050]
    raccoltaVotazioni[id=tit00100]   <- at same level as dibattito, NOT inside one
      votazioni
        votazione
```

**Parser fix required:**
```python
# CURRENT CODE (WRONG — finds 0 votes):
for dib_elem in resoconto.findall('dibattito'):
    for votazioni_elem in dib_elem.findall('.//votazioni'):  # finds nothing

# CORRECT — parse votes at resoconto level:
for rv in resoconto.findall('raccoltaVotazioni'):
    for vs in rv.findall('votazioni'):
        for vot_elem in vs.findall('votazione'):
            vote = self.parse_vote(vot_elem, session_id, vot_index)
            votes.append(vote)
            vot_index += 1
```

**Cypher relationship:** `Session-[:HAS_VOTE]->Vote` (CONTEXT.md says `Debate-[:HAS_VOTE]->Vote` but this is inconsistent with XML structure — the correct relationship is to Session)

### Pattern 5: Argomenti Parsing for Debate-to-Act Links

**What:** Parse `<metadati><argomenti>` to link debates to parliamentary acts.

**XML structure:**
```xml
<metadati>
  <argomenti>
    <argomento idDibattito="tit00050">
      <atti>
        <atto tipologiaAtto="pdl" codiceArgomento="5-A"/>
      </atti>
    </argomento>
  </argomenti>
</metadati>
```

**Parser code:**
```python
def _parse_act_references(self, root) -> dict[str, list[dict]]:
    """Returns {debate_original_id: [{act_type, act_code}]}."""
    result: dict[str, list[dict]] = {}
    metadati = root.find('metadati')
    if metadati is None:
        return result
    for arg in metadati.findall('.//argomento'):
        dib_id = arg.get('idDibattito')
        if not dib_id:
            continue
        for atto in arg.findall('.//atto'):
            tipo = atto.get('tipologiaAtto')
            codice = atto.get('codiceArgomento')
            if tipo and codice:
                result.setdefault(dib_id, []).append({"type": tipo, "code": codice})
    return result
```

**Cypher for linking:**
```cypher
UNWIND $batch AS row
MATCH (d:Debate {id: row.debateId})
MERGE (a:ParliamentaryAct {number: row.actCode})
ON CREATE SET a.type = row.actType, a.isPlaceholder = true
MERGE (d)-[:DISCUSSES]->(a)
```

### Pattern 6: Speaker Role Extraction

**What:** After `<nominativo>`, `<testoXHTML>` may contain an `<emphasis>` child with the speaker's institutional role. Only present when speaker has a formal role.

**Example XML:**
```xml
<testoXHTML>
  <nominativo id="300xxx" cognomeNome="MELONI Giorgia">...</nominativo>
  <emphasis type="italic">Presidente del Consiglio dei Ministri</emphasis>
  ...
</testoXHTML>
```

**Parser code:**
```python
def _extract_speaking_role(self, testo_xhtml) -> str | None:
    """Extract institutional role from <emphasis> tag after <nominativo>."""
    if testo_xhtml is None:
        return None
    children = list(testo_xhtml)
    tags = [c.tag for c in children]
    if 'nominativo' not in tags:
        return None
    nom_idx = tags.index('nominativo')
    if nom_idx + 1 >= len(children):
        return None
    next_elem = children[nom_idx + 1]
    if next_elem.tag != 'emphasis' or not next_elem.text:
        return None
    text = next_elem.text.strip().rstrip('.')
    # Filter out applause/stage directions (they start with '(')
    if text.startswith('('):
        return None
    # Only return if it looks like an institutional role
    role_prefixes = (
        'Ministro', 'Ministra', 'Sottosegretario', 'Sottosegretaria',
        'Viceministro', 'Presidente del Consiglio', 'Relatore', 'Relatrice'
    )
    if any(text.startswith(p) for p in role_prefixes):
        return text
    return None
```

**Confirmed role values found in corpus:** `"Presidente del Consiglio dei Ministri"`, `"Ministro dell'Interno"`, `"Ministra per le Disabilità"`, `"Sottosegretario di Stato per la Salute"`, `"Relatore"`, `"Relatore per la IV Commissione"`, and ~30 other ministry titles.

### Pattern 7: Phase Type Mapping

**What:** Map Italian phase title patterns to English enum values for `Phase.phaseType`.

**Corpus statistics:** 6,199 unique phase titles, but the canonical patterns reduce to ~10 types covering the vast majority of phases.

**Recommended mapping (derive in `xml_parser.py`):**

| Pattern (match in title) | `phaseType` value | Count |
|--------------------------|-------------------|-------|
| `Dichiarazioni di voto` | `"vote_declaration"` | 124+ |
| `Votazioni` or `Votazione` | `"vote"` | 116+ |
| `Discussione sulle linee generali` | `"general_discussion"` | 94+ |
| `Parere del Governo` or `Intervento e parere del Governo` or `Replica e parere del Governo` | `"government_opinion"` | 78+ |
| `Interventi` | `"interventions"` | 39 |
| `Discussione` | `"discussion"` | 33 |
| `Annunzio di risoluzioni` | `"resolution_announcement"` | 32 |
| `Esame` | `"article_examination"` | various |
| `Ordini del giorno` | `"order_of_business"` | various |
| `scrutinio` | `"ballot"` | 4 |
| (no match) | `"other"` | fallback |

```python
import re

_PHASE_PATTERNS: list[tuple[re.Pattern, str]] = [
    (re.compile(r'Dichiarazioni di voto', re.I), "vote_declaration"),
    (re.compile(r'Votazioni|Votazione', re.I), "vote"),
    (re.compile(r'Discussione sulle linee generali', re.I), "general_discussion"),
    (re.compile(r'Parere del Governo|parere del Governo', re.I), "government_opinion"),
    (re.compile(r'Annunzio di risoluzioni', re.I), "resolution_announcement"),
    (re.compile(r'Esame degli ordini del giorno', re.I), "order_of_business"),
    (re.compile(r'Esame', re.I), "article_examination"),
    (re.compile(r'Discussione', re.I), "discussion"),
    (re.compile(r'Interventi', re.I), "interventions"),
    (re.compile(r'scrutinio', re.I), "ballot"),
    (re.compile(r'Replica', re.I), "reply"),
]

def classify_phase_type(title: str) -> str:
    for pattern, phase_type in _PHASE_PATTERNS:
        if pattern.search(title):
            return phase_type
    return "other"
```

### Anti-Patterns to Avoid

- **Never call `StenograficoIngester.__init__()`** — it creates Italian-schema constraints on Neo4j. The current `build_and_update.py` avoids this by using `__new__`. The new `xml_parser.py` must not have this coupling at all.
- **Never search `.//votazioni` inside `dibattito`** — votes are at resoconto level. This bug is in the current parser and produces zero votes silently.
- **Never use string f-interpolation in Cypher** — always parameterize with `$param`. Current `load_groups()` uses string interpolation for node labels (`f"MERGE (d:{node_label}..."`) — acceptable for label switching but document clearly.
- **Never pass alignment_map to chunker** — remove this argument entirely. The new `chunker.py` takes only `(text: str, speech_id: str) -> list[dict]`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Embedding cache | Custom cache | `build/embedding_service.py` (FROZEN) | Already has SHA-256 keyed SQLite cache; changing anything invalidates 329MB |
| Neo4j connection pooling | New pool logic | `build/neo4j_helper.py` | Driver handles pooling; just use `get_neo4j_client()` |
| CSV parsing | Custom CSV reader | `pandas.read_csv()` | Group/committee label cleaning helpers already exist |
| YAML config parsing | Custom INI parser | `pyyaml` | Standard; already how Python projects handle this |
| Chunking sentence splitting | Custom tokenizer | Existing regex in `create_chunks()` | The abbreviation-aware split is already correct for Italian parliamentary text |

**Key insight:** The embedding service and cache are frozen by design (STATE.md decision). Any refactoring that touches `build/embedding_service.py` risks invalidating 329MB of cached embeddings and requires full re-embedding.

---

## Common Pitfalls

### Pitfall 1: Deleting Italian Code Before Extracting Parser

**What goes wrong:** `build_and_update.py` imports `StenograficoIngester` (line 40) and uses `StenograficoIngester.__new__()` to access parser methods. Deleting `ingest_stenografici.py` before creating `xml_parser.py` breaks the entire build.

**How to avoid:** The canonical sequence is:
1. Create `xml_parser.py` with extracted `StenograficoParser` class
2. Update `build_and_update.py` import to use `xml_parser`
3. Verify build works
4. Then delete Italian save path code from `ingest_stenografici.py`

### Pitfall 2: Property Rename Without Updating Backend

**What goes wrong:** Phase 1 renames properties (removes `start_char_raw`, `end_char_raw`, `preprocessed_text`, `complete_date`). The backend still queries these. Neo4j returns `null` silently.

**Affected files (must update simultaneously with schema — Phase 2 task):**
- `dense_channel.py` lines 83–84, 285–286
- `graph_channel.py` lines 285–286
- `engine.py` lines 326–328, 500–501
- `query.py` lines 284–285
- `chat.py` lines 427–428, 910–911
- `evidence.py` lines 97–98, 187–188
- `neo4j_client.py` lines 159–160

**Phase 1 must NOT be deployed without Phase 2 completing these updates.** Build new schema, then update all query sites in the same deployment.

### Pitfall 3: Vote Parsing Finds Zero Votes

**What goes wrong:** Current `parse_xml_file()` uses `dib_elem.findall('.//votazioni')` inside the dibattito loop. Verified: this finds **zero** votes across all 365 XML files because votes live in `raccoltaVotazioni` at resoconto level.

**How to avoid:** New `xml_parser.py` must iterate `resoconto.findall('raccoltaVotazioni')` instead. The existing `parse_votazione()` function is correct; only the traversal path is wrong.

### Pitfall 4: Embedding Cache Key Is Frozen

**What goes wrong:** Cache key is `sha256(EMBEDDING_MODEL + "\n" + normalized_text)`. EMBEDDING_MODEL defaults to `"text-embedding-3-small"`. Any change to this string, or to the normalization logic (`.strip().split()` join), orphans all 329MB of cached embeddings.

**How to avoid:** `build/embedding_service.py` is marked FROZEN in CONTEXT.md. Do not modify it. The `precalculate_embeddings.py` script can be updated for property names but must preserve the `EmbeddingService` import unchanged.

### Pitfall 5: `populate_ruoli.py` Uses Italian Labels

**What goes wrong:** `build/populate_ruoli.py` line 168 queries `MATCH (i:Intervento)` — Italian label. After schema migration this finds zero nodes and silently does nothing.

**How to avoid:** The new `db_builder.py`'s `load_roles()` method (ported from `build_and_update.py`) already uses `Speech`. The standalone `populate_ruoli.py` must be updated to `MATCH (i:Speech)` if it is kept — or it can be deprecated in favor of the inline role loading in `db_builder.py`.

### Pitfall 6: `Speech.surname_name` Used in Orphan Reconciliation

**What goes wrong:** The current `load_roles()` method uses `sp.surname_name` to reconcile unmatched speeches. CONTEXT.md says reconciliation must be complete at build time — so this fallback is being removed. However, if speaker reconciliation in `_save_session_english` misses cases, some speeches will have no `SPOKEN_BY` relationship.

**How to avoid:** The new `db_builder.py` must attempt SPOKEN_BY reconciliation inline in `_save_speech()` before any fallback is needed. The GovernmentMember name fallback in `_save_session_english` (lines 685–691) should be preserved as inline logic in `db_builder.py`.

---

## Code Examples

### Existing `parse_votazione()` (from `ingest_stenografici.py` — reuse verbatim in `xml_parser.py`)

```python
def parse_vote(self, vot_elem, session_id: str, vot_index: int) -> dict:
    """Parse a single <votazione> element."""
    def get_text(tag):
        elem = vot_elem.find(tag)
        return elem.text.strip() if elem is not None and elem.text else None
    def get_int(tag):
        val = get_text(tag)
        try:
            return int(val) if val else None
        except ValueError:
            return None
    return {
        'id': f"{session_id}_vot_{vot_index}",
        'number': get_int('numero'),
        'type': get_text('tipo'),
        'subject': get_text('oggetto'),
        'present': get_int('presenti'),
        'voters': get_int('votanti'),
        'abstained': get_int('astenuti'),
        'majority': get_int('maggioranza'),
        'inFavor': get_int('favorevoli'),
        'against': get_int('contrari'),
        'onMission': get_int('missione'),
        'outcome': get_text('esito'),
    }
```

### UNWIND Vote Ingestion

```cypher
UNWIND $batch AS row
MERGE (v:Vote {id: row.id})
SET v.number = row.number,
    v.type = row.type,
    v.subject = row.subject,
    v.present = row.present,
    v.voters = row.voters,
    v.abstained = row.abstained,
    v.majority = row.majority,
    v.inFavor = row.inFavor,
    v.against = row.against,
    v.onMission = row.onMission,
    v.outcome = row.outcome
WITH v, row
MATCH (s:Session {id: row.sessionId})
MERGE (s)-[:HAS_VOTE]->(v)
```

### UNWIND Speech Ingestion

```cypher
UNWIND $batch AS row
MERGE (sp:Speech {id: row.id})
SET sp.text = row.text,
    sp.speakingRole = row.speakingRole
```

### UNWIND Chunk Ingestion

```cypher
UNWIND $batch AS row
MERGE (c:Chunk {id: row.id})
SET c.text = row.text,
    c.index = row.index
WITH c, row
MATCH (sp:Speech {id: row.speechId})
MERGE (sp)-[:HAS_CHUNK]->(c)
```

### UNWIND Debate→Act Link

```cypher
UNWIND $batch AS row
MATCH (d:Debate {id: row.debateId})
MERGE (a:ParliamentaryAct {number: row.actCode})
ON CREATE SET a.type = row.actType, a.isPlaceholder = true
MERGE (d)-[:DISCUSSES]->(a)
```

### Config YAML

```yaml
# build/config.yaml
chunking:
  chunk_size: 1200          # target chars per chunk
  chunk_overlap: 250        # overlap between consecutive chunks
  min_speech_length: 100    # minimum chars to keep a speech
```

---

## Complete Property Rename Map

This is the authoritative list of every property change. **Phase 2 backend update must align to this exactly.**

### Node: Session
| Current property | New property | Action |
|-----------------|--------------|--------|
| `id` | `id` | no change |
| `legislature` | `legislature` | no change |
| `number` | `number` | no change |
| `year` | `year` | no change |
| `month` | `month` | no change |
| `day` | `day` | no change |
| `chamber` | `chamber` | no change |
| `date` | `date` | no change (Neo4j Date type) |
| `complete_date` | — | REMOVE |

### Node: Debate
| Current property | New property | Action |
|-----------------|--------------|--------|
| `id` | `id` | no change |
| `title` | `title` | no change |
| `order` | `order` | no change |

### Node: Phase
| Current property | New property | Action |
|-----------------|--------------|--------|
| `id` | `id` | no change |
| `title` | `title` | no change |
| `order` | `order` | no change |
| (new) | `phaseType` | ADD |

### Node: Speech
| Current property | New property | Action |
|-----------------|--------------|--------|
| `id` | `id` | no change |
| `text` (raw) | `text` (preprocessed) | rename semantics: now holds preprocessed text |
| `preprocessed_text` | — | REMOVE (was duplicate of new `text`) |
| `char_count` | — | REMOVE |
| `merged_from_ids` | — | REMOVE |
| `surname_name` | — | REMOVE |
| (new) | `speakingRole` | ADD (nullable string) |

### Node: Chunk
| Current property | New property | Action |
|-----------------|--------------|--------|
| `id` | `id` | no change |
| `text` | `text` | no change |
| `index` | `index` | no change |
| `embedding` | `embedding` | no change |
| `char_count` | — | REMOVE |
| `start_char_raw` | — | REMOVE |
| `end_char_raw` | — | REMOVE |

### Node: Vote (NEW)
| Property | Type | Notes |
|----------|------|-------|
| `id` | string | `{sessionId}_vot_{index}` |
| `number` | integer | |
| `type` | string | e.g. "Nominale" |
| `subject` | string | vote subject text |
| `present` | integer | |
| `voters` | integer | |
| `abstained` | integer | |
| `majority` | integer | |
| `inFavor` | integer | |
| `against` | integer | |
| `onMission` | integer | |
| `outcome` | string | e.g. "Appr." |

### Node: Deputy (unchanged — all snake_case properties match backend queries)
These are accessed by the backend as `d.first_name`, `d.last_name`, `d.profession`, `d.education`, `d.photo`, `d.deputy_card`. They are already correct per the backend Cypher queries. No change.

### Node: ParliamentaryAct (unchanged)
`a.type`, `a.title`, `a.description`, `a.presentation_date`, `a.number`, `a.recipient`, `a.eurovoc` — all used by `precalculate_embeddings.py` and the acts ingester. No change.

---

## `precalculate_embeddings.py` Update Scope

This script must be updated for the Speech property rename:

| Script section | Current property | After Phase 1 |
|---------------|-----------------|---------------|
| Phase 3 (Chunks) | `c.id`, `c.text` | unchanged |
| Phase 5 (Speeches) | `sp.text` | unchanged (still `text`, now holds preprocessed content) |
| Phase 4 (Deputies) | `d.profession`, `d.education` | unchanged |
| Phase 2 (Acts) | `a.title`, `a.eurovoc`, `a.description` | unchanged |
| Phase 1 (Committees) | `c.name` | unchanged |

No property changes affect `precalculate_embeddings.py`. The script can be left unchanged for Phase 1. The only change needed is adding Vote constraint to `db_builder.py`'s `create_constraints()`.

---

## Makefile `db-all` Target

**Current target** (lines 261–283 of Makefile): Already well-structured.
1. Stops backend/frontend
2. Starts Neo4j Docker
3. Waits for bolt port
4. Downloads deputy CSVs (`download_deputies_csv.py`)
5. Runs `build_and_update.py build`

**Required change:** Only `build_and_update.py` is being replaced in-place. The Makefile `BUILD_SCRIPT` variable points to `$(BUILD_DIR)/build_and_update.py` which stays at the same path. The Makefile itself needs **no changes** if the new `build_and_update.py` preserves the same CLI interface (`build` and `update` modes with same flags).

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Italian labels (`Seduta`, `Dibattito`, `Intervento`) | English labels (`Session`, `Debate`, `Speech`) | Already done in production schema | Build pipeline lags behind — still writes Italian as dead code in `ingest_stenografici.py::save_to_neo4j()` |
| Per-item `session.run()` writes | Per-item writes | Current state | N round-trips for N items — replace with UNWIND batching |
| Auto-commit transactions | Auto-commit | Current state | No retry on transient errors — replace with `execute_write` |

**Deprecated/outdated:**
- `ingest_stenografici.py::save_to_neo4j()`: Italian-schema save function — dead code, never called from `build_and_update.py`
- `ingest_stenografici.py::create_constraints()`: Italian constraints (`Seduta`, etc.) — dead code
- `ingest_stenografici.py::create_indexes()`: Italian indexes — dead code
- `build/populate_ruoli.py`: Standalone script using Italian labels — superseded by `build_and_update.py::load_roles()`; can be deprecated
- `build/initialize_db.py`: Unclear purpose — inspect before phase begins
- `build/migrate_foti.py`: One-time migration script — dead code, should be deleted
- `build/create_vector_index.py`: Standalone; vector index creation is now in `build_and_update.py::create_vector_index()` — superseded

---

## Open Questions

1. **Vote-to-Debate relationship**
   - What we know: `raccoltaVotazioni` is at session level with no direct `dibattito` reference in the XML
   - What's unclear: CONTEXT.md decision says `Debate-[:HAS_VOTE]->Vote`; actual XML structure only allows `Session-[:HAS_VOTE]->Vote`
   - Recommendation: Use `Session-[:HAS_VOTE]->Vote`. The CONTEXT.md assumption appears to have been based on the current (buggy) code which searches inside `dibattito`. Confirm with user before planning if the relationship direction matters for any Phase 4 feature.

2. **`build/initialize_db.py` purpose**
   - What we know: file exists in `build/` directory
   - What's unclear: whether it overlaps with `build_and_update.py` or is a separate script
   - Recommendation: inspect during Wave 0 setup; likely deprecated or duplicate

3. **`Speech.surname_name` removal and GovernmentMember reconciliation**
   - What we know: `surname_name` is used for orphan speech reconciliation in `load_roles()`
   - What's unclear: exact rate of orphan speeches in corpus (reconciliation completeness)
   - Recommendation: Keep the inline GovernmentMember name fallback in `db_builder.py`'s `_save_speech()` method; remove the post-hoc reconciliation query that relied on `surname_name`

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | pytest (available in backend virtualenv at `backend/venv/bin/pytest`) |
| Config file | none — create `build/tests/conftest.py` in Wave 0 |
| Quick run command | `cd /path/to/project && backend/venv/bin/pytest build/tests/ -x -q` |
| Full suite command | `backend/venv/bin/pytest build/tests/ -v` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BUILD-01 | All properties in output dicts are camelCase | unit | `pytest build/tests/test_xml_parser.py::test_session_properties_camelcase -x` | ❌ Wave 0 |
| BUILD-02 | `save_to_neo4j` not callable from `xml_parser.py` | unit | `pytest build/tests/test_xml_parser.py::test_no_neo4j_dependency -x` | ❌ Wave 0 |
| BUILD-03 | `StenograficoParser.parse_xml_file()` returns correct structure | unit | `pytest build/tests/test_xml_parser.py::test_parse_xml_file -x` | ❌ Wave 0 |
| BUILD-04 | Chunk dict has no `charCount`, `startCharRaw`, `endCharRaw` keys | unit | `pytest build/tests/test_chunker.py::test_chunk_properties -x` | ❌ Wave 0 |
| BUILD-05 | Speech dict has no `preprocessedText` key | unit | `pytest build/tests/test_xml_parser.py::test_speech_no_preprocessed_text -x` | ❌ Wave 0 |
| BUILD-06 | Session dict has no `completeDate` key | unit | `pytest build/tests/test_xml_parser.py::test_session_no_complete_date -x` | ❌ Wave 0 |
| DATA-01 | `parse_xml_file()` returns non-empty votes list for files with votes | unit | `pytest build/tests/test_xml_parser.py::test_votes_parsed -x` | ❌ Wave 0 |
| DATA-02 | `parse_xml_file()` returns act_references dict for files with argomenti | unit | `pytest build/tests/test_xml_parser.py::test_act_references -x` | ❌ Wave 0 |
| DATA-03 | Speech with institutional speaker has `speakingRole` populated | unit | `pytest build/tests/test_xml_parser.py::test_speaking_role -x` | ❌ Wave 0 |
| DATA-04 | Phase with known title returns correct `phaseType` | unit | `pytest build/tests/test_xml_parser.py::test_phase_type_classification -x` | ❌ Wave 0 |
| BUILD-07 | `db_builder.py` uses UNWIND in batch methods (no `session.run()` per item) | unit | `pytest build/tests/test_db_builder.py::test_unwind_pattern -x` | ❌ Wave 0 |
| BUILD-08 | `db_builder.py` uses `execute_write` not raw `session.run()` | unit | `pytest build/tests/test_db_builder.py::test_managed_transactions -x` | ❌ Wave 0 |
| BUILD-09 | `build_and_update.py build --help` exits 0 | smoke | `backend/venv/bin/python build/build_and_update.py build --help` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `backend/venv/bin/pytest build/tests/ -x -q`
- **Per wave merge:** `backend/venv/bin/pytest build/tests/ -v`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `build/tests/__init__.py` — test package init
- [ ] `build/tests/conftest.py` — shared fixtures (sample XML files, expected dicts)
- [ ] `build/tests/test_xml_parser.py` — covers BUILD-01 through BUILD-06, DATA-01 through DATA-04
- [ ] `build/tests/test_chunker.py` — covers BUILD-04
- [ ] `build/tests/test_db_builder.py` — covers BUILD-07, BUILD-08
- [ ] Framework already installed: `backend/venv/bin/pytest` exists

---

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection: `build/build_and_update.py` (1078 lines, full analysis)
- Direct codebase inspection: `build/ingest_stenografici.py` (900 lines, full analysis)
- Direct codebase inspection: `build/embedding_service.py` (full file)
- Direct codebase inspection: `build/precalculate_embeddings.py` (full file)
- Direct codebase inspection: `build/neo4j_helper.py` (full file)
- Direct codebase inspection: `build/app_config.py` (full file)
- XML corpus analysis: 365 files, Python 3 `xml.etree.ElementTree` traversal
- `.planning/research/STACK.md` (project research, UNWIND patterns, naming conventions)
- `.planning/research/PITFALLS.md` (project research, critical risks)

### Secondary (MEDIUM confidence)
- `.planning/phases/01-build-pipeline/01-CONTEXT.md` (user decisions)
- `.planning/REQUIREMENTS.md` (requirement definitions)

### Tertiary (LOW confidence)
- Phase type vocabulary: derived from corpus frequency analysis of 365 XML files; covers top patterns but 6199 unique titles means long tail will map to `"other"`

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all packages already in codebase
- Architecture: HIGH — all patterns derived from direct code inspection
- Property rename map: HIGH — verified against actual property names written in `_save_session_english()`
- XML structure: HIGH — verified by Python traversal of actual corpus files
- Vote relationship correction: HIGH — zero votes found in dibattito, 487 found in raccoltaVotazioni in first 50 files
- Phase type mapping: MEDIUM — top 10 patterns cover most titles; long tail is large

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable domain; XML corpus and code won't change before Phase 1 executes)
