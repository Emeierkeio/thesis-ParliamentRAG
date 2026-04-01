# Feature Landscape: XML Stenographic Data Extraction

**Domain:** Parliamentary RAG — Camera dei Deputati XML stenographic records
**Researched:** 2026-04-02
**Confidence:** HIGH (based on direct inspection of 3 actual XML files)

---

## XML Files Analyzed

| File | Session | Date | Size | Character |
|------|---------|------|------|-----------|
| `stenografico_leg19_0001.xml` | Sed. 1 | 2022-10-13 | 184 KB | Opening session, no acts |
| `stenografico_leg19_0100.xml` | Sed. 100 | 2023-05-09 | 651 KB | Multiple motions + 40 votes |
| `stenografico_leg19_0029.xml` | Sed. 29 | 2022-12-28 | 2.2 MB | Confidence vote, PDL 705 |

---

## Current Extraction (Table Stakes — What Is Already Captured)

These elements are currently parsed and saved to Neo4j. Confirm completeness before the milestone.

### Session-Level (Node: `Session`)

| Property | XML XPath | Status |
|----------|-----------|--------|
| `id` | `seduta/@legislatura` + `seduta/@numero` | Extracted |
| `legislature` | `seduta/@legislatura` | Extracted |
| `number` | `seduta/@numero` | Extracted |
| `date` (Neo4j Date) | `seduta/@anno`, `@mese`, `@giorno` | Extracted |
| `complete_date` (string) | `//metadati/dataEstesa` | Extracted — **marked for removal** in refactoring |
| `chamber` | `seduta/@ramo` | Extracted |

### Debate-Level (Node: `Debate`)

| Property | XML XPath | Status |
|----------|-----------|--------|
| `id` | `//dibattito/@id` | Extracted |
| `title` | `//dibattito/titolo` | Extracted |
| `order` | (positional counter) | Extracted |

### Phase-Level (Node: `Phase`)

| Property | XML XPath | Status |
|----------|-----------|--------|
| `id` | `//fase/@id` | Extracted |
| `title` | `//fase/titolo` | Extracted |
| `order` | (positional counter) | Extracted |

### Speech-Level (Node: `Speech`)

| Property | XML XPath | Status |
|----------|-----------|--------|
| `id` | `//intervento/@id` (prefixed with seduta_id) | Extracted |
| `text` | `//testoXHTML` + `//interventoVirtuale` (raw) | Extracted |
| `preprocessed_text` | post-processing of raw (parenthetical removal) | Extracted — **marked for removal** in refactoring |
| `char_count` | computed | Extracted |
| `surname_name` | `//nominativo/@cognomeNome` | Extracted |
| `merged_from_ids` | set when continuation merge occurs | Extracted |
| SPOKEN_BY → Deputy/GovernmentMember | `//nominativo/@id` → Deputy.id URI | Extracted |

### Chunk-Level (Node: `Chunk`)

| Property | XML XPath | Status |
|----------|-----------|--------|
| `text` | computed from preprocessed_text | Extracted |
| `index` | positional | Extracted |
| `char_count` | computed | Extracted |
| `start_char_raw` / `end_char_raw` | alignment map | Extracted — **marked for removal** (buggy, unused) |

### Vote-Level (Node: `Vote` / `Votazione`)

| Property | XML XPath | Status |
|----------|-----------|--------|
| All vote fields | `//votazioni/votazione/*` | Extracted by `parse_votazione()` in the **old Italian-schema path only** — **NOT saved in `build_and_update.py`** |

**Gap confirmed:** The production English-schema path (`build_and_update.py`) does NOT save Vote nodes at all. The parse logic exists in `ingest_stenografici.py` but `_save_session_english()` never calls it.

---

## Differentiators — New Data That Would Significantly Improve RAG Quality

These elements exist in the XML but are currently ignored. Each has a clear RAG value.

---

### 1. Debate-to-Act Cross-Reference (`<metadati>/<argomenti>`)

**XML XPath:** `//metadati/argomenti/argomento[@idDibattito]/atti/atto[@tipologiaAtto][@codiceArgomento]`

**Example (Sed. 100, lines 8–47):**
```xml
<metadati>
  <argomenti>
    <argomento idDibattito="tit00030">
      <atti>
        <atto tipologiaAtto="mozione" codiceArgomento="1-00098" />
        <atto tipologiaAtto="mozione" codiceArgomento="1-00056" />
      </atti>
    </argomento>
    <argomento idDibattito="tit00080">
      <atti>
        <atto tipologiaAtto="pdl" codiceArgomento="1067-A" />
      </atti>
    </argomento>
  </argomenti>
</metadati>
```

**Act types observed across all 365 files:**
| `tipologiaAtto` | Count | Meaning |
|-----------------|-------|---------|
| `pdl` | 3,493 | Proposta di legge (bill) |
| `interrogazioneRispostaOrale` | 1,016 | Oral question |
| `mozione` | 351 | Motion |
| `doc` | 264 | Document |
| `interpellanza` | 194 | Interpellation |

**RAG Value:** This is the most impactful missing link in the graph. Currently the system has Debate nodes connected to speeches but NO connection to parliamentary acts being debated. Adding this creates:
- Direct `Debate -[:DISCUSSES]-> ParliamentaryAct` edges
- When a user asks "what was said about bill 705", retrieval can start from the act and traverse to debates and speeches — **graph-channel improvement**
- The `codiceArgomento` is the act's ID as used by the Camera API, matching `ParliamentaryAct` nodes already in the graph

**Implementation:** Extract during `parse_xml_file()`. Add to the `data` dict as `debate_act_links: List[{debate_id, tipologia, codice}]`. In `_save_session_english()`, after creating Debate nodes, MATCH the ParliamentaryAct and MERGE the relationship.

---

### 2. Debate Title Encodes Act Number — Extract as Structured Property

**XML XPath:** `//dibattito/titolo` (text content already extracted, but not parsed for structure)

**Example (Sed. 29, line 80):**
```
"Seguito della discussione del disegno di legge: S. 274 - Conversione in legge,
... (A.C. 705)."
```

**RAG Value:** The debate title consistently embeds the act number in parentheses as `(A.C. NNN)` or `(S. NNN)`. Extracting this as `Debate.act_reference` gives a searchable numeric identifier. This is a lower-effort complement to item 1 above (item 1 is the primary source; this is a fallback for debates not in `<argomenti>`).

**Implementation:** Add a regex extraction in debate parsing: `re.search(r'\(A\.C\.\s*(\d[\d\-A-Z]*)\)', titolo)` → store as `Debate.ac_number`. Similarly `(S. NNN)` for Senate-originated bills.

---

### 3. Speaker's Institutional Role at Time of Speech

**XML XPath:** `//testoXHTML/nominativo/following-sibling::text()` followed by `//testoXHTML/emphasis[@type="italic"]`

**Example (Sed. 100, line 102):**
```xml
<nominativo id="307138" cognomeNome="GAVA Vannia">VANNIA GAVA</nominativo>,
<emphasis type="italic"> Vice Ministra dell'Ambiente e della sicurezza energetica</emphasis>.
```

**Example (Sed. 100, line 64):**
```xml
<nominativo id="301577" cognomeNome="DELLA VEDOVA Benedetto">BENEDETTO DELLA VEDOVA</nominativo>,
<emphasis type="italic"> Segretario</emphasis>
```

**RAG Value:** The first `<emphasis type="italic">` immediately following a `<nominativo>` (before the period) encodes the speaker's role at the time of the speech — "Vice Ministra dell'Ambiente", "Segretario", "Relatore", "Sottosegretario di Stato per la Giustizia". This is not the same as the static `institutional_role` on a Deputy node. It captures:
- Which ministry a government member represented on that day
- When a deputy acted as rapporteur (`Relatore`) vs ordinary speaker
- Cross-session role changes

Currently `get_nominativo_info()` reads `id` and `cognomeNome` from `<nominativo>` but ignores the adjacent `<emphasis>` role tag. The role is present in the raw text (so it ends up in speech text) but is not a structured property.

**Implementation:** In `parse_intervento()`, after `get_nominativo_info()`, check the tail text of `<nominativo>` and the first `<emphasis>` child of `<testoXHTML>`. Store as `Speech.speaking_role` (e.g., "Vice Ministra dell'Ambiente e della sicurezza energetica"). Use this to enrich the authority score or as a filter in graph retrieval.

---

### 4. Vote Nodes Saved in Production Schema

**XML XPath:** `//dibattito//votazioni/votazione`

**Example (Sed. 100, lines 2035–2047):**
```xml
<votazione>
  <numero>1</numero>
  <tipo>Nominale</tipo>
  <oggetto>MOZ. 1-98 - P. 1</oggetto>
  <presenti>289</presenti>
  <votanti>289</votanti>
  <astenuti>0</astenuti>
  <maggioranza>145</maggioranza>
  <favorevoli>281</favorevoli>
  <contrari>8</contrari>
  <missione>51</missione>
  <esito>Appr.</esito>
</votazione>
```

**RAG Value:** A session of 100 debates has 40 recorded votes. Currently ALL Vote data is discarded by `build_and_update.py`. Saving votes enables:
- "Did the motion on nuclear energy pass?" → direct answer from graph
- Vote outcome (`esito`) as metadata filter on debate retrieval
- `missione` count shows deputies absent on mission — relevant to authority scoring
- `oggetto` links the vote back to the specific act part (e.g., "MOZ. 1-98 - P. 1")

The parse logic already exists in `parse_votazione()`. The only missing piece is calling it in `_save_session_english()` and adding a MERGE for `Vote` nodes with `[:HAS_VOTE]` from `Debate`.

**Note:** PROJECT.md explicitly says "Keep Vote nodes (renamed from Votazione) for future RAG integration" — this is the implementation of that decision.

---

### 5. Phase Type as Structured Enum (Not Just Free-Text Title)

**XML XPath:** `//fase/titolo`

**Observed patterns across files:**
```
(Dichiarazioni di voto)
(Parere del Governo)
(Discussione sulle linee generali - A.C. 1067-A)
(Repliche - A.C. 1067-A)
(Votazioni)
(Esame degli ordini del giorno - A.C. 705)
(Votazione della questione di fiducia - Articolo unico - A.C. 705)
```

**RAG Value:** Phase titles follow a closed vocabulary. Extracting a `Phase.phase_type` enum (`government_opinion`, `vote_declarations`, `general_discussion`, `replicas`, `voting`, `orders_of_day`, `confidence_vote`) enables:
- Filtering retrieval to "only speeches where the government gave its opinion" or "only vote declarations"
- Authority scoring: a government minister's speech in `(Parere del Governo)` phase is specifically an official government position — high retrieval weight for policy questions
- Filtering out procedural phases from RAG retrieval entirely

**Implementation:** Add a mapping dict in the parser: `PHASE_TYPE_MAP = {"Dichiarazioni di voto": "vote_declarations", "Parere del Governo": "government_opinion", ...}`. Apply via regex since titles include act references. Store as `Phase.phase_type`.

---

### 6. Inline Party Group Code from Speech Text

**XML XPath:** Text content following `</testoXHTML>` nominativo, e.g., `(PD-IDP)`, `(M5S)`, `(A-IV-RE)`

**Example (Sed. 100, line 122):**
```xml
<nominativo id="306398" cognomeNome="FORNARO Federico">FEDERICO FORNARO</nominativo> (PD-IDP).
```

**Frequency observed in Sed. 100:**
```
(M5S): 19 occurrences
(PD-IDP): 14 occurrences
(FDI): 12 occurrences
(A-IV-RE): 8 occurrences
```

**RAG Value:** The party code is embedded as plain text in `<testoXHTML>` after the nominativo, before the period. This is a reliable, machine-readable party affiliation for each speech — derived directly from the stenographic record rather than from CSV join. Useful as a cross-check when deputy-to-group CSV join fails (orphan reconciliation). Could be stored as `Speech.group_code` and used in group-based retrieval filtering.

**Implementation:** In `parse_intervento()`, after extracting `cognome_nome`, check the raw tail text of `<nominativo>` for `^\s*\(([A-Z0-9\-]+)\)` pattern. Store as `Speech.group_code`. Keep as secondary source — primary authority remains Neo4j Deputy→ParliamentaryGroup edge.

---

### 7. Session Start Time and Suspension Events

**XML XPath:** `//resoconto/avviso/titolo` (text content)

**Observed patterns:**
```
"La seduta comincia alle 15,45."
"La seduta, sospesa alle 17,05, è ripresa alle 17,25."
"La seduta, sospesa alle 4,20 del 29 dicembre, è ripresa alle 17."
"La seduta termina alle 20,10."
```

**RAG Value:** Session duration and overnight sittings are factually meaningful. Useful for:
- Answering "how long did the debate on X last?" (low priority but feasible)
- Detecting marathon overnight sessions (Sed. 29 ran past 4am) — could correlate with contentious topics

**Implementation:** Parse `avviso/titolo` with a time regex and store `Session.start_time` (string), `Session.end_time` (string). Suspension/resumption events could be stored on Phase nodes if the `<avviso>` is inside a `<fase>` block.

---

## Nice-to-Have — Marginal RAG Value

These elements exist but adding them is unlikely to improve retrieval quality meaningfully.

### Page Numbers (`<pagina numero="N" />`)

**XML XPath:** `//pagina/@numero`

**Value:** Maps speech position to stenographic record page. Useful for citation, not for retrieval. Could be stored as `Speech.page_start` if human-readable provenance is needed in answers. Low RAG value since embeddings retrieve by semantic similarity, not page.

### Session Presiding Officer (`<presidenza>`)

**XML XPath:** `//resoconto/presidenza`

**Example:** `"PRESIDENZA DEL VICEPRESIDENTE FABIO RAMPELLI"`

**Value:** Already captured implicitly (presiding officer speeches are filtered as PRESIDENTE). Storing as `Session.presiding_officer` (text) gives provenance but adds no retrieval value since the president's interventions are already filtered out.

### `interventoVirtuale` IDs as Structural Markers

**XML XPath:** `//interventoVirtuale/@id`

**Value:** IDs like `iv.1`, `iv.2` provide intra-speech segmentation. Currently the parser concatenates all `interventoVirtuale` into one speech. Splitting could allow finer-grained chunking but would increase chunk count significantly and fragment context windows. Not recommended — current merge strategy is better for RAG.

---

## Anti-Features — Explicitly Not Worth Extracting

| Anti-Feature | Why Avoid |
|--------------|-----------|
| Extracting `<emphasis type="italic">` content as separate nodes | Emphasis marks procedural asides and applause annotations — already stripped by preprocessing. Creating nodes from them adds noise. |
| Parsing individual vote roll-calls (deputy-level yes/no) | The XML does not contain per-deputy vote records — only aggregate counts. Per-deputy vote data comes from a separate system and is out of scope. |
| Storing `<gruppi />` from `<metadati>` | The element appears empty in all three files inspected (`<gruppi />`). No data to extract. |
| Re-segmenting speeches at `interventoVirtuale` boundaries | Would create hundreds of micro-chunks (under 200 chars) and break the coherent speech model. The current ellipsis-based merge is the right approach. |

---

## Feature Dependencies

```
Debate-to-Act Link (#1) → requires ParliamentaryAct nodes to already exist
Vote Nodes (#4)          → requires Debate nodes to already exist (saved before votes)
Phase Type Enum (#5)     → independent of all other features
Speaker Role (#3)        → requires Speech node to already exist
Group Code (#6)          → independent, can supplement existing Deputy→Group edge
```

---

## MVP Recommendation for This Milestone

The refactoring milestone's XML analysis task ("Analyze XML stenographic files for additional extractable information") should prioritize in this order:

**Must implement:**
1. **Debate-to-Act link** (`#1`) — highest graph value, closes the biggest retrieval gap
2. **Vote nodes in production schema** (`#4`) — parse logic already exists, just needs `_save_session_english()` to call it

**Should implement:**
3. **Speaker institutional role** (`#3`) — low implementation cost, enriches Speech metadata for authority scoring
4. **Phase type enum** (`#5`) — enables retrieval filtering by procedural context

**Defer:**
5. **Act number from title** (`#2`) — redundant if `#1` is implemented; use as fallback only
6. **Group code** (`#6`) — supplement to existing data, low priority
7. **Session timing** (`#7`) — minimal retrieval value

---

## Sources

- Direct inspection of `data/xml/stenografico_leg19_0001.xml` (184 KB, 150 lines read)
- Direct inspection of `data/xml/stenografico_leg19_0100.xml` (651 KB, 2,060+ lines read)
- Direct inspection of `data/xml/stenografico_leg19_0029.xml` (2.2 MB, 300+ lines read)
- Full text search across all 365 XML files (`tipologiaAtto` enumeration, group code extraction)
- Code analysis of `build/ingest_stenografici.py` (`parse_intervento`, `parse_votazione`, `parse_xml_file`)
- Code analysis of `build/build_and_update.py` (`_save_session_english` — confirmed Vote nodes NOT saved)
- `.planning/PROJECT.md` for confirmed decisions (Vote node rename, XML analysis task)
